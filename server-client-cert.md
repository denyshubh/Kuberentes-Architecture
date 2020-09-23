
https://kubernetes.io/docs/setup/best-practices/certificates/

### Generating Client and Server Certificates

In this section we will generate client and server certificates for each Kubernetes component and a client certificate for the Kubernetes admin user.

Kubernetes requires PKI for the following operations:

1. Client certificates for the kubelet to authenticate to the API server <br/>
2. Server certificate for the <strong> API server endpoint</strong><br/>
3. Client certificates for administrators of the cluster to authenticate to the API server<br/>
4. Client certificates for the API server to talk to the kubelets<br/>
5. Client certificate for the API server to talk to etcd<br/>
6. Client certificate/kubeconfig for the controller manager to talk to the API server<br/>
7. Client certificate/kubeconfig for the scheduler to talk to the API server.<br/>
8. Client and server certificates for the front-proxy<br/>

----------------------------------------------------------------------------------- <br/>
1. Kube-API Certificate (Server)<br/>
2. Admin Certificate (Client)<br/>
3. Kubelet Certificate (Client)<br/>
4. Kube-proxy Certificate (Client)<br/>
5. Service-account Certificate<br/>
6. Worker Node Certificates<br/>
7. Kube-controller Certificate<br/>
8. Kube Schedular Certificate<br/>


#### 1.  The Admin Client Certificate

1. We'll create an admin-csr.json file
2. Use cfssl and cfssljson to generate certificate

```
{
    cat > admin-csr.json << EOF
    {
    "CN": "admin",
    "key": {
    "algo": "rsa",
    "size": 2048
    },
    "names":[
    {
    "C":"In",
    "L":"Varanasi",
    "O":"system:masters",
    "OU":"kubernetes the hard way",
    "ST":"UP"
    }
    ]
    }
    EOF
    cfssl gencert -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -profile=kubernetes \
    admin-csr.json | cfssljson -bare admin; 

}
```
OUTPUT:
    [INFO] generate received request
    [INFO] received CSR
    [INFO] generating key: rsa-2048
    [INFO] encoded CSR
    [INFO] signed certificate with serial number 118088315988631785703694264035475787978358103177
    [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
            websites. For more information see the Baseline Requirements for the Issuance and Management
            of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
            specifically, section 10.2.3 ("Information Requirements").

### The Kubelet Client Certificates

Kubernetes uses a special-purpose authorization mode called Node Authorizer, that specifically authorizes API requests made by Kubelets. In order to be authorized by the Node Authorizer, Kubelets must use a credential that identifies them as being in the system:nodes group, with a username of system:node:<nodeName>. In this section you will create a certificate for each Kubernetes worker node that meets the Node Authorizer requirements.

Generate a certificate and private key for each Kubernetes worker node:


### 2. Kube-API server certificate

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
CERT_HOSTNAME=10.32.0.1,172.31.113.231,23bcd35c4e1c.mylabserver.com,172.31.113.73,d29dca59cd1c.mylabserver.com,172.31.122.30,94fbee77961c.mylabserver.com

{
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "Varanasi",
      "O": "Kubernetes",
      "OU": "kubernetes the hard way",
      "ST": "UP"
    }
  ]
}
EOF

cfssl gencert \
  -remote="localhost:8888" \
  -hostname=10.32.0.1,${CERT_HOSTNAME},127.0.0.1,localhost,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}

### 3. kube-proxy client
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "Varansi",
      "O": "system:node-proxier",
      "OU": "kubernetes the hard way",
      "ST": "UP"
    }
  ]
}
EOF

cfssl gencert \
  -remote="localhost:8888" \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}

### 4. service-account key pair 

{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "Varanasi",
      "O": "Kubernetes",
      "OU": "kubernetes the hard way",
      "ST": "UP"
    }
  ]
}
EOF

cfssl gencert \
  -remote="localhost:8888" \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}

### 5. Worker Node Certificate
{
declare -A dict
dict+=([ce589041881c.mylabserver.com]=172.31.127.23 [4b0a9a11df1c.mylabserver.com]=172.31.118.38)

for instance in ${!dict[@]}; do
cat > ${instance}-csr.json << EOF
{
"CN": "system:node:${instance}",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
"C": "IN",
"L": "Varanasi",
"O": "system:nodes",
"OU": "kubernetes the hard way",
"ST": "UP"
}
]
}
EOF
cfssl gencert \
-remote="localhost:8888" \
-hostname="${instance},${dict[${instance}]}" \
-profile="kubernetes" \
${instance}-csr.json | cfssljson -bare ${instance}
done
}
### 6. Kube-Controller

{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "Varanasi",
      "O": "system:kube-controller-manager",
      "OU": "kubernetes the hard way",
      "ST": "UP"
    }
  ]
}
EOF

cfssl gencert \
  -remote="localhost:8888" \
  -profile="kubernetes" \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}

### 7. kube schedular 

{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "Varanasi",
      "O": "system:kube-scheduler",
      "OU": "kubernetes the hard way",
      "ST": "UP"
    }
  ]
}
EOF

cfssl gencert \
  -remote="localhost:8888" \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}



### ----------------- COPY CERTS TO THEIR RESPECTIVE LOCATIONS -----------------

1. COPY THE CERTS OF WORKER NODES

for instance in 172.31.113.231 172.31.113.73; do
    scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done



