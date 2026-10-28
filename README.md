# android-re-lab-notes

Reverse-engineering and kernel-level lab notes from my self-directed research on Android and two commercial mobile games, one built on Unity (IL2CPP) and one on Unreal Engine 4. Written in February and March 2026.

## What I worked on

- **Android kernel.** Loading out-of-tree modules on a GKI 6.6 kernel, CFI and LTO, and telling a watchdog bite apart from a CFI failure in a crash log.
- **QEMU lab.** A minimal BusyBox rootfs workflow that tests kernel modules in about a second instead of rebooting a phone.
- **Unity IL2CPP.** Handling protected `global-metadata.dat`, dumping the runtime image, locating methods by RVA.
- **Unreal Engine 4.** Enumerating `GUObjectArray`, decoding `FName`, and using reflection (`UClass` / `FProperty`) to resolve field offsets.
- **Protection mechanisms.** How a commercial anti-cheat probes for root, injection frameworks and debuggers, including the attempts that failed and why.
- **A kernel-side memory-access backend** (research prototype with a small binary protocol), built after studying the design of an existing third-party tool.

## Not public, but demonstrable

Source code, tooling, binaries, memory dumps, version-specific offsets and structures, and in-game material are **not published**. They live in private repositories.

I am happy to walk through them live on a screen share, or to give read access to a specific reviewer, on request. Contact me through GitHub.

## What is here

| Path | Content |
|---|---|
| `notes/` | General engine notes that are not tied to any game (UE4 reflection) |
| `reports/bypass_history/` | Experiment logs, probing history, and a GKI / CFI kernel-module crash analysis |
| `reports/kernel_engine/` | Architecture notes for the memory-access backend, a protocol spec, the QEMU workflow |
| `reports/*.md` | Strategy and brainstorming notes |
| `images/` | Screenshot of the module environment |

The most reusable parts are the GKI crash analysis, the QEMU workflow and `notes/ue4-reflection.md`.

## Status

The notes describe game versions of early 2026. Both games and their protections have changed since, so treat the contents as historical. Not maintained. Several files are exported working conversations and were only lightly cleaned. Language: mostly Chinese, some English.
