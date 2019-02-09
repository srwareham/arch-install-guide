Installing a Window System
-------------
Install a window system and appropriate drivers to facilitate a desktop environment.

### The X Window System

Install the xorg implementation of the X window system.

1. Install xorg and xinit
   ```bash
   sudo pacman -S xorg-server xorg-xinit
   ```

#### Proprietary NVIDIA Drivers

Install NVIDIA's proprietary drivers for recent cards. 

1. Confirm your system properly detects your GeForce 600 or newer card.
   
   ```bash
   lspci | grep -e VGA -e 3D  
   ```
   
2. Install the NVIDIA proprietary drivers.

   ```bash
   sudo pacman -S nvidia
   ```
   
3. Install a pacman hook to support properly updating graphics drivers with pacman.
   ```bash
   # Create directory if not already present
   sudo mkdir -p /etc/pacman.d/hooks/
   ```
   
   ```bash
   # Open file to write hook
   sudo nano /etc/pacman.d/hooks/nvidia.hook
   ```
   
   Copy paste the below file into the terminal and then press `<ctrl o>` followed by `<ctrl x>`.
   ```bash
   [Trigger]
   Operation=Install
   Operation=Upgrade
   Operation=Remove
   Type=Package
   Target=nvidia
   Target=linux

   [Action]
   Description=Update Nvidia module in initcpio
   Depends=mkinitcpio
   When=PostTransaction
   NeedsTargets
   Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
   ```

4. Install NVIDIA's configuration tool

   ```bash
   sudo pacman -S nvidia-settings
   ```

5. Write an NVIDIA-generated xorg configuration file to `/etc/X11/xorg.conf`.

   ```bash
   sudo nvidia-xconfig
   ```

Installing a Desktop Environment
-------------
 
 We will install a desktop environment configured to run using the X Window Server. Note that we will _not_ configure our arch install to automatically boot to the KDE Plasma environment, but instead will boot to the console. The desktop enviornment will be accessible by entering the `startx` command.
 
 ### KDE Plasma
 
 We will install the KDE Plasma desktop environment.
 
 1. Install KDE Plasma.
 
    ```bash
    # Accept all defaults when prompted
    sudo pacman -S plasma
    ```
   
2. Configure xinit to start KDE Plasma.
 
   ```bash
   echo "exec startkde" > $HOME/.xinitrc
   ```
   
3. Reboot to allow your computer to switch to the NVIDIA graphics driver and then launch the KDE Plasma Enviornment.

    ```bash
    sudo reboot
    ```
    
    Wait for your system to boot then start KDE Plasma.
    
    
    ```bash
    startx
    ```
   
4. Manually configure your nvidia settings by launching the `nvidia-settings` tool and configuring to your setup. Suggested tweaks include arranging multi-monitor setups and enabling "Force Full Composition Pipeline" under "X Server Display Configuration -> Advanced" for each display.

   Open a new terminal and enter
   ```
   sudo nvidia-settings
   ```
   
   Manualy modify settings and then press "Save to X Configuration File"
