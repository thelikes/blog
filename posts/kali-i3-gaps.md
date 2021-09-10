# kali + i3-gaps

## resources
- [i3 reference card](https://i3wm.org/docs/refcard.html)
- [0x1-i3-i3gaps-kali (comradecheese)](https://comradecheese.com/2020/06/20/0x1-i3-i3gaps-kali-linux/)
- [lightening the load for kali VMs (jmpesp) (i3 over gnome)](https://jmpesp.me/kali-and-i3-lightening-the-load/)
- [animated background - @xct](https://gist.github.com/xct/cf2d9aa2585c68c2f600f99386e9f815)

## packages
- xorg
- xinit
- i3-gaps
- rofi
- open-vm-tools
- open-vm-tools-desktop
- feh
- compton
- imagemagick
- lxappearance
- flameshot
- eog
- nautilus
- gnome-terminal
- evince
- [sublime-text](https://www.sublimetext.com/docs/3/linux_repositories.html)

## config

```
# full screen , copy&paste
exec_always vmware-user-suid-wrapper
# wallpaper
exec_always feh --bg-scale image.jpg
# transparency
exec_always compton -f
# rofi launcher
bindsym $mod+d exec rofi -show run
# kde screenshot tool
bindsym $mod+p spectacle -g
# gaps
# from: https://bbs.archlinux.org/viewtopic.php?id=252775

# This effectively disables the window title bars alltogether (enabling gaps/gapping)
#for_window [class=".*"] border pixel 0

gaps inner 8
gaps outer -2

# Smart gaps (gaps used if only more than one container on the workspace)
#smart_gaps on

# Smart borders (draw borders around container only if it is not the only container on this workspace)
# on|no_gaps (on=always activate and no_gaps=only activate if the gap size to the edge of the screen is 0)
smart_borders on
```

## setup
```
# chrome
apt install snapd
systemctl start snapd.service
snap install chromium
ln -s /snap/bin/chromium /usr/bin
systemctl enable apparmor
# rofi theme
alt+d -> search 'rofi-theme-selector'
pip3 install pywal

# qemu
# https://superuser.com/questions/1111788/i3-how-to-save-background-and-screen-resolution
# list available
xrandr --query

# set
xrandr --out Virtual-1 --mode 3440x2440
```