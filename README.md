# SMB Proxy Share-Mode VFS

`vfs_smbproxy_sharemode` lets Windows SMB clients and Wine programs on Linux
CIFS mounts see each other's share modes when they use the same files
through the sibling [`smb-proxy-appliance`](../smb-proxy-appliance/).

**Status: design.** No code yet; see [`docs/DESIGN.md`](docs/DESIGN.md).

## Why

File-share databases written for Windows find out whether anyone else is
using them, and keep others out while they rebuild or reset a file, with
deny-write and deny-all opens. The Linux CIFS client sends no deny modes, so
a Wine program behind it is invisible to Windows clients' share modes and
blind to theirs. Wine programs see each other's share modes through a Wine
patch that records them as byte-range locks on reserved bytes. Those locks
reach the proxy's Samba like any other. This module translates between the
two at the proxy.

This is a Samba component, not an appliance component. Like
[`smbproxy-session-vfs`](../smbproxy-session-vfs/), it uses Samba-private
`source3` interfaces, so every binary release is built for one exact Debian
`samba` package revision.

## License

MIT, see [`LICENSE`](LICENSE).
