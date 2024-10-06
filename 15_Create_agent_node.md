# Launch Agent Nodes
Because K3s server nodes are schedulable by default, agent nodes are not required for a HA K3s cluster. However, you may wish to have dedicated agent nodes to run your apps and services.

At this point you should have a K3s Cluster with at least 2 or mode server nodes.

# Before you begin
We will be using three Ubuntu Server 24.04 with Linux Kernel 6.11.0.

# Setup K3s Server Nodes
In this tutorial we will configure three agent nodes.

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|Node|
|----|----|----|----|----|----|----|----|
|agent|k3s1agent1.kloud.lan|10.30.100.61|Ubuntu 24.04|6.11.0|4G|2|pve1|
|agent|k3s1agent2.kloud.lan|10.30.100.62|Ubuntu 24.04|6.11.0|4G|2|pve1|
|agent|k3s1agent3.kloud.lan|10.30.100.63|Ubuntu 24.04|6.11.0|4G|2|pve1|

# Terminal Multiplexer
I'm using `tmux` to access all the VMs at the same time from a bastion host. Create the following script to start a session on each VM. Adjust the variable `ssh_list`:
```sh
FILE="k3s1-agent.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash

ssh_list=( k3s1agent1 k3s1agent2 k3s1agent3 )

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

> [!NOTE]  
> The steps above were executed on `k8s2bastion1` in the directory `~/k3s1/`

# Join Agent Node(s)
Joining agent nodes in an HA cluster is the same as joining agent nodes in a single server cluster. You just need to specify the URL the agent should register to (either one of the server IPs or a fixed registration address) and the token it should use. This could be done on the command line but I prefer going with a configuration file.

> [!NOTE]  
> This has to be executed on an `agent` node not a bastion host.

```sh
sudo mkdir -p /etc/rancher/k3s/
cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml > /dev/null
token: "my_super_secret"
EOF
curl -sfL https://get.k3s.io | sudo sh -s - agent --server https://k3s1api.kloud.lan:6443 --config /etc/rancher/k3s/config.yaml
```

> [!NOTE]  
> Repeat the steps above for every `agent` node.

Output:
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
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
```

## Copy Config file (Optional)
The follwoing steps are optional. I prefere to administer a Kubernetes Cluster from a bastion host.
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
# Adjust the API server address to our load balancer
API_IPv4=$(dig +short k3s1api.kloud.lan)
sed -i s/127.0.0.1:6443/${API_IPv4}:6443/ $HOME/.kube/config
```

> [!IMPORTANT]  
> This WON'T survive after a restart.

## Add node role
I like to have a `ROLES` with `worker`, so I add a node role for each worker node with the command:
```sh
for i in {1..3}; do kubectl label node k3s1agent${i}.kloud.lan node-role.kubernetes.io/worker=''; done
```

> [!NOTE]  
> You can run this command on any node that has access to the cluster. I ran it on my bastion host.

# Congratulation
You should have a six-node K3s cluster in **High Availibility** mode 🍾🎉🥳
- Three (3) K3s Server Nodes
- Three (3) K3s Agent Nodes

Output of the command `kubectl get nodes -o wide`
```
NAME                    STATUS   ROLES                  AGE     VERSION        INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION          CONTAINER-RUNTIME
k3s1agent1.kloud.lan    Ready    worker                 5d      v1.30.5+k3s1   10.30.100.61   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1agent2.kloud.lan    Ready    worker                 5d      v1.30.5+k3s1   10.30.100.62   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1agent3.kloud.lan    Ready    worker                 5d      v1.30.5+k3s1   10.30.100.63   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1server1.kloud.lan   Ready    control-plane,master   5d19h   v1.30.5+k3s1   10.30.100.51   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1server2.kloud.lan   Ready    control-plane,master   5d      v1.30.5+k3s1   10.30.100.52   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
k3s1server3.kloud.lan   Ready    control-plane,master   5d      v1.30.5+k3s1   10.30.100.53   <none>        Ubuntu 24.04.1 LTS   6.11.0-061100-generic   containerd://1.7.21-k3s2
```

# References
[High Availability External DB](https://docs.k3s.io/datastore/ha)  
[k3s server Options](https://docs.k3s.io/cli/server)  
[K3s Configuration Options](https://docs.k3s.io/installation/configuration)  
