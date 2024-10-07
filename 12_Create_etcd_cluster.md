# Set up a High Availability etcd Cluster
`etcd` is distributed, reliable key-value store for the most critical data of a distributed system. It's a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines. It gracefully handles leader elections during network partitions and can tolerate machine failure, even in the leader node.

This guide will cover the instructions for bootstrapping an `etcd` cluster on a X-node cluster from pre-built binaries. This tutorial has nothing to do with Kubernetes but it could eventually be used when installing an H.A Kubernetes Cluster with `External etcd` topology.

# Before you begin
We will be using three Ubuntu Server 24.04 with Linux Kernel 6.11.0.

Prerequisites:
- The three nodes that can communicate with each other on TCP port `2380`.
- The clients of the `etcd` cluster can reach any of them on TCP port `2379`. Here the clients are the Kubernetes Master node or K3s1 Server nodes.
  - TCP port `2379` is the traffic for client requests
  - TCP port `2380` is the traffic for server-to-server communication
  - TCP port `2381` is the traffic for the endpoints `/metrics` and `/health` (Optional)
- Each host must have `systemd` and a `bash` compatible shell installed.
- Some infrastructure to copy files between hosts. For example, `scp` can satisfy this requirement. In the preceding step you should have installed the SSH public key of your bastion host on every etcd VMs.

# Setup an External ETCD Cluster
In this tutorial we will configure a three-node TLS enabled `etcd` cluster that will act as an external datastore for your K3s Kubernetes H.A. Cluster.

|Role|FQDN|IP|OS|Kernel|RAM|vCPU|Node|
|----|----|----|----|----|----|----|----|
|etcd database|k3s1etcd1.kloud.lan|10.30.100.71|Ubuntu 24.04|6.11.0|4G|2|pve1|
|etcd database|k3s1etcd2.kloud.lan|10.30.100.72|Ubuntu 24.04|6.11.0|4G|2|pve1|
|etcd database|k3s1etcd3.kloud.lan|10.30.100.73|Ubuntu 24.04|6.11.0|4G|2|pve1|

> [!NOTE]  
> Everything from here is done on `k8s2bastion1` in the directory `~/k3s1/`. It is **VERY** important that you backup this directory and keep it secure since it will contain the TLS certificates for the communication between the etcd members.

# Terminal Multiplexer
I'm using `tmux` to access all the VMs at the same time. Create the following script to start a session on each VM. Adjust the variable `ssh_list`:
```sh
FILE="k3s1-etcd.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash

ssh_list=( k3s1etcd1 k3s1etcd2 k3s1etcd3 )

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

# Download the binaries
Install the latest version of the `etcd` binaries on **each** of the Linux `etcd` node. Three binaries will be installed on each server in the directory `/usr/local/bin/`.
- etcd: The etcd server
- etcdctl: a command line tool for interacting with the etcd server
- etcdutl: a command line administration utility for etcd server

> [!IMPORTANT]  
> The commands below are executed on **every** etcd host. I used `tmux`.

## Install `ETCD` from binaries
```sh
ARCH=$(dpkg --print-architecture)
VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
printf "etcd version: %s for ARCH:%s\n" "${VER}" "${ARCH}"
curl -LO https://github.com/etcd-io/etcd/releases/download/${VER}/etcd-${VER}-linux-${ARCH}.tar.gz
tar xvf etcd-${VER}-linux-${ARCH}.tar.gz
cd etcd-${VER}-linux-${ARCH}
sudo install etcd etcdctl etcdutl -m 755 -o root -g adm /usr/local/bin/
```

## Verify installation
Check the binaries you just installed:
```sh 
etcd --version
etcdctl version
etcdutl version
```

## Cleanup
Remove the temporary directory and files created in the above step:
```sh
cd ..
rm -rf etcd-${VER}-linux-${ARCH}
rm etcd-${VER}-linux-${ARCH}.tar.gz
unset VER
```

> [!IMPORTANT]  
> Return to the bastion host. just type `exit` in the `tmux` terminal.

# Generating and Distributing TLS Certificates
The following can be done on any server. I used one of my `bastion` host.

We will use the `openssl` tool to generate our own CA and all the `etcd` server certificates and keys. You can use your own certificate manager.

## Define Variables
Define all the variables needed for the scripts below. Make sure you execute the scripts in the same window as the one you defined the variables.
```sh
unset newVM
declare -A newVM
newVM[k3s1etcd1]="10.30.100.71"
newVM[k3s1etcd2]="10.30.100.72"
newVM[k3s1etcd3]="10.30.100.73"
```

## Generate Private CA
This script will generate a CA root certificate with its private key. The argument is the prefix of both files created. This is a simple certificate of authority only for my etcd cluster.
```sh
dnsDomain="kloud.lan"
export CA_CERT="k3s1-etcd-ca"

