# SSH Keys
This script will copy the public key of a Linux host (bastion) on every Kubernetes hosts.

## Define variables
Declare a "kind-of-dictionnary" with the hostname and IP address of the VMs.
```sh
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

DOMAINNAME="kloud.lan"
# Password for the SSH session.
MYPASSWORD="PASSWORD"
```

## Install `sshpass`
You need to have the utility `sshpass` installed on every bastion host. I installed it on `k8s2bastion1` only.
```sh
sudo apt update && sudo apt -y install sshpass
```

## SSH without password
This step will:
- Copy you public key, from your bastion host, to the file `.ssh/authorized_keys` file on the new VMs so you can ssh without needing to enter your password.
- Populate your local (on your bastion host) file `.ssh/known_hosts` so you won't need to answer `yes` to accept the public key of the remote host on your first login.

```sh
for VM in "${!newVM[@]}"
do
  sshpass -p ${MYPASSWORD} ssh-copy-id -o StrictHostKeyChecking=no -i $HOME/.ssh/id_ecdsa.pub ${VM}
  ssh-keyscan ${VM} >> $HOME/.ssh/known_hosts
  ssh-keyscan ${VM}.${DOMAINNAME} >> $HOME/.ssh/known_hosts
  printf "New SSH key for %s\n" "${VM}"
done
```

> [!IMPORTANT]  
> This needs to be done on every hosts that might act as a `bastion` host.

# Security
Passing a password on the CLI is consider a security risk. This is because other users can see the password in various way, like
- `ps` command
- `top` command
- password will also be logged into your shell history file

You can and should delete your history file or at least check for the variable `MYPASSWORD` and delete every reference to it.
