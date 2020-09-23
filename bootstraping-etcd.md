# USE TMUX TO SET ETCD CLUSTER

### 1. How to get public and private Ip dynamically of aws ec2 instance
curl http://169.254.169.254/latest/meta-data/public-ipv4
curl http://checkip.amazonaws.com
curl http://169.254.169.254/latest/meta-data/local-ipv4


### 2. Install the etcd Binary on Both Control Nodes

wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.10/etcd-v3.4.10-linux-amd64.tar.gz"

### 3. Extract the etcd archive & move it to /usr/local/bin dir
{
tar -xvf etcd-v3.4.10-linux-amd64.tar.gz
sudo mv etcd-v3.4.10-linux-amd64/etcd* /usr/local/bin/
}

### 4. create the /etc/etcd and /var/lib/etcd directories
{
  cd ~/certs/
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo chmod 700 /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}

### 5. Set up Environment Variables
Each etcd member must have a unique name within an etcd cluster. Set the etcd name to match the hostname of the current compute instance:
```
# export DNS_NAME=$(hostname -s).mylabserver.com
ETCD_NAME=$(hostname -s)
INTERNAL_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)
CONTROLLER_0_INTERNAL_IP=172.31.113.231
CONTROLLER_1_INTERNAL_IP=172.31.113.73
INITIAL_CLUSTER=23bcd35c4e1c=https://${CONTROLLER_0_INTERNAL_IP}:2380,d29dca59cd1c=https://${CONTROLLER_1_INTERNAL_IP}:2380
```

### 6. Create the etcd.service systemd unit file

cat << EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos/etcd

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${INITIAL_CLUSTER}\\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

### 7. start the etcd server


{
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
}

### 8. See logs of the etcd server
{
sudo systemctl status etcd.service -l --no-pager
sudo journalctl -u etcd.service -l --no-pager|less
sudo journalctl -f -u etcd.service
}

### Verify that the etcd cluster is working correctly.
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem

```
ID,               Status,  Name,         Peer Addrs,                 Client Addrs,               Is Learner
9b51ef0271a921a1, started, d29dca59cd1c, https://172.31.113.73:2380, https://172.31.113.73:2379, false
cd8f7c754d10e2a7, started, 23bcd35c4e1c, https://172.31.113.231:2380, https://172.31.113.231:2379, false
```
this command prints out comma-separated member lists for each endpoint.
        The items in the lists are ID, Status, Name, Peer Addrs, Client Addrs, Is Learner.

As you can see from above the last column is for Is Learner which is false for all of your nodes. ETCD version 3.4 introduces a new node state “Learner”, which joins the cluster as a non-voting member until it catches up to leader’s logs. This means the learner still receives all updates from leader, while it does not count towards the quorum, which is used by the leader to evaluate peer activeness. The learner only serves as a standby node until promoted. This relaxed requirements for quorum provides the better availability during membership reconfiguration and operational safety.

https://etcd.io/docs/v3.3.12/learning/learner/