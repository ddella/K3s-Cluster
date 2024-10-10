<!-- Improved compatibility of back to top link: See: https://github.com/othneildrew/Best-README-Template/pull/73 -->
<a name="readme-top"></a>

# K3s Cluster
This tutorial will show how to set up a High Availability K3s Cluster on Bare-Metal or more precisely on eleven (11) VMs. All the VMs will be running Ubuntu 24.04 LTS on a single physical KVM node running, as you may have guessed, Ubuntu but this time version 22.04.

Two nodes are used for load balancing the K3s API endpoints. I have choosen `Nginx`, but you could have used `HAProxy`, for the load balancing. I will use `Keepalived` for the VIP between the `Nginx` servers.

## High Availibility
There's two different approaches to setting up a highly available K3s cluster:
- With Embedded SQLite or etcd database. This approach requires less infrastructure.
- With an external database cluster. The control plane nodes and database members are on separate VMs. This approach requires more infrastructure.

I've choosen the second option for this tutorial because it's more complicated which is the goal when your learn a new technology 😉 Setting up a cluster with external etcd nodes requires that you setup etcd first, and you need to pass the etcd information in the K3s config file when bootstrapping your first control plane.

## Network
There will be one network to hosts the different components of our K3s cluster. This is purely optional and is not the real world, which will have many different networks. One could think of having the master node and database on different subnets with a firewall in between.

## Installation Outline
The high level steps to create our High Availability K3s Cluster are:
1. Create all the DNS entries.
2. Create eleven (11) Ubuntu 24.04 LTS VMs. I will be using `Cloud-Init`.
3. Create a cluster of three `etcd` nodes. This will be our external datastore for the K3s cluster.
4. Configure a Fixed Registration Address. Two servers will be used as load balancer.
5. Launch the K3s Server Nodes. K3s requires two or more server nodes for an HA configuration. When running the k3s server command, you must set the `datastore-endpoint` parameter so that K3s knows how to connect to the external datastore created in the step above.
6. Join Agent Nodes. You can join as many as you need.

## Here's a table that describes every IP addresses of each servers for this tutorial.
|Role|FQDN|IP|OS|Kernel|RAM|vCPU|Node|
|----|----|----|----|----|----|----|----|
|Load Balancer (VIP)|k3s1api.kloud.lan|10.30.100.100|N/A|N/A|N/A|N/A|N/A|
|Load Balancer|k3s1vrrp1.kloud.lan|10.30.100.101|Ubuntu 24.04|6.11.0|4G|2|pve1|
|Load Balancer|k3s1vrrp2.kloud.lan|10.30.100.102|Ubuntu 24.04|6.11.0|4G|2|pve1|
|server|k3s1server1.kloud.lan|10.30.100.51|Ubuntu 24.04|6.11.0|4G|2|pve1|
|server|k3s1server2.kloud.lan|10.30.100.52|Ubuntu 24.04|6.11.0|4G|2|pve1|
|server|k3s1server3.kloud.lan|10.30.100.53|Ubuntu 24.04|6.11.0|4G|2|pve1|
|agent|k3s1agent1.kloud.lan|10.30.100.61|Ubuntu 24.04|6.11.0|4G|2|pve1|
|agent|k3s1agent2.kloud.lan|10.30.100.62|Ubuntu 24.04|6.11.0|4G|2|pve1|
|agent|k3s1agent3.kloud.lan|10.30.100.63|Ubuntu 24.04|6.11.0|4G|2|pve1|
|etcd database|k3s1etcd1.kloud.lan|10.30.100.71|Ubuntu 24.04|6.11.0|4G|2|pve1|
|etcd database|k3s1etcd2.kloud.lan|10.30.100.72|Ubuntu 24.04|6.11.0|4G|2|pve1|
|etcd database|k3s1etcd3.kloud.lan|10.30.100.73|Ubuntu 24.04|6.11.0|4G|2|pve1|

<a name="step-1"></a>

<p>
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</p>

# Step 1: DNS entries
I'm using Bind9 for my internal DNS. Here's the entries for this tutorial.

My `A` records:
```
; **************************************
; *********** K3s Cluster 1 ************
; **************************************
k3s1api           IN A      10.30.100.100
k3s1vrrp1         IN A      10.30.100.101
k3s1vrrp2         IN A      10.30.100.102

k3s1server1       IN A      10.30.100.51
k3s1server2       IN A      10.30.100.52
k3s1server3       IN A      10.30.100.53

k3s1agent1        IN A      10.30.100.61
k3s1agent2        IN A      10.30.100.62
k3s1agent3        IN A      10.30.100.63

k3s1etcd1         IN A      10.30.100.71
k3s1etcd2         IN A      10.30.100.72
k3s1etcd3         IN A      10.30.100.73
```

