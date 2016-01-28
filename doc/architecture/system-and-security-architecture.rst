================================================
 Ostro (TM) OS System and Security Architecture
================================================

Introduction
============

Ostro (TM) OS is a Linux(TM) based operating system for Internet of
Things (IoT). It provides the basis for building innovative, secure
IoT devices. Because security is crucial for IoT, security mechanisms
are tightly integrated into the system architecture. The primary
customers for Ostro OS are considered to be OEMs, OSVs, Service
Providers, and developers building their own devices.

This document introduces key concepts in Ostro OS and how they fit
together. The target audience are developers who want to learn about
Ostro OS and/or use the OS in their own product. Application
developers will also find useful background information. However,
there will also be a dedicated document more focused on application
development and installation.

It is understood that there is no “one solution that fits all” when
talking about IoT security. Thus, what Ostro OS provides are
(pre-configured) mechanisms and templates for supported use cases. The
recommended configuration is delivered pre-compiled, so building a
full disk image is fast. Less common configurations are still
possible, but may require compiling from source.

The expectation is that most devices will have just a few trusted
applications and not have access to an application store. Compared to,
say, a mobile phone OS, the security focus is moved from protecting
against malicious application to protecting against attacks coming
from outside the device (network or malicious physical
access). However, the security is planned to be scalable. If there is
a problem that the default security setup does not cover, the security
can be stepped up by adding components and configuration.

In addition, supporting multiple real users of the same device is less
important and left to applications to support. Therefore the security
model can use Unix users to distinguish between different
applications. All interaction with the device is supposed to be done
through applications, not by logging into the core system directly.

This architecture documentation explains how the security is currently
integrated. It does not attempt to explain alternative or future
implementations, and therefore can be used as documentation of the
current revision of Ostro OS that the documentation is shipped
with. Some items that are likely to be added soon are already
documented, but clearly marked as *TODO*. Especially covered are the
places where the security model differs from baseline Linux security
that can be expected from any mainstream desktop Linux distribution.

For a discussion of potential other security mechanisms see the
`threat analysis document`_.

.. _`threat analysis document`: security-threat-analysis.rst


Key Security Concepts
=====================

* scalable security: protection mechanisms can be turned on to
  increase security (defence in depth) or turned off to decrease
  overhead and/or complexity. However, not all combinations are
  tested, see `Production and Development Images`_

* Unix DAC is used to separate applications. Each application runs
  under a different uid. Supporting multiple real users of the same
  device is left to applications to support.

* Permission checks are based on Unix group membership.

* When using DAC alone, applications can communicate with each
  other. Application authors must be careful about setting permission
  bits as intended to prevent that. Because applications are trusted,
  this is acceptable. When that is undesirable and/or to mitigate
  risks when applications get compromised, optionally Smack as a MAC
  mechanism can be used to separate applications further (*TODO*).

* Namespaces and containers further restrict what applications can
  access and do.

* An IPv4/IPv6 firewall controls access to local services.

* The core OS is hardened with a combination of suitable compiler
  flags, running services with minimal capabilities, and avoiding
  insecure configurations.

Boot Process + Secure Boot
==========================

1. The first stage is hardware specific. Currently supported is UEFI
   Secure Boot: Ostro OS installs the Linux kernel and initramfs
   combined into a single, signed UEFI blob. That blob gets loaded and
   verified directly by the device firmware. The following stages are
   the same for all devices.

2. The Linux kernel starts up and transfers control to the initramfs.
   The initramfs goes through several steps, the main ones being:

   a) it searches (and if necessary, waits) for the root partition

   b) when using IMA with EVM, and booting for the first time, it is
      necessary to personalize the EVM protection to the current
      device (*TODO*); more on that below.

   c) if used, IMA/EVM policy gets activated

   d) the file system root is changed to the root partition and
      control is transferred to system as PID 1

3. Systemd finishes bringing up the userspace side of the system.

4. Applications get started by systemd because they are also systemd
   service (see `Applications`_).

