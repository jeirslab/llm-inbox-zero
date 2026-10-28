# llm-inbox-zero

Triage large IMAP mailboxes down to the mail you care about, with small models doing the
judgments and plain code doing the decisions. One service per instance: it mirrors the
metadata of every configured mailbox into SQLite, groups mail by sender and domain, asks a
decision model typed questions about the senders the headers do not already settle, turns the
answers into a plan (keep, file, unsubscribe, mark as spam, trash), and applies the plan over
IMAP with a snapshot and an undo log. An MCP server lets an LLM drive all of it.

Status: **specification**. Nothing below is implemented yet.

Nothing in this repository is specific to one person. Accounts, providers, model endpoints,
rules, keys and backup targets are declared per instance (see [Configuration](#configuration)).

## Contents

- [Goals and non-goals](#goals-and-non-goals)
- [Pipeline](#pipeline)
- [Store](#store)
- [Prepared lists](#prepared-lists)
- [Signals settled in code](#signals-settled-in-code)
- [Models](#models)
- [Questions and the decision table](#questions-and-the-decision-table)
- [Plans, actions and undo](#plans-actions-and-undo)
- [Backups](#backups)
- [Rules for future mail](#rules-for-future-mail)
- [MCP server](#mcp-server)
- [CLI](#cli)
- [Configuration](#configuration)
- [Provider notes](#provider-notes)
- [Security](#security)
- [Layout](#layout)
- [Roadmap](#roadmap)

## Goals and non-goals

Goals:

- A reliable list of what matters (bills, receipts, people, the newsletters actually read) out of
  tens of thousands of messages, for a few cents of model time or none.
- Many mailboxes per instance, each with its own IMAP server and provider quirks.
- Every bulk action reviewable before it runs, and reversible after.
- Fully local operation: a self-hosted decision model and a self-hosted chat model are enough.
  Hosted models are an opt-in fallback.
- Driven by a person (CLI) or by an LLM (MCP), with the same operations and the same safety net.

Non-goals:

- A mail client. Read mail in whatever client you use.
- Sending mail, except the one email a `mailto:` unsubscribe requires.
- Spam filtering of the server's kind. The provider already does that; this tool sorts what
  gets through.

## Pipeline

```
sync ──► prep ──► taxonomy ──► classify ──► decide ──► plan ──► apply
 IMAP     code     chat model   decision     code       file     IMAP (+ snapshot, undo log)
 →SQLite           (once)       model                   (review)
                                                                  watch: rules on new mail
```

| Stage | Who does it | Cost | Output |
|---|---|---|---|
| `sync` | code (`imap-tools`) | IMAP time only | `messages`, `folders` in SQLite |
| `prep` | code | none | sender table, one file per domain, header verdicts |
| `taxonomy` | chat model, once per instance (and when asked) | one call over the compact sender table | proposed categories; a person approves them |
| `classify` | decision model, only for senders `prep` did not settle | one request per sender, all questions bundled | typed answers with probabilities |
| `decide` | code: the decision table | none | an action per sender (and per message where needed) |
| `plan` | code, or an LLM through MCP | none | a plan file listing the actions |
| `apply` | code | IMAP time | changed mailboxes, a snapshot, an undo log |
| `watch` | code (IMAP IDLE) | none | the instance's rules applied to new mail |

Token efficiency is the design constraint: the bulk of the work is header parsing and counting.
Models see only compact per-sender summaries, never whole mailboxes, and only for the senders
the headers leave open.

## Store

One SQLite file per instance (`$STATE_DIRECTORY/inbox.db`). A message is identified by
`(account, folder, uidvalidity, uid)` and deduplicated across folders by `Message-ID` (and, on
Gmail, `X-GM-MSGID`, since one message appears in every label's folder).

`sync` fetches, per message, with `BODY.PEEK` so that fetching never marks a message as read:

- the header block (`From`, `Reply-To`, `To`, `Cc`, `Date`, `Subject`, `Message-ID`,
  `List-Id`, `List-Unsubscribe`, `List-Unsubscribe-Post`, `Precedence`, `Auto-Submitted`,
  `Feedback-ID`, `Return-Path`, and the ESP headers listed in `prep/signals.py`);
- the first 2 KiB of the text body, decoded and stripped of HTML;
- the flags (`\Seen`, `\Answered`, `\Flagged`, `$Junk`, keywords), size, internal date;
- the MIME structure (attachment count, names, types, sizes) without the attachment bodies;
- on Gmail, `X-GM-LABELS` and `X-GM-THRID`.

Re-runs are incremental: per folder, `UIDVALIDITY` and the highest UID seen (and `CONDSTORE`
`MODSEQ` where the server supports it, for flag changes). Folders are imported with their
hierarchy, special-use attributes (`\Sent`, `\Trash`, `\Junk`, `\Archive`) and message counts.

Tables: `accounts`, `folders`, `messages`, `senders` (derived), `verdicts` (header and model
answers, with the model and question version), `plans`, `applied` (undo log), `keys`, `audit`.

## Prepared lists

`prep` writes two kinds of list from the store, as TSV (cheaper in tokens than JSON or Markdown
tables, and readable by people and models alike):

`senders.tsv`, one row per sender address:

| column | meaning |
|---|---|
| `sender` | address, lower-cased |
| `name` | most common display name |
| `domain` | registrable domain (`mail.example.com` → `example.com`) |
| `total`, `read`, `unread` | message counts (`\Seen`) |
| `replied` | messages with `\Answered`, plus whether the account ever sent to this address (from the Sent folder) |
| `flagged` | messages with `\Flagged` |
| `filed` | messages the owner moved into a folder of their own |
| `attachments` | messages with at least one attachment |
| `first`, `last` | dates of the first and last message |
| `list`, `automated` | header verdicts (see below) |

`domains/<domain>.tsv`, one file per domain, one row per message:

| column | meaning |
|---|---|
| `date` | |
| `from` | sender address (a domain often has several) |
| `subject` | |
| `snippet` | first 100 characters of the text body |
| `to_count`, `cc_count`, `to_me` | how many recipients, and whether the account was in `To` (not only `Cc` or `Bcc`) |
| `attachments` | names, comma-separated |
| `seen`, `answered`, `flagged` | flags |
| `folder` | where it is now |

Senders with more than `prep.maxRowsPerSender` messages (default 20) are summarised in their
domain file by their most recent rows plus counts, so one bulk sender cannot dominate a file.

Read versus unread is a weak signal alone: preview panes mark mail as read, and mail is
archived without being opened. It is one input among `replied`, `flagged`, `filed`, `to_me` and
how long mail from the sender is kept.

## Signals settled in code

These need no model. `prep` records each as a verdict with its evidence:

| Signal | Evidence | Verdict |
|---|---|---|
| mailing list | `List-Id`, `List-Unsubscribe`, `Precedence: bulk/list` | `list` |
| automated | `Auto-Submitted` other than `no`, `noreply`/`no-reply`/`donotreply` local part, bounce `Return-Path` | `automated` |
| bulk sender | `Feedback-ID`, ESP headers (Mailchimp, SendGrid, Mailgun, Amazon SES, Postmark, ...) | `bulk` |
| correspondent | `replied` > 0, or the account sent to this address | `person` (strong) |
| provider category | Gmail `X-GM-LABELS` or `X-GM-RAW "category:..."` (Promotions, Social, Updates, Forums) | the category |
| one-click unsubscribe | `List-Unsubscribe` with `https:` and `List-Unsubscribe-Post: List-Unsubscribe=One-Click` (RFC 8058) | `unsubscribe: one-click` |
| mailto unsubscribe | `List-Unsubscribe` with `mailto:` only | `unsubscribe: mailto` |

A sender a rule in code already decides (for example a correspondent the account has replied
to) is not sent to a model unless `classify --all` is asked for.

## Models

Two roles, each with its own ordered list of providers. The first available provider in the
list is used; an unavailable one (connection refused, timeout, 5xx) is skipped. Each provider
is marked `local` or `external`.

### Decision model (`classify`)

Typed questions over a state, answered with calibrated probabilities: `noul` (the probability a
statement is true), `choice` (one of named options, with a probability per option) and `score`
(a position on described levels). The wire protocol is TypeSafe's **System One API**,
`POST /v1/systemone` ([docs](https://docs.typesafe.ai/go/api.md),
[OpenAPI](https://api.typesafe.ai/openapi.json)). Two implementations exist:

- **Kev** ([models](https://huggingface.co/collections/jaredpalmer/kev),
  [code](https://github.com/jaredpalmer/kev)): open weights (Apache-2.0) on Qwen bases, served
  by `kev.serve` with the same endpoint. `kev-9b` runs on one 24 GB GPU; `kev-4b` and
  `kev-0.8b` on less. The `local` choice.
- **Jev**, TypeSafe's hosted model (`https://api.typesafe.ai`). The `external` choice.

Any server that implements `/v1/systemone` can be added. The client sends only the fields the
API defines, so the same request goes to either. `classify` records which provider and model
answered each question.

### Chat model (`taxonomy`, summaries, the MCP assistant's own calls)

Any OpenAI-compatible `/v1/chat/completions` endpoint: a LiteLLM gateway, vLLM, llama.cpp,
Ollama, or a hosted API. Used once to propose the category list from `senders.tsv`, and on
demand to summarise a domain or explain a plan. The category list it proposes is written to
`taxonomy.yaml` in the state directory and used only after a person approves it
(`taxonomy approve`); the decision model then picks among those categories.

### Consent for external providers

Before the first request to an `external` provider that would carry mail content (addresses,
subjects, snippets), the run stops with `consent_required`, naming the provider and what would
be sent. Consent is given per provider, per instance, and recorded with who gave it and when:

- in the CLI, `llm-inbox-zero consent grant <provider>` (or answering the prompt);
- through MCP, the `consent` tool, which only a key with the `admin` scope may call;
- ahead of time, in the instance configuration (`providers.<name>.consent = true`).

Per provider, `send` limits what leaves the machine: `full` (the prepared rows), `metadata`
(sender, counts and flags; no subjects or snippets) or `none` (never used for mail content).

## Questions and the decision table

All question wording, options, criteria, thresholds and the table itself live in one module,
`src/llm_inbox_zero/questions.py`, versioned (`QUESTIONS_VERSION`) so that answers from an
older version are known as such and can be re-asked.

The state of a request is one sender: its `senders.tsv` row and its most recent domain rows.
All questions about that sender go in the same request. Questions (first set, to be tuned on
real mailboxes):

| id | type | question |
|---|---|---|
| `category` | choice | Which kind of mail does `sender` send? Options: the approved taxonomy. |
| `fits` | noul | Does any of the categories describe mail from `sender`? |
| `person` | noul | Is mail from `sender` written by a person to the account holder? |
| `money` | noul | Do the messages from `sender` record a payment, a bill, an invoice or a receipt? |
| `security` | noul | Are the messages from `sender` sign-in codes, verification links or security alerts? |
| `action` | noul | Do the recent messages from `sender` ask the account holder to do something by a date? |
| `engaged` | score | How much attention does the account holder give mail from `sender`? Levels described from the counts' point of view ("never opens it", "opens some, replies to none", ...). |
| `wanted` | noul | Would the account holder miss mail from `sender` if it stopped? |

The decision table (code) maps verdicts and answers to an action. Thresholds start at 0.7 and
0.3 for `noul`; the band between is its own row. Illustrative rows:

| person | money | security | list | engaged | wanted | action |
|---|---|---|---|---|---|---|
| ≥ 0.7 | — | — | — | — | — | keep in inbox |
| — | ≥ 0.7 | — | — | — | — | file to `Receipts` (or the instance's mapping) |
| — | — | ≥ 0.7 | — | — | — | file to `Security`, trash after `retention.security` |
| < 0.3 | < 0.3 | < 0.3 | yes | low | < 0.3 | unsubscribe, then trash |
| < 0.3 | < 0.3 | < 0.3 | yes | high | ≥ 0.7 | file to `Newsletters` |
| any row in the 0.3 to 0.7 band | | | | | | `review` (a person or the MCP assistant decides) |

The instance maps categories and actions to its own folder names (`decide.folders`).

## Plans, actions and undo

A plan is a JSON file (`plans/<id>.json`) of actions over message selectors. Selectors resolve
at apply time against the store, after a fresh incremental sync, so a plan never acts on a
message it did not see.

```json
{
  "id": "2026-10-27T18-04-12Z-a1b2",
  "account": "personal",
  "author": "mcp:key-3f9c",
  "actions": [
    {"op": "move", "select": {"sender": "receipts@shop.example"}, "to": "Receipts"},
    {"op": "unsubscribe", "select": {"sender": "news@list.example"}},
    {"op": "trash", "select": {"sender": "news@list.example", "older_than": "30d"}},
    {"op": "flag", "select": {"domain": "bank.example"}, "flag": "\\Flagged"},
    {"op": "spam", "select": {"sender": "x@spam.example"}},
    {"op": "folder_create", "name": "Receipts"}
  ]
}
```

Operations: `move`, `copy`, `trash`, `expunge`, `flag`, `unflag`, `read`, `unread`, `spam`,
`not_spam`, `folder_create`, `folder_rename`, `folder_delete`, `unsubscribe`, `keep` (no-op that
records a decision).

- `plan show` lists what each action would touch (counts per folder and sender, a sample of
  subjects) before anything runs.
- `trash` moves to the account's `\Trash` folder. `expunge`, emptying Trash and deleting a
  non-empty folder are `destroy` operations: they need the `destroy` scope and the backup
  conditions in [Backups](#backups).
- `spam` moves to `\Junk` and sets `$Junk`, which providers learn from.
- `unsubscribe` uses the RFC 8058 one-click POST where offered; otherwise sends the `mailto:`
  message through the account's SMTP settings, if the instance declared them; otherwise lists
  the link for a person. Each attempt is recorded.
- `apply` writes an undo log: per message, the folder and flags before and after, and the new UID
  (from `COPYUID`/`MOVE` responses). `undo <plan>` reverses moves and flag changes. An
  unsubscribe cannot be undone and says so in `plan show`.

## Backups

Three layers, cheapest first:

1. **State snapshot, before every apply (always on).** Per message: `Message-ID`, folder and
   flags. Kilobytes per thousand messages; seconds to take. Enough to undo any move or flag
   change, even one made outside this tool since.
2. **Raw copy of what a plan destroys (always on).** Before a `destroy` operation, and before a
   `trash` into a folder the provider empties automatically, the affected messages are fetched
   whole (`BODY.PEEK[]`) and written to `backups/<plan>/<message-id>.eml.zst`. The cost is the
   size of what is being deleted, nothing more.
3. **Full mirror (optional, `backup.mirror.enable`).** `mbsync` keeps a Maildir copy of every
   account, incrementally; `restic` (or any command the instance names) ships it to the
   instance's backup target. With `backup.mirror.requireFresh = "24h"`, `destroy` operations are
   refused unless the last mirror run is newer than that. The first run over a large mailbox
   (50,000 messages is a few gigabytes) takes hours through a slow IMAP server such as Proton
   Bridge; later runs fetch only new mail.

Retention of layers 1 and 2 is `backup.keep` (default 90 days).

## Rules for future mail

A rule is a match and the same actions a plan has. Rules are declared per instance, never in
this repository, in one provider-neutral form:

```nix
rules = [
  { name = "receipts"; account = "personal";
    match = { category = "receipt"; };            # or sender, domain, list_id, header, subject
    actions = [ { op = "move"; to = "Receipts"; } ]; }
  { name = "old-newsletters"; account = "personal";
    match = { domain = "list.example"; };
    actions = [ { op = "move"; to = "Newsletters"; } { op = "read"; } ]; }
];
```

Backends, chosen per account:

- `watch` (default, every provider): the service keeps an IMAP IDLE connection per account and
  applies the rules to new mail, through the same apply path (snapshot, undo log). A `category`
  match classifies the new message's sender first if it is not known yet.
- `sieve` (export only): renders the rules that need no model (sender, domain, header, subject)
  as a Sieve script to paste into a provider that accepts Sieve (Proton accepts Sieve filters in
  its web settings; it has no API for them).
- `gmail` (later): the same rules as Gmail filters through the Gmail API (needs OAuth, not an
  app password).

Rules can also be proposed from a plan (`rules suggest <plan>`): the actions a person approved
in bulk become rules for the mail that keeps coming.

## MCP server

A network service, not a stdio process: Streamable HTTP at `/mcp` on the configured address,
behind TLS (directly, or through a reverse proxy). Every request carries an API key
(`Authorization: Bearer lmz_...`).

Tools (each needs the scope shown):

| Tool | Scope | |
|---|---|---|
| `accounts`, `folders` | read | accounts, folder tree with counts |
| `senders`, `domain` | read | the prepared lists, paged and filterable |
| `search` | read | messages by sender, domain, subject, flags, date, folder |
| `message` | read | one message's headers and snippet (the body only with `read_body`) |
| `verdicts`, `taxonomy` | read | answers so far; the category list |
| `sync`, `prep` | plan | refresh the store |
| `classify` | plan | run the decision model (may return `consent_required`) |
| `plan_create`, `plan_show`, `plan_diff` | plan | write and inspect plans |
| `apply`, `undo` | apply | run a plan without `destroy` operations, or undo one |
| `apply_destroy` | destroy | run a plan with `destroy` operations |
| `rules_list`, `rules_suggest` | read / plan | the declared rules; proposals |
| `consent`, `taxonomy_approve` | admin | external providers; the category list |

The server tells the client, in its instructions, the rules it enforces: plans before changes,
`trash` before `expunge`, consent before external models.

### API keys

- `llm-inbox-zero keys create --name <label> --scope read,plan,apply --ttl 30d` prints the key
  once. Only its hash (argon2id) and a short prefix are stored.
- `keys list` (prefix, name, scopes, created, expires, last used), `keys revoke <prefix>`,
  `keys rotate <prefix>`.
- Expiry is mandatory (`keys.maxTtl`, default 90 days). An expired key is refused and logged.
- Keys can also be declared in the configuration as files of hashes, for keys made elsewhere.
- Every tool call is recorded in `audit` with the key prefix, the tool and its arguments (not
  the message content returned).

## CLI

```
llm-inbox-zero [--config FILE] <command>

  accounts                          list accounts and whether they connect
  sync [ACCOUNT...] [--full]        incremental IMAP mirror into the store
  prep [ACCOUNT...]                 write senders.tsv and domains/*.tsv, header verdicts
  taxonomy propose|show|approve     the category list
  classify [--all] [--sender S]     decision-model answers for unsettled senders
  decide                            run the decision table, write a draft plan
  plan show|edit|diff ID            inspect a plan
  apply ID [--yes]                  run a plan (snapshot, raw copies, undo log)
  undo ID                           reverse a plan's moves and flag changes
  folders [create|rename|delete]    folder tree and changes (through a plan)
  rules list|suggest|export-sieve   rules for future mail
  watch                             apply rules to new mail (what the service runs)
  backup snapshot|mirror|status     backup layers
  consent list|grant|revoke         external providers
  keys create|list|revoke|rotate    MCP API keys
  serve                             the MCP server (what the service runs)
```

Every command that changes a mailbox goes through a plan; `folders create` is shorthand for a
one-action plan.

## Configuration

A NixOS module (`nixosModules.default`) declares an instance. Secrets are paths to files, never
values. Example (placeholders only):

```nix
services.llm-inbox-zero = {
  enable = true;
  stateDir = "/var/lib/llm-inbox-zero";

  accounts.personal = {
    imap = { host = "127.0.0.1"; port = 1143; tls = "starttls"; user = "me@example.org";
             passwordFile = "/run/secrets/inbox/personal-imap"; caFile = null; };
    smtp = null;                 # set to allow mailto: unsubscribes
    provider = "generic";        # generic | gmail | proton-bridge (quirks, see below)
    rulesBackend = "watch";
    folders = { receipts = "Receipts"; newsletters = "Newsletters"; security = "Security"; };
  };

  providers = {
    decision = [   # tried in order
      { name = "kev";  kind = "systemone"; url = "http://gpu-host.example:8008"; model = "kev-latest";
        locality = "local"; apiKeyFile = null; }
      { name = "jev";  kind = "systemone"; url = "https://api.typesafe.ai"; model = "jev-latest";
        locality = "external"; send = "full"; consent = false;
        apiKeyFile = "/run/secrets/inbox/typesafe"; }
    ];
    chat = [
      { name = "gateway"; kind = "openai"; url = "http://gateway.example:4000/v1"; model = "some-local-model";
        locality = "local"; apiKeyFile = "/run/secrets/inbox/gateway"; }
      { name = "claude"; kind = "openai"; url = "https://api.anthropic.com/v1/"; model = "claude-sonnet-5-5";
        locality = "external"; send = "metadata"; consent = false;
        apiKeyFile = "/run/secrets/inbox/anthropic"; }
    ];
  };

  rules = [ ];

  backup = {
    keep = "90d";
    mirror = { enable = true; requireFresh = "24h"; ship = "restic"; repositoryFile = "/run/secrets/inbox/restic-repo";
               passwordFile = "/run/secrets/inbox/restic-pass"; };
  };

  mcp = {
    enable = true;
    listen = "127.0.0.1:8740";   # put a TLS proxy in front, or set tls.certFile/keyFile
    keys.maxTtl = "90d";
    keys.hashFiles = [ ];
  };
};
```

The module runs two units: `llm-inbox-zero.service` (`serve`: MCP server and `watch`) and
`llm-inbox-zero-mirror.timer` (layer 3 backups). `sync`, `prep` and `classify` run on demand,
from the CLI or through MCP. The package is also usable without NixOS: `uvx llm-inbox-zero
--config config.toml` takes the same settings as TOML.

## Provider notes

- **Gmail**: IMAP needs an app password (2-step verification on) or OAuth. `[Gmail]/All Mail`
  holds every message once; label folders repeat them, so `sync` reads All Mail and records
  labels from `X-GM-LABELS`. Moving out of `INBOX` removes the label; `trash` moves to
  `[Gmail]/Trash`, which Gmail empties after 30 days.
- **Proton Mail through Proton Bridge**: Bridge serves IMAP on loopback (default port 1143) with
  a self-signed certificate made for `127.0.0.1`. Reach it on the same host, or put a TLS proxy
  in front with a certificate the client trusts (`caFile`). Bridge decrypts on fetch, so a full
  sync of a large mailbox is slow; incremental syncs are not. Labels appear as folders under
  `Labels/`, folders under `Folders/`. Proton offers Sieve filters in its web settings but no API
  for them (`rulesBackend = "sieve"` exports a script).
- **Mail forwarded or imported from one provider into another**: triage it in one place only,
  and leave the other account out of the instance (or set it to `read-only`), so the same
  message is not acted on twice.

## Security

- The store holds mail metadata and snippets: it is created `0600`, under a dedicated system
  user, in the unit's `StateDirectory`.
- Mail content leaves the machine only to providers marked `external` with consent recorded,
  and only at the `send` level declared for them.
- Passwords and keys are read from files at use; never logged, never in arguments, never in
  plans or the audit log.
- Plans and rules are data, not code: an LLM writing a plan cannot run anything but the listed
  operations.
- Snippets and subjects are untrusted input when given to a model: the MCP server marks them as
  mail content, not instructions.

## Layout

```
src/llm_inbox_zero/
  cli.py            commands
  store.py          SQLite schema and queries
  sync.py           imap-tools: incremental fetch, folders, flags
  prep/             senders.tsv, domains/*.tsv, signals.py (header verdicts)
  providers/        systemone.py (Kev, Jev, any /v1/systemone), openai.py (chat), consent.py
  questions.py      every question, option, criterion, threshold and the decision table
  plans.py          plan format, selectors, plan show/diff
  apply.py          IMAP operations, snapshots, raw copies, undo log
  unsubscribe.py    RFC 8058 one-click, mailto
  rules/            schema, watch (IMAP IDLE), sieve export
  backup.py         layers 1-3
  mcp/              Streamable HTTP server, tools, keys, audit
nix/module.nix      the NixOS module
skills/llm-inbox-zero/SKILL.md   template skill for an assistant using the MCP server
tests/              fixtures are synthetic mailboxes (no real mail)
```

## Roadmap

1. `sync` and `prep` against one generic IMAP account; the prepared lists.
2. Header verdicts; `taxonomy` through an OpenAI-compatible endpoint.
3. `classify` through `/v1/systemone` (Kev first), provider fallback and consent.
4. Decision table, plans, `apply` with layers 1 and 2, `undo`.
5. MCP server with keys and audit; the skill template.
6. NixOS module; layer 3 mirror.
7. Rules: `watch`, then `sieve` export, then the Gmail API.
8. Unsubscribe (one-click, then mailto).

## Dependencies

[`imap-tools`](https://github.com/ikvk/imap_tools) (IMAP), the Python standard library's `email`
and `sqlite3`, [`mcp`](https://github.com/modelcontextprotocol/python-sdk) (server),
`httpx`, `argon2-cffi`, `zstandard`; `isync` (`mbsync`) and `restic` for the optional mirror.
