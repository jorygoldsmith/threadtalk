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
