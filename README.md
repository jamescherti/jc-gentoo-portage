# jc-gentoo-portage - An opinionated, performance-oriented Gentoo Portage /etc/portage configuration

<img src="https://www.jamescherti.com/wp-content/uploads/2023/04/gentoo-linux-icon-1.png" align="right" />

The [jc-gentoo-portage](https://github.com/jamescherti/jc-gentoo-portage) repository houses an opinionated, performance-oriented Gentoo Portage (`/etc/portage`) configuration.

This repository can be used as an inspiration to build a lean and fast Gentoo operating system.

Features:

- Maximizes execution performance across the system by globally enabling `xs`, `asm`, `orc`, `jit`, and `threads`, forcing packages to use hand-written assembly routines and JIT compilation loops.
- Disables telemetry and remote RPC calls by globally setting `-telemetry`, `-geoclue`, `-geolocation`, `-cloudproviders`, `-google`, and `-gnome-online-accounts`.
- Optimizes Rust compiler outputs by enforcing modern instruction sets via `target-cpu=native`, `opt-level=3`, `strip=symbols`, `lto=thin`, and `codegen-units=1`.
- Links long-running daemons and high-throughput parsing utilities against `jemalloc` to guarantee flat memory profiles and highly predictable multi-threaded performance during extended uptimes.
- Prunes legacy hardware probing and obsolete I/O interfaces by globally disabling optical media (`-cdrom`, `-dvd`, `-dvdr`, `-css`) and deprecated X11 video rendering (`-xv`).
- Removes documentation bloat by globally disabling `-doc`, `-gtk-doc`, and `-handbook`. This instructs the build system to skip the generation of extraneous HTML manuals and localized help files, reducing compilation times and the final disk footprint.
- Optimizes the linker aggressively using `-Wl,-O2 -Wl,--sort-common -Wl,--as-needed -Wl,-z,pack-relative-relocs` to compress relocation tables, drop unused dependencies, and group global variables by alignment for faster binary loads.
- Standardizes the multimedia stack exclusively on PipeWire while enforcing hardware-accelerated video decoding.
- Restricts the GNOME desktop footprint by masking heavy file indexing (`app-misc/localsearch -pdf -playlist`) and stripping network discovery protocols from file managers (`-samba`, `-ads`, `-acl`).
- Isolates high-load compilation processes to background resources using `PORTAGE_SCHEDULING_POLICY="idle"`, `PORTAGE_IONICE_COMMAND="ionice -c 3 \${PID}"`, and a custom `notmpfs.conf` to prevent Out of Memory errors on massive packages.
- Compiles binary packages and man pages using maximum compression via `BINPKG_COMPRESS_FLAGS_ZSTD="-19 -T0"`, utilizing all CPU threads.

## Installation

1. Ensure the system is set to the compatible 23.0 systemd desktop profile:
   ```sh
   eselect profile set default/linux/amd64/23.0/desktop/systemd
   ```

   (This ensures that the base system dependencies, compiler configurations, and default USE flags are aligned for a modern systemd-based graphical desktop environment.)

2. Install requirements:
   ```sh
   emerge -av app-portage/cpuid2cpuflags app-arch/zstd dev-vcs/git
   ```
   (`cpuid2cpuflags` is required to detect your host processor's hardware capabilities. `zstd` is installed to provide high-speed compression for Portage build operations and binary packages, and `git` is required to clone this repository.)

3. Clone the Repository:
   ```sh
   git clone https://github.com/jamescherti/jc-gentoo-portage /etc/portage
   ```

4. Run:
   ```sh
   /etc/portage/scripts/init-portage
   ```

   (This script creates `/var/portage-notmpfs` to prevent compilation failures for massive packages that run out of space when building in RAM. It also generates `/etc/portage/make-local.conf`, which establishes a safe, untracked location for machine-specific overrides. By default, it populates this file with a `MAKEOPTS` setting configured to use half of your system's processors. Finally, the script uses the `cpuid2cpuflags` command to dynamically query your hardware for supported instruction sets, such as AVX2 or SSE4, and writes them to `/etc/portage/package.use/00my-cpu-flags`. This ensures that all subsequently compiled software is fully optimized for your specific processor.)

5. Create make.profile:
   ```sh
   cd /etc/portage
   ln -sf ../../var/db/repos/gentoo/profiles/default/linux/amd64/23.0/desktop/systemd make.profile
   ```

   (Portage relies on the `make.profile` symlink to determine which system profile is currently active. Creating this link manually ensures Portage resolves the dependency graph and default variables accurately from the downloaded Gentoo repository tree.)

6. Recompile GCC using this Portage configuration, which enables Profile-Guided Optimization (PGO) and Link-Time Optimization (LTO) to maximize compilation throughput:
   ```sh
   emerge -av sys-devel/gcc
   ```

7. Begin customizing `/etc/portage` to fit your specific requirements and install packages using `emerge`.

## Repository Structure

To effectively customize this configuration, you need to understand its layout:

* [make.conf](https://github.com/jamescherti/jc-gentoo-portage/blob/main/make.conf): The primary configuration file. It contains global compiler flags (`CFLAGS`, `CXXFLAGS`), `MAKEOPTS`, global `USE` flags, and `FEATURES`.
* [package.use/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.use): A directory containing modular files that define USE flags on a per-package basis. Files are categorized logically (e.g., `gnome`, `sound-server`, `optimize`).
* [package.accept_keywords/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.accept_keywords): Allows the installation of specific testing or unstable packages on a stable system.
* [package.mask/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.mask) and [package.unmask/](https://github.com/jamescherti/jc-gentoo-portage/blob/main/package.unmask): Used to block or allow specific package versions.

## Customizing USE Flags (package.use)

The `package.use/` directory is modular. You should read through the files and remove entries for software you do not intend to install. If you need a feature that is disabled globally in `make.conf` (like `nls` or `bluetooth`), do not enable it globally. Instead, enable it only for the specific package that requires it by creating a new entry in `package.use/`.

### Force English

For users who don't need localization, setting `-nls`, `-cjk`, and `-ibus` globally **forces interfaces to English**, which skips the compilation of thousands of unneeded localization files:

File: `/etc/portage/package.use/00my-just-english`

```
# Global exclusion of the Intelligent Input Bus (IBUS).
#
# Justification:
# - Performance: Prevents unnecessary background daemon processes.
# - Footprint: Eliminates complex dependencies and reduces system bloat.
# - Security: Minimizes the attack surface by removing unused system services.
#
# Note:
# This assumes that no specialized Input Method Editors (IME) are required for
# non-Latin script input. If localized language support is needed in the future,
# this flag must be re-evaluated.
*/* -ibus

# The exclusion of nls (Native Language Support) is a deliberate choice to
# simplify the dependency graph and simplify the package installation process.
# By setting USE="-nls", you instruct Portage to ignore internationalization
# libraries and omit the compilation of localized message files, ensuring that
# all software interfaces default to English. This configuration is particularly
# beneficial on Gentoo because it prevents unnecessary interactions with the
# gettext utility and significantly reduces the total number of files installed
# across your system. Consequently, your updates will finish faster, and you
# will regain valuable disk space that would otherwise be occupied by dozens of
# translation files you do not need, resulting in a cleaner and more efficient
# OS environment.
#
# cjk: cjk (Chinese, Japanese, Korean), safe to disable globally.
*/* -nls -cjk
```

### Disable smartcard

File: `/etc/portage/package.use/00my-no-smartcard`

```
# Disabling smartcard support prevents packages like app-crypt/gnupg and
# net-misc/openssh from linking against smartcard-reading libraries.
#
# Warning: Do not apply this mask if you rely on a physical hardware token (such
# as a YubiKey or Nitrokey) for SSH authentication, GPG commit signing, or LUKS
# disk decryption.
*/* -smartcard
```

### NVIDIA GPU

File: `/etc/portage/package.use/00my-hw-gpu-nvidia`

```
*/* VIDEO_CARDS: -* nvidia

# Opts into native NVIDIA hardware video encoding/decoding
*/* nvenc nvdec vdpau

# Enables VA-API support across applications.
*/* vaapi
```

### Intel CPU


File: `/etc/portage/package.use/00my-hw-cpu-intel`

```
# Tells Portage to only install the microcode files necessary for the host CPU.
sys-firmware/intel-microcode hostonly

# Enables VA-API support across applications.
*/* vaapi

# Enable Intel Quick Sync Video
# Ensure every media application on the system compiles with Intel Quick Sync
# Video support if the package supports it. For a dual-GPU setup containing an
# Intel iGPU and an NVIDIA discrete card, the primary benefit is systematic
# workload isolation. It allows offloading everyday video decoding and
# background encoding tasks across all applications (such as media players,
# transcoders, and broadcasting tools) directly to the Intel processor. This
# strategy keeps the NVIDIA card completely free from media processing overhead,
# reserving its full hardware capacity for demanding tasks like 3D rendering or
# compute workloads, while avoiding the need to configure flags on a per-package
# basis.
*/* qsv
```

### Intel Integrated graphics

File: `/etc/portage/package.use/00my-hw-intel-integrated-graphics`

```
*/* VIDEO_CARDS: -* intel

# Enables VA-API support across applications. This allows the Intel integrated GPU
# to handle hardware acceleration paths natively via Intel Quick Sync.
*/* vaapi

# Enable Intel Quick Sync Video
# Ensure every media application on the system compiles with Intel Quick Sync
# Video support if the package supports it. For a dual-GPU setup containing an
# Intel iGPU and an NVIDIA discrete card, the primary benefit is systematic
# workload isolation. It allows offloading everyday video decoding and
# background encoding tasks across all applications (such as media players,
# transcoders, and broadcasting tools) directly to the Intel processor. This
# strategy keeps the NVIDIA card completely free from media processing overhead,
# reserving its full hardware capacity for demanding tasks like 3D rendering or
# compute workloads, while avoiding the need to configure flags on a per-package
# basis.
*/* qsv

# Disable VDPAU backend
*/* -vdpau
```

### Intel audio + USB audio

File: `/etc/portage/package.use/00my-hw-audio-intel`

```
# Intel audio + USB audio
*/* ALSA_CARDS: -* hda-intel usb-audio
```

### Scanner: Disabling all sane backends

For users who use scanners requiring proprietary drivers, such as those from Brother, it is recommended to disable all SANE backends.

File: `/etc/portage/package.use/00my-hw-scanner`

```
# Disable all sane backends
*/* SANE_BACKENDS: -*
```

### systemd-boot and dracut for sys-kernel/gentoo-kernel or sys-kernel/gentoo-kernel-bin users

The systemd-boot makes `installkernel` manage `bootctl` entries dynamically using the Boot Loader Specification (BLS). This creates individual menu options for each installed kernel version, providing an automatic fallback if a new kernel fails to boot.

First, edit `/etc/kernel/install.conf` and add the following lines to define the kernel installation layout:

```
layout=bls
initrd_generator=dracut
uki_generator=none
```

Next, edit `/etc/portage/package.use/00my-systemd-boot` to apply the required USE flags. This enables dracut and systemd-boot for installkernel and sets the dist-kernel flag globally. Add the following lines:
```sh
sys-kernel/installkernel dracut systemd-boot
sys-apps/systemd boot

# When dist-kernel is set, Portage will automatically trigger Dracut to
# build the initramfs and automatically rebuild out-of-tree modules
# (like nvidia-drivers) whenever a new sys-kernel/gentoo-kernel is installed.
*/* dist-kernel
```

### Dracut + NVIDIA: Force NVIDIA Driver Inclusion in the Early Boot Sequence

By default, Dracut may omit out-of-tree drivers during initramfs generation. Force the inclusion of the essential NVIDIA kernel components, including the core driver, modesetting controls, unified virtual memory, and direct rendering manager layers, directly into the early boot image. This step also ensures the open-source Nouveau driver is explicitly omitted to prevent conflicts.

First, edit `/etc/dracut.conf.d/nvidia.conf` and add the following lines to bind the required drivers:

```
add_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "
omit_drivers+=" nouveau "
```

Next, regenerate all of your initramfs images to apply the configuration. Run your bootloader or kernel management tool to trigger the rebuilding process, or execute your system's standard initramfs generation utility with the force flag to ensure the changes are baked into the early boot sequence.

## Customizing /etc/portage/make-local.conf

The `jc-gentoo-portage` repository tracks the primary `make.conf` file via Git. Modifying `make.conf` directly to add hardware specifics will cause merge conflicts whenever `git pull` is executed to update the repository with upstream changes.

To prevent this, the provided `make.conf` automatically sources `/etc/portage/make-local.conf` at the end of its execution. By placing all system-specific overrides (such as `GOAMD64`, `MAKEOPTS`, or `CFLAGS`) inside `make-local.conf`, these settings successfully override the global defaults while keeping the Git working tree clean. This allows upstream updates to be applied without manual conflict resolution.

Open `/etc/portage/make-local.conf` and modify the variables to match your system resources. For example, modify `MAKEOPTS` based on your CPU core count and available RAM. A common rule is `-jN -lN` where `N` is your logical CPU core count.

### Go Compiler Optimizations (GOAMD64)

The `GOAMD64` environment variable specifies the microarchitecture level of the `amd64` (x86-64) architecture that the Go compiler targets. Setting `GOAMD64="v3"` inside `/etc/portage/make-local.conf` forces the Go compiler to generate machine code leveraging newer CPU instructions, such as AVX2, BMI1, BMI2, F16C, FMA, LZCNT, MOVBE, and OSXSAVE.

While C and C++ compiler optimizations are managed via `CFLAGS` and `CXXFLAGS` (e.g., `-march=x86-64-v3`), the Go compiler ignores these flags. Many modern utilities packaged in Gentoo are written in Go (including Docker, Kubernetes, and Terraform). When Portage builds these packages from source, the ebuilds read Go-specific environment variables.

To determine the correct target level for a specific hardware setup, processor capabilities must be inspected. CPU flags can be checked by running `grep -m 1 '^flags' /proc/cpuinfo` and matching them against the following levels:

* `v1`: The baseline x86-64 architecture. This is appropriate for distributing compiled binaries to unknown hardware or for processors older than 2008.
* `v2`: Requires `popcnt` and `sse4_2`. This is intended for older processors released around 2008 to 2013, such as Intel Nehalem or AMD Jaguar.
* `v3`: Requires `avx2`. This is intended for modern processors released after 2014, such as Intel Haswell or AMD Excavator.
* `v4`: Requires `avx512f`. This is intended for the latest enterprise or high-end desktop processors, such as Intel Skylake-X or AMD Zen 4.

Once the highest supported level is identified, the variable can be appended to the local configuration:

```bash
echo 'GOAMD64="v3"' >> /etc/portage/make-local.conf
```

Explicitly declaring `GOAMD64="v3"` in `/etc/portage/make-local.conf` ensures Portage applies hardware-specific optimizations to all compiled Go binaries. If this variable is omitted, the Go compiler defaults to `v1`, generating universally compatible but unoptimized code. A higher tier should only be set if the target processor explicitly supports the required instruction sets.

### FEATURES: buildpkg

If you manage multiple identical or similar Gentoo machines, use `FEATURES="buildpkg"` on your fastest machine to compile binaries once, then distribute them to your other machines using `emerge --usepkg`.

```sh
echo 'FEATURES="$FEATURES buildpkg"' >> /etc/portage/make-local.conf
```

## Other customizations

### Install the latest testing kernel

Unmask the testing versions (`~amd64`) of the Gentoo distribution kernels. This allows Portage to look past the stable tree and fetch the latest upstream kernel updates.

First, edit `/etc/portage/package.accept_keywords/00my-latest-gentoo-kernel` and add the following lines to accept the testing keywords:

```text
sys-kernel/gentoo-kernel-bin ~amd64
sys-kernel/gentoo-kernel ~amd64
virtual/dist-kernel ~amd64
```

Next, run your standard package upgrade command to pull in the new kernel version, and use your system's kernel selection utility to set the newly installed kernel as the default.

Then run:
```
emerge --ask --with-bdeps=y --update --deep --changed-use @world
```

Finally, use `eselect kernel list` and `eselect kernel set <version>` to select the newly installed kernel.

### Temporary File Systems (tmpfs) Optimization

Gentoo compiles the majority of software from source code via Portage, a process that generates a significant volume of intermediate build artifacts. By default, Portage writes these temporary files to the physical storage device at `/var/tmp/portage`.

Shifting high-volume compilation I/O operations into memory substantially reduces solid-state drive wear while exploiting the superior read and write speeds of RAM to eliminate storage bottlenecks.

To extend hardware longevity and minimize extraneous disk writes across the entire operating environment, offload the standard system temporary directories to RAM. Append these corresponding entries to `/etc/fstab`:

```
tmpfs    /tmp        tmpfs    rw,nodev,nosuid,size=8G    0 0
tmpfs    /var/tmp    tmpfs    rw,nodev,nosuid,size=8G    0 0
```

Configuring the following entry in `/etc/fstab` mounts the directory as a `tmpfs` allocation. The size parameter establishes a hard limit on memory consumption to prevent resource exhaustion during the build process.

```
tmpfs    /var/tmp/portage     tmpfs     size=16G,uid=portage,gid=portage,mode=775,nosuid,noatime,nodev    0  0
```

### Storage Optimization & Encryption (LUKS / SSD)

For systems utilizing an encrypted root filesystem on solid-state storage (SSD/NVMe), specialized kernel parameters are required to maintain storage performance.

By default, dm-crypt/LUKS containers block TRIM requests for security reasons, which can degrade SSD performance and longevity over time. If `sys-kernel/genkernel` is used to manage the initramfs, append the following parameter to the bootloader kernel command line (e.g., in `grub.cfg` or `refind.conf`):

```
root_trim=yes
```

This parameter instructs the initramfs script to pass the `--allow-discards` option to `cryptsetup` during the initial phase of the boot sequence. This allows the root filesystem to successfully pass discard/TRIM commands through the encryption layer down to the underlying physical controller.

This parameter is unique to `genkernel`. If the initramfs is built using `sys-kernel/dracut`, this flag will be ignored; standard Dracut configuration or the `rd.luks.options=discard` kernel parameter must be used instead.

Enabling TRIM on an encrypted device exposes disk usage patterns and filesystem layout to an attacker with physical access to the drive. For standard operational profiles, the performance and hardware longevity benefits outweigh this minor metadata leakage.

### Using the Ninja Build System

Ninja is a efficient build system that evaluates dependencies rapidly and execute multiple build tasks concurrently. Replacing standard `make` with Ninja provides performance improvements during the compilation phase, particularly for large C and C++ projects.

Benefits of using Ninja:

* Faster dependency resolution and startup time.
* Improved management of parallel build processes.
* Noticeably reduced compilation times for heavy packages.

To configure Portage to use Ninja globally for CMake-based ebuilds, you must install the package and declare it as the default generator.

Install Ninja:

```sh
emerge -av dev-build/ninja
```

Append the generator variable to your local configuration:

```sh
echo 'CMAKE_MAKEFILE_GENERATOR="ninja"' >> /etc/portage/make-local.conf
```

This configuration ensures that any package using CMake will use Ninja instead of traditional `make` to process the build.

## Maintenance

After applying this configuration or making your own modifications, you must instruct Portage to evaluate the dependency tree and apply the changes to your live system.

1. Apply the new USE flags and update the system:
   ```bash
   emerge --ask --verbose --update --deep --newuse @world
   ```

2. Remove orphaned dependencies that are no longer required:
   ```bash
   emerge --ask --depclean
   ```

## License

The **jc-gentoo-portage** files were written by James Cherti and are distributed under terms of the MIT license.

Copyright (C) 2022-2026 [James Cherti](https://www.jamescherti.com).

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Links

- [jc-gentoo-portage @GitHub](https://github.com/jamescherti/jc-gentoo-portage)
- [Customizing /etc/portage/make.conf ](https://wiki.gentoo.org/wiki//etc/portage/make.conf)
- [Gentoo Linux x86 Handbook: Installing Gentoo ](https://wiki.gentoo.org/wiki/Handbook:X86/Full/Installation)
- [Gentoo AMD64 Handbook ](https://wiki.gentoo.org/wiki/Handbook:AMD64)
- [Gentoo Wiki](https://wiki.gentoo.org/wiki/)
- [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page)
- [Article: jc-gentoo-portage: An opinionated, performance-oriented Gentoo Portage /etc/portage configuration](https://www.jamescherti.com/jc-gentoo-portage/)

Other projects by the same author:
- [jc-dotfiles @GitHub](https://github.com/jamescherti/jc-dotfiles): A collection of UNIX/Linux configuration files. You can either install them directly or use them as inspiration your own dotfiles.
- [bash-stdops @GitHub](https://github.com/jamescherti/bash-stdops): A collection of Bash helper shell scripts.
- [jc-gnome-settings](https://github.com/jamescherti/jc-gnome-settings): GNOME customizations that can be applied programmatically.
- [jc-firefox-settings @GitHub](https://github.com/jamescherti/jc-firefox-settings): Provides the user.js file, which holds settings to customize the Firefox web browser to enhance the user experience and security.
- [jc-xfce-settings](https://github.com/jamescherti/jc-xfce-settings): GNOME customizations that can be applied programmatically.
- [watch-xfce-xfconf](https://github.com/jamescherti/watch-xfce-xfconf/): A command-line tool that can be used to configure XFCE 4 programmatically using the *xfconf-query* commands displayed when XFCE 4 settings are modified.
