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

Support for open networking hardware platforms is currently distributed over
various different projects. Many platforms, especially older ones, are supported
by OpenNetworkLinux, many are supported by SONiC, and some are supported by
DentOS, which is based on OpenNetworkLinux, but targets different devices. The
open hardware support is usually very tightly integrated in the build systems,
which makes sharing and reusing the support hard.

The intention of this project is not to replace or compete with any of the
existing network operating systems, but rather to provide a common and
extensible layer on which they can be built. Its principles and paradigms are
largely aligned with and adapted from the
[OCP OpenNetworkLinux Project](https://github.com/opencomputeproject/OpenNetworkLinux)
(which has been moved to maintenance mode, pending archival).
This proposal specifically seeks an agreement between the developers and
maintainers of the aforementioned network operating system communities, to be
able to share common implementations to support open networking hardware.

## Requirements and Dependencies

## Design

![example layering with distribution layer on top, the BSP layer in the middle,
and base OS agnostic packages at the bottom](images/layering-example.png "layering example")

The project will be split into three layers:

- At the bottom the OS agnostic hardware platform support. NOS's not based on
  Yocto (NOS C) will directly use it.
- In the middle the BSP Yocto layer packaging the platform support. NOS's based
  on Yocto may chose to just use these parts (NOS B).
- At the top the distribution layer, providing a full example NOS, making use of
  the BSP layer. NOS's trying to enhance it will use this layer.

### OS agnostic hardware platform support

To allow sharing the platform support between different operating systems, the
sources should be managed in an independent repository. This should allow easy
building and installation outside of OS distribution build systems, and allow
building and development on-device. Each OS specific build system should also be
able to consume and utilise the shared platform support sources from this
independently managed repository.

Any packaging decisions like if or and how to split up individual platform
support should be up to the OS distribution packaging it.

This allows non-Yocto based Operation Systems to make use of the platform
support.

### Base platform support Yocto layer (BSP layer)

The base platform support Yocto layer will provide the appropriate recipes and
definitions for running on supported platforms. It will contain recipes for the
kernel, the platform support, and appropriate machine definitions to target
platforms based on their architecture.

### Yocto based example NOS (Distribution layer)

To allow quick testing and easy extension for other users, an example
implementation and support layer for Yocto will be provided. It will contain all
basic building blocks for creating an installable image, i.e. a distribution
definition, as well as appropriate recipes for creating ONIE installable images.

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

For the first iteration, the platform support code can be left as is
in their current source repositories.

Future iterations should focus on improving the platform drivers and support
code to follow modern code style and compile with as few warnings as possible,
ideally none.

New platform support should follow modern coding styles from the beginning (e.g.
kernel coding style for modules).

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

ASIC/dataplane support is out of scope for this proposal and is intentionally
left as possible extensions to be implemented by NOS providers (e.g. SONiC or
DENT) on top of the BSP Layer targeted here.

One exception are switchdev-enabled platforms like Mellanox or Marvell Prestera,
where the ASIC support is already provided by the upstream Linux kernel.

### Targeted Kernel and Yocto versions

Target Yocto version is scarthgarp, which is the next LTS release, planned for
April 2024.

The initial Kernel version target could be 5.15 (minimum kernel version of
Yocto) or 6.6 (the latest LTS version of the Linux kernel). The kernel patch
version will be a moving target, and regularly updated (ideally weekly).

Since there will be many out-of-tree drivers for the platform support, required
API updates should be done via
[coccinelle semantic patches](https://coccinelle.gitlabpages.inria.fr/website/).
This should ensure that the amount of work needed for keeping drivers building
and working with newer kernels stays manageable.

The semantic patches will also help vendors quickly updating their drivers when
submitting new platforms based on older releases.

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
  - coccinelle semantic patches for upgrading drivers to support Linux 6.6 APIs
  - ONLP adapter(?)
  - extensible ONIE NOS installer
- M4:
  - get accepted as
    [official yocto layer](https://www.yoctoproject.org/development/yocto-project-compatible-layers/)
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

### DentOS as a layer extending the base platform support layer

The base platform support will be already provided by the base platform layer
(i.e. custom hardware drivers, platform initialization, etc).

Since DentOS uses a customized version of the Prestera driver that contains
several features not included in the upstream driver, the custom Prestera
driver will be provided as an external kernel module package, while the
in-kernel driver will be disabled.

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
described in
[The Linux kernel firwmware guide](https://docs.kernel.org/firmware-guide/acpi/enumeration.html).

A similar approach to the device tree properties can even be done using
[\_DSD Device Properties](https://docs.kernel.org/firmware-guide/acpi/DSD-properties-rules.html).

These ACPI platform device descriptions do not necessarily need to live in the
BIOS itself, and may be loaded from userspace as Linux
[support upgrading ACPI tables on boot](https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html).

Describing all devices and their layout in the firmware (device tree or ACPI)
will simplify the platform initialization by userspace, and avoid any ambiguity
due to e.g. dynamic  I2C bus number assignment.

## Contributors

- Jonas Gorski <jonas.gorski@bisdn.de>
- Jan Klare <jan.klare@bisdn.de>

## Supporters
