## Setting up MicroOS as a desktop OS
### Pre-installation
1. Download an ISO image from [https://en.opensuse.org/Portal:MicroOS/Downloads](https://en.opensuse.org/Portal:MicroOS/Downloads); _Choose the Tumbleweed base, as the LEAP base does not contains patterns for the desktop._
2. Write the ISO on a USB drive and start the installation proces.
3. Select GNOME or KDE Plasma as desktop environment.
4. Select _Encryption_ if desired. 

Commands starting with `$` can be copied straight into your terminal prompt.
Command starting with `#` can only be run inside a transactional-update shell; _transactional-update shell is a chroot where you can open the snapshot you want to boot into next. And there you can make adjustments to the normally immutable root filesystem. This is mainly for advanced users._

### Ways to install applications
You can install applications in several ways:
* flatpaks from flathub --_preferred solution_
* RPM's via `sudo transactional-update pkg install package_name`
* RPM's in a toolbox `toolbox -u`
* snaps (requires some extra setup to get snapd installed)
* AppImages

### Full drive encryption
If you want a full drive encryption, you can go through the ISO installer as normal, but at the last screen click on _Partitioning_ and use the guided partitioner to enable encryption. _LVM_ is not necessary for it to work. However with full drive encryption you will need to enter your password twice: 
1. the first time to open the grub menu
2. the second time to decrypt during boot.

Note that the second decrypt can also be done automatically with the following [manual](https://en.opensuse.org/SDB:Encrypted_root_file_system):

#### Setting up a key file
The steps below describe how to set up a key file. Please execute the commands and edit the files listed below in a _transactional-update_ shell.

```
$ sudo transactional-update shell
  # touch /.root.key
  # chmod 600 /.root.key
  # dd if=/dev/urandom of=/.root.key bs=1024 count=1
  # cryptsetup luksAddKey /dev/sda1 /.root.key
```
replace `/dev/sda1` with the root partition (`sudo fdisk -l` to see partitions)
```
  # vim /etc/crypttab
```
  Edit `/etc/crypttab`, find the line starting with the `UUID` reference of the root partition (`UUID=...`) and write the path to the the key file in the third column.

    cr_sda1 UUID=... /.root.key

  (Again, the partition name is just an example.)
  Configure `dracut` to add the key file to the initrd:
```
  # echo -e 'install_items+=" /.root.key "' | tee --append /etc/dracut.conf.d/99-root-key.conf > /dev/null
  # mkinitrd
  # exit
$ sudo reboot
```

### Installation: GNOME
Make sure you install a browser via transactional-updates if you want to use GNOME extensions. The easiest way to install Gnome extensions is via the website provided by the [Gnome project](https://extensions.gnome.org/). For this to work however you need a browser that installed directly on the system and in a container (like flatpaks, snaps, AppImages or in a toolbox). In the browser you need to install the plugin that is mentioned on the top of the page when you first open it. 

After installation of the plugin you can reload the page and choose the Gnome Extensions you want to install.

To get Gnome extensions running you need the following command to install Chromium directly:
```
$ sudo transactional-update pkg install chromium
$ sudo reboot
```

* You can also install some other useful packages trough transactional-update and then reboot:
```
$ sudo transactional-update pkg install gnome-remote-desktop gnome-shell-search-provider-nautilus
$ sudo reboot
```

* Some necessary commands for _GNOME Software_: 
```
$ gsettings set org.gnome.software install-bundles-system-wide false
$ gsettings set org.gnome.software allow-updates false
$ gsettings set org.gnome.software download-updates false
$ gsettings set org.gnome.software enable-repos-dialog false
$ gsettings set org.gnome.software first-run true
```
and the following:
```
$ sudo rm -Rf /var/cache/app-info
$ sudo transactional-update shell
  # rpm -e --nodeps libzypp-plugin-appdata
  # zypper al libzypp-plugin-appdata
  # exit
$ sudo reboot
```

### Installation: KDE
* In _Discover_, enable the _flatpak_ repository.
* For theming, normal theme installations don’t work in KDE Plasma. But you can install themes via a transactional-update shell:
    * (For example the Arc theme and papirus-icon-theme)
```
$ sudo transactional-update shell
  # cd /tmp
  # wget -qO- https://raw.githubusercontent.com/PapirusDevelopmentTeam/arc-kde/master/install.sh | sh
  # wget -qO- https://git.io/papirus-icon-theme-install | sh
  # exit
$ sudo reboot
```
* Other themes:
    * [Qogir-icon-theme](https://github.com/vinceliuice/Qogir-icon-theme)
    * [Matcha-gtk](https://github.com/vinceliuice/Matcha-gtk-theme)
    * [Matcha-KDE](https://github.com/vinceliuice/Matcha-kde)
    * [Akwa-gtk](https://github.com/berkiyo/akwa)
    
### General tips
It is recommended to disable automatic updating and rebooting as well. It’s a handy feature (mainly for servers that are running 24/7), and the developers are thinking about to better integrate it within a desktop distribution. For now it is recommended to do the same, at least until you have learned a little bit about it, are more familiar with the idea and with the basics of how it works, and come come-up with the best automatic updating & rebooting strategy for your workflow and use case. So:
```
$ sudo systemctl disable --now transactional-update.timer
$ sudo systemctl disable --now rebootmgr.service
```
And then let’s check:
```
$ sudo rebootmgrctl is-active
*RebootMgr is dead*
$ sudo rebootmgrctl status
*Error: The name org.opensuse.RebootMgr was not provided by any .service files*
```

Updates are now to be initialised manually with:
```
$ sudo transactional-update dup
$ sudo reboot
```
Or
```
$ sudo transactional-update shell
  # zypper dup
  # exit
$ sudo reboot
```


### Installing flatpaks

The `flatpak` package is already installed, all that needs to be done is to add the flathub repo. Within KDE Plasma this is possible from within _Discover_. For GNOME you have to use one of the two options below (also possible for a terminal prompt if you don't like Discover):

If you want to install flathub for only your user (in `/home/~ folder`):
```
$ flatpak remote-add --user --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```
If you want to install flathub for all users (in `/var folder`):
```
$ flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```
Some flatpaks you could install:
- Firefox
- Libreoffice
- VLC
- etc.


### Installing snapds

Snaps are mainly used within Ubuntu, but it is possible to install them as well in openSUSE MicroOS with the following commands.

```
$ sudo transactional-update shell
  # zypper addrepo --refresh https://download.opensuse.org/repositories/system:/snappy/openSUSE_Tumbleweed snappy
  # zypper --gpg-auto-import-keys refresh
  # zypper ref
  # zypper dup --from snappy
  # zypper in snapd
  # exit
$ sudo reboot
$ sudo mksubvolume /snap
$ source /etc/profile
$ sudo systemctl enable --now snapd
$ sudo systemctl enable --now snapd.apparmor
```
You can install snaps via the terminal (for example the snap store where you can find snaps and install them via a graphical interface):
```
$ sudo snap install snap-store
```

### Installing virtualbox
```
$ sudo transactional-update shell
  # zypper ref
  # zypper in virtualbox-qt
  # exit
$ sudo reboot
```

#### Adding the extension pack for the host
```
$ sudo transactional-update shell
  # cd /tmp
  # LatestVirtualBoxVersion=$(wget -qO - https://download.virtualbox.org/virtualbox/LATEST-STABLE.TXT) && wget "https://download.virtualbox.org/virtualbox/${LatestVirtualBoxVersion}/Oracle_VM_VirtualBox_Extension_Pack-${LatestVirtualBoxVersion}.vbox-extpack"
  # VBoxManage extpack install --replace Oracle_VM_VirtualBox_Extension_Pack-${LatestVirtualBoxVersion}.vbox-extpack
  # exit
$ sudo reboot
```

### Adding software from OBS
Find what you are looking for on [https://software.opensuse.org](https://software.opensuse.org). Then choose the openSUSE Tumbleweed repo, as MicroOS desktop is build from the same base as Tumbleweed. Everything that runs on Tumbleweed, should in theory, run without fail on MicroOS Desktop.
  You can't use the 1-click install, as there is no YaST and the 1-click installer has no access to a transactional-update. So     for example if we install `zerotier`, pick a TumbleWeed repository. But click on _Expert Download_ choose the _Add repository and install manually_ option.
```
$ sudo transactional-update shell
  # zypper addrepo https://download.opensuse.org/repositories/home:alphard:RHEL/openSUSE_Tumbleweed/home:alphard:RHEL.repo 
  # zypper ref
  # zypper in zerotier-one
  # exit
$ sudo reboot
```
If you have added repositories to zypper by yourself, it's possible that they will not refresh automatically with a `dup` command. You'll have to give them the flag to autorefresh:
```
$ sudo transactional-update shell
  # zypper lr -p
  # zypper mr -f <number of repo you want to add Yes to in the Refresh colomn>
  # exit
$ sudo reboot
```

### Tips if you're installing on laptop
2 important packages were found missing for some users:

* `tlp` for better battery management

* `ModemManager` for laptops with a WWAN moden

You can install both with:
```
$ sudo transactional-update pkg install tlp ModemManager
$ sudo reboot
```

### Some useful aliases for bash to type the commands faster
edit the file `~/.bashrc`

And add the following lines in the file at the end:
```
#My custom aliases
alias shalt="sudo systemctl halt"
alias sboot="sudo systemctl reboot"
alias sts="sudo transactional-update shell"
alias stsc="sudo transactional-update shell --continue"
alias stdc="sudo transactional-update dup --continue"
alias std="sudo transactional-update dup"
alias fsu="sudo flatpak update && sudo snap refresh"
```
When you for example type `std` in the terminal, you will give the command `sudo transactional-update dup`


## Known Issues
### No graphical session/login screen in Hyper-V
hyper-v needs package "xf86-video-fbdev" for graphical session
```
$ sudo transactional-update pkg install xf86-video-fbdev
$ sudo reboot
```

### Toolbox is not starting ("potentially insufficient UIDs and GIDs")
uid/guid needs to be set for toolbox to work
```
$ sudo transactional-update shell
  # echo "<yourusername>:100000:65536" > /etc/subuid 
  # echo "<yourusername>:100000:65536" > /etc/subgid
  # exit
$ sudo reboot
```

### Graphical applications in the toolbox are missing fonts/icons
They need to be installed inside the toolbox
```
sudo zypper install xorg-x11-fonts-core adwaita-icon-theme
```

## Alternative for Gnome Extensions
There is an alternative to install Gnome Extensions, so no RPM installed browser is necessary.
It's currently under development at [Github](https://github.com/ekistece/GetExtensions)

You can install it via a toolbox; the commands are as follows:
```
$ toolbox -u
$ sudo zypper in git python38-pip
$ cd /tmp
$ git clone https://github.com/ekistece/GetExtensions.git
$ pip3 install ./GetExtensions --user
$ exit
```
You can then open the application `Get Extensions` that is available in your launchpad 

To uninstall:
```pip3 uninstall getextensions```



