# QEMU Cloud-Init Support
`Cloud-Init` is the package that handles early initialization of a virtual machine instance. When the VM starts for the first time, the `Cloud-Init` software inside the VM will apply different settings.

Many Linux distributions provide ready-to-use `Cloud-Init` images. While it may seem convenient to get such ready-to-use images, we will show how to prepare the images by yourself. The advantage is that you will know exactly what you have installed, and this helps you later to easily customize the image for your needs.

Once you have created such a `Cloud-Init` image it is recommended to convert it into a VM template. You just need to configure the network and maybe the ssh keys before you start the new VM.

# Create env variables

```sh
# Name of the Cloud image
IMAGE="noble-server-cloudimg-amd64.img"
# Directory where the VM images are located on the KVM host
LIBVIRT_DIR="/var/lib/libvirt/images"
# Name of the new VM created with the Cloud Image
VM_NAME="cloud-init01"
DOMAINNAME="kloud.lan"
USERNAME="daniel"
FULL_NAME="My full name"
# Change your password
PASSWORD="PASSWORD"
ENC_PASSWORD=$(openssl passwd -6 ${PASSWORD})
# Local SSH public key that will be copied on the new VM
PUB_KEY=$(cat ~/.ssh/id_ecdsa.pub)

# Networking parameters for new VM 
IPv4_CIDR="10.100.2.80/24"
GW="10.100.2.1"
DNS1="192.168.13.10"
DNS2="192.168.13.16"
DNS3="192.168.13.17"
SEARCH="kloud.lan"
# Local bridge where the ethernet intf will be connected for the new VM
BRIDGE="k8s2-bastion"

# Cloned of the new VM. Add many lines as needed.
# key: hostname of the cloned VM
# value: IPv4 address of the cloned VM
unset newVM
declare -A newVM
newVM[k3s1vrrp1]="10.30.100.101"
newVM[k3s1vrrp2]="10.30.100.102"
newVM[k3s1server1]="10.30.100.51"
newVM[k3s1server2]="10.30.100.52"
newVM[k3s1server3]="10.30.100.53"
newVM[k3s1agent1]="10.30.100.61"
newVM[k3s1agent2]="10.30.100.62"
newVM[k3s1agent3]="10.30.100.63"
newVM[k3s1etcd1]="10.30.100.71"
newVM[k3s1etcd2]="10.30.100.72"
newVM[k3s1etcd3]="10.30.100.73"

# Subnet info for the VMs
localSubnet="10.30.100.0"
subnetMask="24"
defaultGateway="10.30.100.1"
localBroadcast="10.30.100.255"

# The bridge for the cloned VM, in case the new Cloud Init VM is not in the same bridge
NEW_BRIDGE="vmnet30100"

########## NO NEED TO CHANGE ANYTHING PASSED THIS LINE ##########
# Remove CIDR (the subnet mask) to get only the IPv4
IPv4=$(echo ${IPv4_CIDR} |  sed "s/\/.*//g")

# Each VM is in it's own subdirectory
BASE_DIR=${VM_NAME}
```

## Create working directory
Create a local temporary directory for all the files needed to build the VM.

> [!NOTE]  
> All the commands are executed on the **KVM server** unless otherwise specified.

```sh
mkdir Cloud-Init
cd Cloud-Init
```

## Download the image
Download the `Cloud-Init` image and the SHA256 file. In this example, I'm using Ubuntu 24.04-1 LTS image. Feel free to use the image of your choice.

