# 1. Create a WSL Image with a GUI Desktop

>⚠️ **WARNING**: This procedure was designed for **Windows 11 22H2 and early builds (and probably early 23H2 builds**. 
>
>Starting with **Windows 11 24H2/25H2**, Microsoft's built-in **WSLg** (GUI support) has become more deeply integrated into WSL2, making it harder to disable and >potentially causing interference with external X11 servers. 
>
>**Note**: WSLg remains unsuitable for this use case because it **still does not support full desktop environments in a single window**—it only launches individual >applications. Therefore, X11 forwarding is still required, but now works against the system's architecture rather than with it.
>
>**Recommendation**: This method may become increasingly unreliable with future Windows updates. Test thoroughly and expect potential conflicts.



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

## 2.4 Start the **MATE** Desktop

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

## 2.5  startup script

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
