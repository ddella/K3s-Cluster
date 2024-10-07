# Launch Server Nodes
K3s requires two or more server nodes for this HA configuration. See the Requirements guide for minimum machine requirements.

When running the k3s server command on these nodes, you must set the datastore-endpoint parameter so that K3s knows how to connect to the external datastore. The token parameter can also be used to set a deterministic token when adding nodes. When empty, this token will be generated automatically for further use.

For example, a command like the following could be used to install the K3s server with a MySQL database as the external datastore and set a token:

# Before you begin
We will be using three Ubuntu Server 24.04 with Linux Kernel 6.11.0.

# Setup K3s Server Nodes
In this tutorial we will configure three K3s servers to build a K3s Cluster in H.A.mode 😉

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|Node|
|----|----|----|----|----|----|----|----|
|server|k3s1server1.kloud.lan|10.30.100.51|Ubuntu 24.04|6.11.0|4G|2|pve1|
|server|k3s1server2.kloud.lan|10.30.100.52|Ubuntu 24.04|6.11.0|4G|2|pve1|
|server|k3s1server3.kloud.lan|10.30.100.53|Ubuntu 24.04|6.11.0|4G|2|pve1|

> [!NOTE]  
> Everything from here is done on `k8s2bastion1` in the directory `$HOME/k3s1/`

# Terminal Multiplexer
I'm using `tmux` to access all the VMs at the same time. Create the following script to start a session on each VM. Adjust the variable `ssh_list`:
```sh
FILE="k3s1-server.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash

ssh_list=( k3s1server1 k3s1server2 k3s1server3 )

split_list=()
for ssh_entry in "${ssh_list[@]:1}"; do
  split_list+=( split-pane ssh "$ssh_entry" ';' )
done

tmux new-session ssh "${ssh_list[0]}" ';' \
  "${split_list[@]}" \
  select-layout tiled ';' \
  set-option -w synchronize-panes
EOF
chmod +x ${FILE}
```

### Copy certificates
When I created the `etcd` cluster I created certificates from my own CA (bunch of scripts with OpenSSL 😉). All the certificates were created on my bastion `k8s2bastion1` node. Copy the `etcd` CA certificate and private key, from `k8s2bastion1` to `k3s1server1` in the directory `$HOME`.

The following commands must be executed **from** the server where the `etcd` certificates were generated. In my case I used my `k8s2bastion1` host. Each line will copy the necessary TLS certificate/private key to each K3s server.
- k3s1-etcd-ca.crt is the CA certificate
- apiserver-etcd-client.crt is the client certificate to access any `etcd` server
- apiserver-etcd-client.key is the client key that macthes the certificate above

```sh
# ssh k8s2bastion1
# cd in the directory where you generated the certificates
# cd /home/daniel/k3s1
rsync --mkpath --rsync-path="sudo rsync" k3s1-etcd-ca.crt apiserver-etcd-client.crt apiserver-etcd-client.key daniel@k3s1server1:/home/daniel/
rsync --mkpath --rsync-path="sudo rsync" k3s1-etcd-ca.crt apiserver-etcd-client.crt apiserver-etcd-client.key daniel@k3s1server2:/home/daniel/
rsync --mkpath --rsync-path="sudo rsync" k3s1-etcd-ca.crt apiserver-etcd-client.crt apiserver-etcd-client.key daniel@k3s1server3:/home/daniel/
```

You should have the following files on all K3s server node.
```
/home/daniel
├── k3s1-etcd-ca.crt
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
```

# Install K3s Server node(s)
K3s can use a config file. By default, values present in a YAML file located at `/etc/rancher/k3s/config.yaml` will be used on the initial bootstrap. Multiple configuration files are supported. By default, configuration files are read from `/etc/rancher/k3s/config.yaml` and `/etc/rancher/k3s/config.yaml.d/*.yaml` in alphabetical order.

> [!IMPORTANT]  
> You have to be on `k3s1server1` to bootstrap the server.