My `PTR` records:
```
; **************************************
; *********** K3s Cluster 1 ************
; **************************************
100.100.30        IN PTR    k3s1api.kloud.lan.
101.100.30        IN PTR    k3s1vrrp1.kloud.lan.
102.100.30        IN PTR    k3s1vrrp2.kloud.lan.

51.100.30         IN PTR    k3s1server1.kloud.lan.
52.100.30         IN PTR    k3s1server2.kloud.lan.
53.100.30         IN PTR    k3s1server3.kloud.lan.

61.100.30         IN PTR    k3s1agent1.kloud.lan.
62.100.30         IN PTR    k3s1agent2.kloud.lan.
63.100.30         IN PTR    k3s1agent3.kloud.lan.

71.100.30         IN PTR    k3s1etcd1.kloud.lan.
72.100.30         IN PTR    k3s1etcd2.kloud.lan.
73.100.30         IN PTR    k3s1etcd3.kloud.lan.
```

<a name="step-2"></a>

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Step 2: Create the VMs
I'm using `Cloud-Init` to create my template VM and then use `virt-clone` to clone the eleven (11) VMs needed. The VMs will have a standard user account with `sudo` privileges. The IP address/DNS/hostname will be configured, after le cloning step, with `virt-sysprep`. At the end of this step you will have eleven (11) VMs that you can SSH to it ready to install K3s or etcd.

[Create the VMs](10_KVM_Cloud-Init_VMs.md)

[push the public SSH key from your bastion host to all the VMs](11_SSH_Keys.md)

<a name="step-3"></a>

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Step 3: Create the etcd cluster
At this stage you should have all the VMs needed to build your K3s1 Cluster in high availibility. Let's create your etcd cluster.

[Create the etcd cluster](12_Create_etcd_cluster.md)

<a name="step-4"></a>

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Step 4: Create the load balancer
It's time to create our load balancer for the K3s1 API. Let's use two servers with NGINX and Keealived to achieve this task.
- NGINX will act as our TCP load balancer
- Keealived will act as our VRRP process for the VIP between the two servers

[Create the load balancer](13_Create_API-LB4.md)

<a name="step-5"></a>

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Step 5: Launch K3s Server nodes
It's time to launch are K3s Server Nodes. This is the same as a Master node in pure Kubernetes 😉 Here I'm launcing the first K3s Server node and two more are added at the end.

[Launch server nodes](14_Create_server_cluster.md)

<a name="step-6"></a>

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Step 6: Join agent node(s)

[Launch agent nodes](15_Create_agent_node.md)

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# Congratulation
You should have a six-node K3s cluster in **High Availibility** mode 🍾🎉🥳
- Three (3) K3s Server Nodes (control-plane,master)
- Three (3) K3s Agent Nodes (worker)

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

<!-- LICENSE -->
## License

This project is licensed under the [MIT license](/LICENSE).  

<div id="links">
  <p style="float:left;"><a href="README.md">🏠 back to README</a></p>
  <p style="float:right;"><a href="#readme-top">🎬 back to Top of file</p>
</div>

# References
[virt-install - provision new virtual machines](https://manpages.ubuntu.com/manpages/noble/man1/virt-install.1.html)  
[Ubuntu 24.04 LTS (Noble) daily](https://cloud-images.ubuntu.com/noble/current/)  
[Cloud config examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html#yaml-examples)  
[Domain XML format](https://libvirt.org/formatdomain.html)  
[Reset, unconfigure or customize a virtual machine so clones can be made](https://libguestfs.org/virt-sysprep.1.html)  

[etcd main website](https://etcd.io/)  
[Set up a High Availability etcd Cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)  
[Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)  
[Configuration file for etcd server](https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample)  

[GitHub](https://github.com/acassen/keepalived)  
[RedHat Simple Configuratiomn](https://www.redhat.com/sysadmin/keepalived-basics)  

[High Availability External DB](https://docs.k3s.io/datastore/ha)  
[High Availability K3s Cluster Setup](https://www.the38.page/posts/high-availability-k3s-cluster-setup/)  
[Cluster Load Balancer](https://docs.k3s.io/datastore/cluster-loadbalancer)  
[Configuration Options](https://docs.k3s.io/installation/configuration)  
[Cluster Datastore](https://docs.k3s.io/datastore)  
[Designing and Building HA Kubernetes on Bare-Metal](https://thebsdbox.co.uk/2020/01/02/Designing-Building-HA-bare-metal-Kubernetes-cluster/)  