printf "\nMaking ETCD Root CA...\n"
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:secp521r1 -pkeyopt ec_param_enc:named_curve -out ${CA_CERT}.key

openssl req -new -sha256 -x509 -key ${CA_CERT}.key -days 7300 \
-subj "/C=CA/ST=QC/L=Montreal/O=RootCA/OU=${CA_CERT}/CN=k3s1etcd.${dnsDomain}" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:${CA_CERT}.${dnsDomain},IP:127.0.0.1" \
-addext "basicConstraints = critical,CA:TRUE" \
-addext "keyUsage = critical, digitalSignature, cRLSign, keyCertSign" \
-addext "subjectKeyIdentifier = hash" \
-addext "authorityKeyIdentifier = keyid:always, issuer" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out ${CA_CERT}.crt

# Print the certificate
# openssl x509 -text -noout -in ${CA_CERT}.crt
# Verification of certificate and private key. The next 2 checksum must be identical!
printf "The checksums MUST be identical\n"
openssl x509 -pubkey -in ${CA_CERT}.crt -noout | openssl sha256 | awk -F '= ' '{print $2}'
openssl pkey -pubout -in ${CA_CERT}.key | openssl sha256 | awk -F '= ' '{print $2}'
```

This results in two files:
- The file `k3s1-etcd-ca.crt` is the CA certificate.
- The file `k3s1-etcd-ca.key` is the CA private key. Keep this file in a safe place.

## Generate Node Certificates
You could use the same client certificate of every `etcd` node and it will work fine. I've decided to use different certificate for each node. The following script will generate the certificate and key for every `etcd` node in the cluster.

```sh
cat <<'EOF' > gen_cert.sh
#!/bin/bash

# Generate a new private/public key pair and a certificate for a server.
#
# ***** This script assume the following: *****
#     Certificate filename: <name>.crt
#     CSR filename:         <name>.csr
#     Private key filename: <name>.key
#
# This is a bit dirty, but helps when you want to generate lots of certificate
# for testing.
#
# HOWTO use it: ./gen_cert.sh k8setcd1 172.31.11.10 etcd-ca
#
# This will generate the following files:
#  -rw-r--r--   1 username  staff  1216 01 Jan 00:00 server.crt >> certificate
#  -rw-r--r--   1 username  staff   976 01 Jan 00:00 server.csr >> CSR
#  -rw-------   1 username  staff   302 01 Jan 00:00 server.key >> key pair
#
# Works with 'bash' and 'zsh' shell on macOS and Linux. Make sure you have OpenSSL *** in your PATH ***.
#
# *WARNING*
# This script was made for educational purposes ONLY.
# USE AT YOUR OWN RISK!"
DOMAIN='kloud.lan'
EXTRA_SAN=''

