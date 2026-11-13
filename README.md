# threadtalk

**Harness-agnostic messaging between AI agent threads via a shared mailbox of plain files.**

No server. No SDK. No schema. No daemon required (a tiny watcher helps).

If an agent can read/write files and run a shell command, it can join a threadtalk box.

## Why

Agent harnesses (DeepSeek Harness, Claude Code, Codex, …) can't talk to each other.
But they *all* can read a file. So the protocol is: **a folder, a naming convention,
and a trigger**. That's it.

This is not a replacement for [A2A](https://github.com/a2aproject/A2A) or MCP — those are
formal interop protocols. threadtalk is the "two machines, a folder, and a convention"
tier. It is deliberately tiny.

## The spec (five lines)

```
1. mailbox:   exchange/to-<name>/   + consumed/
2. message:   *.instr with a first line of PROMPT: / CMD: / NOTE:, or a bare file
3. reply:     <name>.reply written back to the sender's mailbox
4. consume:   move the message to consumed/ only after delivery succeeded
5. trigger:   a watcher (or a direct invoke) runs  <agent> -p "<task>"
```

Full details in [SPEC.md](SPEC.md).

## Layout

```
<box>/
├── exchange/
│   ├── to-alice/      # mailbox for alice
│   ├── to-bob/        # mailbox for bob
│   └── consumed/      # delivered items (audit trail)
└── .staging/          # atomic transfer staging (never read by receivers)
```

## Quickstart

Two machines, SSH reachable, key-based auth:

```bash
# on each side
export THREADTALK_ME=alice          # this side's mailbox name
export THREADTALK_PEER=bob          # the other side
export THREADTALK_HOST=user@host    # ssh target (omit for local/shared-FS use)
export THREADTALK_BOX=/path/to/box  # box path (on the remote if HOST is set)

# send a file or a task
bin/threadtalk notes.txt
bin/threadtalk "PROMPT: summarise the attached data and reply."

# receive (once)
bin/threadtalk get

# receive continuously (5s)
bin/threadtalk-watch 5
```

## Intros (pairing two threads)

Two threads that have never met can't guess each other's mailbox name. So there is a
tiny, deliberate pairing step — an **intro** — and the *acceptance* is the act of
opening it in the other thread.

```bash
# in thread A
threadtalk intro bob "build task"
#   intro ba8ade: alice -> bob
#   -> in the 'bob' thread, run: threadtalk intro-accept ba8ade

# in thread B (the user carries the id across, or thread B lists pending intros)
threadtalk intros                      # bob sees: ba8ade from alice -> accept it
threadtalk intro-accept ba8ade
#   paired: you are 'bob', peer is 'alice'   (pair ba8ade)
#   export THREADTALK_ME='bob'
#   export THREADTALK_PEER='alice'
```

A box can hold **many** pairings. `threadtalk intros` lists all accepted pairings
(with a `<- you are …` marker) and every pending intro, split into *for me* (actionable)
and *waiting on them*.

A pair id can also drive the mailbox config directly:

```bash
THREADTALK_ME=bob THREADTALK_PAIR=ba8ade threadtalk status   # me=bob peer=alice
```

Notes:

- **No handshake, no daemon, no registry.** An intro is a file; an accept is a file.
- The accept is a **deliberate gesture** — you choose to open it in the other thread.
- Unaccepted intros just sit there. Nothing expires; nothing rots.

## Adapters

The only harness-specific piece is "run this instruction". Adapters are one-liners:

| Harness | Adapter |
|---|---|
| DeepSeek Harness | `adapters/dsh` → `dsh --profile headless "$1"` |
| Claude Code | `adapters/claude` → `claude -p "$1"` |
| Codex | `adapters/codex` → `codex exec "$1"` |

Point `THREADTALK_AGENT` at one, and `threadtalk-watch` will run incoming
`PROMPT:` instructions through it.

## Design rules (keep it small)

- **Additive, never rename.** Paths and mailboxes are long-lived; vocabulary is cheap.
- **Atomic delivery.** Stage, then `mv` into the mailbox. Never let a receiver see a
  partial file (this is what causes truncation bugs with large files).
- **Consume only after success.** If delivery failed, leave the message where it is.
- **No registry, no handshake.** Identity is a mailbox name. Nothing to register.

## License

MIT