## Create Configuration file
```sh
sudo mkdir -p /etc/rancher/k3s/

cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml > /dev/null
cluster-cidr: "100.66.0.0/16"
cluster-domain: "k3s1-prod.lan"
service-cidr: "198.18.8.0/22"
write-kubeconfig-mode: "0644"
token: "my_super_secret"

tls-san:
  - "k3s1api.kloud.lan"
  - "10.30.100.100"
  - "k3s1server1.kloud.lan"
  - "10.30.100.51"
  - "k3s1server2.kloud.lan"
  - "10.30.100.52"
  - "k3s1server3.kloud.lan"
  - "10.30.100.53"
datastore-endpoint: "https://k3s1etcd1.kloud.lan:2379,https://k3s1etcd2.kloud.lan:2379,https://k3s1etcd3.kloud.lan:2379"
datastore-cafile: "/home/daniel/k3s1-etcd-ca.crt"
datastore-certfile: "/home/daniel/apiserver-etcd-client.crt"
datastore-keyfile: "/home/daniel/apiserver-etcd-client.key"
EOF
```

## Install K3s
```sh
curl -sfL https://get.k3s.io | sudo sh -s - server --config /etc/rancher/k3s/config.yaml
```

Output should be similar to this:
```
[INFO]  Finding release for channel stable
[INFO]  Using v1.30.5+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.30.5+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.30.5+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

Enable `kubectl` autocompletion for Bash (Optional):
```sh
sudo kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
source ~/.bashrc
```

## Copy K3s Config file
The K3s configuration files contains the necessary information to connect to the K3s Cluster API.
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config-k3s1
sudo chown $(id -u):$(id -g) $HOME/.kube/config-k3s1
chmod 600 ~/.kube/config-k3s1
export KUBECONFIG=$HOME/.kube/config-k3s1
# Adjust the API server address to our load balancer
API_IPv4=$(dig +short k3s1api.kloud.lan)
sed -i s/127.0.0.1:6443/${API_IPv4}:6443/ $HOME/.kube/config-k3s1

echo 'alias k=kubectl' >>$HOME/.bashrc
echo 'complete -o default -F __start_kubectl k' >>$HOME/.bashrc
```

> [!IMPORTANT]  
> This variable `KUBECONFIG` WON'T survive after a restart.

## Verify K3s service

```sh
sudo systemctl status k3s.service
```

