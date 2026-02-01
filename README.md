# Samba-CTDB-CTDB_SOCKET-Security-Issue
Security analysis and disclosure of a CTDB socket handling vulnerability in Samba (Bugzilla #15921)

---


# Samba CTDB_SOCKET Security Issue (Bugzilla #15921)

## Overview
This repository documents a security issue identified in the **CTDB (Clustered Trivial Database)** component of Samba, related to the handling of the `CTDB_SOCKET` environment variable outside of test mode.

The issue was responsibly reported and later acknowledged and fixed by the Samba Team.  
This document serves as a technical archive and reference for the vulnerability and its resolution.

---

## Affected Component
- **Project:** Samba
- **Subsystem:** CTDB (ctdb-common)
- **Relevant Function:** `ux_socket_bind()` in `ctdb_daemon.c`
- **Configuration Variable:** `CTDB_SOCKET`

---

## Issue Summary
At the time of discovery, the `CTDB_SOCKET` environment variable could be used **outside of test mode**, despite not being intended for such usage.

This behavior introduced a **potential security risk** due to how the CTDB daemon secured the UNIX domain socket associated with this variable.

Specifically:
- The socket path could be influenced via `CTDB_SOCKET`
- After binding the socket, the code applied `chown(2)` and `chmod(2)` operations on the socket path
- This pattern can potentially enable a **symlink race condition**, depending on filesystem timing and control

---

## Technical Details
To secure the CTDB UNIX socket, the following operations were performed in `ux_socket_bind()`:

- `bind(2)` on a filesystem path
- Followed by `chown(2)` and `chmod(2)` on that path

When the socket path is externally controllable, post-bind ownership and permission changes may expose a **race window**, where an attacker could attempt to replace the socket path with a symbolic link.

While this behavior did not immediately constitute a known exploit, it represented an **unnecessary and avoidable attack surface**.

---

## Fix and Code Changes

### Important Diff Reference
The critical change that resolved the issue is available in the merge request diff below:

🔗 **Diff highlighting the restriction of `CTDB_SOCKET` to test mode only:**  
https://gitlab.com/samba-team/samba/-/merge_requests/4239/diffs?commit_id=692cdabe3a43bb9cc9e3e050415bd0ef6be9ee83

This diff shows:
- Removal of support for `CTDB_SOCKET` outside of `CTDB_TEST_MODE`
- Adjustments ensuring that the socket path handling is only respected when testing is explicitly enabled
- Warnings added to discourage unintended use

---

## Resolution
The Samba Team resolved the issue by enforcing the intended design constraint:

- `CTDB_SOCKET` is now respected **only when `CTDB_TEST_MODE` is explicitly enabled**
- The clustered member self-test environment ensures `CTDB_TEST_MODE` is set
- Additional warnings were added to discourage unintended usage

This change removes the vulnerable code path from non-test environments, effectively mitigating the potential security issue.

---

## Design Considerations
An alternative approach using `fchown(2)` and `fchmod(2)` on the socket file descriptor was considered.  
However:
- These system calls are not well-defined for sockets
- Previous implementations relied on Linux-specific behavior
- Such code was removed earlier due to portability concerns

Given these constraints, restricting `CTDB_SOCKET` usage to test mode was the safest and most portable solution.

---

## Official References
- **Bugzilla Report:** Bug #15921  
  https://bugzilla.samba.org/show_bug.cgi?id=15921

- **Merge Request (Fix):**  
  https://gitlab.com/samba-team/samba/-/merge_requests/4239

---

## Credit
**Reported by:** GUIAR OQBA  
📧 techokba@gmail.com

The issue was acknowledged and fixed by the Samba Team.

---

## Disclosure
This issue was reported following responsible disclosure practices.  
Public documentation was created after an official fix was merged upstream.

