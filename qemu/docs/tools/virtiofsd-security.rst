========================
Virtiofsd Security Guide
========================

Introduction
============
This document covers security topics for users of virtiofsd, the daemon that
implements host<->guest file system sharing.  Sharing files between one or more
guests and the host raises questions about the trust relationships between
these entities.  By understanding these topics users can safely deploy
virtiofsd and control access to their data.

Architecture
============
The virtiofsd daemon process acts as a vhost-user device backend, implementing
the virtio-fs device that the corresponding device driver inside the guest
interacts with.

There is one virtiofsd process per virtio-fs device instance.  For example,
when two guests have access to the same shared directory there are still two
virtiofsd processes since there are two virtio-fs device instances.  Similarly,
if one guest has access to two shared directories, there are two virtiofsd
processes since there are two virtio-fs device instances.

Files are created on the host with uid/gid values provided by the guest.
Furthermore, virtiofsd is unable to enforce file permissions since guests have
the ability to access any file within the shared directory.  File permissions
are implemented in the guest, just like with traditional local file systems.

Security Requirements
=====================
Guests have root access to the shared directory.  This is necessary for root
file systems on virtio-fs and similar use cases.

When multiple guests have access to the same shared directory, the guests have
a trust relationship.  A broken or malicious guest could delete or corrupt
files.  It could exploit symlink or time-of-check-to-time-of-use (TOCTOU) race
conditions against applications in other guests.  It could plant device nodes
or setuid executables to gain privileges in other guests.  It could perform
denial-of-service (DoS) attacks by consuming available space or making the file
system unavailable to other guests.

Guests are restricted to the shared directory and cannot access other files on
the host.

Guests should not be able to gain arbitrary code execution inside the virtiofsd
process.  If they do, the process is sandboxed to prevent escaping into other
parts of the host.

Daemon Sandboxing
=================
The virtiofsd process handles virtio-fs FUSE requests from the untrusted guest.
This attack surface could give the guest access to host resources and must
therefore be protected.  Sandboxing mechanisms are integrated into virtiofsd to
reduce the impact in the event that an attacker gains control of the process.

As a general rule, virtiofsd does not trust inputs from the guest, aside from
uid/gid values.  Input validation is performed so that the guest cannot corrupt
memory or otherwise gain arbitrary code execution in the virtiofsd process.

Sandboxing adds restrictions on the virtiofsd so that even if an attacker is
able to exploit a bug, they will be constrained to the virtiofsd process and
unable to cause damage on the host.

Seccomp Whitelist
-----------------
Many system calls are not required by virtiofsd to perform its function.  For
example, ptrace(2) and execve(2) are not necessary and attackers are likely to
use them to further compromise the system.  This is prevented using a seccomp
whitelist in virtiofsd.

During startup virtiofsd installs a whitelist of allowed system calls.  All
other system calls are forbidden for the remaining lifetime of the process.
This list has been built through experience of running virtiofsd on several
flavors of Linux and observing which system calls were encountered.

It is possible that previously unexplored code paths or newer library versions
will invoke system calls that have not been whitelisted yet.  In this case the
process terminates and a seccomp error is captured in the audit log.  The log
can typically be viewed using ``journalctl -xe`` and searching for ``SECCOMP``.

Should it be necessary to extend the whitelist, system call numbers from the
audit log can be translated to names through a CPU architecture-specific
``.tbl`` file in the Linux source tree.  They can then be added to the
whitelist in ``seccomp.c`` in the virtiofsd source tree.

Mount Namespace
---------------
During startup virtiofsd enters a new mount namespace and releases all mounts
except for the shared directory.  This makes the file system root `/` the
shared directory.  It is impossible to access files outside the shared
directory since they cannot be looked up by path resolution.

Several attacks, including `..` traversal and symlink escapes, are prevented by
the mount namespace.

The current virtiofsd implementation keeps a directory file descriptor to
/proc/self/fd open in order to implement several FUSE requests.  This file
descriptor could be used by attackers to access files outside the shared
directory.  This limitation will be addressed in a future release of virtiofsd.

Other Namespaces
----------------
Virtiofsd enters new pid and network namespaces during startup.  The pid
namespace prevents the process from seeing other processes running on the host.
The network namespace removes network connectivity from the process.

Deployment Best Practices
=========================
The shared directory should be a separate file system so that untrusted guests
cannot cause a denial-of-service by using up all available inodes or exhausting
free space.

If the shared directory is also accessible from a host mount namespace, it is
recommended to keep a parent directory with rwx------ permissions so that other
users on the host are unable to access any setuid executables or device nodes
in the shared directory.  The `nosuid` and `nodev` mount options can also be
used to prevent this issue.
