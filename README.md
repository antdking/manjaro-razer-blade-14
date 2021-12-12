# Manjaro on Razer Blade 14 (2021)

This guide worked for the RTX 3070 edition of the Blade 14, however it should be applicable to the others in the series.


## Installing Manjaro

Choose your flavour, I went with KDE.

The easiest way to boot is to use the proprietary drivers.
The arch[^1] guide makes mention of `nomodeset` as a kernel parmeter, so this will likely help you boot using Open drivers.


No other special requirements needed for installing; for reference, I did a clean install with LUKS encrytion + ext4.

## Notes

- Not everything here is required for a working system, it's just how I've configured mine. Feel free to adapt this howber you want.
- For accessing the AUR, I will be using `yay`,



## Working out the box

Pretty much everything "just worked", with some notable exceptions:

- RGB Control requires the use of an unreleased version of OpenRazer[^2]
- OpenRGB did not work, I instead used razer-cli[^3] to manage the keyboard, and dbus to turn the logo off (not exposed directly in OpenRazer yet).
- HDMI + Displayport require Reverse-Prime[^4] to be configured
- Some artifacting on the screen during boot


## Fixing the bugs

### Reverse-Prime

Out of the box, Manjaro had the nvidia 495 drivers installed.

There are 2 things that need to be done to get HDMI working correctly while using the integrated GPU.

#### Xorg Config

As per the Reverse-Prime config, but tweaked a little until it worked:

```conf
Section "ServerLayout"
        Identifier "layout"
        Screen 0 "igpu"
        Option "AllowNVIDIAGPUScreens"  # Allow passing through HDMI
EndSection

Section "Device"
        Identifier "igpu"
        Driver "amdgpu"
        BusID "PCI:4:0:0"  # This might be different for yours, run 'lspci | grep VGA'. It MUST be specified.

        Option "TearFree" "true"  # Fixes screen artifacting at boot
        Option "VariableRefresh" "true"  # Enables Freesync on the monitor
        Option "DRI" "3"  # probably doesn't do anything?
EndSection

# Associate the screen to the igpu. Xorg will work out the rest.
Section "Screen"
        Identifier "igpu"
        Device "igpu"
EndSection

# dGPU needs to be explicitly noted, otherwise Xorg doesn't know what to do.
Section "Device"
        Identifier "dGPU"
        Driver "nvidia"
        Option "DRI" "3"
EndSection
```

This will get the HDMI working, however you first need to make sure the dGPU is turned on.

running `xrandr --listproviders` will wake up the card, and HDMI will work.

Or, you can enable the nvidia persistence service (installed by default):

```sh
systemctl enable --now nvidia-persistenced.service
```

### RGB

The current release (3.1.0) of OpenRazer doesn't not support this laptop, but the next one will.

In the mean time, you can install `openrazer-git` from the AUR, which will use the development version of OpenRazer.


As mentioned before, OpenRGB wasn't working for me, maybe someone else has insight. `razer-cli` did the job I wanted, so I'm not that fussed.

## Optional extras

These aren't required to make things run, it's just how I have my machine configured.

```sh
pamac install base-devel git  # lets us use AUR
yay -S spotify  # tunes
yay -S manjaro-pipewire  # better Bluetooth stack

# Rootless Container volumes
yay -S fuse-overlayfs
# User managed Namespaces
echo "anthony:100000:65536" > /etc/subgid
echo "anthony:100000:65536" > /etc/subuid

# going dockerless with this machine
yay -S buildah podman podman-docker podman-dnsname podman-compose

# setup the docker repository
echo "unqualified-search-registries = ['docker.io']" > /etc/containers/registries.conf.d/docker.conf

# get fake docker service running:
systemctl --user enable --now podman.service

# TODO: commonise
echo 'export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/podman/podman.sock"' >> ~/.bashrc
echo 'export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/podman/podman.sock"' >> ~/.zshrc

# emulation support, this takes a while.
yay -S binfmt-qemu-static qemu-user-static binfmt-qemu-static-all-arch 

# better kernel for my needs
# Custom kernel needs nvidia dkms
yay -S nvidia-dkms
yay -S linux-zen-versioned-bin linux-zen-versioned-headers-bin

# It's best to actually remove the default kernel too:
yay -R linux513  # this will be different for you dependening.

# Modify ramdisk to load modules early:
# TODO: automate

# modify /etc/mkinitcpio.conf
# MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm amdgpu)
mkinitcpio -P

# Modify grub to put NVIDIA DRM into modeset
# TODO: automate
# add nvidia-drm.modeset=1 to GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub
update-grub

# My time wasn't working weirdly, so I jsut setup Chrony
yay -S chrony
systemctl enable --now chronyd.service

# Personal use, I'm using mozilla vpn
yay -S mozillavpn

# HW Acceleration
yay -S nvidia-utils libva-mesa-driver mesa-vdpau libva-utils vdpauinfo

# I was not able to get HW Acceleration working consistently through prime.

# Gaming support, though most was there already
yay -S manjaro-steam lib32-mesa lib32-nvidia-utils

# coding
yay -S visual-studio-code-bin

# Not bothering with wayland, but it does work.

# Vulkan support, works fine
yay -S vulkan-icd-loader lib32-vulkan-icd-loader amdvlk lib32-amdvlk vulkan-tools

# RGB:
gpasswd -a $USER plugdev
yay -S openrgb-git

## reboot

# yes, yes, I know. don't install system level for pip.
# TODO: fix
pip install razer-cli
razer-cli -c dddddd -b 40
```

[^1]: https://wiki.archlinux.org/title/Razer_Blade_14_RZ09-0370
[^2]: https://openrazer.github.io/
[^3]: https://github.com/LoLei/razer-cli
[^4]: https://wiki.archlinux.org/title/PRIME#Reverse_PRIME
