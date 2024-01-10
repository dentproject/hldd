# Unified Open Networking Hardware Support

[[_TOC_]]

## Scope

The current open network operating system (NOS) market is fragmented and has
multiple competing projects with different focus, architecture and level of
maturity. Despite those differences, all of the currently available,
open network operating systems are utilizing a mainline Linux kernel and
need to build and maintain platform-specific abstractions (e.g., out-of-tree
drivers) for peripheral switch hardware.

This proposal aims to reduce the fragmentation by providing a common base for
networking hardware support, without addressing the question of how to abstract
and control the switch forwarding plane itself.

The intention of this project is not to replace or compete with any of the
existing network operating systems, but rather to provide a common and
extensible layer on which they can be built. Its principles and paradigms are
largely aligned with and adapted from the
[OCP OpenNetworkLinux Project](https://github.com/opencomputeproject/OpenNetworkLinux) (which has been moved to maintenance mode, pending archival).
This proposal specifically seeks an agreement between the developers and
maintainers of the aforementioned network operating system communities, to be
able to share common implementations to support open networking hardware.

Support for open networking hardware platforms is currently distributed over
various different projects. Many platforms, especially older ones, are supported
by OpenNetworkLinux, many are supported by SONiC, and some are supported by
DentOS, which is based on OpenNetworkLinux, but targets different devices. The
open hardware support is usually very tightly integrated in the build systems,
which makes sharing and reusing the support hard.

## Requirements and Dependencies

## Functional Description

### Base Open Networking Hardware Support

The base networking hardware support combines hardware support from multiple
sources, and therefore will need to support multiple kinds of APIs.

Possible APIs are:

- SONiC PDDF, due to the large amount of available platforms
  https://github.com/sonic-net/SONiC/blob/master/doc/platform/brcm_pdk_pddf.md
- S3IP as described in https://github.com/sonic-net/SONiC/pull/1068
- ONLP, as used by ONL and DentOS

The base networking hardware support is organized in multiple repositories:

- kernel: Linux fork with required base changes (backports of important
submissions, etc)
- hardware-platform-support: drivers, libraries and userspace code for
  initializing hardware platforms (do we need more?)
- TBD: organization of platform support
- TBD: platform initialization

In the first iteration, the platform support is fine in it the state as it is
in their source repository.

Future iterations should focus on improving the platform drivers and support
code to follow modern code style and compile with as few warnings as possible,
ideally none.

New platform support should follow modern coding styles from the beginning (e.g.
kernel coding style for modules).

## Design

![example layering with operating layers on top like meta-dent, the proposed
hardware support layer in the middle, and base Yocto and Open Embedded layers
at the bottom](images/layering-example.png "layering example")

### OS-agnostic hardware platform support

To allow sharing the platform support between different operating systems, the
support is contained in an independent repository. It should allow easy building
and installation outside of OS distribution build systems, and allow building
and development on-device.

Any packaging decisions like if or and how to split up individual platform
support should be up to the OS distribution packaging it.

### Yocto based example and testing OS

To allow quick testing and easy extension for other users, an example
implementation and support layer for Yocto will be provided. It will contain all
basic building blocks for creating an installable image:

- recipes for the basic kernel mentioned above and for platform support
- class and image recipe for an ONIE installable image
- machine definitions as generic OS-based

### Kernel recipe and configuration handling

The kernel configuration will be done via
https://docs.yoctoproject.org/kernel-dev/advanced.html to allow large
flexibility in how other projects using the layer can customize the kernel.

TBD: should we have patches or branches with backport commits? Maybe both?

- patches make required changes due to upstream changes more obvious (provide a
“conflict diff”) and easier to review
- branches are closer to the upstream approach, but may be harder to track
  changes needed due to upstream changes

### Platform support recipes

To allow easier extension by vendors or other users, the platform support will
be split into multiple subpackages:

- a base package for the userspace utilities and API libraries
- one or multiple support packages for the platforms

### ASIC/dataplane support

ASIC/dataplane support is out of scope for now, but may be provided by a future
project or extension.

The only exception are switchdev-enabled platforms like Mellanox or Marvell
Prestera, where the ASIC support is already provided by the upstream Linux kernel.

### Targeted Kernel and Yocto versions

Target Yocto version is scarthgarp, which is the next LTS release, planned for
April 2024.

The initial Kernel version target could be 5.15 (minimum kernel version of Yocto)
or 6.6 (the latest LTS version of the Linux kernel). The kernel patch version will
be a moving target, and regularly updated (ideally weekly).

Since there will be many out-of-tree drivers for the platform support, required
API updates should be done via [coccinelle semantic patches](https://coccinelle.gitlabpages.inria.fr/website/). This should ensure that the
amount of work needed for keeping drivers building and working with newer
kernels stays manageable.

The semantic patches will also help vendors quickly updating their drivers when
submitting new platforms based on older releases.

### DentOS as a layer extending the base platform support layer

The base platform support will be already provided by the base platform layer
(i.e. custom hardware drivers, platform initialization, etc).

Since DentOS uses a customized version of the prestera driver that contains
several features not included in the upstream driver, the custom prestera
driver will be provided as an external kernel module package, while the
in-kernel driver will be disabled.

## Milestones

- M1:
  - initial platform support repository with hand-selected/-crafted platform
    support (>= 1 platform)
  - initial machine definitions
  - initial layer with kernel with defconfig, and package for platform support
  - initial kernel 5.15?
  - ramdisk image bootable on selected platform(s)
- M2:
  - refined kernel configuration via kernel-meta
  - import of various platforms via scripts from SONiC
  - initial ONIE NOS installer
- M3:
  - import missing platforms from ONL (convert them to PDDF?)
  - import missing platforms from DentOS (convert them to PDDF?)
  - kernel 6.6 support(?)
  - coccinelle semantic patches for upgrading drivers to supprot linux 6.6 APIs
  - ONLP adapter(?)
  - extensible ONIE NOS installer
- M4: get accepted as official yocto layer
  - are we fine with LTS kernels, or do we need to support non-LTS ones?
  - figure out long-term maintenance strategy

## Possible Future Extensions

### ASIC / dataplane support

While providing ASIC / dataplane support is out of scope for this project,
a follow up project could provide appropriate SAI or similar packaging.

### Common userspace API / utilities

Currently there are several competing ways of providing access to sensor data
and other values. To avoid forcing users to know which one is used for their
current platform, it would be nice to have a generic/common utilities for
querying the data, hiding the implementation details.

### Conformance testing or DIAG utilities

To allow verifying that various drivers and libraries work, it would be good
to have some frameworks or testing utilities for OEMs to ensure that everything
works from the API side.

### Improved driver/platform architecture

Currently the need for platform initialization code in user space is due to
insufficient hardware description from the platform.

To reduce the need for custom platform initialization code in user space, and in
best case even remove it, we should work for a more complete description of the
platform from the platform itself.

For device-tree enabled platforms, the device tree should describe the hardware
layout fully. Any configuration values set by the platform init script should be
configurable via appropriate node attributes.

For ACPI platforms, devices should be described in their appropriate tables like
described in [The Linux kernel firwmware guide](https://docs.kernel.org/firmware-guide/acpi/enumeration.html).

A similar approach to the device tree properties can even be done using
[\_DSD Device Properties](https://docs.kernel.org/firmware-guide/acpi/DSD-properties-rules.html).

These ACPI platform device descriptions do not necessarily need to live in the
BIOS itself, and may be loaded from userspace as Linux [support upgrading ACPI
tables on boot](https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html).

Describing all devices and their layout in the firmware (device tree or ACPI)
will simplify the platform initialization by userspace, and avoid any ambiguity
due to e.g. dynamic  I2C bus number assignment.

## Contributors

- Jonas Gorski <jonas.gorski@bisdn.de>
- Jan Klare <jan.klare@bisdn.de>

## Supporters
