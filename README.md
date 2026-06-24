# jc-gentoo-portage - A highly opinionated, performance-oriented Gentoo Portage /etc/portage desktop configuration

<img src="https://www.jamescherti.com/wp-content/uploads/2023/04/gentoo-linux-icon-1.png" align="right" />

The [jc-gentoo-portage](https://github.com/jamescherti/jc-gentoo-portage) repository houses a highly opinionated, performance-oriented Gentoo Portage (`/etc/portage`) desktop configuration. It can be used to build a lean and fast operating system by explicitly stripping away legacy dependencies, unneeded background daemons, and redundant toolkit libraries.

These configuration files serve as a strict baseline to maximize hardware utilization, minimize compile times, and keep the system footprint as small as possible.

**Key Benefits:**

* Pure GTK desktop environment (specifically `gnome-base/gnome-light`). It strictly prevents Qt dependencies from bleeding into the system via applications like LibreOffice or Matplotlib. In addition to that, it actively debloats the GNOME shell by pruning heavy file-indexing hooks from `localsearch`, removing Samba/Active Directory integrations from Nautilus, and stripping out weather daemons, cloud providers, and background sensor calibrations.
* The system dependency graph is significantly reduced. Setting `-nls` globally forces interfaces to English, which skips the compilation of thousands of unneeded localization files and reduces the time required for system updates. The configuration also drops KDE libraries, optical media decryption routines, and obsolete X11 hardware overlay paths.
* Core system utilities, including GCC, Bash, and Python, are compiled using `pgo` (Profile-Guided Optimization) and `lto` (Link-Time Optimization). Global flags such as `xs`, `asm`, `orc`, `jit`, and `threads` ensure applications use hand-optimized assembly routines and multi-core parallelism to execute code as close to the bare metal as possible.
* Network chatter is bounded. The configuration explicitly disables upstream telemetry, background analytics reporting, and zero-configuration local service scanning like Avahi. It also prevents NetworkManager from leaking your IP address through periodic background HTTP connectivity checks.
* The audio system is firmly standardized on PipeWire, actively disabling the legacy PulseAudio daemon. For video, the setup relies entirely on FFmpeg's optimized internal decoders and the industry-reference `dav1d` AV1 decoder, which prevents Portage from pulling in redundant, legacy external codecs.
* Unnecessary UI layers are explicitly masked to prevent dependency bloat. For example, Qt6 is restricted from being pulled into GTK-based environments via Python data science libraries. The configuration also trims LibreOffice extensions, heavy database indexing hooks, and large redundant typography packages for a minimalist graphical environment.

## Prerequisites

Before installing, ensure your system is set to the compatible 23.0 systemd desktop profile:

```
eselect profile set default/linux/amd64/23.0/desktop/systemd
```

Then, create this required directory:

```
mkdir -p /var/portage-notmpfs
```

(Creating /var/portage-notmpfs is used to prevent compilation failures for massive packages that run out of space when building in RAM. Many users configure Portage to compile packages inside a tmpfs, a temporary filesystem in RAM, to accelerate build times, but exceptionally large software like web browsers, gcc, or compilers can easily exceed available memory during the build process. Creating this physical directory on a hard drive or SSD allows you to configure Portage to redirect the build location specifically for those giant packages, providing them with enough physical disk space to finish compiling successfully.)

## Installation

1. Install requirements:
   ```
   emerge -av app-portage/cpuid2cpuflags
   ```

2. Clone the Repository:

   ```bash
   git clone https://github.com/jamescherti/jc-gentoo-portage /etc/portage
   ```

3. Update CPU flags:
   ```
   echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
   ```

4. Create make.profile:
   ```
   cd /etc/portage
   ln -sf ../../var/db/repos/gentoo/profiles/default/linux/amd64/23.0/desktop/systemd make.profile
   ```

5. Recompile GCC using this Portage configuration, which explicitly enables Profile-Guided Optimization (PGO) and Link-Time Optimization (LTO) to maximize compilation throughput:
   ```
   emerge -av sys-devel/gcc
   ```

6. Begin customizing it to fit your specific requirements.

7. Install packages:
   ```
   emerge -av gnome-base/gnome-light
   ```

## Repository Structure

Understanding the layout of this configuration is necessary for effective customization:

* [make.conf](https://github.com/jamescherti/jc-gentoo-portage/blob/main/make.conf): The primary configuration file. It contains global compiler flags (`CFLAGS`, `CXXFLAGS`), `MAKEOPTS`, global `USE` flags, and `FEATURES`.
* [package.use/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.use): A directory containing modular files that define USE flags on a per-package basis. Files are categorized logically (e.g., `gnome`, `sound-server`, `optimize`).
* [package.accept_keywords/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.accept_keywords): Allows the installation of specific testing or unstable packages on a stable system.
* [package.mask/](https://github.com/jamescherti/jc-gentoo-portage/tree/main/package.mask) and [package.unmask/](https://github.com/jamescherti/jc-gentoo-portage/blob/main/package.unmask): Used to explicitly block or allow specific package versions.

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

### Managing Per-Package USE Flags (package.use)

The `package.use/` directory is modular. You should read through the files and remove entries for software you do not intend to install. If you need a feature that is disabled globally in `make.conf` (like `nls` or `bluetooth`), do not enable it globally. Instead, enable it only for the specific package that requires it by creating a new entry in `package.use/`.

### Hardware-Specific Examples

Below are examples of how to configure `package.use` for specific hardware setups. You can create new files in `/etc/portage/package.use/` to store these directives.

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

Other project by the same author:
- [jc-dotfiles @GitHub](https://github.com/jamescherti/jc-dotfiles): A collection of UNIX/Linux configuration files. You can either install them directly or use them as inspiration your own dotfiles.
- [bash-stdops @GitHub](https://github.com/jamescherti/bash-stdops): A collection of Bash helper shell scripts.
- [jc-gnome-settings](https://github.com/jamescherti/jc-gnome-settings): GNOME customizations that can be applied programmatically.
- [jc-firefox-settings @GitHub](https://github.com/jamescherti/jc-firefox-settings): Provides the user.js file, which holds settings to customize the Firefox web browser to enhance the user experience and security.
- [jc-xfce-settings](https://github.com/jamescherti/jc-xfce-settings): GNOME customizations that can be applied programmatically.
- [watch-xfce-xfconf](https://github.com/jamescherti/watch-xfce-xfconf/): A command-line tool that can be used to configure XFCE 4 programmatically using the *xfconf-query* commands displayed when XFCE 4 settings are modified.
