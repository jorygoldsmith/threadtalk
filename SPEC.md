# threadtalk — specification

Version 1. Deliberately small. If a change makes this longer, reconsider it.

## 1. The box

A `box` is a directory, reachable by every participant by *some* transport
(shared filesystem, SSH, rsync, …). Transport is not part of this spec.

```
<box>/
├── exchange/
│   ├── to-<name>/     one mailbox per participant
│   └── consumed/      delivered items, prefixed by the receiving side
└── .staging/          transfer staging; receivers MUST ignore this
```

A participant is identified only by its `<name>` (e.g. `alice`, `mac`, `cluster`).
There is no registry and no handshake.

## 2. Messages

A message is any file dropped into `exchange/to-<name>/`.

If its content begins with one of these keywords on the first non-empty line, it is
an **instruction**:

| Keyword | Meaning |
|---|---|
| `PROMPT:` / `AGENT:` | the receiver's agent should perform the task; everything after the keyword (and following lines) is the task text |
| `CMD:` | the receiver should run the given shell command |
| `NOTE:` | informational; no action |

Anything else is a **payload** — data to be delivered as-is.

## 3. Replies

The receiver writes its answer to the sender's mailbox as `<name>.reply`, where
`<name>` is the instruction's filename.

A reply SHOULD begin with a short header:

```
To: <peer>
From: <self>
Date: <ISO-ish>
In response to: <original name>
Status: <status>
---
<body>
```

`Status: NO-AGENT` is reserved for "this side has no agent configured" — it is a
valid, non-error outcome.

## 4. Delivery and consumption

```
deliver:  copy/stage the message into the receiver's local receive area
consume:  move it from exchange/to-<name>/ to exchange/consumed/<self>.<name>
```

**Rules:**

1. **Atomic transfer.** Never write directly to the final mailbox path with a
   long-running transfer. Write to `.staging/`, then `mv` into the mailbox.
   A receiver must never observe a partially-written message.
2. **Consume only after successful delivery.** If the copy failed, leave the
   message in the mailbox and retry on the next pass.
3. **Idempotence.** `consumed/<self>.<name>` existing means "already handled";
   skip it.

## 5. Trigger

The box does not push. Something must run:

- **Watcher** (preferred): every N seconds, scan `exchange/to-<self>/` and process
  new messages.
- **Direct invoke**: a sender may trigger a single receiver pass over a side
  channel (e.g. `ssh host 'threadtalk get'`).

A one-shot trigger is preferred over polling when the sender knows the receiver
is idle: it costs nothing when nothing happens.

## 6. Adapters

Executing an instruction is the only harness-specific concern.

**Contract:** an adapter takes the instruction text on stdin (or as `$1`) and
prints the agent's answer on stdout, exiting non-zero on failure.

Reference adapters:

```
dsh (DeepSeek Harness)   dsh --profile headless "$task"
claude (Claude Code)     claude -p "$task"
codex (Codex)            codex exec "$task"
```

## 7. Non-goals

Not in scope, on purpose:

- discovery, capability negotiation, authentication (use the transport's)
- typed schemas or versioned envelopes
- ordering guarantees across mailboxes
- a daemon or registry