The chain of trust in Secure Boot cannot stop at the kernel. To
protect against offline attacks, all files loaded for booting the core
system must also be checked for integrity before using them. IMA/EVM
ensure that as follows:

* IMA hashes the content of the file and stores the hash in a
  security.ima xattr attached to the file’s inode. This hash can
  already be calculated on the build host for read-only files and be
  transferred to the device together with the file content, in which
  case it is possible to sign it with a secret private key.

* This is the most secure method: the kernel then checks the signature
  using a public key rooted in a CA that got compiled into the kernel
  and thus is anchored in the UEFI Secure Boot chain. Not even a root
  process can create new files with a valid signed security.ima xattr.

* The less secure approach is to allow file creation and have the
  kernel create the content hash on file closing. This mode is
  necessary for files which have to be created or modified on the
  device. It depends on the IMA policy where such files are accepted.

* EVM hashes inode meta data like protection bits and xattrs like
  security.ima and stores the hash in security.evm. The inode number
  is also included in the hashed data, to prevent copying security.evm
  from one inode to another. Because the update mechanism is file
  based, inode numbers vary between devices and therefore signing the
  EVM hash on the build host is impossible. Instead, Ostro OS relies
  on a per-device secret key stored in a TPM, with a less secure
  software-only solution as fallback (see below). That key is used to
  encrypt and decrypt the EVM hash. Because the key is sealed in the
  TPM, even an attacker with physical access to the device and the
  root partition cannot access it, which makes it possible to detect
  offline attacks because files created or modified by an attacker
  with not have a security.evm that passes the runtime checks. The
  secret key needed for EVM is set up by the initramfs. This all
  happens automatically, without the need for user interaction.

* All files are checked on access and if the hashes does not match the
  current content or meta data, access fails with a “permission
  denied” error. When listing directories with files that do not pass
  the check, only the file names will be visible. Additional details
  that belong to the inode, like size or protection bits, cannot be
  shown because access to them gets denied.

* Without a TPM, Ostro OS falls back to software encryption keys for
  EVM. This still protects against online attacks (because the kernel
  can limit access to the secret key) but is not sufficient to prevent
  offline attacks.


Filesystem Layout
=================

Ostro OS needs to protect data differently, depending on sensitivity
and usage patterns. Files used by the core system change infrequently
and can be protected by IMA/EVM. But IMA/EVM changes the performance,
semantic and error handling of the filesystem and thus is less
suitable for application data with unknown usage patterns.

Here is an overview of the different parts of the virtual file
system. Specific Ostro devices will likely map this to different
partitions because that way the filesystem UID can be used in the IMA
policy to treat files differently depending on their
location. However, a simpler Ostro OS configuration could also drop
IMA and use a simpler partition layout where everything is stored in
the same writable partition.


``/``
  Includes everything that is not explicitly listed
  below. Conceptually this is read-only and will only be mounted
  read/write during system software updates (*TODO*: in practice,
  currently / is mounted read/write all the time but all services
  except software update use systemd's ``ProtectSystem=full`` to make the
  root filesystem appear read-only to them). All files are using
  signed IMA hashes and thus cannot be modified on the device (*TODO*:
  because we have not finished the transition to a clean separation
  between read-only and read/write files in different partitions, the
  current IMA policy also allows hashes created on the device, which
  allows circumventing the offline protection).

``/var``
  Persistent data which can be written on the device. Protected by
  IMA/EVM with hashes created on-the-fly by the kernel on the device.

``/tmp`` ``/var/run``
  A tmpfs which will not survive a reboot.

``/home``
  Persistent, read/write, no IMA/EVM. Each application gets its own
  home directory with access limited to the application.

/etc and the files in it are part of the core OS and thus considered
read-only. However, there are a few noteworthy exceptions:

/etc/ld.so.cache
 Its content depends on the currently installed shared libraries,
 which may vary by device. Therefore it needs to be updated on the
 device after system software installation or updates. Its real
 location thus is in /var (*TODO*).

