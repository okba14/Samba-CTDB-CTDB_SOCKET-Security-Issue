# Samba-CTDB-CTDB_SOCKET-Security-Issue

![Security](https://img.shields.io/badge/category-security-critical?style=flat-square)
![Project](https://img.shields.io/badge/project-Samba-blue?style=flat-square)
![Component](https://img.shields.io/badge/component-CTDB-orange?style=flat-square)
![Disclosure](https://img.shields.io/badge/disclosure-responsible-success?style=flat-square)
![Status](https://img.shields.io/badge/status-fixed-brightgreen?style=flat-square)

Security analysis and disclosure of a CTDB socket handling vulnerability in Samba  
(**Bugzilla #15921**)

---

## Overview

This repository documents a security issue identified in the  
**CTDB (Clustered Trivial Database)** component of **Samba**, related to the handling of the
`CTDB_SOCKET` environment variable outside of test mode.

The issue was **responsibly reported**, officially **acknowledged**, and **fixed upstream** by the Samba Team.

> This repository exists as a **technical archive and long-term reference**, ensuring the issue is publicly indexed and searchable.
> This documentation is strictly based on upstream acknowledgements and merged fixes.

---

## Affected Components

| Field                  | Details |
|-----------------------|---------|
| **Project**            | Samba |
| **Subsystem**          | CTDB (`ctdb-common`) |
| **Source File**        | `ctdb_daemon.c` |
| **Relevant Function**  | `ux_socket_bind()` |
| **Environment Variable** | `CTDB_SOCKET` |

---

## Issue Summary

At the time of discovery, the environment variable: 

```bash
CTDB_SOCKET
```

---


could be used **outside of test mode**, even though it was **never intended** for production usage.

This behavior introduced a **potential security risk** due to how the CTDB daemon secured
the associated UNIX domain socket.

Specifically:

- The socket path could be influenced externally via `CTDB_SOCKET`
- After binding the socket, the code applied:
  - `chown(2)`
  - `chmod(2)`
- Performing ownership and permission changes on a filesystem path that may be externally influenced can expose a **symlink race window**

> While no exploit was demonstrated, this represented an **unnecessary and avoidable attack surface**.

---

## Technical Details

The function:

```bash
ux_socket_bind()
```

performed the following sequence:

```text
bind(2)
chown(2)
chmod(2)
```
on a UNIX domain socket path.

---

> When the socket path is controllable, post-bind permission changes may allow a symbolic
> link to be swapped during the operation, potentially redirecting ownership or permissions
> to an unintended filesystem object.
>
> This pattern is a well-known class of filesystem race condition and is generally avoided
> in security-sensitive code paths.

---

## Fix and Code Changes

### Important Diff Reference

The exact fix implemented by the Samba Team can be reviewed here:

🔗 **Merge Request Diff (commit-specific):**  
https://gitlab.com/samba-team/samba/-/merge_requests/4239/diffs?commit_id=692cdabe3a43bb9cc9e3e050415bd0ef6be9ee83

This diff shows:

- Restriction of `CTDB_SOCKET` usage to **test mode only**
- Enforcement of `CTDB_TEST_MODE` in relevant self-test environments
- Additional warnings to discourage unintended or unsafe usage

---

## Resolution

The issue was resolved by enforcing the original design intent:

> `CTDB_SOCKET` is now respected **only** when `CTDB_TEST_MODE` is explicitly enabled.

As a result:

- The vulnerable code path is no longer reachable in non-test environments
- The potential symlink race condition is eliminated from production usage

---

## Design Considerations

An alternative design using:

```c
fchown(2)
fchmod(2)
```
on the socket file descriptor was discussed. However:

- These system calls are **not well-defined for sockets**
- Previous implementations relied on **Linux-specific behavior**
- Such code had already been removed earlier due to **portability concerns**

Given these constraints, restricting the environment variable to test mode
was the safest and most portable solution.

---

## Official References

- **Bugzilla Report:**  
  https://bugzilla.samba.org/show_bug.cgi?id=15921

- **Merge Request:**  
  https://gitlab.com/samba-team/samba/-/merge_requests/4239

---

## Credit 🇩🇿

**Reported by:**  
**GUIAR OQBA**  
📧 techokba@gmail.com  

> Security researcher  🇩🇿  
> The issue was officially acknowledged and fixed by the Samba Team.


---

## Disclosure

This issue was reported following **responsible disclosure practices**.  
Public documentation was created **after the fix was merged upstream**.
