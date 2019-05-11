#### To Do

* move /home to an encripted partition
* clone disk (wiki.archlinux.org/index.php/Dd#Disk_cloning_and_restore)
* clean config files (wiki.archlinux.org/index.php/System_maintenance#Clean_the_filesystem)
* decorate virtual console (github.com/uobikiemukot/yaft)
* evaluate file manager (github.com/jarun/nnn#cmdline-options)
* consolidate colorisation (wiki.archlinux.org/index.php/Color_output_in_console#pacman)

#### Installation notes

```
loadkeys it
setfont latarcyrheb-sun32 // use bigger font in HiDMI monitor (see also terminus fonts)
wifi-menu
ping www.archlinux.org // test connection
timedatectl set-ntp true

// Disk partition
lsblk
cfdisk // sda1 5M linux boot and sda2 remaining size linux
mkfs.ext4 /dev/sda1
mkfs.ext4 /dev/sda2

mount /dev/sda2 /mnt
pacstrap /mnt base base-devel
genfstab -U /mnt >> /mnt/etc/fstab

// CHROOT
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
hwclock --systohc

// Setup locales
// uncomment `it_IT.UTF-8` in `/etc/locale.gen`
echo 'LANG="it_IT.UTF-8"' >> /etc/locale.conf
echo 'LC_COLLATE="C"' >> /etc/locale.conf 

// Set Hostname

// Install packages
pacman -S iw wpa_supplicant dialog

// Set admin psw
passwd

// Setup Grub
pacman -S intel-ucode grub
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

// Add user
pacman -S zsh
useradd -m -s /bin/zsh riccardo
passwd riccardo
// edit sudoers file using visudo

// Use terminus font in the Virtual console
pacman -S terminus-font
setfont ter-132n (add FONT=ter-132n in /etc/vconsole.conf)
```

#### Autostart X at login

Add the following to `.zshrc` :

```
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
	exec startx
fi
```

Ref: [link](wiki.archlinux.org/index.php/Xinit#Autostart_X_at_login)

#### Pacman commands

Upgrade the system:

```
pacman -Syu
```

Check for packages that were installed as a dependency but now no other packages depend on them:

```
pacman -Qtd
```

Query foreign packages, i.e. packages that were not found in the sync database (AUR packages for example).

```
pacman -Qm
```

#### Cleaning the package cache

```
systemctl enable paccache.timer
systemctl start paccache.timer
```

#### Swap

Check swap status using `swapon --show` or `free -h`.

Create a swap partition with:

```
cfdisk /dev/sdb // use partition type 82
mkswap /dev/sdb2
```

The `mkswap` command should return the UUID of the swap space. Enable this swap partition on boot, add an entry to `/etc/fstab`:

```
// /etc/fstab
# /dev/sdb2
UUID=8970229c-3b13-449a-b55d-3407afa339ad       none            swap            defaults,discard        0 0
```

Parameters `default,discard` are related to the use of TRIM support on SSD.

#### LOCALE

Display the currently set locale:

```
locale
```

List available locales which have been previously generated:

```
localedef --list-archive
```

Edit locale in `/etc/locale.conf`, to make locale changes immediate run:

```
unset LANG
source /etc/profile.d/locale.sh
```

#### Audio (TODO)

```
// Verifying correct sound modules are loaded
lsmod | grep '^snd' | column -t

// check the directory /dev/snd/ for the right device files
ls -l /dev/snd

// install pulseaudio and utils
pacman -S pulseaudio pulseaudio-alsa alsa-utils

// test
speaker-test -c 2
```

#### Image viewer

```
sxiv -t -z 75 ./
```

Use key `Enter` to switch between Thumbnail mode and Image mode, use `n` and `p` for the next and previous image in Image mode.

#### NVIDIA

Make sure to have the latest linux kernel:

```
pacman -Syu
```

Get your card name:

```
lspci -k | grep -A 2 -E "(VGA|3D)"
```

Install the nvidia driver for your card. For card code names, have a look at https://nouveau.freedesktop.org/wiki/CodeNames/#NV130.

```
pacman -S nvidia
```

#### Nouveau open source driver

If present, uninstall nvidia driver.

```
pacman -Rs nvidia
```

Install `mesa` pkg.

```
pacman -S mesa
```

Ref: [link](wiki.archlinux.org/index.php/Nouveau)

Aside note: in my case (GeForce GTX 1050 Ti) nouveau driver has poor performance.


#### Blacklist intel modules in /etc/default/grub (black screen issue):

After installing the nvidia driver and rebooted I got a black screen (where actually is still possible to blindly login).
The issue is related to my Intel integrated GPU.

Blacklinsting modules in `/etc/modprobe.d/blacklist.conf` did not work for me, module `i915` is still loaded (`lsmod | grep "i915"`), therefore I opted for the kernel command line.

```
// /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet module_blacklist=i915,intel_agp"
```

Generate the grub config file and reboot:

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

Ref: [link](https://wiki.archlinux.org/index.php/NVIDIA/Troubleshooting#Black_screen_on_systems_with_Intel_integrated_GPU)

#### Setting the framebuffer resolution (TODO):

```
// /etc/default/grub
GRUB_GFXMODE=1920x1080x32
```

Generate the grub config file and reboot:

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
reboot
```

For some reason the above method does not work, therefore I use old `vga=` way.

To view the list of supported modes:

```
sudo hwinfo --framebuffer
```

Convert to decimal number, for example the `0x0371` returns `881`:

```
printf "%d\n" 0x0371
```

Update grub configuration file:

```
// /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet module_blacklist=i915,intel_agp vga=881"
```

Regenerate the main configuration file `/boot/grub/grub.cfg`:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Composite

Make sure the Composite extension is enabled for the X Server:

```
xdpyinfo | grep Composite
```

`xdpyinfo` can be installed with `pacman -S xorg-xdpyinfo`.

Install Compton:

```
pacman -S compton
```

Exec Compton when X starts (with i3 window manager):

```
// ~/.config/i3/config
# Automatically starting applications on i3 startup
exec compton -CGb
```

Ref: [link](wiki.archlinux.org/index.php/Xcompmgr#Installation), [link](wiki.archlinux.org/index.php/Compton)

#### Nodejs

```
pacman -S nodejs npm yarn
```

Allow global package installations for the current user with:

```
// .profile
PATH="$HOME/.node_modules/bin:$PATH"
export npm_config_prefix=~/.node_modules
```

#### Managing .pac* files


Locate .pac* files:

```
locate --existing --regex "\.pac(new|save)$"
```

Search all .pacnew and .pacsave files and ask for any actions on them:

```
pacdiff
```

Ref: [link](wiki.archlinux.org/index.php/Pacman/Pacnew_and_Pacsave)

#### i3

Install the i3 window manager:

```
pacman -S i3
```

Start with X:

```
// .xinitrc
exec i3
```

Configuration file: `~/.config/i3/config`.
Ref card: [https://i3wm.org/docs/refcard.html](https://i3wm.org/docs/refcard.html).

#### ZSH

Install (oh my) zsh:

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

Add this to .zshrc to automatically rehash new executables, this avoid to restart the shell for having auto-completion of newly installed programs:

```
// .zshrc
zstyle ':completion:*' rehash true
```

Update with:

```
upgrade_oh_my_zsh
```

Replace file `~/.profile` with `~/.zprofile`. When starting, Zsh will source it.

#### Termite

Keybinding: [github.com/thestinger/termite#keybindings](github.com/thestinger/termite#keybindings)
Config file: `~/.config/termite`.

#### Gtk-3 theme

```
pacman -S arc-gtk-theme
pacman -S arc-icon-theme
pacman -S lxappearance
```

#### Locate files

```
pacman -S mlocate
updatedb
```

Locate a file

```
locate ...
```

#### MIME

```
pacman -S xdg-utils
```

Determine a file's MIME type:

```
xdg-mime query filetype filename.md
```

Default application for a MIME type:

```
xdg-mime query default image/jpeg
```

Change the default application for a MIME type:

```
xdg-mime default feh.desktop image/jpeg
```

#### Add fonts

Place fonts in the user directory `~/.local/share/fonts`.

Update the fontconfig font cache:

```
fc-cache
```

#### Imagemagick

Scale an image:

```
convert image.png -scale 33% rescaled-image.png
```

#### Set Wallpaper Image

Install feh:

```
pacman -S feh
```

Set a wallpaper image:

```
feh --bg-scale /path/to/image.file
```

Make it persistent adding `~/.fehbg &` in `~/.config/i3/config`:

```
exec --no-startup-id ~/.fehbg
```

#### HiDPI

Edit `.profile` and `.Xresources` as following:

```
// ~/.profile
export GDK_SCALE=2
export GDK_DPI_SCALE=0.5
export QT_AUTO_SCREEN_SET_FACTOR=0
export QT_SCALE_FACTOR=2
export QT_FONT_DPI=96
```

```
// ~/.Xresources
Xft.dpi: 192
Xcursor.size: 32
```

#### Miltihead

See the state of the outputs:

```
xrandr
```

Configure two monitors, one HDMI and one DP:

```
xrandr --output HDMI-3 --mode 1920x1080 --pos 3840x0 --output DP-3 --mode 3840x2160 --pos 0x0
```

In ended up setting xrandr in the `.xinitrc` file:

```
// .xinitrc
xrandr --output HDMI-0 --mode 1920x1080 --pos 3840x0 --scale 2x2 --output DP-0 --mode 3840x2160 --pos 0x0 &
exec i3
```

Note the presence of `--scale` for the HDMI output, it changes the dimensions of the output picture, so that the dimension of the HDMI output is equal to the dimension of the DV output.

#### Set keyboard layout in Xorg

View keyboard settings:

```
setxkbmap -print -verbose 10
```

Set keyboard layout it:

```
setxkbmap -layout it
```

Edit Xorg conf to have a system-wide configuration which is persistent across reboots:

```
// /etc/X11/xorg.conf.d/00-keyboard.conf
Section "InputClass"
		Identifier "system-keyboard"
		MatchIsKeyboard "on"
		Option "XkbLayout" "it"
EndSection
```

Ref: [link](wiki.archlinux.org/index.php/Xorg/Keyboard_configuration)

#### Framebuffer terminal (TODO)

wiki.archlinux.org/index.php/List_of_applications#Terminal

#### Keyboard bindings

Bind keysym with i3 window manager. To interactively enter a key and see what keysym it is configured to, use xev (`pacman -S xorg-xev`).

```
// ~/.config/i3/config
bindsym $mod+XF86Eject exec --no-startup-id systemctl suspend

bindsym $mod+F12 exec --no-startup-id amixer set Master 5%+
bindsym $mod+F11 exec --no-startup-id amixer set Master 5%-
bindsym $mod+F10 exec --no-startup-id amixer set Master toggle
```

Ref: [link](i3wm.org/docs/userguide.html#keybindings)

#### Jekyll

```
sudo pacman -S ruby
```

Allow RubyGems to be executed:

```
// ~/.profile
PATH="$PATH:$(ruby -e 'puts Gem.user_dir')/bin"
```

Update with `source ~/.profile` for changes to apply.

Install jekyll and bundler:

```
gem update
gem install jekyll bundler
```

#### Manage browser profile in tmpfs

To manage the browser profile in tmpfs install profile-sync-daemon:

```
// yay is an AUR helper, visit github.com/Jguer/yay
yay -S profile-sync-daemon
```

Then activate the service:

```
psd // and edit the indicated file
psd
systemctl --user start psd.service
systemctl --user enable psd.service
```

Ref: [link](wiki.archlinux.org/index.php/Profile-sync-daemon)

#### Enable Trim on SSD

Verify trim support as described in [wiki.archlinux.org/index.php/Solid_state_drive#TRIM](wiki.archlinux.org/index.php/Solid_state_drive#TRIM).

Check if unit `fstrim.timer` is installed:

```
systemctl list-unit-files | grep "fstrim"
```

Enable the timer unit:

```
systemctl enable fstrim.timer
```

The timer will activate the service weekly.

Ref: [en.wikipedia.org/wiki/Trim_(computing)](en.wikipedia.org/wiki/Trim_(computing)).

#### Netctl

Network profiles are in `/etc/netctl`, in case you want to remove a profile just delete the relevant file.

List available profiles:

```
netctl list
```

Use `wifi-menu` to generate profiles.

Start a profile:

```
netctl start <profile>
```

Enable a profile on boot:

```
netctl enable <profile>
```

#### Disable pc speaker beep

```
echo "blacklist pcspkr" > /etc/modprobe.d/nobeep.conf
```

#### Redshift

```
pacman -S redshift
```

Edit file `~/.config/redshift/redshift.conf` with latitude and longitude coordinates.

Start with X, before the window manager:

```
// ~/.xinitrx
redshift &
exec i3
```

Note that the systemctl user service fails (`systemctl --user`) (TODO: see arch docs)

Ref: [link](wiki.archlinux.org/index.php/Redshift)


#### Enable F<num> keys on Apple Keyboard

```
echo "options hid_apple fnmode=2" > /etc/modprobe.d/hid_apple.conf
```

#### Set custom Keymap in the Virtual Console

Assuming `it` is the loaded keymap, the file to customise is `/usr/local/share/kbd/keymaps/it.map`.
Add some custom keymaps, for example:

```
shift altgr keycode 26 = braceleft
shift altgr keycode 27 = braceright
```

#### Environment variables

```
~/.profile
export EDITOR="nvim"
export VISUAL="nvim"
```

#### Update the mirror list with Reflector

Install reflector:

```
pacman -Syu reflector
```

Update the mirror list with the HTTPS mirrors located in either France, Germany or Italy, rate and sort the 20 most recently synchronized mirrors by download speed, and overwrite the file `/etc/pacman.d/mirrorlist`:

```
reflector --country France --country Germany --country Italy --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Schedule a systemd service that runs reflector at each boot once the network is up:

```
// /etc/systemd/system/reflector.service
[Unit]
Description=Pacman mirrorlist update
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/reflector --country France --country Germany --country Italy --latest 20 --protocol https --sort rate --save /etc/pacman.d/mirrorlist

[Install]
RequiredBy=multi-user.target
```

Enable the service and the network wait service of `netctl`:

```
systemctl enable reflector.service
systemctl enable netctl-wait-online.service
```

Ref: [link](wiki.archlinux.org/index.php/Reflector)
