# SecurityOnion-ISO-Customization
Security Onion ISO Customization Process
In this repo, we will present a method (one of many) to customize the Security Onion 2.3 ISO.  The reason for this is to bypass certain options and apply new options to the kickstart script (ks.cfg) in order to automate the process of installing Security Onion 2.3 in special circumstances. The circumstances in this case are deploying SO 2.3 to multiple ESXi systems without user intervention.
## Setup Security Onion ISO for custom configuration
The first step to creating a custom SO ISO is to download the ISO to a system that can create an ISO.  In this case, we use either a Ubuntu system or CentOS system.
The current version of SO is 2.3.30.
On a Ubuntu system ensure the following packages are installed:
<pre><code>sudo apt install isohybrid
sudo apt install syslinux-utils
sudo apt install isolinux
</code></pre>
On a CentOS system ensure the following packages are installed:
<pre><code>sudo yum install mkisofs
sudo yum install syslinux
</code></pre>
Copy the Security Onion 2.3.x ISO to your Linux platform.
Mount the ISO.
Create a directory to which you will copy the ISO files.
Copy all files and directories to the newly created directory.
<pre><code>sudo mount -o loop securityonion-2.3.30.iso /mnt
sudo mkdir /tmp/seconionCustom
sudo cp -Rvp /mnt/* /tmp/seconionCustom/
cd /tmp/seconionCustom
</code></pre>
## Make Security Onion ISO Configuration Changes
We are primarily concerned with two files: isolinux.cfg and ks.cfg
### isolinux.cfg changes
The isolinux.cfg file is the boot menu that allows the user to select how they want to boot the system.  Since we are installing to ESXi using the ISO mounted on a CDROM, we want to make changes to ensure it is going to read our custom ks.cfg file from the CDROM.

Change the following
<pre><code>
label linux
  menu label ^Install Security Onion 2.3.30
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 ks=file:///ks.cfg quiet

label linux
  menu label ^Install Security Onion 2.3.30 in basic graphics mode
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 ks=file:///ks.cfg nomodeset quiet

label check
  menu label Test this ^media & install Security Onion 2.3.30
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check ks=file:///ks.cfg quiet
</code></pre>
Changes to make are lines related to the *ks* option.  Basically changing **ks=file:///ks.cfg** to **ks=cdrom:/ks.cfg**.
<pre><code>
label linux
  menu label ^Install Security Onion 2.3.30
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 ks=cdrom:/ks.cfg

label linux
  menu label ^Install Security Onion 2.3.30 in basic graphics mode
  menu default
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 ks=cdrom:/ks.cfg nomodeset

label check
  menu label Test this ^media & install Security Onion 2.3.30
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 rd.live.check ks=cdrom:/ks.cfg
</code></pre>
The SO install also has a boot message that waits for the user to press ENTER before continuing.  We want to comment out that line in the isolinux.cfg file, also.
<pre><code>
#display boot.msg
</code></pre>
Save the file.  Next we want to customize the kickstart script.
### ks.cfg changes
The following is the current ks.cfg file installed on the ISO
<pre><code>
# Set the firewall to allow SSH
firewall --enabled --port=22:tcp

# Install Mode
install

# Lock the root account
rootpw --lock

# Force this to use the shadow file
auth --useshadow --passalgo=sha512

# Install via Text Mode
text

# Disable firstboot. We don't have a GUI
firstboot --disable

# Set the keyboard
keyboard us

# Set the language
lang en_US

# Turn on SELinux
selinux --enforcing

# No X Windows please
skipx

# Installation logging level
logging --level=info

# Reboot after install
reboot --eject

# System timezone to UTC
timezone --isUtc Etc/UTC

#%include /tmp/uefi
%include /tmp/part-include

%pre

#!/bin/sh
exec < /dev/tty6 > /dev/tty6

# Switch to the tty so we can type stuff
chvt 6

# Get the megarams
mem=$(($(free -m | grep Mem | awk '{print $2}')+2000))

# Set drives to 0 for now
NUMDRIVES=0

# Block device directory
DIR="/sys/block"

# Minimum drive size in GIGABYTES
MINSIZE=99

# Set the root drive to blank for now
ROOTDRIVE=""

if [ -d /sys/firmware/efi ]; then
  is_uefi=true
fi

TEMPDRIVE=/tmp/part-include

# Ask some basic questions

for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
  if [ -d $DIR/$DEV ]; then

    # Find removeable devices so we don't install on them
    REMOVABLE=$(cat $DIR/$DEV/removable)

    if (( $REMOVABLE == 0 )); then
      NUMDRIVES=$((NUMDRIVES+1))
      SIZE=$(cat $DIR/$DEV/size)
      GB=$(($SIZE/2**21))
    fi
  fi
done

# If there is a single drive move forward
if [ $NUMDRIVES -lt 2 ]; then
  for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
    if [ -d $DIR/$DEV ]; then
      REMOVABLE=$(cat $DIR/$DEV/removable)
      if (( $REMOVABLE == 0 )); then
        SIZE=$(cat $DIR/$DEV/size)
        GB=$(($SIZE/2**21))
        if [ $GB -gt $MINSIZE ]; then
          ROOTDRIVE=$DEV
        else
          echo "Not enough space to install Security Onion. You need at least $MINSIZE GB to proceed"
          read drivetoosmall
        fi
      fi
    fi
  done

  ROOTSIZE=$(($GB/3))

  # Set the volume size to 300GB if it's larger than 300GB
  if [ $ROOTSIZE -gt 300 ]; then
    ROOTPART=300
  else
  # If there isn't at least 300GB set it to what is there.
    ROOTPART=$ROOTSIZE
  fi

# Determine if we need to use gpt
  if [ $GB -gt 1900 ]; then
    parted -s /dev/$ROOTDRIVE mklabel gpt
  fi
  echo 'zerombr' > $TEMPDRIVE
  echo -e "clearpart --all --drives=$ROOTDRIVE --initlabel" >> $TEMPDRIVE
  if [ ! $is_uefi ]; then
  echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part biosboot --fstype=biosboot --size=1 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part /boot --asprimary --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  fi
  if [ $is_uefi ]; then
  echo -e "part /boot/efi --asprimary --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part /boot --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  fi
  echo -e "part pv.1 --size 1 --grow --fstype=xfs --ondrive=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "volgroup system pv.1" >> $TEMPDRIVE
  echo -e "logvol swap --vgname system --size=8192 --name=swap" >> $TEMPDRIVE
  echo -e "logvol /tmp --fstype xfs --vgname system --size=2000 --name=tmp --fsoptions=\"nodev,nosuid,noexec\"" >> $TEMPDRIVE
  echo -e "logvol / --fstype xfs --vgname system --size=$(($ROOTPART*1000)) --name=root" >> $TEMPDRIVE
  echo -e "logvol /nsm --fstype xfs --vgname system --grow --size=1 --name=nsm" >> $TEMPDRIVE

else
  echo "Multiple drives detected.  Let's answer some questions."

  #If there's more than one drive we need to make some choices
  NUMDRIVES=0
  while [ -z $ROOTDRIVE ]
  do
    echo -e "Device\tSize"
    for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd; do
      if [ -d $DIR/$DEV ]; then
        REMOVABLE=$(cat $DIR/$DEV/removable)
        if (( $REMOVABLE == 0 )); then
          NUMDRIVES=$((NUMDRIVES+1))
          SIZE=$(cat $DIR/$DEV/size)
          GB=$(($SIZE/2**21))
          echo -e "$DEV\t$(($SIZE/2**21))GB"
        fi
      fi
    done

  echo "There are $NUMDRIVES available disk(s)"

    echo "Which device would you like to use as the operating system filesystem (must be larger than $MINSIZE GB):"
    read rootchoice
    if [ -d $DIR/$rootchoice ]; then
      REMOVABLE=$(cat $DIR/$rootchoice/removable)
      if (( $REMOVABLE == 0 )); then
        SIZE=$(cat $DIR/$rootchoice/size)
        GB=$(($SIZE/2**21))
        if [ $GB -gt $MINSIZE ]; then
          echo -e "$rootchoice\t$(($SIZE/2**21))GB - OS Drive"
          ROOTDRIVE=$rootchoice
        else
          echo "Available volume does not meet size requirements.  Please provide a volume greater than $MINSIZE GB"
          NUMDRIVES=0
        fi
      else
        echo "That is a removable drive. Please provide a device name for a fixed disk"
        NUMDRIVES=0
      fi
    else
      echo "That device does not exist.  Please pick one of the above listed devices"
      NUMDRIVES=0
    fi
  done
  SAMEDRIVE=""
  NSMDRIVE=""
  while [ -z $SAMEDRIVE ]
  do
    echo "Would you like to use the same device for NSM storage? (yes/no)"
    read SAMEDRIVE
    if [ "$SAMEDRIVE" == "yes" ]; then
      NSMDRIVE=$ROOTDRIVE
      echo -e "$rootchoice\t$(($SIZE/2**21))GB - NSM Drive"
    elif [ "$SAMEDRIVE" == "no" ]; then
      NUMDRIVES=0
      while [ -z $NSMDRIVE ]
      do
        echo -e "Device\tSize"
        for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
          if [ -d $DIR/$DEV ]; then
            REMOVABLE=$(cat $DIR/$DEV/removable)
            if (( $REMOVABLE == 0 )); then
              NUMDRIVES=$((NUMDRIVES+1))
              SIZE=$(cat $DIR/$DEV/size)
              GB=$(($SIZE/2**21))
              if [ "$DEV" != "$ROOTDRIVE" ]; then
                echo -e "$DEV\t$(($SIZE/2**21))GB"
              fi
            fi
          fi
        done
        echo "Which device would you like to use for NSM storage:"
        read whichnsm
        if [ -d $DIR/$whichnsm ]; then
          REMOVABLE=$(cat $DIR/$whichnsm/removable)
          if (( $REMOVABLE == 0 )); then
            echo -e "$whichnsm\t$(($SIZE/2**21))GB - NSM Drive"
            NSMDRIVE=$whichnsm
            PCAPSIZE=$(cat $DIR/$NSMDRIVE/size)
            PCAPGB=$(($PCAPSIZE/2**21))
          else
            echo "That is a removable drive. Please provide a device name for a fixed disk"
            NUMDRIVES=0
          fi
        else
          echo "That device does not exist.  Please pick one of the above listed devices"
          NUMDRIVES=0
        fi
      done
    else
      SAMEDRIVE=""
    fi
  done
  ROOTRAWSIZE=$(cat $DIR/$ROOTDRIVE/size)
  ROOTGB=$(($ROOTRAWSIZE/2**21))
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    ROOTSIZE=$(($ROOTGB/3))
    if [ $ROOTSIZE -gt 300 ]; then
      ROOTPART=300
    else
      ROOTPART=$ROOTSIZE
    fi
  else
    ROOTPART=$ROOTGB
  fi
  if [ $ROOTGB -gt 1900 ]; then
    parted -s /dev/$ROOTDRIVE mklabel gpt
  fi
  echo 'zerombr' > $TEMPDRIVE
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    echo -e "clearpart --all --drives=$ROOTDRIVE" >> $TEMPDRIVE
  else
    echo -e "clearpart --all --drives=$ROOTDRIVE,$NSMDRIVE --initlabel" >> $TEMPDRIVE
  fi
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    if [ ! $is_uefi ]; then
      echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part biosboot --fstype=biosboot --size=1" >> $TEMPDRIVE
      echo -e "part /boot --asprimary --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    else
      echo -e "part /boot/efi --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part /boot --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    fi
  else
    if [ ! $is_uefi ]; then
      echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE,$NSMDRIVE" >> $TEMPDRIVE
      echo -e "part biosboot --fstype=biosboot --size=1" >> $TEMPDRIVE
      echo -e "part /boot --asprimary --fstype=ext4 --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    else
      echo -e "part /boot/efi --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part /boot --fstype=ext4 --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    fi
  fi
  echo -e "part pv.1 --size 1 --grow --fstype=xfs --ondrive=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "volgroup system pv.1" >> $TEMPDRIVE
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    echo -e "logvol / --fstype xfs --vgname system --size=$(($ROOTPART*1000)) --name=root" >> $TEMPDRIVE
    echo -e "logvol /nsm --fstype xfs --vgname system --grow --size=1 --name=nsm" >> $TEMPDRIVE
  else
    if [ $PCAPGB -gt 1900 ]; then
      parted -s /dev/$NSMDRIVE mklabel gpt
    fi
    echo -e "part pv.2 --size 1 --grow --fstype=xfs --ondrive=$NSMDRIVE" >> $TEMPDRIVE
    echo -e "volgroup nsm pv.2" >> $TEMPDRIVE
    echo -e "logvol /nsm --fstype xfs --vgname nsm --grow --size=1 --name=nsm" >> $TEMPDRIVE
    echo -e "logvol / --fstype xfs --vgname system --grow --size=1 --name=root" >> $TEMPDRIVE
  fi
fi

manufacturer=$(dmidecode -s system-manufacturer)
family=$(dmidecode -s system-family)
product=$(dmidecode -s system-product-name)

INSTALL=no
PWMATCH=no

if [[ "$manufacturer" == "Security Onion Solutions" && "$family" == "Automated" ]]; then
  INSTALL=yes
  PWMATCH=yes
  SOUSER=onion
  echo "PRODUCT=$product" >> /tmp/variables.txt
  echo "SOUSER=$SOUSER" >> /tmp/variables.txt
  echo "PASSWORD=automation" >> /tmp/variables.txt
fi

while [[ "$INSTALL" != "yes" ]]; do
  clear
  echo "###########################################"
  echo "##          ** W A R N I N G **          ##"
  echo "##    _______________________________    ##"
  echo "##                                       ##"
  echo "##  Installing the Security Onion ISO    ##"
  echo "## on this device will DESTROY ALL DATA  ##"
  echo "##            and partitions!            ##"
  echo "##                                       ##"
  echo "##      ** ALL DATA WILL BE LOST **      ##"
  echo "###########################################"
  echo "Do you wish to continue? (Type the entire word 'yes' to proceed.) "
  read INSTALL
done

userPattern="^[a-z_]([a-z0-9_-]{0,31}|[a-z0-9_-]{0,30}\$)$"
firstAttempt=1
invalidUsers=("root bin daemon adm lp sync shutdown halt mail operator games ftp nobody systemd-network dbus polkitd sshd postfix chrony socore soremote ntp tcpdump elasticsearch stenographer suricata zeek curator kratos kibana elastalert ossecm ossecr ossec logstash")
while [[ ! $SOUSER =~ $userPattern ]]; do
  echo ""
  if [ $firstAttempt -eq 1 ]; then
    echo "A new administrative user will be created. This user will be used for setting up and administering Security Onion."
  else
    echo "The provided username is not valid, try again."
  fi
  echo ""
  echo -n "Enter an administrative username: "
  read SOUSER
  firstAttempt=0
  if [[ " ${invalidUsers[@]} " =~ " ${SOUSER} " ]]; then
    SOUSER=
  fi
done

while [ $PWMATCH != yes ]; do
  echo ""
  echo "Let's set a password for the $SOUSER user:"
  echo ""
  echo -n "Enter a password: "
  PASSWORD1=''
  while IFS= read -r -s -n1 char; do
    [[ -z $char ]] && { printf '\n'; break; } # ENTER pressed; output \n and break.
    if [[ $char == $'\x7f' ]]; then # backspace was pressed
      # Remove last char from output variable.
      [[ -n $PASSWORD1 ]] && PASSWORD1=${PASSWORD1%?}
      # Erase '*' to the left.
      printf '\b \b'
    else
      # Add typed char to output variable.
      PASSWORD1+=$char
      # Print '*' in its stead.
      printf '*'
    fi
  done

  echo -n "Re-enter the password: "
  PASSWORD2=''
  while IFS= read -r -s -n1 char; do
    [[ -z $char ]] && { printf '\n'; break; } # ENTER pressed; output \n and break.
    if [[ $char == $'\x7f' ]]; then # backspace was pressed
      # Remove last char from output variable.
      [[ -n $PASSWORD2 ]] && PASSWORD2=${PASSWORD2%?}
      # Erase '*' to the left.
      printf '\b \b'
    else
      # Add typed char to output variable.
      PASSWORD2+=$char
      # Print '*' in its stead.
      printf '*'
    fi
  done

  if [ $PASSWORD1 == $PASSWORD2 ]; then
    echo PASSWORD=$PASSWORD1 >> /tmp/variables.txt
    echo SOUSER=$SOUSER >> /tmp/variables.txt
    PWMATCH=yes
  else
    echo "Passwords don't match. Press enter to try again. "
    read -p
  fi
done
exec < /dev/tty1 > /dev/tty1
chvt 1
%end

%post --nochroot
cp /tmp/variables.txt /mnt/sysimage/tmp/variables.txt
mkdir /mnt/sysimage/root/SecurityOnion
mkdir -p /mnt/sysimage/nsm/docker-registry/docker
mkdir -p /mnt/sysimage/nsm/repo
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/SecurityOnion/* /mnt/sysimage/root/SecurityOnion/
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/docker/* /mnt/sysimage/nsm/docker-registry/docker/
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/Packages/* /mnt/sysimage/nsm/repo/
chmod +x /mnt/sysimage/root/SecurityOnion/setup/so-setup

%end

%post
source /tmp/variables.txt
rm -f /tmp/variables.txt

useradd $SOUSER
cp -Rv /root/SecurityOnion /home/$SOUSER/
chown -R $SOUSER:$SOUSER /home/$SOUSER/SecurityOnion/
chmod +x /home/$SOUSER/SecurityOnion/so-setup
echo $SOUSER:$PASSWORD | chpasswd --crypt-method=SHA512
echo "$SOUSER   ALL=(ALL)       ALL" >> /etc/sudoers
echo "$SOUSER   ALL=(ALL) NOPASSWD: /home/$SOUSER/SecurityOnion/setup/so-setup" >> /etc/sudoers
echo "sudo /home/$SOUSER/SecurityOnion/setup/so-setup iso" >> /home/$SOUSER/.bash_profile

if [ ! -z $PASSWORD ]; then
  echo $SOUSER:$PASSWORD | chpasswd --crypt-method=SHA512
fi

if [ ! -z $PRODUCT ]; then
  echo "@reboot sudo /home/$SOUSER/SecurityOnion/setup/so-setup iso $PRODUCT" | crontab -u $SOUSER -
fi

# SSHD Banner
touch /etc/ssh/sshd-banner
echo "##########################################" > /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner
echo "###                                    ###" >> /etc/ssh/sshd-banner
echo "###   UNAUTHORIZED ACCESS PROHIBITED   ###" >> /etc/ssh/sshd-banner
echo "###                                    ###" >> /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner

# Set the SSHD banner
echo "Banner /etc/ssh/sshd-banner" >> /etc/ssh/sshd_config

if [ -z $PRODUCT ]; then
  exec < /dev/tty6 > /dev/tty6
  chvt 6
  clear
  echo "Initial Install Complete. Press [Enter] to reboot!"
  read -p "Initial Install Complete. Press [Enter] to reboot!"
  exec < /dev/tty1 > /dev/tty1
  chvt 1
fi
%end

%packages --nobase
@core
%end
</code></pre>
Before we discuss changing the kickstart script, let's discuss our goals
1. Since we are using Ansible to install our SO virtual machines directly to ESXi, we need to add a line to ensure our network interfaces activate properly.
2. We want the SO install to be automatic with no user intervention.  We will need to comment out several sections of the kickstart script for this to happen.
3. We want to  make sure a preliminary username and password is implemented so the Ansible playbooks can make the necessary SSH communications

#### Goal 1
For goal 1, we will add the following lines to the ks.cfg script
<pre><code>
# Network information
network  --bootproto=dhcp --device=link --onboot=on --activate
</code></pre>

#### Goal 2
For goal 2, we need to comment out various lines that cause the install to wait for user input.  We don't want this to happen since it halts our autmated install. Note that the lines are already commented (preceeded by a hashtag).
<pre><code>
#while [[ "$INSTALL" != "yes" ]]; do
#  clear
#  echo "###########################################"
#  echo "##          ** W A R N I N G **          ##"
#  echo "##    _______________________________    ##"
#  echo "##                                       ##"
#  echo "##  Installing the Security Onion ISO    ##"
#  echo "## on this device will DESTROY ALL DATA  ##"
#  echo "##            and partitions!            ##"
#  echo "##                                       ##"
#  echo "##      ** ALL DATA WILL BE LOST **      ##"
#  echo "###########################################"
#  echo "Do you wish to continue? (Type the entire word 'yes' to proceed.) "
#  read INSTALL
#done

#userPattern="^[a-z_]([a-z0-9_-]{0,31}|[a-z0-9_-]{0,30}\$)$"
#firstAttempt=1
#invalidUsers=("root bin daemon adm lp sync shutdown halt mail operator games ftp nobody systemd-network dbus polkitd sshd postfix chrony socore soremote ntp tcpdump #elasticsearch stenographer suricata zeek curator kratos kibana elastalert ossecm ossecr ossec logstash")
#while [[ ! $SOUSER =~ $userPattern ]]; do
#  echo ""
#  if [ $firstAttempt -eq 1 ]; then
#    echo "A new administrative user will be created. This user will be used for setting up and administering Security Onion."
#  else
#    echo "The provided username is not valid, try again."
#  fi
#  echo ""
#  echo -n "Enter an administrative username: "
#  read SOUSER
#  firstAttempt=0
#  if [[ " ${invalidUsers[@]} " =~ " ${SOUSER} " ]]; then
#    SOUSER=
#  fi
#done

#while [ $PWMATCH != yes ]; do
# echo ""
#  echo "Let's set a password for the $SOUSER user:"
#  echo ""
#  echo -n "Enter a password: "
#  PASSWORD1=''
#  while IFS= read -r -s -n1 char; do
#    [[ -z $char ]] && { printf '\n'; break; } # ENTER pressed; output \n and break.
#    if [[ $char == $'\x7f' ]]; then # backspace was pressed
#      # Remove last char from output variable.
#      [[ -n $PASSWORD1 ]] && PASSWORD1=${PASSWORD1%?}
#      # Erase '*' to the left.
#      printf '\b \b'
#    else
#      # Add typed char to output variable.
#      PASSWORD1+=$char
#      # Print '*' in its stead.
#      printf '*'
#    fi
#  done

#  echo -n "Re-enter the password: "
#  PASSWORD2=''
#  while IFS= read -r -s -n1 char; do
#    [[ -z $char ]] && { printf '\n'; break; } # ENTER pressed; output \n and break.
#    if [[ $char == $'\x7f' ]]; then # backspace was pressed
#      # Remove last char from output variable.
#      [[ -n $PASSWORD2 ]] && PASSWORD2=${PASSWORD2%?}
#      # Erase '*' to the left.
#      printf '\b \b'
#    else
#      # Add typed char to output variable.
#      PASSWORD2+=$char
#      # Print '*' in its stead.
#      printf '*'
#    fi
#  done
</code></pre>
We also want to make sure that an automatic reboot occurs after the install is complete.  To ensure this happens, we will comment out a few lines at the end of the ks.cfg script and add a reboot command at the end of the script as follows.
<pre><code>
#  echo "Initial Install Complete. Press [Enter] to reboot!"
#  read -p "Initial Install Complete. Press [Enter] to reboot!"
  exec < /dev/tty1 > /dev/tty1
  chvt 1
fi
%end

%packages --nobase
@core
%end
reboot
</code></pre>
#### Goal 3
Finally we want to add a username and password to ensure that our Ansible palybooks can connect to the SO nodes. In the section below, we add the SOUSER, PASSWORD1, and PASSWORD2 lines after the section we previously commented out asking for a username and password input.
<pre><code>
...
#      PASSWORD2+=$char
#      # Print '*' in its stead.
#      printf '*'
#    fi
#  done
SOUSER="MYUSERNAME"
PASSWORD1="MYPASSWORD"
PASSWORD2="MYPASSWORD"
  if [ $PASSWORD1 == $PASSWORD2 ]; then
    echo PASSWORD=$PASSWORD1 >> /tmp/variables.txt
...
</code></pre>
Below is an example of the customized ks.cfg file. The sections we commented out earlier are completely removed from this example.
<pre><code>
# Set the firewall to allow SSH
firewall --enabled --port=22:tcp

# Install Mode
install

# Lock the root account
rootpw --lock

# Force this to use the shadow file
auth --useshadow --passalgo=sha512

# Install via Text Mode
text

# Disable firstboot. We don't have a GUI
firstboot --disable

# Set the keyboard
keyboard us

# Set the language
lang en_US

# Turn on SELinux
selinux --enforcing

# No X Windows please
skipx

# Installation logging level
logging --level=info

# Reboot after install
reboot --eject

# System timezone to UTC
timezone --isUtc Etc/UTC

#%include /tmp/uefi
%include /tmp/part-include

%pre

#!/bin/sh
exec < /dev/tty6 > /dev/tty6

# Switch to the tty so we can type stuff
chvt 6

# Get the megarams
mem=$(($(free -m | grep Mem | awk '{print $2}')+2000))

# Set drives to 0 for now
NUMDRIVES=0

# Block device directory
DIR="/sys/block"

# Minimum drive size in GIGABYTES
MINSIZE=99

# Set the root drive to blank for now
ROOTDRIVE=""

if [ -d /sys/firmware/efi ]; then
  is_uefi=true
fi

TEMPDRIVE=/tmp/part-include

# Ask some basic questions

# Network information
network  --bootproto=dhcp --device=link --onboot=on --activate

for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
  if [ -d $DIR/$DEV ]; then

    # Find removeable devices so we don't install on them
    REMOVABLE=$(cat $DIR/$DEV/removable)

    if (( $REMOVABLE == 0 )); then
      NUMDRIVES=$((NUMDRIVES+1))
      SIZE=$(cat $DIR/$DEV/size)
      GB=$(($SIZE/2**21))
    fi
  fi
done

# If there is a single drive move forward
if [ $NUMDRIVES -lt 2 ]; then
  for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
    if [ -d $DIR/$DEV ]; then
      REMOVABLE=$(cat $DIR/$DEV/removable)
      if (( $REMOVABLE == 0 )); then
        SIZE=$(cat $DIR/$DEV/size)
        GB=$(($SIZE/2**21))
        if [ $GB -gt $MINSIZE ]; then
          ROOTDRIVE=$DEV
        else
          echo "Not enough space to install Security Onion. You need at least $MINSIZE GB to proceed"
          read drivetoosmall
        fi
      fi
    fi
  done

  ROOTSIZE=$(($GB/3))

  # Set the volume size to 300GB if it's larger than 300GB
  if [ $ROOTSIZE -gt 300 ]; then
    ROOTPART=300
  else
  # If there isn't at least 300GB set it to what is there.
    ROOTPART=$ROOTSIZE
  fi

# Determine if we need to use gpt
#  if [ $GB -gt 1 ]; then
    parted -s /dev/$ROOTDRIVE mklabel gpt
#  fi
  echo 'zerombr' > $TEMPDRIVE
  echo -e "clearpart --all --drives=$ROOTDRIVE --initlabel" >> $TEMPDRIVE
  if [ ! $is_uefi ]; then
  echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part biosboot --fstype=biosboot --size=1 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part /boot --asprimary --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  fi
  if [ $is_uefi ]; then
  echo -e "part /boot/efi --asprimary --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "part /boot --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
  fi
  echo -e "part pv.1 --size 1 --grow --fstype=xfs --ondrive=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "volgroup system pv.1" >> $TEMPDRIVE
  echo -e "logvol swap --vgname system --size=8192 --name=swap" >> $TEMPDRIVE
  echo -e "logvol /tmp --fstype xfs --vgname system --size=2000 --name=tmp --fsoptions=\"nodev,nosuid,noexec\"" >> $TEMPDRIVE
  echo -e "logvol / --fstype xfs --vgname system --size=$(($ROOTPART*1000)) --name=root" >> $TEMPDRIVE
  echo -e "logvol /nsm --fstype xfs --vgname system --grow --size=1 --name=nsm" >> $TEMPDRIVE

else
  echo "Multiple drives detected.  Let's answer some questions."

  #If there's more than one drive we need to make some choices
  NUMDRIVES=0
  while [ -z $ROOTDRIVE ]
  do
    echo -e "Device\tSize"
    for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd; do
      if [ -d $DIR/$DEV ]; then
        REMOVABLE=$(cat $DIR/$DEV/removable)
        if (( $REMOVABLE == 0 )); then
          NUMDRIVES=$((NUMDRIVES+1))
          SIZE=$(cat $DIR/$DEV/size)
          GB=$(($SIZE/2**21))
          echo -e "$DEV\t$(($SIZE/2**21))GB"
        fi
      fi
    done

  echo "There are $NUMDRIVES available disk(s)"

    echo "Which device would you like to use as the operating system filesystem (must be larger than $MINSIZE GB):"
    read rootchoice
    if [ -d $DIR/$rootchoice ]; then
      REMOVABLE=$(cat $DIR/$rootchoice/removable)
      if (( $REMOVABLE == 0 )); then
        SIZE=$(cat $DIR/$rootchoice/size)
        GB=$(($SIZE/2**21))
        if [ $GB -gt $MINSIZE ]; then
          echo -e "$rootchoice\t$(($SIZE/2**21))GB - OS Drive"
          ROOTDRIVE=$rootchoice
        else
          echo "Available volume does not meet size requirements.  Please provide a volume greater than $MINSIZE GB"
          NUMDRIVES=0
        fi
      else
        echo "That is a removable drive. Please provide a device name for a fixed disk"
        NUMDRIVES=0
      fi
    else
      echo "That device does not exist.  Please pick one of the above listed devices"
      NUMDRIVES=0
    fi
  done
  SAMEDRIVE=""
  NSMDRIVE=""
  while [ -z $SAMEDRIVE ]
  do
    echo "Would you like to use the same device for NSM storage? (yes/no)"
    read SAMEDRIVE
    if [ "$SAMEDRIVE" == "yes" ]; then
      NSMDRIVE=$ROOTDRIVE
      echo -e "$rootchoice\t$(($SIZE/2**21))GB - NSM Drive"
    elif [ "$SAMEDRIVE" == "no" ]; then
      NUMDRIVES=0
      while [ -z $NSMDRIVE ]
      do
        echo -e "Device\tSize"
        for DEV in sda sdb sdc sdd hda hdb hdc hdd vda vdb vdc vdd nvme0n1 nvme1n1 nvme2n1 nvme3n1 xvda xvdb xvdc xvdd ; do
          if [ -d $DIR/$DEV ]; then
            REMOVABLE=$(cat $DIR/$DEV/removable)
            if (( $REMOVABLE == 0 )); then
              NUMDRIVES=$((NUMDRIVES+1))
              SIZE=$(cat $DIR/$DEV/size)
              GB=$(($SIZE/2**21))
              if [ "$DEV" != "$ROOTDRIVE" ]; then
                echo -e "$DEV\t$(($SIZE/2**21))GB"
              fi
            fi
          fi
        done
        echo "Which device would you like to use for NSM storage:"
        read whichnsm
        if [ -d $DIR/$whichnsm ]; then
          REMOVABLE=$(cat $DIR/$whichnsm/removable)
          if (( $REMOVABLE == 0 )); then
            echo -e "$whichnsm\t$(($SIZE/2**21))GB - NSM Drive"
            NSMDRIVE=$whichnsm
            PCAPSIZE=$(cat $DIR/$NSMDRIVE/size)
            PCAPGB=$(($PCAPSIZE/2**21))
          else
            echo "That is a removable drive. Please provide a device name for a fixed disk"
            NUMDRIVES=0
          fi
        else
          echo "That device does not exist.  Please pick one of the above listed devices"
          NUMDRIVES=0
        fi
      done
    else
      SAMEDRIVE=""
    fi
  done
  ROOTRAWSIZE=$(cat $DIR/$ROOTDRIVE/size)
  ROOTGB=$(($ROOTRAWSIZE/2**21))
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    ROOTSIZE=$(($ROOTGB/3))
    if [ $ROOTSIZE -gt 300 ]; then
      ROOTPART=300
    else
      ROOTPART=$ROOTSIZE
    fi
  else
    ROOTPART=$ROOTGB
  fi
#  if [ $ROOTGB -gt 1900 ]; then
    parted -s /dev/$ROOTDRIVE mklabel gpt
#  fi
  echo 'zerombr' > $TEMPDRIVE
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    echo -e "clearpart --all --drives=$ROOTDRIVE" >> $TEMPDRIVE
  else
    echo -e "clearpart --all --drives=$ROOTDRIVE,$NSMDRIVE --initlabel" >> $TEMPDRIVE
  fi
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    if [ ! $is_uefi ]; then
      echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part biosboot --fstype=biosboot --size=1" >> $TEMPDRIVE
      echo -e "part /boot --asprimary --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    else
      echo -e "part /boot/efi --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part /boot --fstype=xfs --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    fi
  else
    if [ ! $is_uefi ]; then
      echo -e "bootloader --location=mbr --driveorder=$ROOTDRIVE,$NSMDRIVE" >> $TEMPDRIVE
      echo -e "part biosboot --fstype=biosboot --size=1" >> $TEMPDRIVE
      echo -e "part /boot --asprimary --fstype=ext4 --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    else
      echo -e "part /boot/efi --fstype="fat32" --size=1024 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
      echo -e "part /boot --fstype=ext4 --size=500 --ondisk=$ROOTDRIVE" >> $TEMPDRIVE
    fi
  fi
  echo -e "part pv.1 --size 1 --grow --fstype=xfs --ondrive=$ROOTDRIVE" >> $TEMPDRIVE
  echo -e "volgroup system pv.1" >> $TEMPDRIVE
  if [ "$ROOTDRIVE" == "$NSMDRIVE" ]; then
    echo -e "logvol / --fstype xfs --vgname system --size=$(($ROOTPART*1000)) --name=root" >> $TEMPDRIVE
    echo -e "logvol /nsm --fstype xfs --vgname system --grow --size=1 --name=nsm" >> $TEMPDRIVE
  else
#    if [ $PCAPGB -gt 1900 ]; then
      parted -s /dev/$NSMDRIVE mklabel gpt
#    fi
    echo -e "part pv.2 --size 1 --grow --fstype=xfs --ondrive=$NSMDRIVE" >> $TEMPDRIVE
    echo -e "volgroup nsm pv.2" >> $TEMPDRIVE
    echo -e "logvol /nsm --fstype xfs --vgname nsm --grow --size=1 --name=nsm" >> $TEMPDRIVE
    echo -e "logvol / --fstype xfs --vgname system --grow --size=1 --name=root" >> $TEMPDRIVE
  fi
fi

manufacturer=$(dmidecode -s system-manufacturer)
family=$(dmidecode -s system-family)
product=$(dmidecode -s system-product-name)

INSTALL=no
PWMATCH=no

if [[ "$manufacturer" == "Security Onion Solutions" && "$family" == "Automated" ]]; then
  INSTALL=yes
  PWMATCH=yes
  SOUSER=onion
  echo "PRODUCT=$product" >> /tmp/variables.txt
  echo "SOUSER=$SOUSER" >> /tmp/variables.txt
  echo "PASSWORD=automation" >> /tmp/variables.txt
fi

SOUSER="MYUSERNAME"
PASSWORD1="MYPASSWORD"
PASSWORD2="MYPASSWORD"
  if [ $PASSWORD1 == $PASSWORD2 ]; then
    echo PASSWORD=$PASSWORD1 >> /tmp/variables.txt
    echo SOUSER=$SOUSER >> /tmp/variables.txt
    PWMATCH=yes
  else
    echo "Passwords don't match. Press enter to try again. "
    read -p
  fi
done
exec < /dev/tty1 > /dev/tty1
chvt 1
%end

%post --nochroot
cp /tmp/variables.txt /mnt/sysimage/tmp/variables.txt
mkdir /mnt/sysimage/root/SecurityOnion
mkdir -p /mnt/sysimage/nsm/docker-registry/docker
mkdir -p /mnt/sysimage/nsm/repo
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/SecurityOnion/* /mnt/sysimage/root/SecurityOnion/
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/docker/* /mnt/sysimage/nsm/docker-registry/docker/
rsync -avh --exclude 'TRANS.TBL' /run/install/repo/Packages/* /mnt/sysimage/nsm/repo/
chmod +x /mnt/sysimage/root/SecurityOnion/setup/so-setup

%end

%post
source /tmp/variables.txt
rm -f /tmp/variables.txt

useradd $SOUSER
cp -Rvp /root/SecurityOnion /home/$SOUSER/
chown -R $SOUSER:$SOUSER /home/$SOUSER/SecurityOnion/
chmod +x /home/$SOUSER/SecurityOnion/so-setup
chmod -R +x /home/$SOUSER/SecurityOnion/salt/common/tools/sbin/
echo $SOUSER:$PASSWORD | chpasswd --crypt-method=SHA512
echo "$SOUSER   ALL=(ALL)       ALL" >> /etc/sudoers
echo "$SOUSER   ALL=(ALL) NOPASSWD: /home/$SOUSER/SecurityOnion/setup/so-setup" >> /etc/sudoers
#echo "sudo /home/$SOUSER/SecurityOnion/setup/so-setup iso" >> /home/$SOUSER/.bash_profile

if [ ! -z $PASSWORD ]; then
  echo $SOUSER:$PASSWORD | chpasswd --crypt-method=SHA512
fi

#if [ ! -z $PRODUCT ]; then
#  echo "@reboot sudo /home/$SOUSER/SecurityOnion/setup/so-setup iso $PRODUCT" | crontab -u $SOUSER -
#fi

# fix onboot line so interface will come up.
sed -i 's/ONBOOT=no/ONBOOT=yes/g' /etc/sysconfig/network-scripts/ifcfg-ens192


# SSHD Banner
touch /etc/ssh/sshd-banner
echo "##########################################" > /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner
echo "###                                    ###" >> /etc/ssh/sshd-banner
echo "###   UNAUTHORIZED ACCESS PROHIBITED   ###" >> /etc/ssh/sshd-banner
echo "###                                    ###" >> /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner
echo "##########################################" >> /etc/ssh/sshd-banner

# Set the SSHD banner
echo "Banner /etc/ssh/sshd-banner" >> /etc/ssh/sshd_config

if [ -z $PRODUCT ]; then
  exec < /dev/tty6 > /dev/tty6
  chvt 6
  clear
  exec < /dev/tty1 > /dev/tty1
  chvt 1
fi
%end

%packages --nobase
@core
%end
reboot
</code></pre>
Save the ks.cfg file once customization is complete
## Compile the Custom ISO
From our custom ISO directory, we want to compile the new ISO. 
<pre><code>
mkisofs -o securityonionCustom-2.3.30.iso -allow-limited-size -b isolinux.bin -J -R -l -c boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table -eltorito-alt-boot -e images/efiboot.img -no-emul-boot -graft-points -joliet-long -R -V "CentOS 7 x86_64" .
</code></pre>
Finally, we make the ISO bootable and prepare it to be copied to our ESXi datastore.
<pre><code>
isohybrid --uefi securityonionCustom-2.3.30.iso
chmod 777 securityonionCustom-2.3.30.iso
</code></pre>
