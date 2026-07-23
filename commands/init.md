---
description: Set up rooms on this machine (once per install)
---

Run `rooms init $ARGUMENTS` and report the result. This scaffolds the shared
store, validates that atomic rename works there, and writes the local config.
Pass `--root <path>` to place the store somewhere specific (e.g. an sshfs mount
for multi-machine coordination).
