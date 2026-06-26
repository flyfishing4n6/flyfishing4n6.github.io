---
title: "Extending Magnet AXIOM: a Python custom artifact for WhatsApp chats"
focus: Mobile Forensics
date: 2026-06-26
summary: "Writing a Magnet AXIOM custom artifact in Python to parse a WhatsApp Android msgstore.db into examiner-ready chat records — with the forensic logic kept auditable and tested outside the tool."
tools: "Python 3 (standard library), SQLite, Magnet AXIOM custom-artifact API"
dataset: "Self-generated synthetic msgstore.db"
repo: "https://github.com/flyfishing4n6/whatsapp-axiom-artifact"
repo_label: "whatsapp-axiom-artifact"
---

## Problem

Commercial DFIR suites like **Magnet AXIOM** parse a huge catalog of apps out of
the box — but never *every* app, and never instantly when an app changes its
storage format. When an examiner hits an app AXIOM doesn't fully cover, the
professional move isn't to give up or hand-triage the database by eye; it's to
**extend the tool**. AXIOM supports exactly this through its custom-artifact
framework, including custom artifacts written in Python.

This project builds one end to end: a custom artifact that reads a WhatsApp
Android `msgstore.db` and surfaces each chat message — direction, sender, body,
and a normalized UTC timestamp — as AXIOM records, the same way a built-in
artifact would.

The goal is to demonstrate the skill that separates a tool *operator* from a tool
*builder*: understanding the underlying artifact well enough to parse it yourself,
and packaging that parser so it slots into the examiner's existing workflow.

## The artifact: WhatsApp `msgstore.db`

WhatsApp on Android stores conversations in a SQLite database, `msgstore.db`. In
the long-standing legacy schema, the `messages` table holds one row per message
with the fields that matter for reconstruction:

| Column | Meaning |
|---|---|
| `key_remote_jid` | The chat: `<number>@s.whatsapp.net` (1:1) or `<id>@g.us` (group) |
| `key_from_me` | `1` = sent by the device owner, `0` = received |
| `timestamp` | Send/receive time, **milliseconds since the Unix epoch (UTC)** |
| `data` | The message text (`NULL` for media-only / system rows) |
| `remote_resource` | In **group** chats, the participant who actually sent the message |
| `key_id` | WhatsApp's message identifier |

Two details drive most of the parsing logic:

- **Sender attribution is not a single column.** For a *sent* message the sender
  is the device owner; for a *received group* message the real sender is in
  `remote_resource`, not `key_remote_jid` (which only names the group); for a
  *received 1:1* message the sender is the chat jid.
- **Not every row is a message.** Media-only and system rows carry a `NULL`
  `data` field and must be filtered out so they don't show up as empty records.

## Approach

I built the artifact in **two layers**, deliberately, so the forensic logic stays
auditable and testable *outside* AXIOM:

1. **A pure standard-library parser** (`parse_messages`) that opens `msgstore.db`
   **read-only**, reconstructs each message, and returns plain records. This is
   the part that does the forensic reasoning, and it runs anywhere Python runs —
   no AXIOM, no third-party packages.

2. **A thin AXIOM binding** — `Artifact` / `Hunter` classes that feed those
   records into AXIOM through its `axiom` module. AXIOM Process provides `axiom`
   at runtime; when it's absent (standalone runs, unit tests) the binding is
   simply skipped, so the file stays importable and the parser stays exercisable.

This split is the whole point: the interesting, checkable work — *how a message
is reconstructed* — doesn't require a license to read, run, or verify.

### The parser core

Opening the evidence database read-only is a basic soundness requirement — the
tool must never modify the source:

```python
def parse_messages(db_path):
    conn = sqlite3.connect("file:%s?mode=ro" % db_path, uri=True)
    try:
        conn.row_factory = sqlite3.Row
        rows = conn.execute(_QUERY).fetchall()
    finally:
        conn.close()
    ...
```

The `WHERE data IS NOT NULL` clause drops media/system rows, and sender
attribution is handled explicitly:

```python
def _resolve_sender(from_me, chat_jid, remote_resource):
    if from_me:
        return "Local user"
    if remote_resource:        # populated only for group messages
        return remote_resource
    return chat_jid
```

### The AXIOM binding

The binding declares the columns AXIOM will display and emits one hit per
message. It loads only inside AXIOM:

