<a name="readme-top"></a>

# Nginx as a load balancer
This section is about configuring an Nginx reverse proxy/load balancer to front the K3s Kubernetes API server. We are building a K3s cluster in high availability with three (3) master nodes. When a request arrives for Kubernetes API, Nginx becomes a proxy and further forward that request to any healthy K3s Master node, then it forwards the response back to the client.

This assumes that:
- K3s API server runs on port 6443 with HTTPS
- All K3s Master node runs the API via the URL: https://<master node>:6443

Nginx will run on a bare metal/virtual Ubuntu server outside the K3s cluster.

##  What is a Reverse Proxy
Proxying is typically used to distribute the load among several servers, seamlessly show content from different websites, or pass requests for processing to application servers over protocols other than HTTP.

When NGINX proxies a request, it sends the request to a specified proxied server, fetches the response, and sends it back to the client.

In this tutorial, Nginx Reverse proxy receive inbound `HTTPS` requests and forward those requests to the K3s master nodes. It receives the outbound `HTTPS` response from the API servers and forwards those requests to the original requester.

# Setup K3s Server Nodes
In this tutorial we will configure a two servers that will act as a load balancer in front the Kubernetes APIU Cluster.

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|Node|
|----|----|----|----|----|----|----|----|
|Load Balancer (VIP)|k3s1api.kloud.lan|10.30.100.100|Ubuntu 24.04|6.11.0|4G|2|N/A|
|Load Balancer|k3s1vrrp1.kloud.lan|10.30.100.101|Ubuntu 24.04|6.11.0|4G|2|pve1|
|Load Balancer|k3s1vrrp2.kloud.lan|10.30.100.102|Ubuntu 24.04|6.11.0|4G|2|pve1|

> [!NOTE]  
> Everything from here is done on `k8s2bastion1` in the directory `$HOME/k3s1/`

# Terminal Multiplexer
I'm using `tmux` to access all the VMs at the same time. Create the following script to start a session on each VM. Adjust the variable `ssh_list`:
```sh
FILE="k3s1-vrrp.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash

ssh_list=( k3s1vrrp1 k3s1vrrp2 )

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

## Installing from the Official NGINX Repository

> [!IMPORTANT]  
> NGINX has to be installed on both `k3s1vrrp1` and `k3s1vrrp2`. I used `tmux` to install on both at the same time.

NGINX Open Source is available in two versions:

- **Mainline** - Includes the latest features and bug fixes and is always up to date. It is reliable, but it may include some experimental modules, and it may also have some number of new bugs.
- **Stable** - Doesn't include all of the latest features, but has critical bug fixes that are always backported to the mainline version. We recommend the stable version for production servers.

Of course I choosed the `Mainline` version to get all the latest and the greatest features 😀

Install the prerequisites:
```sh
sudo apt -y install curl gnupg2 ca-certificates lsb-release debian-archive-keyring
```

Import an official nginx signing key so `apt` could verify the packages authenticity. Fetch the key with the command:
```sh
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
| sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```
Verify that the downloaded file contains the proper key:
```sh
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```

If the output is different **stop** and try to figure out what happens:
```
pub   rsa4096 2024-05-29 [SC]
      8540A6F18833A80E9C1653A42FD21310B49F6B46
uid                      nginx signing key <signing-key-2@nginx.com>

pub   rsa2048 2011-08-19 [SC] [expires: 2027-05-24]
      573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62
uid                      nginx signing key <signing-key@nginx.com>

pub   rsa4096 2024-05-29 [SC]
      9E9BE90EACBCDE69FE9B204CBCDCD8A38D88A2B3
