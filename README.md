# lolm-reverse-notes

Research notes from a reverse-engineering study of an Android Unity (IL2CPP) game and its anti-cheat, written in February 2026.

- **Notes only. No source code, binaries, offsets tables or tooling are included.**
- The notes describe one game version of that time. The game and its protection have since changed, so treat everything here as historical.
- Language: mostly Chinese, some English. Several files are exported working conversations and were only lightly cleaned.
- Written out of curiosity, studying how an existing public project worked. Not maintained.

## Layout

| Path | Content |
|---|---|
| `reports/bypass_history/` | Experiment logs: dynamic-debugging attempts, probing history, a GKI / CFI kernel-module crash analysis |
| `reports/kernel_engine/` | Architecture notes for a kernel-side memory-access backend, a protocol spec, a QEMU minimal-rootfs test workflow |
| `reports/*.md` | Strategy and brainstorming notes |
| `images/` | Screenshot of the module environment |

The generic, reusable parts are the GKI analysis and the QEMU workflow.