```python
class WhatsAppChatArtifact(Artifact):
    def GetName(self):
        return "WhatsApp Chat Messages (Custom)"
    def CreateFragments(self):
        for label, type_name in COLUMNS:
            self.AddFragment(label, _FRAGMENT_TYPES[type_name])

class WhatsAppChatHunter(Hunter):
    def Register(self, registrar):
        registrar.RegisterFileName("msgstore.db")
    def Hunt(self, context):
        for m in parse_messages(context.FileName):
            hit = Hit()
            hit.SetLocation("messages._id = %d" % m.row_id)
            for label, value in _row_values(m):
                hit.AddValue(label, value)
            context.Publish(hit)

RegisterArtifact(WhatsAppChatArtifact())
```

## Findings

Run against the synthetic database, the parser produces examiner-ready records.
Abbreviated capture (full output is committed in the repo's `data/output.txt`):

```
WhatsApp Chat Messages (Custom) -- 7 records from .../data/msgstore.db

Record 1  [messages._id = 1]
  Chat (JID):  15551237890@s.whatsapp.net
  Direction:   Received
  Sender:      15551237890@s.whatsapp.net
  Message:     Hey, are we still on for the handoff tomorrow?
  Timestamp:   2026-06-20 14:30:12 UTC
  Message ID:  3A1F0001

Record 2  [messages._id = 2]
  Chat (JID):  15551237890@s.whatsapp.net
  Direction:   Sent
  Sender:      Local user
  Message:     Yep. I'll bring the drive.
  Timestamp:   2026-06-20 14:31:05 UTC
  Message ID:  3A1F0002

Record 5  [messages._id = 6]
  Chat (JID):  120363041234567890@g.us
  Direction:   Received
  Sender:      15558675309@s.whatsapp.net      # group sender, not the @g.us chat
  Message:     Morning all -- status check in 10.
  Timestamp:   2026-06-21 09:02:50 UTC
  Message ID:  3A2B0001
```

Two things to notice, both of which validate the parsing logic:

- **The media-only row is gone.** Eight rows were generated; seven records are
  surfaced. The dropped row (`_id = 4`, a `NULL`-body media row) is why the
  record IDs jump from 3 to 5 — empty rows never reach the examiner.
- **Group attribution is correct.** Record 5 comes from an `@g.us` group chat,
  but the `Sender` is the individual participant jid from `remote_resource`, not
  the group identifier. Getting this wrong is a classic way to misattribute who
  said what in a group conversation.

## Validation & integrity

Because this project ships **no AXIOM license**, the in-AXIOM GUI load is *not*
click-tested. Instead, the parsing core — the part that does the forensic work —
is validated directly by an automated test suite (pure-stdlib `unittest`, 8
tests, all passing) covering:

- media-only rows filtered out,
- sent/received direction mapping,
- sender attribution for sent, 1:1-received, and group-received messages,
- millisecond-epoch → UTC formatting (including an exact known value),
- chronological ordering, and
- **read-only soundness** — the database's modification time is unchanged after
  parsing.

<div class="note" markdown="1">
**Honest scope.** The custom-artifact API used here (`Artifact` / `Hunter` /
`Hit` / `RegisterArtifact`) matches Magnet's published Python custom-artifact
structure. Fragment-type constants and the `Hunt` context accessor can vary
between AXIOM versions, so the binding layer should be treated as
version-adaptable glue around the verified parser. What is proven here is the
parsing and attribution logic; what is *designed but not click-tested* is the
in-AXIOM load. That line is drawn deliberately rather than papered over.
</div>

## Provenance

**100% self-generated synthetic data.** There is no real WhatsApp account, no
real export, and no casework of any kind. A generator script
(`scripts/gen_dataset.py`) builds a small `msgstore.db` whose `messages` table
follows the publicly documented legacy WhatsApp Android schema. It deliberately
includes a 1:1 chat, a group chat (to exercise `remote_resource` attribution),
and a media-only row (to exercise filtering), so the test data covers every code
path. The database and the captured output are committed alongside the code.

## Run it

Pure Python 3 standard library — nothing to install.

```bash
python3 scripts/gen_dataset.py                      # build the synthetic msgstore.db
python3 axiom_artifact/whatsapp_chat.py             # parse + print AXIOM-style records
python3 -m unittest discover -s tests -v            # run the test suite
```

## Code

The full, self-contained tool — parser, AXIOM binding, dataset generator, tests,
and captured output — is in the repository:
[**flyfishing4n6/whatsapp-axiom-artifact**](https://github.com/flyfishing4n6/whatsapp-axiom-artifact).