/etc/machineid
 Currently systemd creates a machine ID when booting and writes it to
 /etc/machineid when /etc becomes writeable. When moving to the strict
 IMA policy, we need to prevent that (because the file would become
 unreadable, which breaks several systemd services) or move it to /var
 (*TODO*).


User, Group and Privilege Management
====================================

User and group management files (like ``/etc/passwd``) are
read-only. That means that the core system can only have static system
users. It is not possible to set a root password.

To become root in the core system:

  * After installation and before booting for the first time, add a
    public key to the ``~root/.ssh/authorized_keys`` file (*TODO*:
    create the directory and file with correct permissions, document
    the exact location, which may vary between development and
    production image)

  * *Only in the development image*: log in via a local console and or
    serial port as root. A PAM module allows root to log in without
    password. Because development and production image use different
    signing keys (*TODO*), that module and its configuration cannot be
    copied from a development image to a production image.

Most groups are used to control access to certain resources like
files, devices or privileged operations in system daemons. Device node
ownerships are set using udev rules, similar to how ``audio`` and
``video`` are handled in traditional Linux desktop systems.

Here is a list of existing groups and the corresponding resources:

============ ===============================================================================================
Unix Group   Resource
============ ===============================================================================================
*TODO*: adm  operations typically reserved for root, like rebooting and starting/stopping systemd services
audio        audio devices
video        video devices
rfkill       ``/dev/rfkill`` (*TODO* - patch pending in pohly/ostro/passwd)
============ ===============================================================================================


Process Handling
================

Directly after booting, systemd as PID 1 is the only running
process. Nothing potentially started in the initramfs survives.

All processes are started by systemd, including
applications. systemd’s interfaces (``systemctl`` and the `D-Bus API
of systemd`_) are the currently supported interfaces for listing and
controlling processes.

.. _`D-Bus API of systemd`: http://www.freedesktop.org/wiki/Software/systemd/dbus/


Applications
============

At the moment, applications are only supported when integrated already
into the image (“pre-installed applications”) that gets installed on a
device. Such applications can use the normal mechanisms for creating
the user they run under, install files in the normal root file system,
cause additional system packages they depend on to be added to the
image, etc.

What distinguishes applications from regular system services is that
they provide a manifest file. That manifest file is translated by the
application framework in Ostro OS into a systemd service file
(``/run/systemd/system/app-$ID.service``). The long-term goal is to limit
where applications can install files and rely exclusively on the
application manifest file.

The generated systemd service file contains settings that are used to
isolate the application from other applications. In a system that runs
with only basic Unix DAC, every application is run as a different user
and the user can belong to different Unix groups. These groups specify
the access the application will have to different system resources. As
applications run as different Unix users, ptrace-based attacks are
prevented.

Application manifest security content (*TODO*):

 * firewall configuration (ports that need to be open etc.)
 * user for running the application
 * groups that the user should belong to for access to system services
 * sensor provisioning information
 * container information
 * which namespaces should be isolated (systemd-nspawn: FS or everything)
 * which parts from system rootfs should be bind-mounted
 * Support for systemd’s security features, such as network isolation, apparently read-only directories, etc.

Some applications can request to be run in containers. These
applications bring with themselves a (complete or partial) root
filesystem and systemd executes that container using a suitable
container mechanism, probably systemd-nspawn.

Since applications are run with different user accounts but MAC is
optional, applications can arrange to share data between themselves in
some cases when they are running outside of containers, inside the
same container, or when the containers do not isolate IPC
namespaces. The applications can, for instance, use abstract Unix
domain sockets, loopback network interface, or System V message queues
for connecting to each other. Note that this behavior is not as such
encouraged or documented by Ostro -- it’s just not explicitly
disallowed. If the system integrator wants to prevent this behavior,
using MAC or containers for application isolation is recommended.


System Updates
==============

