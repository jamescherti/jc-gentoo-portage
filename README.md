# jc-gentoo-portage

This repository houses James Cherti's [Gentoo](https://www.gentoo.org/) Portage (`/etc/portage`), which enables the compilation and installation of software packages on a Gentoo Linux system.

The configurations in this repository can serve as a foundation for your own Portage setup or inspire you to enhance your existing configuration.

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