```sh
curl -LO https://cloud-images.ubuntu.com/noble/current/${IMAGE}
curl -LO https://cloud-images.ubuntu.com/noble/current/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

Output expected for the checksum:
```
noble-server-cloudimg-amd64.img: OK
```

## Images directory
On KVM with Ubuntu, the virtual machine's images are placed in `/var/lib/libvirt/images` by default. You can use your own but for this tutorial I'll use this one. Don't forget to setup the correct permission.

```sh
sudo cp ${IMAGE} ${LIBVIRT_DIR}/.
sudo chown -R libvirt-qemu:libvirt ${LIBVIRT_DIR}/${IMAGE}
```

# Create your configuration
In the next steps we need to create three configuration files. You can leave some configuration files empty.

- network-config
- meta-data
- user-data 

## Define our user data - Cloud config
Cloud-config is a YAML-based configuration type that tells cloud-init how to configure the virtual machine instance. Multiple different format types are supported by cloud-init. The first line starts with `#cloud-config`, which tells cloud-init what type of user data is in the config. For more information, see the [Cloud config examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html#yaml-examples) documentation describing different formats.

> [!IMPORTANT]  
> The file must be valid `YAML` syntax.

The data source `NoCloud` is a flexible datasource that can be used in multiple different ways. With `NoCloud`, the user can provide user data and metadata to the instance without running a network service (or even without having a network at all). We will use the `Local filesystem` to provide cloud-init configurations from the local filesystem, a labeled `vfat` or `iso9660` filesystem containing user data and metadata may be used. For this method to work, the filesystem volume must be labelled **CIDATA**.

These user data and metadata files are required as separate files at the same base URL:
```
/user-data
/meta-data
/network-config
```

> [!IMPORTANT]  
> All files must be present for it to be considered a valid seed ISO.

