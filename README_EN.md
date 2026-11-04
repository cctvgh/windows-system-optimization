# windows-system-optimization

Whole-machine Windows optimization skill: startup-item audit and cleanup, disk-space recovery (cache cleanup + symlink migration + pagefile relocation), runtime tuning, and activation status check. Built for **enterprise-managed machines** — security, compliance and endpoint-management software is never touched, and uncertain items are always confirmed by the user.

## Features

- **Six-source startup audit**: HKLM/HKCU Run and RunOnce, startup folders, logon/boot scheduled tasks, auto services, StartupApproved leftovers
- **Three-way classification**: never touch (security agents, core services) / clearly redundant (legacy preloaders, cloud-init, duplicate updaters) / ask the user (daily-use apps, grouped multi-select)
- **Disk recovery**: cache & log cleanup, large-folder `robocopy` + `mklink` migration (delete source only if size delta < 5%), pagefile relocation to a data drive
- **Runtime tuning**: visual effects and startup delay, TCP parameters, event-log size caps, defragmentation (analyze first, run in background), `sfc /scannow` verified via CBS.log
- **Safety rails**: `reg export` backup before registry edits, scheduled tasks disabled rather than deleted, migration never uninstalls, reboot-required changes flagged
- **Protected-service detection**: access denied usually means vendor registry protection, not a permission bug — stop escalating immediately to avoid triggering security alerts

## Quick start

1. Place this folder in your skills directory (or let the agent load `SKILL.md`).
2. Describe the goal, e.g.:

```
C盘只剩 8G 了，帮我看哪些能清理、哪些能迁到 D 盘
```

```
开机太慢，帮我审计一下自启项，能关的列出来让我选
```

3. The skill follows: scan (read-only) → classify → user confirmation → backup → execute → verify → report.

## Structure

```
windows-system-optimization/
├── SKILL.md
├── references/
│   ├── enterprise-security-constraints.md   # security agent identification, protected services
│   └── symlink-migration.md                 # five-step migration, command templates, pitfalls
├── README.md
├── README_EN.md
├── LICENSE
└── .gitignore
```

## License

MIT License. See `LICENSE`.

## Disclaimer

This skill modifies system configuration via scripts and registry operations and carries risk. Always back up first and pilot in a non-production machine. The user bears responsibility for system faults, data loss or compliance issues caused by misuse, environment differences or ignoring the safety rules above.