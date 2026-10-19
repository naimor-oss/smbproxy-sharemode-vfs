# Design

## The problem

Two kinds of client use one share through the proxy:

- **Windows clients** (SMB2/3 with share modes). Samba enforces their access
  and deny modes against each other as Windows would.
- **Wine programs on Linux CIFS mounts** (one mount per container, each its
  own session). The Linux CIFS client opens every file with all sharing
  allowed and sends no deny modes, so Samba sees none of the Wine program's
  denials.

Measured between a Windows client and a Wine program, each on its own session
through the proxy, against an SMB1 backend:

| Wine program has… | Windows client… | Windows would | Result |
| --- | --- | --- | --- |
| the file open read-write | opens it deny-write | refuse | refused (Samba sees the CIFS open's access) |
| the file open deny-all or deny-write | opens it for writing | refuse | **allowed** |

| Windows client has… | Wine program… | Windows would | Result |
| --- | --- | --- | --- |
| the file open deny-all or deny-write | opens it for writing | refuse | refused, but as `LOCK_VIOLATION` |
| the file open read-write | opens it deny-write | refuse | **allowed** |

The last row is the one that hurts. A program that probes "am I alone?" with
a deny-write open concludes it is, while Windows users are in. It then resets
shared state or runs maintenance under them. Between two Wine programs, on
separate mounts, share modes already work: see the next section.

## The contract with Wine

The Wine fork's share-mode patch (`naimor-oss/wine`, branch
`naimor/wine-11.0`, commit `7c5455764bf0`, `server/fd.c`) records each open as
**shared** OFD locks on reserved bytes, held on a read-only descriptor of
its own:

| Byte | Meaning: this open… |
| --- | --- |
| `0x7fffffff00000000 + 0` | has read access (`FILE_READ_DATA`, `FILE_EXECUTE`) |
| `+ 1` | has write access (`FILE_WRITE_DATA`, `FILE_APPEND_DATA`) |
| `+ 2` | has delete access |
| `+ 3` | denies read (no `FILE_SHARE_READ`) |
| `+ 4` | denies write |
| `+ 5` | denies delete |

Before an open, it tests the conflicting bytes with an exclusive
`F_OFD_GETLK`. Its access is tested against others' denials, and its denials
against others' access. An open with none of read, write, execute or delete
access neither tests nor locks. Over a CIFS mount, these locks and tests
reach the proxy as ordinary SMB byte-range locks. A test is a lock attempt
followed by an unlock.

This layout is **contract version 1**. The module and the patch change it
only together.

## What the module does

1. **Windows open, checked against Wine's bytes** (every open, any client).
   - On create, if the open has read, write, execute or delete access, read
     the brlock records on the file's reserved bytes held by other opens.
   - Refuse with `NT_STATUS_SHARING_VIOLATION` when this open's access meets a
     held deny byte, or its denials meet a held access byte.
   - Samba's own check runs as usual; this only adds the conflicts it can't
     see.
2. **Windows open, mirrored as Wine's bytes** (opens from mirrored clients
   only).
   - After a successful create with data access, take shared byte-range locks
     on the bytes for its access and denials, owned by that open. That way
     they go away when it closes or its session drops.
   - A Wine program's test then finds them and gets a sharing violation, as
     on Windows.
   - Mirroring is **only** for clients that send share modes themselves.
     Every open from a Linux CIFS client also arrives as a real SMB open
     (share-all), and so does Wine's read-only share descriptor. Mirroring
     those would make each Wine open conflict with itself.
   - **Open question:** how a mirrored client is recognised (an `smb.conf`
     parameter naming the non-mirrored client addresses, or an account
     group, or another attribute Samba can see at tree connect).
3. **Identity.** The checks and locks use the same file identity as
   `smbproxy-session-vfs`'s per-tree mounts, which Samba's `fileid` module
   normalises (`fileid:algorithm = fsname`). The order in `vfs objects`
   relative to `smbproxy_session fileid` is part of the configuration, and
   is tested.

## Limits

- **Not atomic.** As in the Wine patch, two opens racing on different
  clients can both succeed.
- **Error codes.** It doesn't change error codes the Linux CIFS client
  produces itself (the `LOCK_VIOLATION` row above, `ERROR_5`/`ERROR_21` for
  reads and writes in a locked range). Those belong to the client side.

## Acceptance

A release gate like `smbproxy-session-vfs`'s:
- the exact-Samba build;
- the sibling appliance's lock scenarios;
- a two-client share-mode matrix, run against the Wine fork revision the
  contract names:
  - Windows client against Wine program, both directions;
  - Wine against Wine on separate mounts;
  - Wine against Wine in one wineserver.

Each case passes when a conflicting open is refused, and a non-conflicting
one allowed, as on Windows.
