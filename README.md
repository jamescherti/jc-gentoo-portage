# jc-gentoo-portage - An opinionated, performance-oriented Gentoo Portage /etc/portage configuration

<img src="https://www.jamescherti.com/wp-content/uploads/2023/04/gentoo-linux-icon-1.png" align="right" />

The [jc-gentoo-portage](https://github.com/jamescherti/jc-gentoo-portage) repository houses an opinionated, performance-oriented Gentoo Portage (`/etc/portage`) configuration.

This repository can be used as an inspiration to build a lean and fast Gentoo operating system.

- The configuration also drops KDE libraries, smartcard dependencies, and physical optical media (CD/DVD) support to prevent hardware probing utilities from linking against media players.
- Core system utilities are heavily optimized. Applications are compiled using `pgo` (Profile-Guided Optimization) and `lto` (Link-Time Optimization). Global flags such as `xs`, `asm`, `orc`, `jit`, `threads`, `kms`, and `native-extensions` ensure applications use hand-optimized assembly routines and multi-core parallelism. This portage uses `jemalloc` to reduce memory fragmentation, for a specific list of packages. Furthermore, specific linker flags (`-Wl,--as-needed`, `-Wl,-z,pack-relative-relocs`) shrink binaries, and `-fno-semantic-interposition` is used to accelerate the Python interpreter.
- Network chatter is bounded. The configuration disables upstream telemetry, background analytics reporting, geolocation (`-geoclue`), cloud provider integrations, and zero-configuration local service scanning like Avahi. It also prevents NetworkManager from leaking IP addresses through periodic background HTTP connectivity checks.
- The multimedia stack is standardized on PipeWire, actively disabling the legacy PulseAudio daemon. For video, the setup relies entirely on FFmpeg's optimized internal decoders, hardware acceleration (`x264`, `x265`, `vpx`, `aom`), and the industry-reference `dav1d` AV1 decoder. Audio processing is centralized using plugins like `lsp-plugins` and `rnnoise` for neural network noise reduction within EasyEffects.
- Unnecessary UI layers are masked to prevent dependency bloat. For example, Qt5 and Qt6 are restricted from being pulled into GTK-based environments via data science libraries or applications like LibreOffice. LibreOffice is compiled purely for GTK3 without MariaDB, Java-based scripting macros, or Bluetooth support. The configuration also trims heavy database indexing hooks and large redundant typography packages for a minimalist graphical environment.
- The setup features a pure GTK and Wayland desktop environment (specifically `gnome-base/gnome-light`). It actively debloats the GNOME shell by pruning heavy file-indexing hooks from `localsearch`, removing Samba and Active Directory integrations (`-samba`, `-ads`, `-acl`) from Nautilus, and stripping out weather daemons. (For users interested in installing an alternative desktop environment such as KDE, remove `/etc/portage/package.use/desktop-gnome`.)

## Installation

1. Ensure the system is set to the compatible 23.0 systemd desktop profile:
   ```
   eselect profile set default/linux/amd64/23.0/desktop/systemd
   ```

2. Create required file and directory:
   ```
   mkdir -p /var/portage-notmpfs
   touch /etc/portage/make-local.conf
   ```

   (Creating /var/portage-notmpfs is used to prevent compilation failures for massive packages that run out of space when building in RAM. Many users configure Portage to compile packages inside a tmpfs, a temporary filesystem in RAM, to accelerate build times, but exceptionally large software like web browsers, gcc, or compilers can easily exceed available memory during the build process. Creating this physical directory on a hard drive or SSD allows configuring Portage to redirect the build location specifically for those giant packages, providing them with enough physical disk space to finish compiling successfully.)

3. Install requirements:
   ```
   emerge -av app-portage/cpuid2cpuflags app-arch/zstd dev-vcs/git
   ```

4. Clone the Repository:

   ```bash
   git clone https://github.com/jamescherti/jc-gentoo-portage /etc/portage
   ```

5. Update CPU flags:
   ```
   echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
   ```

6. Create make.profile:
   ```
   cd /etc/portage
   ln -sf ../../var/db/repos/gentoo/profiles/default/linux/amd64/23.0/desktop/systemd make.profile
   ```

7. Recompile GCC using this Portage configuration, which enables Profile-Guided Optimization (PGO) and Link-Time Optimization (LTO) to maximize compilation throughput:
   ```
   emerge -av sys-devel/gcc
   ```

8. Begin customizing `/etc/portage` to fit your specific requirements.

9. Install packages, such as:
   ```
   emerge -av gnome-base/gnome-light
   ```

## Repository Structure

To effectively customize this configuration, you need to understand its layout:

* [make.conf](https://github.com/jamescherti/jc-gentoo-portage/blob/main/make.conf): The primary configuration file. It contains global compiler flags (`CFLAGS`, `CXXFLAGS`), `MAKEOPTS`, global `USE` flags, and `FEATURES`.
* [package.use/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.use): A directory containing modular files that define USE flags on a per-package basis. Files are categorized logically (e.g., `gnome`, `sound-server`, `optimize`).
* [package.accept_keywords/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.accept_keywords): Allows the installation of specific testing or unstable packages on a stable system.
* [package.mask/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.mask) and [package.unmask/](https://github.com/jamescherti/jc-gentoo-portage/blob/main/package.unmask): Used to block or allow specific package versions.

## Usage and Maintenance

After applying this configuration or making your own modifications, you must instruct Portage to evaluate the dependency tree and apply the changes to your live system.

1. Apply the new USE flags and update the system:
   ```bash
   emerge --ask --verbose --update --deep --newuse @world
   ```

2. Remove orphaned dependencies that are no longer required:
   ```bash
   emerge --ask --depclean
   ```

## Customizations

This setup is highly opinionated. You are expected to review the configurations and adjust them to match your hardware and workflow.

### Adjusting Global Settings (make.conf)

Open `/etc/portage/make.conf` and modify the variables to match your system resources:

* `MAKEOPTS`: Adjust this based on your CPU core count and available RAM. A common rule is `-jN -lN` where `N` is your logical CPU core count.

### /etc/portage/make-local.conf: Separating Local Overrides from Git-Managed Configuration

The `jc-gentoo-portage` repository tracks the primary `make.conf` file via Git. Modifying `make.conf` directly to add hardware specifics will cause merge conflicts whenever `git pull` is executed to update the repository with upstream changes.

To prevent this, the provided `make.conf` is automatically source `/etc/portage/make-local.conf` at the end of its execution. By placing all system-specific overrides (such as `GOAMD64`, `MAKEOPTS`, or `CFLAGS`) inside `make-local.conf`, these settings successfully override the global defaults while keeping the Git working tree clean. This allows upstream updates to be applied without manual conflict resolution.

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

### Managing Per-Package USE Flags (package.use)

The `package.use/` directory is modular. You should read through the files and remove entries for software you do not intend to install. If you need a feature that is disabled globally in `make.conf` (like `nls` or `bluetooth`), do not enable it globally. Instead, enable it only for the specific package that requires it by creating a new entry in `package.use/`.

### Hardware-Specific and User-Specific Examples

Below are examples of how to configure `package.use` for specific hardware setups. You can create new files in `/etc/portage/package.use/` to store these directives.

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

### Gentoo Linux: Unlocking a LUKS Encrypted LVM Root Partition at Boot Time using a Key File stored on an External USB Drive

Gentoo can be configured to use a key file stored on an external USB drive to unlock a LUKS encrypted LVM root partition.

We will explore in this article the general steps involved in configuring Gentoo to use an external USB drive as a key file to unlock a LUKS encrypted LVM root partition.

Read: [Gentoo Linux: Unlocking a LUKS Encrypted LVM Root Partition at Boot Time using a Key File stored on an External USB Drive](https://www.jamescherti.com/gentoo-linux-unlock-lvm-root-partition-at-boot-key-file-external-usb-stick/)

### Forcing English

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

#### Intel Processor + NVIDIA GPU

File: `/etc/portage/package.use/00my-hardware-video`

```
# Tells Portage to only install the microcode files necessary for the host CPU.
sys-firmware/intel-microcode hostonly

# Enables drivers for both the NVIDIA discrete GPU and the Intel integrated
# GPU. This setup allows using the Intel iGPU for power-efficient
# hardware video decoding.
*/* VIDEO_CARDS: -* nvidia intel

# Opts into native NVIDIA hardware video encoding/decoding while disabling the legacy,
# X11-bound VDPAU backend.
*/* nvenc nvdec -vdpau

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
```

#### Intel audio + USB audio

File: `/etc/portage/package.use/00my-hardware-audio`

```
# Intel audio + USB audio
*/* ALSA_CARDS: -* hda-intel usb-audio
```

#### Scanner: Disable all sane backends

File: `/etc/portage/package.use/00my-hardware-scanner`

```
# Disable all sane backends
*/* SANE_BACKENDS: -*
```

### Installing Debian onto a separate partition from an existing distribution, such as Arch Linux or Gentoo, without using the Debian installer

There are various scenarios in which one might need to install a Debian-based system (e.g., Debian, Ubuntu, etc.) from another distribution (e.g., Arch Linux, Gentoo, Debian/Ubuntu distributions, Fedora, etc.). One common reason is when a user wants to set up a Debian-based system alongside an existing distribution. This could be for the purpose of testing software compatibility, development, or simply to have a dual-boot.

Read: [Installing Debian onto a separate partition from an existing distribution, such as Arch Linux or Gentoo, without using the Debian installer](https://www.jamescherti.com/how-to-install-debian-using-arch-linux-or-gentoo/)

## License

The **jc-gentoo-portage** files have been written by James Cherti and is distributed under terms of the MIT license.

Copyright (C) 2022-2026 [James Cherti](https://www.jamescherti.com).

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Links

- [jc-gentoo-portage @GitHub](https://github.com/jamescherti/jc-gentoo-portage)
- [Customizing /etc/portage/make.conf ](https://wiki.gentoo.org/wiki//etc/portage/make.conf)
- [Gentoo Wiki](https://wiki.gentoo.org/wiki/)
- [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page)
- [Article: jc-gentoo-portage: An opinionated, performance-oriented Gentoo Portage /etc/portage configuration](https://www.jamescherti.com/jc-gentoo-portage/)

Other project by the same author:
- [jc-dotfiles @GitHub](https://github.com/jamescherti/jc-dotfiles): A collection of UNIX/Linux configuration files. You can either install them directly or use them as inspiration your own dotfiles.
- [bash-stdops @GitHub](https://github.com/jamescherti/bash-stdops): A collection of Bash helper shell scripts.
- [jc-gnome-settings](https://github.com/jamescherti/jc-gnome-settings): GNOME customizations that can be applied programmatically.
- [jc-firefox-settings @GitHub](https://github.com/jamescherti/jc-firefox-settings): Provides the user.js file, which holds settings to customize the Firefox web browser to enhance the user experience and security.
- [jc-xfce-settings](https://github.com/jamescherti/jc-xfce-settings): GNOME customizations that can be applied programmatically.
- [watch-xfce-xfconf](https://github.com/jamescherti/watch-xfce-xfconf/): A command-line tool that can be used to configure XFCE 4 programmatically using the *xfconf-query* commands displayed when XFCE 4 settings are modified.