Create a `user-data` file for Cloud-Init. Check [this site](https://cloudinit.readthedocs.io/en/latest/explanation/format.html#user-data-formats) for all the 
https://cloudinit.readthedocs.io/en/latest/reference/examples.html#yaml-examples


```sh
cat <<EOF > user-data
#cloud-config

ssh_pwauth: true
users:
# This creates a full root user
- name: ${USERNAME}
  # gecos: The user name's real name, i.e. "Bob B. Smith"
  gecos: ${FULL_NAME}
  # groups:  Optional. Additional groups to add the user to. Defaults to none
  groups: adm,cdrom,dip,lxd,plugdev,sudo
  # lock_passwd: Defaults to true. Lock the password to disable password login
  lock_passwd: false
  plain_text_passwd: ${PASSWORD}
  # passwd: The hash of the password you want to use for this user.
  # passwd: ${ENC_PASSWORD}
  shell: /bin/bash
  # sudo: Defaults to none. Accepts a sudo rule string, a list of sudo rule
  sudo: ['ALL=(ALL) NOPASSWD:ALL']
  # ssh_authorized_keys: Optional. [list] Add keys to user's authorized keys file
  ssh_authorized_keys:
    - ${PUB_KEY}
    - ecdsa-sha2-nistp521 AAAAE2VjZHNhLXNoYTItbmlzdHA1MjEAAAAIbmlzdHA1MjEAAACFBAFdLXJm+pGkYdUTfzpmC8Zg/yYGTwVepcQu05CAvDRgjtjARqxNL7s2DjJCFy8PjE8W2ZjBKm+bCejFv8LflMflPADPtdO7RVVTe9DbNmr2qQQQZNxZ2dx5MVbqNYLdwLmbZ0Ke8KMLNZTFDC7y6d0XQ53V+gBRMXFWJOgkIM5vLbL1eg== daniel@MacBook-Dan.local

# configure pub key for ssh server
ssh_genkeytypes: ['ed25519']
allow_public_ssh_keys: true
ssh_deletekeys: false

# set timezone for VM
timezone: America/Montreal

# Install additional packages on first boot
packages:
  - bash-completion
  - iputils-tracepath
  - iputils-ping
  - iputils-arping
  - tshark
  - vim
  - jq
  - traceroute
  - qemu-guest-agent

runcmd:
  - echo 127.0.1.1 ${VM_NAME} >> /etc/hosts
  - [ sed, '-ie', 's/127.0.0.1 localhost/127.0.0.1 localhost ${VM_NAME}/', '/etc/hosts' ]
  - [ hostnamectl, 'hostname', '${VM_NAME}.${DOMAINNAME}' ]

# Update apt database on first boot (run 'apt-get update').
# Note, if packages are given, or package_upgrade is true, then
# update will be done independent of this setting.
#
# Default: false
package_update: true

# Upgrade the instance on first boot
#
# Default: false
package_upgrade: true
package_reboot_if_required: true
output:
  all: ">> /var/log/cloud-init-output.log"
EOF
```

To ensure the keys and values in your `user-data` file are correct, you can run:
```sh
# sudo apt install cloud-init
sudo cloud-init schema -c user-data --annotate
```

Expected output:
```
Valid schema user-data
```

## Define our meta data - NoCloud
Each cloud provider presents unique configuration metadata to a launched cloud instance. Cloud-init crawls this metadata and then caches and exposes this information as a standardised and versioned JSON object known as instance-data. This instance-data may then be queried or later used by cloud-init in templated configuration and scripts.

```sh
cat <<EOF > meta-data
local-hostname: ${VM_NAME}
image: ${IMAGE}
distro: ubuntu
distro_release: nobble
distro_version: 24.04
machine: x86_64
creation_date: $(date '+%Y-%m-%d-%H:%M:%S')
EOF
```

## Create network
On a system with Netplan present, cloud-init will pass Version 2 configuration through to Netplan without modification.

```sh
# Some variables, adjust to your needs
cat <<EOF > network-config
# Network Cloud-Init config
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: false
      addresses:
      - ${IPv4_CIDR}
      nameservers:
        addresses:
        - ${DNS1}
        - ${DNS2}
        - ${DNS3}
        search:
        - ${SEARCH}
      routes:
        - to: default
          via: ${GW}
EOF
```

## Create an ISO disk
This disk is used to pass configuration to cloud-init. Create it with the `genisoimage` command. The disk must contain the files:
- `user-data`, 
- `meta-data` and 
- `network-config`:

```sh
genisoimage -output ${VM_NAME}.iso -volid cidata -joliet -rock user-data meta-data network-config
# Keep a copy of each file
cp user-data user-data-${VM_NAME}
cp meta-data meta-data-${VM_NAME}
cp network-config network-config-${VM_NAME}
```

# Create Virtual Machine directory
Create directories to store the actual disk (instance images) for the running VM and copy all the files needed to create the image:
```sh
sudo install -v -d -g libvirt -o libvirt-qemu ${LIBVIRT_DIR}/${BASE_DIR}
sudo install -v -g libvirt -o libvirt-qemu ${IMAGE} ${LIBVIRT_DIR}/${BASE_DIR}/${VM_NAME}.qcow2
sudo install -v -g libvirt -o libvirt-qemu ${VM_NAME}.iso ${LIBVIRT_DIR}/${BASE_DIR}/.
# resize the VM
sudo qemu-img resize ${LIBVIRT_DIR}/${BASE_DIR}/${VM_NAME}.qcow2 20G
```

## Verify the image (Optional)
Give information about the disk image. This is optional.
```sh
sudo qemu-img info ${LIBVIRT_DIR}/${BASE_DIR}/${VM_NAME}.qcow2
```

## Start the VM
`virt-install` is a command line tool for creating new KVM, Xen, or Linux container guests using the `libvirt` hypervisor management library. If using any other hypervisor, please you the tools provided with it.
```sh
virt-install \
--connect qemu:///system \
--virt-type kvm \
--name ${VM_NAME} \
--ram 4096 \
--vcpus=2 \
--os-variant ubuntu24.04 \
--disk path=${LIBVIRT_DIR}/${BASE_DIR}/${VM_NAME}.qcow2,format=qcow2 \
--disk path=${LIBVIRT_DIR}/${BASE_DIR}/${VM_NAME}.iso,device=cdrom,bus=scsi \
--import \
--network bridge=${BRIDGE},model=virtio \
--graphics none \
--noautoconsole
```

> [!IMPORTANT]  
> Wait until it has rebooted a second time and then eject the CD-ROM. It won't be needed anymore.

> [!NOTE]  
> The commands `virsh` are executed on the KVM hosts **NOT inside the VM**.
> The commands below will shutdown and restart the VM.

Eject and detach the CD-ROM, it won't be needed anymore. The lines below are a ugly hack but it works 😀
```sh
# Eject the CD-ROM
virsh --connect qemu:///system change-media --domain ${VM_NAME} --eject /var/lib/libvirt/images/${BASE_DIR}/${VM_NAME}.iso --config --live --force
# Remove the CD-ROM. You will need to reboot the VM for the change to take effect
virsh --connect qemu:///system detach-disk --domain ${VM_NAME} --persistent --live --print-xml --target sda > cdrom.xml
virsh --connect qemu:///system detach-device --domain ${VM_NAME} --config --file cdrom.xml

# Ugly hack to shutdown the VM and restart it after. This is required to remove the CD-ROM completly
virsh --connect qemu:///system shutdown --domain ${VM_NAME}
STATUS=$(virsh --connect qemu:///system domstate ${VM_NAME})
while ([ "${STATUS}" != "shut off" ] )
do
  STATUS=$(virsh --connect qemu:///system domstate ${VM_NAME})
  printf "   Status of VM [%s] is: %s\n" "${VM_NAME}" "${STATUS}"
  sleep 2
done
sleep 1
virsh --connect qemu:///system start --domain ${VM_NAME}
```

# Test the new VM
Try to ssh to the new VM. The irst line is to remove the old SSH key in your `known_hosts` file, if it exist. It's really optional and I used it since it took me more than one try to get the template VM running 😉 The second line will accept the new key from the VM. This should never be used in a production environment.
```sh
ssh-keygen -f "/home/${USER}/.ssh/known_hosts" -R "${IPv4}"
ssh -o StrictHostKeyChecking=accept-new ${IPv4}
```

# Post Install
Customise the new VM according to your needs and preferences. Add any configration or other packages.

```sh
curl -LO https://raw.githubusercontent.com/pimlie/ubuntu-mainline-kernel.sh/master/ubuntu-mainline-kernel.sh
sudo install -v --group=adm --owner=root --mode=755 ubuntu-mainline-kernel.sh /usr/local/bin/
rm ubuntu-mainline-kernel.sh
# Don't create an SSH key pair. It will be created for each of the cloned VM
# ssh-keygen -q -t ed25519 -N '' -f ~/.ssh/id_ed25519 <<<y >/dev/null 2>&1

sudo apt -y remove wpasupplicant
sudo apt purge wpasupplicant
sudo apt autoremove
sudo systemctl disable systemd-networkd-wait-online.service
sudo systemctl mask systemd-networkd-wait-online.service

cat >> .bashrc <<'EOF'
# Taken from: https://robotmoon.com/bash-prompt-generator/
PS1="\[\e[38;5;39m\]\u\[\e[38;5;81m\]@\[\e[38;5;77m\]\h \[\e[38;5;226m\]\w \[\033[0m\]$ "
EOF
source .bashrc
sudo ubuntu-mainline-kernel.sh --yes -i
```

```sh
sudo init 6
```

Log back into the VM after the reboot.
```sh
ssh -o StrictHostKeyChecking=accept-new ${IPv4}
```

```sh
sudo apt --fix-broken -y install
sudo apt -y purge $(dpkg-query --show 'linux-headers-*' | cut -f1 | grep -v "$(uname -r | cut -f1 -d '-')")
sudo apt -y purge $(dpkg-query --show 'linux-modules-*' | cut -f1 | grep -v "$(uname -r | cut -f1 -d '-')")
sudo apt update && sudo apt -y upgrade
sudo apt -y autoremove
```

# Cleanup Ubuntu 24.04 LTS (Optional)
You do the following steps only if you know what yoy're doing, you've been warned.

## Reduce Log size
Reduce log size to 50 MB and make it permanent::
```sh
sudo journalctl --vacuum-time=50m
sudo sed -i 's/#SystemMaxUse=/SystemMaxUse=50M/' /etc/systemd/journald.conf
sudo sed -i 's/#SystemMaxFiles=100/SystemMaxFiles=5/g' /etc/systemd/journald.conf
sudo journalctl --rotate
```

```sh
# keep 1 week worth of backlogs
sudo sed -i 's/rotate 4/rotate 1/g' /etc/logrotate.conf
# rotate log files daily
sudo sed -i 's/weekly/daily/g' /etc/logrotate.conf
```

## Check Services
Check for unwanted services:
```sh
service --status-all
```

## Remove Cloud-Init from Ubuntu
Those are the commands to completly remove Cloud init.

```sh
# Uninstall and purge cloud-init from the server or workstation.
sudo apt -y purge cloud-init
# Remove the /etc/cloud/ directory.
sudo rm -rf /etc/cloud/
# Remove the /var/lib/cloud/ directory.
sudo rm -rf /var/lib/cloud/
```

# Remove unattended upgrades [CAUTION]
> [!IMPORTANT]  
> Don't do this unless you **KNOW** what you are doing and the impacts.

This removes the `unattended-upgrades` package and the associated services which are reponsible for automatically updating packages in the system.
```sh
sudo apt -y purge --auto-remove unattended-upgrades
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl mask apt-daily-upgrade.service
sudo systemctl disable apt-daily.timer
sudo systemctl mask apt-daily.service
```

Comment `unattended-upgrades` that you judge unnecessary.
```sh
# sudo vim /etc/apt/apt.conf.d/50unattended-upgrades
```

## Remove orphan packages
```sh
sudo apt -y autoremove --purge
sudo apt-get autoclean
```

## Remove Plymouth
```sh
sudo apt -y purge --auto-remove plymouth
# sudo systemctl disable plymouth.service
sudo systemctl mask plymouth.service
sudo systemctl daemon-reload
```

## Remove ufw
You can remove `ufw`. It will not affect your iptables configuration. UFW (Uncomplicated Firewall) was simply developed to ease some configurations done with iptables.

To purge it, in cases where it's wasting space, you can type the following:
```sh
sudo apt -y purge --auto-remove ufw
```

# Clone VM

## Shutdown
Shutdown the new VM you just created. It will serve has a template VM:
```sh
virsh --connect qemu:///system shutdown ${VM_NAME}
STATUS=$(virsh --connect qemu:///system domstate ${VM_NAME})
while ([ "${STATUS}" != "shut off" ] )
do
  STATUS=$(virsh --connect qemu:///system domstate ${VM_NAME})
  printf "   Status of VM [%s] is: %s\n" "${VM_NAME}" "${STATUS}"
  sleep 2
done
printf "[%s] is shutdown: %s\n" "${VM_NAME}" "${STATUS}"
```

## Clone
You can clone the VM created (template). The variable `newVM` has been set at the beggining of this tutorial.

```sh
for NEW in "${!newVM[@]}"
do
  # Create the new directory
  sudo install -v -d -g libvirt -o libvirt-qemu ${LIBVIRT_DIR}/${NEW}

  # Clone the VM
  sudo virt-clone --connect qemu:///system \
  --original ${VM_NAME} --name ${NEW} \
  --file ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2

  # Change permission
  sudo chown libvirt-qemu:libvirt ${LIBVIRT_DIR}/${NEW}/${NEW}.qcow2
done
```

### Customize the VMs
Once you have cloned the VM, you can and should customize the new virtual machines with `virt-sysprep` utility. Make sure the utility is installed with the command `sudo apt install guestfs-tools`
```sh
OPERATIONS=$(virt-sysprep --list-operations | egrep -v 'bash-history|lvm-uuids|fs-uuids|machine-id|net-hwaddr|ssh-hostkeys|ssh-userdir' | awk '{ printf "%s,", $1}' | sed 's/,$//')
for NEW in "${!newVM[@]}"
do
  sudo virt-sysprep -d ${NEW} \
  --hostname ${NEW}.${DOMAINNAME} \
  --enable ${OPERATIONS} \
  --keep-user-accounts ${USERNAME} \
  --run-command "ssh-keygen -q -t ed25519 -N '' -f /home/${USERNAME}/.ssh/id_ed25519" \
  --run-command "chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}/.ssh/" \
  --run-command "sed -i \"s/127.0.1.1.*/127.0.1.1 ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/127.0.0.1.*/127.0.0.1 localhost ${NEW}/\" /etc/hosts" \
  --run-command "sed -i \"s/${IPv4}/${newVM[${NEW}]}/\" /etc/netplan/50-cloud-init.yaml" \
  --run-command "sed -i \"s/${GW}/${defaultGateway}/\" /etc/netplan/50-cloud-init.yaml"
done
```

> [!NOTE]  
> A new ssh key pair is created for each VM.

### Attach VM to bridge
> [!NOTE]  
> You can skip this section if your VM template has the correct bridge network configured.

Make it permanent:
```sh
for NEW in "${!newVM[@]}"
do
  virsh --connect qemu:///system dumpxml ${NEW} > ${NEW}.xml
  sed -i "s/${BRIDGE}/${NEW_BRIDGE}/g" ${NEW}.xml
  virsh --connect qemu:///system define --file ${NEW}.xml
  virsh --connect qemu:///system autostart ${NEW}
  rm -f ${NEW}.xml
done
```

## Start the VMs
Start the VMs.

> [!NOTE]  
> By default the VMs won't auto start. You can add the line `virsh autostart ${NEW}` for them to start automaticaly.

```sh
for NEW in "${!newVM[@]}"
do
  printf "Starting VM %s\n" "${NEW}"
  virsh --connect qemu:///system start ${NEW}
done
```

Try to ssh to the new VMs you just cloned. Again I added a line to remove the old SSH key in case 😉
```sh
TEST_IP="10.30.100.102" # Choose one VM
ssh-keygen -f "/home/${USER}/.ssh/known_hosts" -R "${TEST_IP}"
ssh -o StrictHostKeyChecking=accept-new ${TEST_IP}
```

# Destroy the VMs (Optional)
If you want to completly destroy the VMs and remove the disk image, use the script below:

> [!WARNING]  
> - Make sure the environment variables are set!
> - Instead of `rm -rf` maybe you could use `rm -ri` for interactive mode which prompt before every removal!

```sh
for VM in "${!newVM[@]}"
do
  virsh --connect qemu:///system shutdown ${VM}
  STATUS=$(virsh --connect qemu:///system domstate ${VM})
  while ([ "${STATUS}" != "shut off" ] )
  do
    STATUS=$(virsh --connect qemu:///system domstate ${VM})
    printf "   Status of VM [%s] is: %s\n" "${VM}" "${STATUS}"
    sleep 2
  done
  printf "[%s] is shutdown: %s\n" "${VM}" "${STATUS}"
  sudo virsh --connect qemu:///system undefine ${VM} --remove-all-storage --wipe-storage
  # Make sure both vars exist before deleting directory
  if ! [[ -z "${LIBVIRT_DIR}" || -z "${VM}" ]] ; then
    sudo rm -rf ${LIBVIRT_DIR}/${VM}
  else
    printf "Error: Can't delete directory %s/%s.\n" "${LIBVIRT_DIR}" "${VM}"
  fi
  virsh --connect qemu:///system pool-destroy ${VM}
  virsh --connect qemu:///system pool-undefine ${VM}
done
```

# References
[virt-install - provision new virtual machines](https://manpages.ubuntu.com/manpages/noble/man1/virt-install.1.html)  
[Ubuntu 24.04 LTS (Noble) daily](https://cloud-images.ubuntu.com/noble/current/)  
[Cloud config examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html#yaml-examples)  
[Domain XML format](https://libvirt.org/formatdomain.html)  
[Reset, unconfigure or customize a virtual machine so clones can be made](https://libguestfs.org/virt-sysprep.1.html)  
