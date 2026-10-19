# Agent Guide

This repository owns `vfs_smbproxy_sharemode`, a private Samba `source3` VFS
module for the sibling SMB proxy appliance. It bridges Windows share modes
between SMB clients that send them and Wine programs on Linux CIFS mounts,
which can't. Read [`docs/DESIGN.md`](docs/DESIGN.md) first, then
[`../dev-commons/CONTEXT.md`](../dev-commons/CONTEXT.md) and
[`../dev-commons/STYLE.md`](../dev-commons/STYLE.md).

## Non-negotiable contracts

- A release artifact is compatible with exactly one Debian `samba` binary
  package revision, as for `../smbproxy-session-vfs`. Never weaken the exact
  dependency or the runtime guard.
- The reserved-byte layout (`docs/DESIGN.md`, "The contract with Wine") is
  shared with the Wine fork's share-mode patch. Change it only together with
  that patch, under a new contract version.
- Mirror locks are taken only for opens from SMB clients that send share
  modes themselves, never for the Linux CIFS clients that carry Wine's own
  reserved-byte locks: those would conflict with themselves.
- Opens without read, write, execute or delete data access take no locks and
  are never refused, as in Wine and Windows.
- Interoperability observations are evidence, not guarantees: record the
  client, server, Samba package, kernel and a reproducer.

## Repository boundary

The module, its build and packaging, and its compatibility ledger live here.
Appliance configuration, deployment and lab scenarios live in
`../smb-proxy-appliance`; per-tree upstream sessions in
`../smbproxy-session-vfs`; the Wine side in `../wine` (fork) and
`../wine-webapp`.
