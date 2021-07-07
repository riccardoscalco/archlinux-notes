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

#### Configure the nftables firewall

```
sudo pacman -Syu nftables
```

Note that the system must be updated and rebooted since a kernel upgrade:

```
ls -l /usr/lib/modules/$(uname -r)/kernel/net/netfilter | grep nft
```

If the above return no files then probably you nee to reboot.

Rules are listed in `/etc/nftables.conf`.

Start and enable the service as usual with:

```
systemctl enable --now  nftables.service
```

Check the status with:

```
systemctl status nftables.service
```

Ref: [link](wiki.archlinux.org/index.php/Nftables#Simple_stateful_firewall)

#### Security and system auditing

[Lynis](cisofy.com/lynis/) is a security auditing tool for Linux. The tool checks the system and the software configuration, to see if there is any room for improvement the security defenses.

Github repo: [github.com/CISOfy/lynis](github.com/CISOfy/lynis).

```
pacman -R lynis
```

Perform local security scan:

```
lynis audit system
```

TODO: apply suggested actions.

#### tmux

Most used keys:on panes:

* `C-b "` or `tmux split-window`: splits the window into two vertical panes.
* `C-b %` or `tmux·split-window -h`: splits the window into two horizontal panes.
* `C-b <arrow>`: move to a pane.
* `C-b C-<arrow>`: resize a pane.
* `C-b z`: toggle fullscreen a pane.
* `C-d` or `exit` to close a pane.

Most used keys:on sessions:

* `tmux new -s <session-name>`: creates a new tmux session.
* `tmux attach -t <session-name>`: attaches to an existing tmux session.
* `C-b d`: detach current session.
* `C-b C-s`: save a session (tmux-resurrect plugin).
* `C-b C-r`: restore the last saved session (tmux-resurrect plugin).
* `C-b s` or `tmux ls`: list sessions.

Most used keys:on windows:

* `C-b c` or `tmux new-window`: open a new window.
* `C-b p`: go to previous window.
* `C-b n`: go to next window.
* `C-b 3`: go to the third window.

Commands:

```
// Reloads the current configuration:
tmux source ~/.tmux.conf

// Rename session 3
tmux rename-session -t 3 <session-name>

// Kill session 9
tmux kill-session -t 9

// lists out every bound key and the tmux command it runs
tmux list-keys

// lists out every tmux command and its arguments
tmux list-commands
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

#### Change owner and group ownership of a directory

Change (recursively) the directory owner as the current user:

```
chown -R $USER /path/to/dir
```

Change (recursively) the directory group as the current user group:

```
chgrp -R $USER /path/tp/dir
```

#### Mount a partition at boot

The fstab file can be used to define how disk partitions should be mounted into the filesystem.

Mount a give partition at boot:

```
# /etc/fstab
# /dev/sdc3
UUID=d6c15483-a2f1-459c-975d-3b9015b0267b       /mnt/Torrent    ext4            rw,relatime             0 2
```

Get the UUID of a partition with:

```
lsblk -no UUID /dev/sdc3
```

Ref: [link](wiki.archlinux.org/index.php/Fstab)

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

#### Hibernate

Systemd provides commands to hibernate using the kernel's native suspend/resume functionality:

```
systemctl hibernate
```

In order to use hibernation is necessary to 1) create a swap partition, 2) set kernel parameters and 3) configure the initramfs.

Set the kernel parameter `resume` in `/etc/default/grub`, usingthe UUID of the swap space [link](wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Required_kernel_parameters):

```
// /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet module_blacklist=i915,intel_agp vga=881 resume=UUID=8970229c-3b13-449a-b55d-3407afa339ad"
```

Reconfigure the main grub configuration file:

```
grub-mkconfig -o /boot/grub/grub.cfg
```

Configure the initramfs adding the `resume` hook after `udev` [link](wiki.archlinux.org/index.php/Power_management/Suspend_and_hibernate#Configure_the_initramfs):

```
// /etc/mkinitcpio.conf
HOOKS=(base udev autodetect keyboard modconf block filesystems resume fsck)
```

Regenerate the initramfs:

```
mkinitcpio -p linux
```

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

// install alsa utils
pacman -S alsa-utils

// test
speaker-test -c 2
```

#### Pulseaudio


Pulseaudio should not be necessary. If needed, install pulseaudio with:

```
pacman -S pulseaudio pulseaudio-alsa
```

Kill and restart pulseaudio with:

```
pulseaudio -k
pulseaudio --start
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

#### Convert svg to png

```
svgexport file.svg file.png 100% 2048:2048
```

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

Install Picom:

```
pacman -S picom
```

Exec Picom when X starts (with i3 window manager):

```
// ~/.config/i3/config
# Automatically starting applications on i3 startup
exec picom -CGb
```

Ref: [link](wiki.archlinux.org/index.php/Xcompmgr#Installation), [compton (replaced by picom)](wiki.archlinux.org/index.php/Compton), [picom](https://wiki.archlinux.org/index.php/Picom)

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

#### Imagemagick / Graphicsmagick

Scale an image:

```
// imagemagick
convert image.png -scale 33% rescaled-image.png

// gm
gm mogrify -resize 256x170! image.jpg
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

Edit file `$XDG_CONFIG_HOME/psd/psd.conf`.

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

To delete a profile:

* remove it in directory `/etc/netctl`
* remove related file in `/etc/systemd/system/multi-user.target.wants`
* remove related directory in `/etc/systemd/system/`

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

or, with zsh shell,

```
~/.zprofile
export EDITOR="nvim"
export VISUAL="nvim"
```

Use `source .zprofile` to update global environment variables. 

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

#### Firefox Developer Edition

Enable spell checking:

```
pacman -S hunspell hunspell-it
// then restart the browser
```

#### Vscode

Disable GPU acceleration adding the following line to file `argv.json` (Ctrl-Shif-P -> Preferences: Configure Runtime Argument), after that restart vscode:

```
"disable-hardware-acceleration": true
```

#### Full system backup with rsync

```
rsync -aAXv --delete --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found","/home/riccardo/Git","/home/*/.cache/mozilla/*","/home/*/.cache/chromium/*"} / /mnt
```

#### Synchronizing the system clock across the network

Start/enable `systemd-timesyncd` service:

```
systemctl start systemd-timesyncd.service
systemctl enable systemd-timesyncd.service
```

Check the service status with:

```
timedatectl status
```

Ref: [link](https://wiki.archlinux.org/index.php/Systemd-timesyncd)

#### Make a screenshot

```
scrot -d 5 -c
```

#### Make a screencast

Install tools:

```
sudo pacman -S ffmpeg mkvtoolnix-cli
```

Find display resolution with:

```
xrandr -q --current | grep '*' | awk '{print$1}'
```

Make a sccreencast:

```
ffmpeg -f x11grab  -s 3840x2160 -i :0.0 -r 25 -vcodec libx264  output.mp4
```

Make a screencsat on the right screen:

```
ffmpeg -f x11grab  -s 3840x2160 -i :0.0+3840,0 -r 25 -vcodec libx264  output.mp4
```

Make a screencast at a lower resolution:

```
ffmpeg -f x11grab  -s 3840x2160 -i :0.0 -r 25 -vcodec libx264 -vf scale=1920:-1 output.mp4
```

View the screencast:

```
mpv output.mp4
```

#### Convert djvu to pdf

Install djvulibre

```
sudo pacman -S djvulibre
```

Convert to pdf

```
ddjvu --format=pdf input.djvu output.pdf
```

#### Tor
	
```
sudo pacman -S torbrowser-launcher
```
	
Enable and start tor:

```
systemctl enable --now tor
```

Run with:
	
```
torbrowser-launcher
```