uid                      nginx signing key <signing-key-3@nginx.com>
```

Run the following command to use `mainline` Nginx packages:
```sh
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/mainline/ubuntu/ `lsb_release -cs` nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
```

Set up repository pinning to prefer our packages over distribution-provided ones:
```sh
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
| sudo tee /etc/apt/preferences.d/99nginx
```

Update the repo and install NGINX:
```sh
sudo apt update && sudo apt -y install nginx
```

## Checking Nginx
Start and check that Nginx is running:
```sh
sudo systemctl start nginx
sudo systemctl status nginx
```

Try with `cURL`, you should receive the Nginx welcome page:
```sh
curl http://127.0.0.1
```

## Configure Nginx for layer 4 Load Balancing
This will be the initial configuration of Nginx. I've never been able to bootstrap a Kubernetes Cluster with a layer 7 Load Balancer due the `mTLS` configuration.

Create another directory for our layer 4 Load Balancer.
```sh
sudo mkdir /etc/nginx/tcpconf.d/
```

Create the configuration file. This one will be active:
```sh
cat <<'EOF' | sudo tee /etc/nginx/tcpconf.d/k3s1api.conf
stream {
    log_format k3s1apilogs '[$time_local] $remote_addr:$remote_port $server_addr:$server_port '
        '$protocol $status $bytes_sent $bytes_received '
        '$session_time';
    upstream k3s1-api {
        # server k3s1server1.kloud.lan:6443;
        # server k3s1server2.kloud.lan:6443;
        # server k3s1server3.kloud.lan:6443;
        server 10.30.100.51:6443;
        server 10.30.100.52:6443;
        server 10.30.100.53:6443;
    }
    server {
        listen 6443;

        proxy_pass k3s1-api;
        access_log /var/log/nginx/k3s1api.access.log k3s1apilogs;
        error_log /var/log/nginx/k3s1api.error.log warn;
    }
}
EOF
```
> [!IMPORTANT]  
>  Don't forget the quote around `'EOF'`. We need the variables inside the file not the values of those variables

Add a directive in the `nginx.conf` to parse all the `.conf` file in the new directory we created:
```sh
cat <<EOF | sudo tee -a /etc/nginx/nginx.conf >/dev/null
include /etc/nginx/tcpconf.d/*.conf;
EOF
```

**Important**: Verify Nginx configuration files with the command:
```sh
sudo nginx -t
```
> [!IMPORTANT]  
>  If you don't use `sudo`, you'll get some weird alerts

Restart and check status Nginx server:
```sh
sudo systemctl restart nginx
sudo systemctl status nginx
```

## Verify the load balancer
On any server check Nginx logs with the command:
```sh
sudo tail -f /var/log/nginx/k3s1api.error.log
```

When a client tries to connect, you should see this output. Since we don't have a K8s cluster yet, Nginx will try all the servers in the group and it will receive a `Connection refused`.
```
2024/10/06 10:03:08 [error] 690#690: *3 connect() failed (111: Connection refused) while connecting to upstream, client: 10.30.100.61, server: 0.0.0.0:6443, upstream: "10.30.100.52:6443", bytes from/to client:0/0, bytes from/to upstream:0/0
2024/10/06 10:03:08 [warn] 690#690: *3 upstream server temporarily disabled while connecting to upstream, client: 10.30.100.61, server: 0.0.0.0:6443, upstream: "10.30.100.52:6443", bytes from/to client:0/0, bytes from/to upstream:0/0
2024/10/06 10:03:08 [error] 689#689: *4 connect() failed (111: Connection refused) while connecting to upstream, client: 10.30.100.62, server: 0.0.0.0:6443, upstream: "10.30.100.51:6443", bytes from/to client:0/0, bytes from/to upstream:0/0
2024/10/06 10:03:08 [warn] 689#689: *4 upstream server temporarily disabled while connecting to upstream, client: 10.30.100.62, server: 0.0.0.0:6443, upstream: "10.30.100.51:6443", bytes from/to client:0/0, bytes from/to upstream:0/0
```

From another machine, try to connect to the K3s API loab balancer, with the command:
```sh
curl --insecure --max-time 3 https://k3s1api.kloud.lan:6443
```

Output on the client:
```
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.1</center>
</body>
</html>
```

>Both outputs are normal, since we don't have a K3s master node yet 😀

When the load balancer will be running the message should look like this one:
```
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}
```

# Conclusion
You have a Ubuntu server that acts as a load balancer for all API requests to K3s (ex.: `kubectl` command).

# References
[Nginx Load Balancer](https://nginx.org/en/docs/http/load_balancing.html)  
[Installing NGINX Open Source](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)  

---

# Install Keepalived
`keepalived` is a daemon that implements the VRRP protocol. VRRP is a fundamental brick for router failover. The Virtual Router Redundancy Protocol (VRRP) is a networking protocol that provides for automatic assignment of a VIP and MAC address to participating hosts. Let's get our hands dirty and learn about the installation and basic configuration of `keepalived`. This section applies to both server, `k3s1vrrp1` and `k3s1vrrp2`.

This is a short guide on how to install `keepalived` package on Ubuntu 24.04:
```sh
# sudo snap install --classic keepalived
sudo apt install keepalived
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Configure `keepalived` on `k3s1vrrp1`
Create a Keepalived configuration file named `/etc/keepalived/keepalived.conf` on `k3s1vrrp1`.

> [!NOTE]  
> Make sure you have all you DNS records for the cluster. I'm using it to set the `VIP` in `keepalived` configuration file.

```sh
INTERFACE=enp1s0
VIP=$(dig +short +search k3s1api | tail -1)
SUBNET_MASK=24

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf > /dev/null
global_defs {
  # optimization option for advanced use
  max_auto_priority
  enable_script_security
  script_user nginx  
}

# Script to check whether Nginx is running or not
vrrp_script check_nginx {
  script "/etc/keepalived/check_nginx.sh"
  interval 3
}

vrrp_instance VRRP_1 {
  state MASTER
  interface ${INTERFACE}
  virtual_router_id 70
  priority 255
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass secret
  }
  virtual_ipaddress {
    ${VIP}/${SUBNET_MASK}
  }
  # Use the script above to check if we should fail over
  track_script {
    check_nginx
  }
}
EOF
```

Create the script that checks if Nginx is running:
```sh
cat <<'EOF' | sudo tee /etc/keepalived/check_nginx.sh > /dev/null
#!/bin/sh
if [ -z "$(pidof nginx)" ]; then
  exit 1
fi
EOF
```

Set owner/permission for the script and restart `keepalived`:
```sh
sudo chown nginx:nginx /etc/keepalived/check_nginx.sh
sudo chmod 700 /etc/keepalived/check_nginx.sh 
sudo systemctl restart keepalived
sudo systemctl status keepalived
```

`keepalived `Parameters:

- state MASTER/BACKUP: the state that the router will start in.
- interface ens3: interface VRRP protocol messages should flow.
- virtual_router_id: An ID that both servers should agreed on.
- priority: number for master/backup election – higher numerical means higher priority.
- advert_int: backup waits this long (multiplied by 3) after messages from master fail before becoming master
- authentication: a clear text password authentication, no more than 8 caracters.
- virtual_ipaddress: the virtual IP that the servers will share
- track_script: Check if `Nginx` is running, if not `keepalived` fail to the backup

With the above configuration in place, you can start Keepalived on both servers using `systemctl start keepalived` and observe the IP addresses on each machine. Notice that `k3s1vrrp1` has started up as the VRRP master and owns the shared IP address (192.168.13.71), while `k3s1vrrp2` IP addresses remain unchanged:

<p align="right">(<a href="#readme-top">back to top</a>)</p>

## Configure `keepalived` on `k3s1vrrp2`
Create a Keepalived configuration file named `/etc/keepalived/keepalived.conf` on `k3s1vrrp2`. The only parameter to change here is the `state`.
```sh
INTERFACE="enp1s0"
VIP=$(dig +short +search k3s1api | tail -1)
SUBNET_MASK="24"
SECRET="secret"

cat <<EOF | sudo tee /etc/keepalived/keepalived.conf > /dev/null
global_defs {
  # optimization option for advanced use
  max_auto_priority
  enable_script_security
  script_user nginx  
}

# Script to check whether Nginx is running or not
vrrp_script check_nginx {
  script "/etc/keepalived/check_nginx.sh"
  interval 3
}

vrrp_instance VRRP_1 {
  state MASTER
  interface ${INTERFACE}
  virtual_router_id 70
  priority 250
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass ${SECRET}
  }
  virtual_ipaddress {
    ${VIP}/${SUBNET_MASK}
  }
  # Use the script above to check if we should fail over
  track_script {
    check_nginx
  }
}
EOF
```

Create the script that checks if Nginx is running:
```sh
cat <<'EOF' | sudo tee /etc/keepalived/check_nginx.sh > /dev/null
#!/bin/sh
if [ -z "$(pidof nginx)" ]; then
  exit 1
fi
EOF
```

Set owner/permission for the script and restart `keepalived`:
```sh
sudo chown nginx:nginx /etc/keepalived/check_nginx.sh
sudo chmod 700 /etc/keepalived/check_nginx.sh 
sudo systemctl restart keepalived
sudo systemctl status keepalived
```

- `k3s1vrrp1` should start as the VRRP master and owns the shared VIP address `10.30.100.100`
- `k3s1vrrp2` should start as the VRRP backup

You can check with the command:
```sh
ip add show dev enp1s0 | grep inet | grep -v inet6
```

Output for the primary:
```
    inet 10.30.100.101/24 brd 10.30.100.255 scope global enp1s0
    inet 10.30.100.100/24 scope global secondary enp1s0
```

Output for the secondary:
```
    inet 10.30.100.102/24 brd 10.30.100.255 scope global enp1s0
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

# Verify keepalived and VRRP
## Logs
### Filter Service
The `-u` flag is used to specify the service you are looking for:
```sh
journalctl -u keepalived.service
```

### View log details
Detailed messages with explanations can be viewed by adding the `-x` to help you understand the logs:
```sh
journalctl -u keepalived.service -x
```

## View logs between a given time period
```sh
journalctl -u keepalived.service  --since "2023-07-23 10:27:00" --until "2023-07-23 10:28:00"
```

View logs from yesterday until now:
```sh
journalctl --since yesterday --until now
```

## IP address
If you check the IP addresses on the master node, you should see the VRRP address:
```sh
ip -brief address show
```

>You won't see the VRRP address for `backup` node in normal situation.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

# Simulate a failure
Let's simulate a failure by shutting the service `keepalived` on the primary node, `k3s1vrrp1`, like so:
```sh
sudo systemctl stop keepalived
```

If you monitor the logs on the backup node, `k3s1vrrp2`, with the command:
```sh
journalctl -f -u keepalived
```

You should see a event like this one:
```
Jul 23 11:13:57 k3s1vrrp2.kloud.lan Keepalived_vrrp[2112]: (VRRP_1) Entering MASTER STATE
```

If you check the IP addresses on the backup node, you should now see the VRRP address:
```sh
ip -brief address show
```

Output on backup node with the master node "down":
```
lo               UNKNOWN        127.0.0.1/8 ::1/128 
enp1s0           UP             10.30.100.102/24 10.30.100.100/24 fe80::5054:ff:fecb:87ff/64 
```

## Arp
From another server, **on the same subnet**, check the *arp* table with the command:
```sh
ip neighbor
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

# References
[GitHub](https://github.com/acassen/keepalived)  
[RedHat Simple Configuratiomn](https://www.redhat.com/sysadmin/keepalived-basics)  
