# jc-gentoo-portage - Optimized /etc/portage Setup for Gentoo Linux

The [jc-gentoo-portage](https://github.com/jamescherti/jc-gentoo-portage) repository houses a highly opinionated, performance-oriented Gentoo Portage configuration. It can be used to build a lean and fast operating system by explicitly stripping away legacy dependencies, unneeded background daemons, and redundant toolkit libraries.

These configuration files serve as a strict baseline to maximize hardware utilization, minimize compile times, and keep the system footprint as small as possible.

Key Benefits:

* Pure GTK desktop environment (specifically `gnome-base/gnome-light`). It strictly prevents Qt dependencies from bleeding into the system via applications like LibreOffice or Matplotlib. In addition to that, it actively debloats the GNOME shell by pruning heavy file-indexing hooks from `localsearch`, removing Samba/Active Directory integrations from Nautilus, and stripping out weather daemons, cloud providers, and background sensor calibrations.
* The system dependency graph is significantly reduced. Setting `-nls` globally forces interfaces to English, which skips the compilation of thousands of unneeded localization files and reduces the time required for system updates. The configuration also drops KDE libraries, optical media decryption routines, and obsolete X11 hardware overlay paths.
* Core system utilities, including GCC, Bash, and Python, are compiled using Profile-Guided Optimization (PGO) and Link-Time Optimization (LTO). Global flags such as `xs`, `asm`, `orc`, `jit`, and `threads` ensure applications use hand-optimized assembly routines and multi-core parallelism to execute code as close to the bare metal as possible.
* Network chatter is strictly bounded. The configuration explicitly disables upstream telemetry, background analytics reporting, and zero-configuration local service scanning like Avahi. It also prevents NetworkManager from leaking your IP address through periodic background HTTP connectivity checks.
* The audio system is firmly standardized on PipeWire, actively disabling the legacy PulseAudio daemon. For video, the setup relies entirely on FFmpeg's optimized internal decoders and the industry-reference `dav1d` AV1 decoder, which prevents Portage from pulling in redundant, legacy external codecs.
* Unnecessary UI layers are explicitly masked to prevent dependency bloat. For example, Qt6 is restricted from being pulled into GTK-based environments via Python data science libraries. The configuration also trims LibreOffice extensions, heavy database indexing hooks, and large redundant typography packages for a minimalist graphical environment.

## Prerequisites

Before installing, ensure your system is set to the compatible 23.0 systemd desktop profile:
```bash
eselect profile set default/linux/amd64/23.0/desktop/systemd
```

## Installation

Here's how to install James Cherti's portage in a new Gentoo installation:

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

5. Begin customizing it to fit your specific requirements.

## Customizations

### Intel Processor + NVIDIA GPU

File: `/etc/portage/package.use/00my-hardware-video`

```
# Tells Portage to only install the microcode files necessary for the host CPU.
sys-firmware/intel-microcode hostonly

# Enables drivers for both the NVIDIA discrete GPU and the Intel integrated
# GPU. This setup allows leveraging the Intel iGPU for power-efficient
# hardware video decoding.
*/* VIDEO_CARDS: -* nvidia intel

# Opts into native NVIDIA hardware video encoding/decoding while disabling the legacy,
# X11-bound VDPAU backend.
*/* nvenc nvdec -vdpau

# Enables VA-API support across applications. This allows the Intel integrated GPU
# to handle hardware acceleration paths natively via Intel Quick Sync.
*/* vaapi
```

### Intel audio + USB audio

File: `/etc/portage/package.use/00hardware-audio`

```
# Intel audio + USB audio
*/* ALSA_CARDS: -* hda-intel usb-audio
```

### Scanner: Disable all sane backends

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
