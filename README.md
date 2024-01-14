# k8s-hard-way

In this setup to create a k8s cluster from scratch, we have provissioned 3 droplets; 1 CP 1 Worker and 1 for load balancer.
We then go on to configure them to create a pure vanilla k8s cluster using 3 nodes. 
We are using this documentation for [kubenetes the hard way](https://medium.com/@DrewViles/kubernetes-the-hard-way-on-bare-metal-vms-setting-up-the-controllers-d5e4c5d47bcd).

### Certificates

- ~/ca: Contains CA certs and config files
- ~/admin: certs for kubectl
- ~/clients: worker node certs
- ~/controller: kube-controler certs
- ~/proxy: kube-proxy certs
- ~/scheduler: kube-scheduler certs
- ~/front-proxy: certs so that kube-proxy can support extension API certs
- ~/api: These are used so that the kube-api-server can be authenticated against.
- ~/service-account: These are used so that service accounts can authenticate with the kube-api-server.

Certiicates are generated using the PKI management tool, `cfssl` and its complementary `cfssljson` to extract the certs and private-keys.
Here we are generating a CSR for controller manager for a given CA using cfssl and then extracting certs and keys using cfssljson.

```
cat > pki/controller/kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "${TLS_C}",
      "L": "${TLS_L}",
      "O": "system:kube-controller-manager",
      "OU": "${TLS_OU}",
      "ST": "${TLS_ST}"
    }
  ]
}
EOF
cfssl gencert \
  -ca=pki/ca/ca.pem \
  -ca-key=pki/ca/ca-key.pem \
  -config=pki/ca/ca-config.json \
  -profile=kubernetes \
  pki/controller/kube-controller-manager-csr.json | cfssljson -bare pki/controller/kube-controller-manager
```

### Making kubeconfigs for control-plane and worker node

The following `kubectl` commands configure the kubeconfig for worker node, for example.
- set-cluster: Sets the cluster information.
- set-credential: Sets the user credential for the user `system:node:${instance}`.
- set-context: Sets the context for a user and cluster.
- use-context: Sets the current-context in the kubeconfig file.
```
kubectl config set-cluster Drewbernetes\
    --certificate-authority=pki/ca/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-credentials system:node:${instance} \
    --client-certificate=pki/clients/${instance}.pem \
    --client-key=pki/clients/${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config set-context default \
    --cluster=Drewbernetes \
    --user=system:node:${instance} \
    --kubeconfig=configs/clients/${instance}.kubeconfig
kubectl config use-context default --kubeconfig=configs/clients/${instance}.kubeconfig
```

To encrypt the data b/w nodes, use:
```
cat > data-encryption/encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: ${ENCRYPTION_KEY}
    - identity: {}
EOF
```

### Setting up CP components

We set the following componets in the CP node:
- etcd
- api-server
- controller-manager
- scheduler

The general process involves installing the binaries, making them as systemd services with proper configuration and kubeconfigs and starting these service daemons.
We then use `nginx` to act as a reverse-proxy for `healtz` checks. 
We also create a `ClusterRole` and `ClusterRoleBinding` to falicitate api-to-kubelet communication.

### Setting up the LB node

We need to load balance the externel traffic to the cluster. We have a pool of CP nodes and we need to make them L7 enabled for traffic routing.

> **ASIDE NOTE**: This is a very good article for understanding HAproxy, load balancing terminologies and diffeernce b/w L4 and L7 load balancers: https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts.
In simple terms, L4 load balancing is routing of traffic based on IP and port of the while L7 load balancing depends on routing based in content path (eg /about or /blog path) of user but are under the same IP and port.

After *apt* installing HAProxy in the lb server, we generate a self signed certificate in the node:
```
openssl req  -nodes -new -x509  -keyout /etc/haproxy/api_key.pem -out /etc/haproxy/api_cert.pem -days 365
```
and combine the key and cert in one file called `/etc/haproxy/k8s_api.pem` which is later set into the HAProxy config.

> Important step here is to set the `Common Name` to your domain name (or just the node hostname and a domain like '*hostname.domain*') and set this Common Name in your `/etc/hosts` file on your local machine to map to your load balancer node.

Set the HAProxy configs to add the CP nodes in the backend pool and restart haproxy.
Hit `curl -k https://hostname.domain/healthz` and expect `ok`.

### Setting up worker nodes

> ASIDE: `Containerd` uses `runc` as its default container execution driver. This means that when containerd needs to run a container, it leverages runc to perform the actual container execution, even though both are container runtimes.

- CNI: Install CNI with bridge and loopback network configs.
- Containerd: Make the containerd configs (making `runc` as the underlying container execution engine) and  a containerd service.
- Kubelet: Make the configs and the systemd service.
- Kube-Proxy: Make the config and systemd service.

Before running these services turn the swap off `swapoff -a` to ensure that your worker nodes have sufficient physical memory to handle the workload without relying on swap which can cause unpredictable memory issues and hence reduced stability. If you run nodes with (traditional to-disk) swap, you lose a lot of the isolation properties that make sharing machines viable. You have no predictability around performance or latency or IO:
```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

### Remote access from local machine
generated the certs, kubeconfigs etc to use kubectl client to communicate with the remote deployment.
```
kubectl config set-cluster Drewbernetes \
        --certificate-authority=pki/ca/ca.pem \
        --embed-certs=true \
        --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config set-credentials admin \
        --client-certificate=pki/admin/admin.pem \
        --client-key=pki/admin/admin-key.pem \
        --embed-certs=true \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config set-context Drewbernetes \
        --cluster=Drewbernetes \
        --user=admin \
        --kubeconfig=configs/admin/admin-remote.kubeconfig
kubectl config use-context Drewbernetes --kubeconfig=configs/admin/admin-remote.kubeconfig
```

### Pod-to-pod communication
To falicitate pod-to-pod communication, we use calico as the CNI plugin. More on k8s networking [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model).

Get the `calico-etcd.yaml` and modify the `etcd-*` vars on it.
```
curl https://docs.projectcalico.org/manifests/calico-etcd.yaml -o calico.yaml
# Set ETCD IPs
sed -i 's/etcd_endpoints:\ "http:\/\/<ETCD_IP>:<ETCD_PORT>"/etcd_endpoints:\ "https:\/\/192.168.0.110:2379,https:\/\/192.168.0.111:2379,https:\/\/192.168.0.112:2379"/g' calico.yaml
# Set Certificate data in secret
sed -i "s/# etcd-cert: null/etcd-cert: $(cat pki\/api\/kubernetes.pem | base64 -w 0)/g" calico.yaml
sed -i "s/# etcd-key: null/etcd-key: $(cat pki\/api\/kubernetes-key.pem | base64 -w 0)/g" calico.yaml
sed -i "s/# etcd-ca: null/etcd-ca: $(cat pki\/ca\/ca.pem | base64 -w 0)/g" calico.yaml
# Setup Config map with secret information
sed -i "s/etcd_ca: \"\"/etcd_ca: \"\/calico-secrets\/etcd-ca\"/g" calico.yaml
sed -i "s/etcd_cert: \"\"/etcd_cert: \"\/calico-secrets\/etcd-cert\"/g" calico.yaml
sed -i "s/etcd_key: \"\"/etcd_key: \"\/calico-secrets\/etcd-key\"/g" calico.yaml
# Setup the POD CIDR ENV
sed -i 's/# - name: CALICO_IPV4POOL_CIDR/- name: CALICO_IPV4POOL_CIDR/g' calico.yaml
sed -i 's/#   value: "192.168.0.0\/16"/  value: "10.200.0.0\/16"/g' calico.yaml
```

### Configure DNS
We will use coredns for DNS in cluster (a tatoulogy).
```
curl https://raw.githubusercontent.com/coredns/deployment/coredns-1.14.0/kubernetes/coredns.yaml.sed -o coredns.yaml.sed
curl https://raw.githubusercontent.com/coredns/deployment/coredns-1.14.0/kubernetes/deploy.sh -o deploy.sh
bash ./deploy.sh -s -i 10.32.0.10 > coredns.yaml
kubectl apply -f coredns.yaml
```

Check the working of dns: `kubectl exec -ti dnsutils -- nslookup kubernetes`

And we'er done with a basic 3 node cluster.