if [[ $# -lt 3 || $# -gt 4 ]]
then
   printf "\nUsage: %s <filename prefix> <IP address> <filename prefix of CA>\n" $(basename $0)
   printf "Ex.: %s k8s2etcd1 172.31.11.10 etcd-ca\n\n" $(basename $0)
   exit -1
fi

# Undocumented argument 😉
EXTRA_SAN=$4

printf "\nMaking certificate: $1 ...\n"
openssl ecparam -name prime256v1 -genkey -out $1.key
# Comment the line above and uncomment the line below if you want an RSA key
# openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out $1.key

openssl req -new -sha256 -key $1.key -subj "/C=CA/ST=QC/L=Montreal/O=$1/OU=IT/CN=$1.$DOMAIN" \
-addext "subjectAltName = DNS:localhost,DNS:*.localhost,DNS:$1,DNS:$1.$DOMAIN,IP:127.0.0.1,IP:$2$EXTRA_SAN" \
-addext "basicConstraints = CA:FALSE" \
-addext "extendedKeyUsage = clientAuth,serverAuth" \
-addext "subjectKeyIdentifier = hash" \
-addext "keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment" \
-addext "authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp" \
-addext "crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8080/crl/Intermediate-CA.crl" \
-out $1.csr

openssl x509 -req -sha256 -days 365 -in $1.csr -CA $3.crt -CAkey $3.key -CAcreateserial \
-extfile - <<<"subjectAltName = DNS:localhost,DNS:*.localhost,DNS:$1,DNS:$1.$DOMAIN,IP:127.0.0.1,IP:$2$EXTRA_SAN
basicConstraints = CA:FALSE
extendedKeyUsage = clientAuth,serverAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid, issuer
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyAgreement, dataEncipherment
authorityInfoAccess = caIssuers;URI:http://localhost:8000/Intermediate-CA.cer,OCSP;URI:http://localhost:8000/ocsp
crlDistributionPoints = URI:http://localhost:8000/crl/Root-CA.crl,URI:http://localhost:8000/crl/Intermediate-CA.crl" \
-out $1.crt

# To verify that etcd node certificate $1.crt is the CA $3.crt:
# openssl verify -no-CAfile -no-CApath -partial_chain -trusted %3.crt $1.crt

# Verification of certificate and private key. The next 2 checksum must be identical!
CRT_PUB=$(openssl x509 -pubkey -in $1.crt -noout | openssl sha256 | awk -F '= ' '{print $2}')
KEY_PUB=$(openssl pkey -pubout -in $1.key | openssl sha256 | awk -F '= ' '{print $2}')

if [[ "${CRT_PUB}" != "${KEY_PUB}" ]]
then
   printf "\nERROR: Public Key of certificate [%s] doesn't match Public Key of private key [%s].\n" $1.crt $1.key
   exit -1
else
   printf "\nSUCCESS: Public Key of certificate [%s] matches Public Key of private key [%s].\n" $1.crt $1.key
   exit 0
fi
EOF
chmod +x gen_cert.sh
```

```sh
for NEW in "${!newVM[@]}"
do
  ./gen_cert.sh "${NEW}" "${newVM[${NEW}]}" "${CA_CERT}"
done

# Generate the certificate for the server node
./gen_cert.sh k3s1server1 10.30.100.51 k3s1-etcd-ca ',DNS:k3s1server1,DNS:k3s1server2,DNS:k3s1server3,IP:10.30.100.51,IP:10.30.100.52,IP:10.30.100.53'
rm -f *.csr
```

I copied the master node certificate and key to `apiserver-etcd-client.{crt,key}` since they will be client certificates on each of the K3s Server node to authenticate on any `etcd` server.
```sh
cp k3s1server1.crt apiserver-etcd-client.crt
cp k3s1server1.key apiserver-etcd-client.key
```

> [!IMPORTANT]  
> Keep those files as they will be required on all K3s server nodes.

The last command is for our Kubernetes Control Plane nodes:

This results in two files per `etcd` node. The `.crt` is the certificate and the `.key` is the private key:
- k3s1etcd1.crt
- k3s1etcd1.key
- k3s1etcd2.crt
- k3s1etcd2.key
- k3s1etcd3.crt
- k3s1etcd3.key
- k3s1server1.crt
- k3s1server1.key
- apiserver-etcd-client.crt
- apiserver-etcd-client.key

> [!IMPORTANT]  
> The private keys are not encrypted as `etcd` needs a non-encrypted `pem` file.

At this point, we have the certificates and keys generated for the CA and all the three `etcd` nodes. The nodes certifcate has a SAN with the hostname, FQDN and IP address. See example below for node `k3s1etcd1`
- DNS:k3s1etcd1
- DNS:k3s1etcd1.kloud.lan
- IP Address:10.30.100.71

# Distribute Certificates
We need to to distribute those certificates and keys to each `etcd` node in the cluster. I've made a script. Adusts the nodes and execute.

> [!IMPORTANT]  
> - Adjust the variables `ETCD_NODES` and `CA_CERT` in the script below.
> - It is assume that SSH doesn't require a password.

```sh
FILE="scp-etcd.sh"
cat <<'EOF' > ${FILE}
#!/bin/bash
#
# This script copy the certificates and private keys to each etcd node.
#
# HOWTO use it: ./${FILE}
#
# *WARNING*
# This script was made for educational purposes ONLY.
# USE AT YOUR OWN RISK!"

# Modify the variables to reflect your environment
USER=daniel
ETCD_NODES=( k3s1etcd1 k3s1etcd2 k3s1etcd3 )
CA_CERT=${CA_CERT}.crt

for NODE in "${ETCD_NODES[@]}"; do
  printf "Copying files on: %s\n" ${NODE}
  scp -oStrictHostKeyChecking=accept-new ${CA_CERT} ${USER}@${NODE}:/home/${USER}/.
  scp -oStrictHostKeyChecking=accept-new ${NODE}.crt ${USER}@${NODE}:/home/${USER}/.
  scp -oStrictHostKeyChecking=accept-new ${NODE}.key ${USER}@${NODE}:/home/${USER}/.
  printf "\n"
done
EOF

chmod +x ${FILE}
./${FILE}
```

# Move Certificate and Key
SSH into each `etcd` node and run the below commands to move the certificate and key into the `/etc/etcd/pki` directory. I will be using `tmux` to run the commands on every node at the same time. Just paste the following in each node. The certificates and keys have the prefix of the short hostname. The variable `ETCD_NAME` will be different on each node.

> [!NOTE]  
> I'm using `tmux` to paste the commands in all `etcd` server at the same time.
> Adjust the variable ${CA_CERT}

```sh
# Adjust this variable
CA_CERT="k3s1-etcd-ca"
ETCD_NAME=$(hostname -s)
echo ${ETCD_NAME}
sudo mkdir -p /etc/etcd/pki
sudo mv ${CA_CERT}.crt /etc/etcd/pki/.
sudo mv ${ETCD_NAME}.{crt,key} /etc/etcd/pki/.
sudo chmod 600 /etc/etcd/pki/${ETCD_NAME}.key
sudo chown root:root /etc/etcd/pki/*
```

We have generated and copied all the certificates/keys on each node. In the next step, we will create the configuration file and the `systemd` unit file for each node.

# Create `etcd` configuration file
This is the configuration file `/etc/etcd/etcd.conf` that needs to be copied on each node. The command can be pasted simultaneously on all the nodes. I will be using `tmux` again. The file below will be different for every etcd nodes.
```sh
ETCD_CLUSTER_NAME=k3s-etcd-cluster-1
ETCD_IP=$(hostname -I | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
ETCD_NAME=$(hostname -s)
ETCD1_IP=10.30.100.71
ETCD2_IP=10.30.100.72
ETCD3_IP=10.30.100.73

cat <<EOF | sudo tee /etc/etcd/etcd.conf > /dev/null
# Human-readable name for this member.
name: "${ETCD_NAME}"

# List of comma separated URLs to listen on for peer traffic.
listen-peer-urls: "https://${ETCD_IP}:2380"

# List of comma separated URLs to listen on for client traffic.
listen-client-urls: "https://${ETCD_IP}:2379,https://127.0.0.1:2379"

# List of additional URLs to listen on, will respond to /metrics and /health endpoints
listen-metrics-urls: "http://${ETCD_IP}:2381,http://127.0.0.1:2381"

# Initial cluster token for the etcd cluster during bootstrap.
initial-cluster-token: o3ZBeUqBgjAMArh8c5BQmuK

# Comma separated string of initial cluster configuration for bootstrapping.
initial-cluster: "k3s1etcd1=https://${ETCD1_IP}:2380,k3s1etcd2=https://${ETCD2_IP}:2380,k3s1etcd3=https://${ETCD3_IP}:2380"

# List of this member's peer URLs to advertise to the rest of the cluster.
# The URLs needed to be a comma-separated list.
initial-advertise-peer-urls: "https://${ETCD_IP}:2380"

# List of this member's client URLs to advertise to the public.
# The URLs needed to be a comma-separated list.
advertise-client-urls: "https://${ETCD_IP}:2379,https://127.0.0.1:2379"

client-transport-security:
  # Path to the client server TLS cert file.
  cert-file: "/etc/etcd/pki/${ETCD_NAME}.crt"

  # Path to the client server TLS key file.
  key-file: "/etc/etcd/pki/${ETCD_NAME}.key"

  # Path to the client server TLS trusted CA cert file.
  trusted-ca-file: "/etc/etcd/pki/${CA_CERT}.crt"

  # Enable client cert authentication.
  client-cert-auth: true

peer-transport-security:
  # Path to the peer server TLS cert file.
  cert-file: "/etc/etcd/pki/${ETCD_NAME}.crt"

  # Path to the peer server TLS key file.
  key-file: "/etc/etcd/pki/${ETCD_NAME}.key"

  # Enable peer client cert authentication.
  client-cert-auth: true

  # Path to the peer server TLS trusted CA cert file.
  trusted-ca-file: "/etc/etcd/pki/${CA_CERT}.crt"

# Path to the data directory.
data-dir: "/var/lib/etcd"

# Initial cluster state ('new' or 'existing').
initial-cluster-state: 'new'
EOF
```

> [!IMPORTANT]  
> Any locations specified by `listen-metrics-urls` will respond to the `/metrics` and `/health` endpoints in `http` only.
> This can be useful if the standard endpoint is configured with mutual (client) TLS authentication (`listen-client-urls: "https://...`), but a load balancer or monitoring service still needs access to the health check with `http` only.
> The endpoint `listen-client-urls` still answers to `https://.../metrics`.

# Configuring and Starting the `etcd` Cluster
On every node, create the file `/etc/systemd/system/etcd.service` with the following contents. I will be using `tmux` to execute the command once on all the `etcd` servers:
```sh
cat <<EOF | sudo tee /lib/systemd/system/etcd.service > /dev/null
[Unit]
Description=etcd is a strongly consistent, distributed key-value store database
Documentation=https://github.com/etcd-io/etcd
After=network-online.target local-fs.target remote-fs.target time-sync.target
Wants=network-online.target local-fs.target remote-fs.target time-sync.target

 
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf
Restart=always
RestartSec=5s
LimitNOFILE=40000
 
[Install]
WantedBy=multi-user.target
EOF
```

> [!NOTE]  
> The `etcd` service file can be found [here](https://github.com/etcd-io/etcd/blob/main/contrib/systemd/etcd.service)

# Start `etcd` service
Start the `etcd` service on all VMs, again using `tmux`:
```sh
sudo systemctl daemon-reload
sudo systemctl enable --now etcd
sudo systemctl status etcd
```

# Testing and Validating the Cluster
To interact with the cluster we will be using `etcdctl`. It's the utility to interact with the `etcd` cluster. This utility as been installed in `/usr/local/bin` on all `etcd` servers. Let's also install it on our bastion host.

Install `etcdctl` on the bastion host:
```sh
# Install both etcd utilities, etcdctl and etcdutl, in "/usr/local/bin/"
ARCH=$(dpkg --print-architecture)
VER=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
printf "etcd version: %s for ARCH:%s\n" "${VER}" "${ARCH}"
curl -LO https://github.com/etcd-io/etcd/releases/download/${VER}/etcd-${VER}-linux-${ARCH}.tar.gz
tar xvf etcd-${VER}-linux-${ARCH}.tar.gz
cd etcd-${VER}-linux-${ARCH}
sudo install etcdctl etcdutl -m 755 -o root -g adm /usr/local/bin/
cd ..
rm -rf etcd-${VER}-linux-${ARCH}
rm etcd-${VER}-linux-${ARCH}.tar.gz
unset VER
```

You can export these environment variables and connect to the clutser without specifying the values each time on the command line.
- Pick any one of the etcd server, it doesn't matter which one
- You need to have the CA TLS certificate used to sign the `etcd` TLS server certificate
- You need to be in the directory where you have the TLS certificate and private key for the `etcd` server

```sh
# Pick any one of the etcd server, it doesn't matter which one
ETCD_NAME=k3s1etcd1
export ETCDCTL_ENDPOINTS=https://k3s1etcd1.kloud.lan:2379,https://k3s1etcd2.kloud.lan:2379,https://k3s1etcd3.kloud.lan:2379
export ETCDCTL_CACERT=./k3s1-etcd-ca.crt
export ETCDCTL_CERT=./${ETCD_NAME}.crt
export ETCDCTL_KEY=./${ETCD_NAME}.key
```

> [!NOTE]  
> The variable `export ETCDCTL_API=3` is not needed anymore with version >3.4.x  
> You can use any of the three client certificate/key with `ETCDCTL_CERT` and `ETCDCTL_KEY` because they are all signed by the same CA.  
> You can generate another certificate/key pair for your bastion host, as long as the certificate is signed by the `k3s1-etcd-ca`.  

## Check Cluster status
To execute the next command, you can be on any host that:
- can reach the `etcd` servers on port `TCP/2379`
- has the client certificate, the CA certificate and private key

And now its a lot easier
```sh
etcdctl --write-out=table member list
etcdctl --write-out=table endpoint status
etcdctl --write-out=table endpoint health
```

See below for the ouput of the three commands above:
```
etcdctl --write-out=table member list
+------------------+---------+-----------+---------------------------+--------------------------------------------------+------------+
|        ID        | STATUS  |   NAME    |        PEER ADDRS         |                   CLIENT ADDRS                   | IS LEARNER |
+------------------+---------+-----------+---------------------------+--------------------------------------------------+------------+
| 28d3f24ed962993d | started | k3s1etcd2 | https://10.30.100.72:2380 | https://10.30.100.72:2379,https://127.0.0.1:2379 |      false |
| c22184bdcacf3141 | started | k3s1etcd1 | https://10.30.100.71:2380 | https://10.30.100.71:2379,https://127.0.0.1:2379 |      false |
| c7eb0db5fde51895 | started | k3s1etcd3 | https://10.30.100.73:2380 | https://10.30.100.73:2379,https://127.0.0.1:2379 |      false |
+------------------+---------+-----------+---------------------------+--------------------------------------------------+------------+
```

```
etcdctl --write-out=table endpoint status
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|             ENDPOINT             |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://k3s1etcd1.kloud.lan:2379 | c22184bdcacf3141 |  3.5.16 |   20 kB |     false |      false |         5 |         14 |                 14 |        |
| https://k3s1etcd2.kloud.lan:2379 | 28d3f24ed962993d |  3.5.16 |   20 kB |     false |      false |         5 |         14 |                 14 |        |
| https://k3s1etcd3.kloud.lan:2379 | c7eb0db5fde51895 |  3.5.16 |   20 kB |      true |      false |         5 |         14 |                 14 |        |
+----------------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

> [!NOTE]  
> Raft is leader-based; the leader handles all client requests which need cluster consensus. However, the client does not need to know which node is the leader. Any request that requires consensus sent to a follower is automatically forwarded to the leader. Requests that do not require consensus (e.g., serialized reads) can be processed by any cluster member.

```
etcdctl --write-out=table endpoint health
+----------------------------------+--------+-------------+-------+
|             ENDPOINT             | HEALTH |    TOOK     | ERROR |
+----------------------------------+--------+-------------+-------+
| https://k3s1etcd3.kloud.lan:2379 |   true |  19.36339ms |       |
| https://k3s1etcd1.kloud.lan:2379 |   true | 19.923875ms |       |
| https://k3s1etcd2.kloud.lan:2379 |   true | 28.707317ms |       |
+----------------------------------+--------+-------------+-------+
```

>For the `--endpoints`, enter all of your nodes. I used the short hostname, FQDN and IP address.

## Check the logs
> [!NOTE]  
> You need to be on an `etcd` host to execute the following command:

Check for any warnings/errors on every nodes:
```sh
journalctl -xeu etcd.service
```

## Test Metrics
```sh
METRIC='http://10.30.100.71:2381'
curl -L ${METRIC}/metrics
```

Check the `/health` endpoint for all your nodes:
```sh
ETCD_NODES=( k3s1etcd1 k3s1etcd2 k3s1etcd3 ); for NODE in "${ETCD_NODES[@]}"; do curl -L http://${NODE}:2381/health; echo ""; done
```

> [!IMPORTANT]  
> If you get a `Connection refused` on the node where the script is run, it used IP address `127.0.1.1` instead of `127.0.0.1`.
> Use the FQDN to test it.

```
127.0.0.1 localhost k8s2etcd1
127.0.1.1 k3s1etcd1
...
```

```
{"health":"true","reason":""}
{"health":"true","reason":""}
{"health":"true","reason":""}
```

# Congratulation
You should have a three-node `etcd` cluster in **High Availibility** mode 🍾🎉🥳

# References
[etcd main website](https://etcd.io/)  
[Set up a High Availability etcd Cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/setup-ha-etcd-with-kubeadm/)  
[Options for Highly Available Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)  
[Configuration file for etcd server](https://github.com/etcd-io/etcd/blob/main/etcd.conf.yml.sample)  