```
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Tue 2024-10-01 08:34:35 EDT; 5min ago
       Docs: https://k3s.io
    Process: 1842 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service 2>/dev/null (code=exited, status=0/SUCCESS)
    Process: 1844 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 1846 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 1848 (k3s-server)
      Tasks: 98
     Memory: 1.1G (peak: 1.1G)
        CPU: 33.298s
     CGroup: /system.slice/k3s.service
             ├─1848 "/usr/local/bin/k3s server"
             ├─1880 "containerd "
             ├─2440 /var/lib/rancher/k3s/data/ac0baecab6b7fa399482b08daa7117e7f2a0b1a739da5c31131bea4ebfaedfec/bin/containerd-shim-runc-v2 -namespace k8s.io -id 58b1da175546a5cff820db319d5bea35b935d5929576c9e850224af95e3341a6 -address /run/k3s/containerd/containerd.sock
             ├─2454 /var/lib/rancher/k3s/data/ac0baecab6b7fa399482b08daa7117e7f2a0b1a739da5c31131bea4ebfaedfec/bin/containerd-shim-runc-v2 -namespace k8s.io -id d08ee4efa4787a33b32f82f051d6395e8e410d3155280925621c9987935f093b -address /run/k3s/containerd/containerd.sock
             ├─2485 /var/lib/rancher/k3s/data/ac0baecab6b7fa399482b08daa7117e7f2a0b1a739da5c31131bea4ebfaedfec/bin/containerd-shim-runc-v2 -namespace k8s.io -id 89b5c9c10ef08f4472e08af034ced6468d03d889f3d6e98806b5bf481405a130 -address /run/k3s/containerd/containerd.sock
             ├─2492 /var/lib/rancher/k3s/data/ac0baecab6b7fa399482b08daa7117e7f2a0b1a739da5c31131bea4ebfaedfec/bin/containerd-shim-runc-v2 -namespace k8s.io -id 23fa7fac0a84b3c9b1dffb8e1c123101085ef597e46affc761ed1d2008dcf5ac -address /run/k3s/containerd/containerd.sock
             └─2608 /var/lib/rancher/k3s/data/ac0baecab6b7fa399482b08daa7117e7f2a0b1a739da5c31131bea4ebfaedfec/bin/containerd-shim-runc-v2 -namespace k8s.io -id 1bea6c919ac02090becd1b43c27958248bded897335971d466e76d60474f4f21 -address /run/k3s/containerd/containerd.sock

Oct 01 08:35:08 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:08.652249    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/local-path-provisioner-6795b5f9d8" duration="55.574µs"
Oct 01 08:35:12 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:12.636669    1848 scope.go:117] "RemoveContainer" containerID="64ee5d8c07a91f1291c779350eaa24b952a7b321ecd6a78a1fa1c94b16c8893a"
Oct 01 08:35:12 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:12.786046    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/metrics-server-cdcc87586" duration="37.41µs"
Oct 01 08:35:21 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:21.636509    1848 scope.go:117] "RemoveContainer" containerID="50c9e5216abe0f4e770fc883ab1822a4ae63cdf6e95cda8089764898a7e95791"
Oct 01 08:35:21 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:21.807509    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/local-path-provisioner-6795b5f9d8" duration="9.062895ms"
Oct 01 08:35:21 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:21.807579    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/local-path-provisioner-6795b5f9d8" duration="40.015µs"
Oct 01 08:35:28 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:28.883028    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/metrics-server-cdcc87586" duration="11.399116ms"
Oct 01 08:35:28 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:28.883570    1848 replica_set.go:676] "Finished syncing" kind="ReplicaSet" key="kube-system/metrics-server-cdcc87586" duration="36.328µs"
Oct 01 08:35:28 k3s1server1.kloud.lan k3s[1848]: time="2024-10-01T08:35:28-04:00" level=warning msg="Proxy error: write failed: write tcp 127.0.0.1:6443->127.0.0.1:43970: write: broken pipe"
Oct 01 08:35:28 k3s1server1.kloud.lan k3s[1848]: I1001 08:35:28.911269    1848 handler.go:286] Adding GroupVersion metrics.k8s.io v1beta1 to ResourceManager
```

## Other Server node(s)
You can repeat the same process for every `server` node you want to add to the K3s Cluster. Don't forget that for H.A. you need at least two K3s Server nodes. I repeated this step two (2) times to get three (3) K3s Server nodes.

```sh
# Make sure you can SSH to your K3s server without password
# ssh-copy-id -i ~/.ssh/id_ed25519.pub k3s1server[1..3]
# TOKEN=$(ssh k3s1server1 -- sudo cat /var/lib/rancher/k3s/server/token)
sudo mkdir -p /etc/rancher/k3s/

cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml > /dev/null
cluster-cidr: "100.66.0.0/16"
cluster-domain: "k3s1-prod.lan"
service-cidr: "198.18.8.0/22"
write-kubeconfig-mode: "0644"
token: "my_super_secret"

tls-san:
  - "k3s1api.kloud.lan"
  - "10.30.100.100"
  - "k3s1server1.kloud.lan"
  - "10.30.100.51"
  - "k3s1server2.kloud.lan"
  - "10.30.100.52"
  - "k3s1server3.kloud.lan"
  - "10.30.100.53"
datastore-endpoint: "https://k3s1etcd1.kloud.lan:2379,https://k3s1etcd2.kloud.lan:2379,https://k3s1etcd3.kloud.lan:2379"
datastore-cafile: "/home/daniel/k3s1-etcd-ca.crt"
datastore-certfile: "/home/daniel/apiserver-etcd-client.crt"
datastore-keyfile: "/home/daniel/apiserver-etcd-client.key"
EOF

curl -sfL https://get.k3s.io | sudo sh -s - server --config /etc/rancher/k3s/config.yaml
# K3S_VER=$(curl -s https://api.github.com/repos/k3s-io/k3s/releases/latest|grep tag_name | cut -d '"' -f 4)
# curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v${K3S_VER} sudo sh -s - server --config /etc/rancher/k3s/config.yaml
```

