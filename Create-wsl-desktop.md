# 1. Create a WSL Image with a GUI Desktop

>⚠️ **WARNING**: This procedure was designed for **Windows 11 22H2 and early builds (and probably early 23H2 builds**. 
>
>Starting with **Windows 11 24H2/25H2**, Microsoft's built-in **WSLg** (GUI support) has become more deeply integrated into WSL2, making it harder to disable and >potentially causing interference with external X11 servers. 
>
>**Note**: WSLg remains unsuitable for this use case because it **still does not support full desktop environments in a single window**—it only launches individual >applications. Therefore, X11 forwarding is still required, but now works against the system's architecture rather than with it.
>
>
> **CURRENT FIX for Windows 11 25H2**
> This fix was generated with AI and may not be the cleanest solution, but it works for now:
> 1. In section 2.1 "Install the Windows Manager", make sure xephyr is installed and the installation line is uncommented.
> 2. You **must** start XLaunch on the Windows host with Display:0 (see section 2.4.2).
> 3. The script has been updated; see **2.5.2 Script for Windows 11 25H2**. (see section 2.5.2)
>
> Todo\Improve: Create separated instruction for Windows 11 (25H2)
>

This document describes how to create a Linux WSL image with a GUI desktop, forwarding the output to an X11 server on the host. Note that we use the **X11 protocol**, not WSLg, because WSLg does not support running a full desktop GUI in a single window. Instead, WSLg opens each Linux application in its own window. This X11-based setup behaves more like a virtual machine, but with proper resolution support (unlike Hyper-V under Windows 11).

This instruction uses the **Debian** distribution in combination with **MATE** desktop, but you can of course choose other combinations.

> The **Debian** distribution source can be found [here](https://learn.microsoft.com/en-us/windows/wsl/install-manual).

> I use the pre included folder **Exported-versions** for pre-created versions.

```sh
# Syntax:
# wsl --import <name> <target-folder> <install-tar>
wsl --import debian-gui d:\wsl\wsl-data\debian-gui ./Debian/install.tar.gz
```

This creates a WSL distribution named debian-gui in the folder d:\wsl\wsl-data\. Note that the folder name is automatically created.


## Start and Use the WSL Container
The following command can be used for handling the WSL, included for your conveniece
> Start and use the WSL container <br>
>  To start and manage your WSL container , use the following commands:
> 
>  <pre class="nje-cmd-multi-line">
> 
> wsl -d distribution_name            #  using this command from your prompt will start the container
>                                     #  Use 'exit' to return to Windows. while it remains started
> wsl --list --verbose                #  Optional. Check if it is running (in other Windows CMD)
> wsl --terminate distribution_name   #  Stops the distribution
> wsl --unregister distribution_name  #  Removes the container, careful!
> wsl --set-default distribution_name #  Set default when running command; wsl
> 
</pre>

<br>

# 2 . Add MATE Desktop to Debian WSL

## 2.1 Install the Windows  Manager

``` sh
sudo apt update
sudo apt upgrade
sudo apt install mate-desktop-environment
sudo apt install mate-terminal
sudo apt install nano
sudo apt install dbus
sudo dbus-daemon --system
# sudo apt install -y xserver-xephyr  # USE THIS FOR Windows 11 25H2 
```

## 2.2 Create user

The window manager does not run by default under root.
To create a normal user:
- login as root
  - `adduser nico`
  - `usermod -aG sudo nico`
  - `su - nico`	# switch to the user 
  -  Or
  - `wsl -d debian-gui -u nico`    # to start WSL with the user    
    - `wsl --manage debian-gui --set-default-user nico ` # set the default user
    - `cat /etc/passwd | cut -d: -f1 `          # Display all available users

## 2.3 Configure X11 Forwarding

This allows displaying the full desktop, which is **not** possible with **WSLg.**

- Edit `nano ~/.bashrc` and add at the bottom:
``` bashrc
export DISPLAY=192.168.50.139:0		# s Use HOST IP Address
export WAYLAND_DISPLAY=
export XDG_SESSION_TYPE=x11
export GDK_BACKEND=x11
export QT_QPA_PLATFORM=xcb
export LIBGL_ALWAYS_INDIRECT=1 
```

- Enable `systemd`. To enable brightness and avoid power management crashes, Edit `nano /etc/wsl.conf` and add this:
  - `[boot]`
    - `systemd=true`

After enabling systemd (verify with ps -p 1 -o comm=), install required packages, this enables mounting, PoliciKit, network filesystems, shares and others:
  - `sudo apt install upower policykit-1 udisks2 gvfs gvfs-backends gvfs-fuse`

Then, configure Polkit to support shutdown and user permissions. Create the file:
`~/.config/autostart/polkit-mate-authentication-agent-1.desktop` With the following content:    
 ```
      [Desktop Entry]
      Type=Application
      Name=PolicyKit Authentication Agent
      Exec=/usr/libexec/polkit-mate-authentication-agent-1 
      OnlyShowIn=MATE;
      NoDisplay=true
      X-GNOME-Autostart-enabled=true
      Comment=Authenticate actions that require root permissions
```
Make the file executable: - `chmod +x ~/.config/autostart/polkit-mate-authentication-agent-1.desktop`

## 2.4.1 Start the **MATE** Desktop (Windows 11 22H2/23H2)

Start XLaunch on the host with the following options:
  - Display in a large window
  - Start no client
  - Native opengl
  - Disable access control

Then, in WSL, start the Mate desktop:
- `wsl -d debian-gui`
- In the WSL start  `mate-session`
  - or use the **`./start-mate.s`h** script (root nico)

You can also run individual MATE apps such as:
  - mate-terminal
  - caja
  - pluma

## 2.4.2 Start the **MATE** Desktop (Windows 11 25H2)

Same as above but make sure to use **DISPLAY:0**! 

## 2.5.1  startup script  (Windows 11 22H2/23H2)

Create ~/start-mate.sh:

 ```
#!/bin/bash
export $(dbus-launch)
echo "DBUS_SESSION_BUS_ADDRESS=$DBUS_SESSION_BUS_ADDRESS"
echo "DBUS_SESSION_BUS_PID=$DBUS_SESSION_BUS_PID"


# Start dbus if needed
if ! pgrep -x "dbus-daemon" > /dev/null; then
  sudo mkdir -p /run/dbus
  sudo dbus-daemon --system --fork
fi

## Start  MATE with dbus
dbus-launch --exit-with-session mate-session

# For this script: chmod +x ~/start-mate.sh

```

## 2.5.2  startup script (Windows 11 25H2)

Use this script and update the line `-screen 3840x2160 \ ` to match your screen resolution. Note that fullscreen mode does not work properly, and creating a window smaller than your maximum screen size may also cause issues. If you resize the window, font scaling (DPI) may become incorrect.

```
#!/bin/bash

echo "Starting MATE desktop (fixed for Win11 23H2+)"

# Fix /tmp permissions (WSL2 issue)
sudo mkdir -p /tmp/.X11-unix /tmp/.ICE-unix
sudo chmod 1777 /tmp/.X11-unix /tmp/.ICE-unix
sudo chown root:root /tmp/.X11-unix /tmp/.ICE-unix

# Step 1: connect to XLaunch
export DISPLAY=:0

# Kill any existing Xephyr
pkill -9 Xephyr 2>/dev/null

# Start Xephyr with fixed large window (moveable, not resizeable)
Xephyr :1 \
  -screen 3840x2160 \
  -dpi 96 \
  -ac \
  -noreset \
  -extension MIT-SHM &
sleep 2

# Step 2: switch to nested display
export DISPLAY=:1

# Force DPI settings
xrdb -merge <<EOF
Xft.dpi: 96
Xft.autohint: 0
Xft.lcdfilter: lcddefault
Xft.hintstyle: hintfull
Xft.hinting: 1
Xft.antialias: 1
Xft.rgba: rgb
EOF

# Complete session environment
export XDG_SESSION_TYPE=x11
export XDG_CURRENT_DESKTOP=MATE
export DESKTOP_SESSION=mate
export XDG_SESSION_DESKTOP=mate
export XDG_SESSION_CLASS=user
export XDG_SESSION_ID=$$
export XDG_RUNTIME_DIR=/run/user/$(id -u)

# Disable hardware acceleration
export LIBGL_ALWAYS_SOFTWARE=1
export GALLIUM_DRIVER=llvmpipe

# Disable problematic features
export ALSA_LOG_LEVEL=0
export G_MESSAGES_DEBUG=none
export GTK_OVERLAY_SCROLLING=0
export GTK_USE_PORTAL=0

# Force GDK scaling
export GDK_SCALE=1
export GDK_DPI_SCALE=1
export QT_SCALE_FACTOR=1

# DBus - clean session
killall dbus-daemon 2>/dev/null
rm -f /tmp/dbus-* 2>/dev/null
eval "$(dbus-launch --sh-syntax --exit-with-session)"

# Create minimal MATE session config
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/disable-power-manager.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Disable Power Manager
Exec=/bin/true
Hidden=true
NoDisplay=true
X-MATE-Autostart-enabled=false
EOF

# Disable power manager, screensaver
gsettings set org.mate.power-manager idle-dim-ac false 2>/dev/null
gsettings set org.mate.screensaver idle-activation-enabled false 2>/dev/null

# Start session
exec mate-session 2>&1 | grep -v -E "(wnck_set_client_type|unsetenv|portal.desktop.gtk|Module initialization failed)"
```



<br>

# 3 Update from Debian 11 to 12
As of June 2025 the debian download location, mentioned in the top of this document still contains a Debian 11 version . To upgrade to 12 use:

## Step 1. Do the full upgrade to the latest 11 version

- `sudo apt update`
- `sudo apt upgrade`
- `sudo apt full-upgrade`
- `sudo apt autoremove`
- `sudo reboot`

## Step 2: Update APT Sources

Edit `/etc/apt/sources.list`
- Replace all instances of `bullseye` with `bookworm`

## Step 3: Upgrade to Debian 12

- `sudo apt update`
- `sudo apt upgrade`
- `sudo apt full-upgrade`

## Step 4: Confirm Upgrade
`cat /etc/os-release.`


End.