Ostro OS binaries are delivered as bundles, as in Clear(TM) OS.
Bundles are a bit like traditional packages, but can overlap with
other bundles and come with less meta data. Instead of thousands of
packages, the entire distro consist of about 10 to 20 (*TODO*:
double-check this guess) bundles. There is a core bundle with all the
essential files required to boot the system. Several optional bundles
contain individual runtimes and applications that were built together
with the OS.

Installing bundles must not change files contained in other bundles,
i.e. if a file is contained in more than one bundle, it must have
exactly the same content and attributes in all those bundles.. So
conceptually, one can imagine the bundle creation as installing all
components of the OS in an image, configuring the image and then
splitting up the installed files and their attributes as found in that
image (for example, the signed security.ima xattr) into different
bundles according to some policy (core OS bundle, application bundles
where each bundle contains the application and all non-core files it
depends on).

When compiling a new revision of the OS, new bundles and binary deltas
against older revisions of the bundles are calculated and published on
a download server. The Clear OS swupd tool is then responsible for
downloading the deltas and applying them to the local copy of the
bundles.

(*TODO*): write more about signature handling, how swupd is run, etc.


Core OS Hardening
=================

*TODO*: describe compiler flags

*TODO*: noexec tmpfs mounts

*TODO*: list daemons running as non-root and how that was achieved (ambient capabilities, rfkill group for connman)

*TODO*: instructions how to deal with services needing to talk with each other, D-Bus policies etc.

*TODO*: Recommended Systemd options for services.

*TODO*: Security processes followed by Ostro OS? Information about found vulnerabilities etc.


Network Security
================

Ostro has a firewall that out-of-the-box protects the system services
using both IPv4 and IPv6. The applications and services need to open
holes into the firewall if they require to be accessible from the
network, that is to offer services to the network. If the Ostro device
is meant to be an Internet gateway or otherwise have a complex network
setup, the system integrator has to change the initial firewall
ruleset. *TODO*: finalize the iptables/nftables transition question

*TODO*: sensor security

*TODO*: system provisioning and first boot security


Production and Development Images
=================================

By default, building an image results in something that is locked-down
and secure. This is how real products should be built. Unless some
kind of application gets installed during image creation, one cannot
do much with the running image (no user interface, no way to log into
the system).

During development, a more open image is more useful. The Ostro project
contains a ``ostro-os-development.inc`` file that can be included
in a build configuration's ``local.conf`` to produce "development"
images.

*IMPORTANT*: such development images are intentionally not built to be
perfectly secure! Do not use them in products built for end-customers and
use them only in secure environments.


The Ostro Project provides two different pre-compiled images,
``ostro-os`` and ``ostro-os-dev``. Despite the name, currently *both*
are compiled as development images. The only difference is that
``ostro-os-dev`` already includes development (gcc) and debugging
tools (strace, valgrind, etc.).

The following table summarizes the differences between the default
configuration for production images and images built with
``ostro-os-development.inc``:

============================= ================================ ==========================================
\                             production image                 development image 
============================= ================================ ==========================================
Target audience               End-customers                    Developers
----------------------------- -------------------------------- ------------------------------------------
Usage                         Reference platform for products  Experimenting with Ostro OS, developing
                                                               Ostro OS or applications
Kernel                        Production kernel                Development kernel
IMA signing key               Product-specific, secret         Published together with the Ostro OS
                                                               source code
swupd signature validation    TBD
----------------------------- ---------------------------------------------------------------------------
Kernel debug interfaces       Disabled                         Enabled (*TODO*: document)
Root password                 Not set
----------------------------- ---------------------------------------------------------------------------
Local login as root           Disabled                         Enabled for console (tty) and serial port,
                                                               automatic login
SSH                           Installed, but disabled (*TODO*) Installed and running, but authorized keys
                                                               must be set up before it becomes usable
============================= ================================ ==========================================

For more information about signing, see the `certificate handling`_ document.

.. _`certificate handling`: ../howtos/certificate-handling.rst