Output from `k3s1server2`:
```
[INFO]  Finding release for channel stable
[INFO]  Using v1.30.5+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.30.5+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.30.5+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Skipping installation of SELinux RPM
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```

Enable `kubectl` autocompletion for Bash (Optional):
```sh
sudo kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null
source ~/.bashrc
```

## Copy Config file
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config-k3s1
sudo chown $(id -u):$(id -g) $HOME/.kube/config-k3s1
chmod 600 $HOME/.kube/config-k3s1
export KUBECONFIG=$HOME/.kube/config-k3s1

# Adjust the API server address to our load balancer
API_IPv4=$(dig +short k3s1api.kloud.lan)
sed -i s/127.0.0.1:6443/${API_IPv4}:6443/ $HOME/.kube/config-k3s1

echo 'alias k=kubectl' >>$HOME/.bashrc
echo 'complete -o default -F __start_kubectl k' >>$HOME/.bashrc
```

> [!IMPORTANT]  
> This variable `KUBECONFIG` WON'T survive after a restart.

# Congratulation
You should have a three-node K3s cluster in **High Availibility** mode 🍾🎉🥳

Output of the command `kubectl get nodes -o wide`
```
NAME                    STATUS   ROLES                  AGE     VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION          CONTAINER-RUNTIME
k3s1server1.kloud.lan   Ready    control-plane,master   5d19h   v1.30.5+k3s1   10.30.100.51   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1server2.kloud.lan   Ready    control-plane,master   5d      v1.30.5+k3s1   10.30.100.52   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1server3.kloud.lan   Ready    control-plane,master   5d      v1.30.5+k3s1   10.30.100.53   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
```

# Troubleshoot

## Re installing K3s
If you use `etcd` outside the K3s cluster and try to re-install it, you might endup with the error message: `msg="starting kubernetes: preparing server: bootstrap data already found and encrypted with different token"`. The K3s unsintall script can't modify the `etcd` database if it's outside the K3s cluster. To fully uninstall K3s, you need to delete the `/bootstrap` key in the database.

Prepare to connect to any `etcd` node. You need the binary `etcdctl`, the CA certificate and a client certificate and private key for the `etcd` cluster:
```sh
ETCD_NAME=k3s1etcd1
export ETCDCTL_ENDPOINTS=https://k3s1etcd1.kloud.lan:2379,https://k3s1etcd2.kloud.lan:2379,https://k3s1etcd3.kloud.lan:2379
export ETCDCTL_CACERT=./k3s1-etcd-ca.crt
export ETCDCTL_CERT=./${ETCD_NAME}.crt
export ETCDCTL_KEY=./${ETCD_NAME}.key
```

## Get the Key
```sh
KEY=$(etcdctl get --prefix /bootstrap | head -n 1)
printf "The key to delete is: [%s]\n" "${KEY}"
```

Output:
```
The key to delete is: [/bootstrap/a77d1191a7da]
```

## Delete the Key
```sh
etcdctl del ${KEY}
```

A value of `1` means the key has been deleted successfully. 
```
1
```

After this steps, you should be able to reinstall K3s using the same `etcd` Cluster.

# References
[High Availability External DB](https://docs.k3s.io/datastore/ha)  
[k3s server Options](https://docs.k3s.io/cli/server)  
[K3s Configuration Options](https://docs.k3s.io/installation/configuration)  
