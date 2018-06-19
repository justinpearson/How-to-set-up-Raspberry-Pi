# Setting up a new Raspberry Pi from scratch

# Table of Contents

1. [Hardware setup](#1-hardware-setup)
    1. [Download Raspbian](#download-raspbian)
    1. [burn micro-SD card ](#burn-micro-sd-card )
        1. [Option 1: etcher](#option-1-etcher)
        1. [Option 2: dd](#option-2-dd)
    1. [MPEG license keys (Optional)](#mpeg-license-keys-optional)
    1. [Key generation for `ssh`](#key-generation-for-ssh)
    1. [Power up with keyboard, mouse, monitor](#power-up-with-keyboard-mouse-monitor)
2. [Configure linux (AS ROOT)](#2-configure-linux-as-root)
    1. [`raspi-config`](#raspi-config)
    1. [Change passwords](#change-passwords)
    1. [Require pw when sudo](#require-pw-when-sudo)
    1. [Capslock to ctrl](#capslock-to-ctrl)
    1. [key-repeat](#key-repeat)
    1. [bashrc](#bashrc)
    1. [sshd](#sshd)
    1. [root crontab](#root-crontab)
3. [Update software (AS ROOT)](#3-update-software-as-root)
    1. [plug in internet](#plug-in-internet)
    1. [update](#update)
    1. [firewall](#firewall)
    1. [install packages](#install-packages)
        1. [exim4 & mutt](#exim4-mutt)
4. [Configure a user](#4-configure-a-user)
    1. [user's bashrc](#users-bashrc)
    1. [python](#python)
    1. [git](#git)
        1. [set username & email](#set-username-email)
        1. [bash prompt shows git branch etc](#bash-prompt-shows-git-branch-etc)
        1. [aliases](#aliases)
        1. [colors](#colors)
5. [For headless web-browser driving (Selenium)](#5-for-headless-web-browser-driving-selenium)
    1. [install](#install)
        1. [geckodriver](#geckodriver)
    1. [Test headless (xvfb) selenium installation](#test-headless-xvfb-selenium-installation)




# 1. Hardware setup
## Download Raspbian

<https://www.raspberrypi.org/downloads/raspbian/>

Latest download: <https://downloads.raspberrypi.org/raspbian_latest>

## burn micro-SD card 
###  Option 1: etcher

<https://etcher.io/>

![](Images/etcher.png)

- Don't have to extract the .zip file, can give it directly to etcher.

###  Option 2: dd

1. Unzip `.zip` to extract the `.img` file
2. Insert micro-SD card
3. Find the location of the SD card in `/dev/`:

        (python3.4.1)Computron:~ justin$ diskutil list
        /dev/disk0
           #:                       TYPE NAME                    SIZE       IDENTIFIER
           0:      GUID_partition_scheme                        *500.3 GB   disk0
           1:                        EFI EFI                     209.7 MB   disk0s1
           2:                  Apple_HFS Macintosh HD            499.4 GB   disk0s2
           3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3
        /dev/disk1
           #:                       TYPE NAME                    SIZE       IDENTIFIER
           0:     FDisk_partition_scheme                        *15.5 GB    disk1
           1:                 DOS_FAT_32 UNTITLED                15.5 GB    disk1s1

    In this example the 16 GB SD card is `/dev/disk1`. The other one, `/dev/disk0`, is my laptop HDD. 

4. Unmount the SD card by using the disk identifier to prepare copying data to it:

        diskutil unmountDisk /dev/disk1

    (or can use Mac's Disk Utility.)

5. Use the "raw" location `/dev/rdisk1` (note the extra `r` in `rdisk1`) in the `dd` command:

        (python3.4.1)Computron:~ justin$ date; sudo dd bs=1m \
        if=/Users/justin/Projects/Raspberry-pi-Raspbian-images/2018-04-18-raspbian-stretch.zip \
        of=/dev/rdisk1; date;

        Tue Nov  3 17:57:00 PST 2015
        292+0 records in
        292+0 records out
        306184192 bytes transferred in 8.355207 secs (36645914 bytes/sec)
        Tue Nov  3 17:57:08 PST 2015

Notes:

- Takes 10 min with class 4 SD card, 10 sec with class 10.
- You can check the progress by sending a SIGINFO signal pressing Ctrl+T.
- If the command fails somehow, try using `disk` instead of `rdisk`.
- If `resource busy` error, you need to unmount the SD card (but do not eject it) before running `dd`.
- If `dd: invalid number '1m'` error, use `1M` instead of `1m`.
- If `dd: /dev/rdisk1: Input/output error`, but `dd` continues to run, it might be ok.

## MPEG license keys (Optional)

**Optional: Hardware-decoding of MPG video**

Only do this part if you'll be playing or processing video on the RPi.

The following section comes from this YouTube video (note: in the video, he ssh's into the RPi to edit `/boot/config.txt`. It is easier to plug the SD card into a normal computer and edit it directly.)

<https://www.youtube.com/watch?v=3zToVK6U7AM>

The RPi is not powerful enough to play HD video without stuttering.

However, there is a solution:

The Raspberry Pi has an MPEG-decoder chip that allows you to hardware-accelerate the decoding of MPEG video. However, due to MPEG licensing reasons, this chip is disabled by default; you can pay $5 for an MPG license key to enable it. If you observe stuttering or skipping of MPG video, consider enabling the chip with the following procedure:

1. Find the RPI's serial number.

    - Option 1: from terminal

            (python3.2.3)pi@raspberrypi:/boot$ cat /proc/cpuinfo
            processor       : 0
            model name      : ARMv6-compatible processor rev 7 (v6l)
            BogoMIPS        : 2.00
            Features        : half thumb fastmult vfp edsp java tls
            CPU implementer : 0x41
            CPU architecture: 7
            CPU variant     : 0x0
            CPU part        : 0xb76
            CPU revision    : 7

            Hardware        : BCM2708
            Revision        : 000e
            Serial          : 0000000012345678   <--- this is the serial number


    - Option 2: from Kodi / OpenELEC / LibreELEC

        OpenElec > Settings > Hardware > get serial number of RPi


2. Go to <http://www.raspberrypi.com/mpeg-2-license-key/>  and give it the serial # and pay $5. They'll email you a license key. 

    - The license is 8 hex characters of the form `0x31415927`. 
    - It is specific to your Pi's serial number, so you can't transfer it between RPis.

3. Plug the Pi's SD card into a normal computer, open the file `config.txt`, change the line

        # decode_MPG2=0x00000000

    to have no `#` and have your license key:

        decode_MPG2=0x31415927

4. Plug the SD card back into the Pi and now MPG video should play smoothly.


Without MPG license, one of the 4 CPU cores runs close to 100%:

![](Images/without-mpeg-decoder.png)

(note the `dc:ff-mpeg2video`, which apparently is the software-based codec)

With the MPG license, the CPU load on all cores is less than 5%:

![](Images/with-mpeg-decoder.png)

(note the new codec in use: `dc:omx-mpeg2`)



## Key generation for `ssh`

From the computer that you'll `ssh` into your RPi from, generate a ssh keypair:

    ssh-keygen -t rsa

    Generating public/private rsa key pair.
    Enter file in which to save the key (/Users/justin/.ssh/id_rsa): /Users/justin/.ssh/rpi-2018
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in /Users/justin/.ssh/rpi-2018.
    Your public key has been saved in /Users/justin/.ssh/rpi-2018.pub.

Copy it onto the SD card:

    cp -a ~/.ssh/rpi-2018.pub /Volumes/boot/

## Power up with keyboard, mouse, monitor

Should see rainbow screen:

![](Images/rpi-rainbow-boot.png)

# 2. configure linux (AS ROOT)

**Important**: do not give network access yet.

## `raspi-config`

Start the `raspi-config` configuration tool:

    $ sudo raspi-config

Use arrow keys & ENTER to move around.

![](Images/raspi-config.png)

1. Do NOT **Change User Password** yet, the keyboard layout isn't set up. We will come back to it after setting locale & keyboard.
2. Network Options
    1. Hostname: `raspberrypi3`
3. Boot Options
    1. Desktop / CLI: "Desktop GUI, requiring user to login"
    3. Splash Screen: No
4. Localisation Options
    1. Change Locale
        - Uncheck `en_GB`
        - Check `en_US ISO-8859-1`, `en_US.ISO-8859-15 ISO-8859-15`, and `en_US.UTF-8 UTF-8`   
        - Default locale: `en_US`
    2. Change Timezone
        - Geographic area: `US`
        - Time zone: Pacific Ocean
    3. Change Keyboard Layout
        - Generic 105-key (Intl) PC
        - Keyboard layout: English (US)
        - Key to function as AltGr: "The default for the keyboard layout"
        - Compose key: No compose key
        - Use Control+Alt+Backspace to terminate the X server? Yes.
    4. Change Wi-fi Country: "US United States"

**Important**: "Finish", "Reboot now"

*(reboots)*

*(Default Raspbian user / password is 'pi / raspberry')*

- Verify: desktop comes up & asks for a password before logging in
- Verify `$ hostname` returns `raspberrypi3` 
- Verify locale:

        pi@raspberrypi3:~ $ locale -a
        C
        C.UTF-8
        en_US
        en_US.iso88591
        en_US.iso885915
        en_US.utf8
        POSIX

- Verify correct time and timezone:

        pi@raspberrypi3:~ $ date
        Sun Jun 10 11:46:24 PDT 2018    

- Verify keyboard layout & ctrl+alt+bksp kills X server:

        pi@raspberrypi3:~ $ cat /etc/default/keyboard
        # KEYBOARD CONFIGURATION FILE

        # Consult the keyboard(5) manual page.

        XKBMODEL="pc105"   
        XKBLAYOUT="us"     
        XKBVARIANT=""
        XKBOPTIONS="terminate:ctrl_alt_bksp"  

        BACKSPACE="guess"    

## Record RPi serial number

    $ cat /proc/cpuinfo
    processor       : 0
    model name      : ARMv6-compatible processor rev 7 (v6l)
    BogoMIPS        : 2.00
    Features        : half thumb fastmult vfp edsp java tls
    CPU implementer : 0x41
    CPU architecture: 7
    CPU variant     : 0x0
    CPU part        : 0xb76
    CPU revision    : 7

    Hardware        : BCM2708
    Revision        : 000e
    Serial          : 0000000012345678   <--- this is the serial number

## Change passwords

    $ sudo passwd root
    $ sudo passwd pi


## Require pw when sudo

    $ sudo visudo

- Replace

        Defaults        env_reset

    with

        # store root pw for 3 mins:
        Defaults        env_reset,timestamp_timeout=3  

- Change

        root  ALL=(ALL:ALL) ALL

    to

        root  ALL=(ALL:ALL) ALL      # ask for pw when sudo  
        pi    ALL=(ALL) PASSWD: ALL  # ask for pw when sudo

- Remove the lines

        # See sudoers(5) for more information on "#include" directives:
        #includedir /etc/sudoers.d

- In the nano text editor, Ctrl-o to save, Ctrl-x to quit

- Remove contents of the fancy `sudoers.d` dir:

        $ sudo rm /etc/sudoers.d/*


**Verify**: Log out & back in, try to `sudo ls`, it should ask for a pw. Give it your login pw for `pi` user, it should work (perform `ls`). Do `sudo ls` again, it shouldn't ask for pw. In 3 mins, the 'no pw' grace period expires and `sudo ls` should ask for pw again.

## Keyboard configuration

Next we make `capslock` act as a `control` key, and set the key repeat nice and fast.

It's important to note that many keyboard config settings that people suggest only work in a graphical X session, and do not work at a raw terminal session like the terminal you get with `ctrl-alt-F1` (do `ctrl-alt-F7` to get back to your window session).

### Capslock to ctrl

In `/etc/default/keyboard`, change `XKBOPTIONS`:

    $ sudo nano /etc/default/keyboard

    XKBOPTIONS="terminate:ctrl_alt_bksp,ctrl:nocaps"

Reboot.

*Note: This command should apply this new keyboard setting, but it didn't work for me:*

    invoke-rc.d keyboard-setup start

Verify: in the window manager, in a terminal window, `capslock-p` moves to the previous command.

### key-repeat

Put in the global xinitrc so it works in any X session:

    sudo nano /etc/X11/xinit/xinitrc

At the bottom, put
    
    xset r rate 130 80

This means: After holding down a key for 130 ms, the key will be repeated at a rate of 80 chars per sec.

Reboot.

*Note: This command should apply this new keyboard setting, but it didn't work for me:*

    invoke-rc.d keyboard-setup start

- If not using X, not sure how to set key repeat.

If you ctrl-alt-F1 to get to a non-X terminal, the `kbdrate` command is how you change the key repeat:

    GLOBAL_BASHRC="/etc/bash.bashrc"
    echo "kbdrate -r $KEYBD_REPEAT_CHARS_PER_SEC -d $KEYBD_DELAY_UNTIL_REPEAT_MS " >> $GLOBAL_BASHRC


Verify: at a terminal window, see if key repeat is nice and fast.

## bashrc

Add to `/etc/bash.bashrc` (the global bashrc file):

    alias ll="ls -lstra" 
    alias emacs="emacs -nw" 
    export HISTTIMEFORMAT="%F %T  " 
    export EDITOR="emacs -nw" 

**Verify:**

- as root user
    - `ll` works
    - `history` shows timestamps
    - `emacs` opens in a terminal, not a X window
- test again for non-root user.

## sshd

Earlier in this tutorial, you created a keypair and copied it to the RPi's SD card.

Now, copy it to the correct place on the RPi:

    $ cat /home/pi/boot/rpi-2018.pub >> /home/pi/.ssh/authorized_keys

Modify these values in `/etc/ssh/sshd_config`:

    Port 1234
    PubkeyAuthentication yes
    ChallengeResponseAuthentication no
    PasswordAuthentication no
    UsePAM no
    PrintMotd yes
    PermitRootLogin forced-commands-only


Enable the ssh daemon:

    update-rc.d ssh enable 
    invoke-rc.d ssh start

or

    invoke-rc.d ssh restart

if it was already running.    

Now you should be able to ssh into the RPi with

    ssh -i ~/.ssh/rpi-2018.pub -p 1234 pi@XXX

where the IP address `XXX` is found with

    ifconfig | grep inet 

(Optional:) Add this entry in `~/.ssh/config` on the machine you're sshing in from:

        Host rpi3
            HostName 192.168.11.XXX
            Port 1234  (or just 22 if you left it default)
            User pi
            IdentityFile ~/.ssh/rpi-2018

Then the command to ssh into the RPi is simply

    $ ssh rpi3


**Verify:**

- no password-based login; only public-key login
- motd prints 
- pw-based login fails 
- root pw-based login fails 
- user and root can log in w ssh key"
- use non-standard port
- set sshd to start at boot

## root crontab

(Optional) If you will be leaving the RPi on all the time, consider restarting it once a day to kill zombie processes:

Run `crontab -e` (as root) and add:

    # Reboot once a day, just to kill off weird zombie processes 
    30 3 * * * reboot


# 3. Update software (AS ROOT)
## plug in internet
## update

    apt-get update
    apt-get dist-upgrade -y

## firewall

    apt-get install -y ufw
    ufw enable
    ufw default deny incoming
    ufw allow from 192.168.11.0/24 to any port 1234 proto tcp # ssh only on local network
    ufw reload
    ufw status

## install packages

    apt-get install -y mlocate emacs ffmpeg logwatch exim4-daemon-light mutt
    updatedb  # so 'locate' works
    apt-get autoremove -y

### exim4 & mutt

It's usefult to have the local mail system set up:

- ouptut from cronjobs get mailed to your local user; this helps debug `cron`.
- the `logwatch` package summarizes your system logs every day to detect mischief (like `ssh` attempts and users `sudo`ing) and mails them to your local user.

Configure the mail system:

    dpkg-reconfigure exim4-config

You can accept all defaults except 'what user should receive mail for 'postmaster' or 'root'' ? I said 'pi'.    

Test the mail system (as non-root user):

    echo "hello world, this message should appear in pi's local mail box." | mail -s "test message sent at $(date +"%Y_%m_%d_%H_%M_%S")" pi

In `mutt`, you should see that message appear.



# 4. Configure a user

### user's bashrc

Add to `/home/pi/.bashrc`:

    alias ll="ls -lstra" 
    alias emacs="emacs -nw" 
    export HISTTIMEFORMAT="%F %T  " 
    export EDITOR="emacs -nw" 

### python 

    pip install --upgrade pip
    pip install virtualenv

**As non-root user,** Set up virtualenv and python packages.

    cd /home/pi
    mkdir .virtualenv
    cd .virtualenv
    virtualenv -p python2 python2
    virtualenv -p python3 python3

    echo "Enabling Python 3..."
    source /home/pi/.virtualenv/python3/bin/activate

    echo "Now using this python:"
    which python
    ls -l `which python`
    python --version

    echo "This is python2:"
    which python2
    ls -l `which python2`
    python2 --version

    echo "This is python3:"
    which python3
    ls -l `which python3`
    python3 --version

Append virtualenv cmds to `/home/pi/.bashrc`:

    alias switchpy2="source ~/.virtualenv/python2/bin/activate"
    alias switchpy3="source ~/.virtualenv/python3/bin/activate"
    # Turn on python3:
    source ~/.virtualenv/python3/bin/activate  # Warning: Must come after path-mangling lines like export PATH=usr/local/bin:$PATH"

**NOTE: How ensure python 3 set up right?**


### git 
#### set username & email
#### bash prompt shows git branch etc
#### aliases
##### co
##### st
##### lol
#### colors


# 5. For headless web-browser driving (Selenium)

## install

    apt-get install -y xvfb firefox-esr 

    pip3 install pyvirtualdisplay bs4 selenium matplotlib

Check versions:

    echo "Versions:"
    echo "py3 Selenium:"
    python3 -c "import selenium ; print(selenium.__version__)"
    echo "py3 Matplotlib:"
    python3 -c "import matplotlib ; print(matplotlib.__version__)"

*(why need `firefox-esr`? Why built-in browser not good enough?)*

### geckodriver

Geckodriver is middleware that allows Selenium to drive Firefox.

Note: You may need to use a different combination of Geckodriver, Firefox, and Selenium. See this thread:

<https://raspberrypi.stackexchange.com/questions/75409/running-selenium-firefox-geckodriver-on-raspberry-pi-2-model-b>

"""
I got a working configuration:

- bought a $35 Raspberry Pi 3 from Adafruit
    - Ships with Python 3.5.3
- lscpu shows it runs ARM7 cpu architecture (required by geckodriver)
- Using OS: Raspbian based on Debian Stretch (Sep 7, 2017, linux kernel v4.9)
- Firefox 52.5.0 (`sudo apt-get install firefox-esr`)
- geckodriver v0.17.0 for ARM7
- Note: Don't use the latest geckodriver -- you need to pick the one that matches your version of Firefox. This is hard because the geckodriver release notes aren't consistent about saying which version of Firefox and Selenium they're compatible with.
"""

        cd /home/pi/Downloads
        mkdir geckodriver
        cd geckodriver
        GECKODRIVER_URL=https://github.com/mozilla/geckodriver/releases/download/v0.17.0/geckodriver-v0.17.0-arm7hf.tar.gz
        wget $GECKODRIVER_URL
        tar xzvf $(basename $GECKODRIVER_URL)
        cp -a geckodriver /usr/local/bin/
        geckodriver --version


## Test headless (xvfb) selenium installation

If get error 

    selenium.common.exceptions.WebDriverException: Message: connection refused

this may mean you haven't gotten past initial firefox 'import your bookmarks' screen.
Must somehow start firefox to click past the "import from chromium" screen.
(Can do `ssh -X` and `$ firefox &`)


    cd /home/pi
    source /home/pi/.virtualenv/python3/bin/activate

Check versions:

    uname -a
    which firefox
    firefox --version
    which python
    python --version
    which geckodriver
    geckodriver --version
    which xvfb-run

Test webdriver:

    xvfb-run python -c "import time; import selenium; print('Selenium: ' + selenium.__version__); from selenium import webdriver; d = webdriver.Firefox(); d.get('http://www.google.com'); print(d.page_source); time.sleep(5); d.quit()"

This should print the HTML source of google.com.

