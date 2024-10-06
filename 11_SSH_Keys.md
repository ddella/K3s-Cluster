# SSH Keys
This script will copy the public key of a Linux host (bastion) on every Kubernetes hosts.

## Define variables
The password to connect on the VMs.
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
MYPASSWORD="PASSWORD"
```

## On every bastion host
This should be executed an every bastion host. In my lab in ran the commands on `k8s2bastion1` only.

Make sure you have `sshpass` installed on your bastion host(s):
```sh
sudo apt update && sudo apt -y install sshpass
```

## SSH without password
You will need to install the `sshpass` package with the command `sudo apt update && sudo apt -y install sshpass`.

- Copy you public key to the "authorized_keys" file on the remote host so you don't need to enter the password
- This will populate your local `known_hosts` so you won't need to answer `yes` to accept the public key for your first login.

```sh
for VM in "${!newVM[@]}"
do
  sshpass -p ${MYPASSWORD} ssh-copy-id -o StrictHostKeyChecking=no -i ~/.ssh/id_ecdsa.pub ${VM}
  ssh-keyscan ${VM} >> ~/.ssh/known_hosts
  ssh-keyscan ${VM}.${DOMAINNAME} >> ~/.ssh/known_hosts
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

You can and should delete your history file.
