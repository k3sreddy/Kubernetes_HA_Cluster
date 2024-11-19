
# Comprehensive Kubernetes HA Cluster Setup Guide for Air-gapped Environments

## Table of Contents

1. [Introduction to Kubernetes and HA Clusters](#1-introduction-to-kubernetes-and-ha-clusters)
2. [Prerequisites](#2-prerequisites)
3. [Air-gapped Environment Setup](#3-air-gapped-environment-setup)
   3.1 [Setting up a Local Package Repository](#31-setting-up-a-local-package-repository)
   3.2 [Setting up a Local Container Registry](#32-setting-up-a-local-container-registry)
   3.3 [Configuring Nodes to Use Local Resources](#33-configuring-nodes-to-use-local-resources)
4. [Setting up the Load Balancer (HAProxy)](#4-setting-up-the-load-balancer-haproxy)
5. [Creating the ETCD Cluster](#5-creating-the-etcd-cluster)
   5.1 [Preparing ETCD Nodes](#51-preparing-etcd-nodes)
   5.2 [Configuring ETCD](#52-configuring-etcd)
   5.3 [Creating the ETCD Systemd Service](#53-creating-the-etcd-systemd-service)
   5.4 [Starting and Enabling ETCD](#54-starting-and-enabling-etcd)
   5.5 [Verifying the ETCD Cluster](#55-verifying-the-etcd-cluster)
6. [Setting up the Kubernetes Control Plane](#6-setting-up-the-kubernetes-control-plane)
7. [Adding Worker Nodes](#7-adding-worker-nodes)
8. [Configuring Networking](#8-configuring-networking)
9. [Setting up Dev/Staging/Production Environments](#9-setting-up-devstaging-production-environments)
10. [Real-time Project Examples](#10-real-time-project-examples)
11. [Additional Kubernetes Topics](#11-additional-kubernetes-topics)
12. [Troubleshooting and Best Practices](#12-troubleshooting-and-best-practices)
13. [Conclusion](#13-conclusion)
14. [Architectural Diagrams](#14-architectural-diagrams)
    14.1 [Overall Kubernetes HA Cluster Architecture](#141-overall-kubernetes-ha-cluster-architecture)
    14.2 [Air-gapped Environment Setup](#142-air-gapped-environment-setup)
    14.3 [CI/CD Pipeline in Air-gapped Environment](#143-cicd-pipeline-in-air-gapped-environment)

## 1. Introduction to Kubernetes and HA Clusters

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. A High Availability (HA) Kubernetes cluster ensures that your applications remain available and resilient, even in the face of hardware failures or other issues.

This guide will walk you through the process of setting up a production-ready Kubernetes HA cluster in an air-gapped environment. An air-gapped environment is isolated from the public internet, which presents unique challenges but also offers enhanced security. This setup is suitable for organizations with strict security requirements or those operating in environments with limited or no internet access.

Key concepts covered in this guide include:
- Air-gapped environment setup
- High Availability cluster architecture
- Load balancing with HAProxy
- ETCD cluster configuration
- Kubernetes control plane and worker node setup
- Networking in air-gapped Kubernetes clusters
- DevOps practices in isolated environments

## 2. Prerequisites

Before starting the setup process, ensure you have the following:

### 2.1 Hardware Requirements

- Three or more machines for control-plane nodes (each with at least 2 CPUs, 4GB RAM, and 50GB storage)
- Three or more machines for worker nodes (resources depend on your workload requirements)
- One machine for the load balancer (2 CPUs, 4GB RAM, and 20GB storage)
- Three machines for the ETCD cluster (can be the same as control-plane nodes for a stacked topology)
- All machines should be running Rocky Linux 9.4

### 2.2 Network Requirements

- Full network connectivity between all machines in the cluster
- A private network for inter-node communication
- Static IP addresses for all nodes
- A load balancer IP address

### 2.3 Software Requirements

- Rocky Linux 9.4 installed on all machines
- SSH access to all nodes
- Superuser (root) access on all machines
- Basic understanding of Linux command line and networking concepts

### 2.4 Air-gapped Environment Requirements

- A machine with internet access for initial package and image downloads
- Storage device or file server to transfer packages and images to the air-gapped environment
- Local package repository server
- Local container image registry

### 2.5 Additional Tools

- Git (for version control)
- A text editor (e.g., vim, nano)
- wget and curl utilities

### 2.6 Documentation and References

- Kubernetes documentation
- ETCD documentation
- HAProxy documentation
- Rocky Linux documentation

Ensure all these prerequisites are met before proceeding with the setup. In the next section, we'll begin by preparing the air-gapped environment, which is crucial for the successful deployment of a Kubernetes HA cluster in an isolated network.

""" + airgapped_setup_content + """


## 4. Setting up the Load Balancer (HAProxy)

In an air-gapped Kubernetes HA cluster, the load balancer plays a crucial role in distributing traffic across the control plane nodes. We'll use HAProxy for this purpose and configure it to work with sslip.io for easier DNS management.

### 4.1 Installing HAProxy

1. Install HAProxy using the local repository we set up earlier:

```bash
sudo dnf install -y haproxy
```

2. Enable and start the HAProxy service:

```bash
sudo systemctl enable haproxy
sudo systemctl start haproxy
```

### 4.2 Configuring HAProxy

1. Back up the original HAProxy configuration file:

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

2. Create a new HAProxy configuration file:

```bash
sudo tee /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend kubernetes-frontend
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-master-0 k8s-master-0.<load-balancer-ip>.sslip.io:6443 check fall 3 rise 2
    server k8s-master-1 k8s-master-1.<load-balancer-ip>.sslip.io:6443 check fall 3 rise 2
    server k8s-master-2 k8s-master-2.<load-balancer-ip>.sslip.io:6443 check fall 3 rise 2

frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:admin  # Change this to a secure password
EOF
```

Replace `<load-balancer-ip>` with your actual load balancer IP address.

3. Verify the HAProxy configuration:

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```

4. If the configuration is valid, restart HAProxy to apply the changes:

```bash
sudo systemctl restart haproxy
```

### 4.3 Configuring sslip.io DNS

1. Add entries to the /etc/hosts file on each node in the cluster:

```bash
sudo tee -a /etc/hosts <<EOF
<load-balancer-ip> k8s-master-0.<load-balancer-ip>.sslip.io k8s-master-0
<load-balancer-ip> k8s-master-1.<load-balancer-ip>.sslip.io k8s-master-1
<load-balancer-ip> k8s-master-2.<load-balancer-ip>.sslip.io k8s-master-2
EOF
```

Replace `<load-balancer-ip>` with your actual load balancer IP address.

### 4.4 Verifying the Load Balancer Setup

1. Check if HAProxy is running:

```bash
sudo systemctl status haproxy
```

2. Test the connection to the Kubernetes API server (this will fail until the control plane is set up, but it should connect to HAProxy):

```bash
nc -v <load-balancer-ip> 6443
```

You should see a connection established, although it will be closed immediately as the Kubernetes API server is not yet running.

3. Access the HAProxy stats page:

Open a web browser and navigate to `http://<load-balancer-ip>:8404/stats`. You should see the HAProxy statistics page.

With these steps completed, you have successfully set up HAProxy as the load balancer for your Kubernetes HA cluster in an air-gapped environment. The load balancer is now ready to distribute traffic to your control plane nodes once they are set up.




## 5. Creating the ETCD Cluster

In an air-gapped environment, setting up the ETCD cluster requires some additional considerations. We'll use the local container registry we set up earlier to pull the ETCD images and configure a highly available ETCD cluster.

### 5.1 Preparing ETCD Nodes

On each ETCD node, perform the following steps:

1. Install necessary packages:

```bash
sudo dnf install -y wget
```

2. Download and install ETCD from the local repository:

```bash
ETCD_VER=v3.5.1
DOWNLOAD_URL=http://<your-local-repo-ip>/etcd

sudo mkdir -p /opt/etcd
sudo wget $DOWNLOAD_URL/etcd-$ETCD_VER-linux-amd64.tar.gz -O /opt/etcd/etcd-$ETCD_VER-linux-amd64.tar.gz
sudo tar xzvf /opt/etcd/etcd-$ETCD_VER-linux-amd64.tar.gz -C /opt/etcd --strip-components=1
sudo ln -s /opt/etcd/etcd /usr/local/bin/
sudo ln -s /opt/etcd/etcdctl /usr/local/bin/
```

Replace `<your-local-repo-ip>` with the IP address of your local package repository.

### 5.2 Configuring ETCD

1. Create the ETCD configuration file on each node:

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd
sudo tee /etc/etcd/etcd.conf <<EOF
name: etcd-<node-id>
data-dir: /var/lib/etcd
listen-client-urls: https://<node-ip>:2379
advertise-client-urls: https://<node-ip>:2379
listen-peer-urls: https://<node-ip>:2380
initial-advertise-peer-urls: https://<node-ip>:2380
initial-cluster: etcd-0=https://<node-0-ip>:2380,etcd-1=https://<node-1-ip>:2380,etcd-2=https://<node-2-ip>:2380
initial-cluster-token: etcd-cluster-1
initial-cluster-state: new
client-transport-security:
  cert-file: /etc/etcd/etcd-server.crt
  key-file: /etc/etcd/etcd-server.key
  trusted-ca-file: /etc/etcd/ca.crt
peer-transport-security:
  cert-file: /etc/etcd/etcd-server.crt
  key-file: /etc/etcd/etcd-server.key
  trusted-ca-file: /etc/etcd/ca.crt
EOF
```

Replace `<node-id>`, `<node-ip>`, `<node-0-ip>`, `<node-1-ip>`, and `<node-2-ip>` with the appropriate values for your ETCD nodes.

2. Generate TLS certificates for ETCD:

```bash
sudo mkdir -p /etc/etcd/ssl
cd /etc/etcd/ssl
sudo openssl genrsa -out ca.key 2048
sudo openssl req -new -x509 -key ca.key -out ca.crt -days 365 -subj "/CN=etcd-ca"
sudo openssl genrsa -out server.key 2048
sudo openssl req -new -key server.key -out server.csr -subj "/CN=etcd-server"
sudo openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
sudo cp ca.crt /etc/etcd/ca.crt
sudo cp server.crt /etc/etcd/etcd-server.crt
sudo cp server.key /etc/etcd/etcd-server.key
```

### 5.3 Creating the ETCD Systemd Service

Create the ETCD systemd service file on each node:

```bash
sudo tee /etc/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
EOF
```

### 5.4 Starting and Enabling ETCD

1. Create the etcd user:

```bash
sudo useradd -r -s /sbin/nologin etcd
```

2. Set the correct permissions:

```bash
sudo chown -R etcd:etcd /var/lib/etcd /etc/etcd
```

3. Start and enable the ETCD service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

### 5.5 Verifying the ETCD Cluster

1. Check the status of the ETCD service:

```bash
sudo systemctl status etcd
```

2. Verify the ETCD cluster health:

```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/ca.crt   --cert=/etc/etcd/etcd-server.crt   --key=/etc/etcd/etcd-server.key   endpoint health
```

3. List the ETCD cluster members:

```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/etcd/ca.crt   --cert=/etc/etcd/etcd-server.crt   --key=/etc/etcd/etcd-server.key   member list
```

If all nodes report "healthy" and are listed as members, your ETCD cluster is set up correctly.

By following these steps, you have successfully set up a highly available ETCD cluster in your air-gapped environment. This ETCD cluster will serve as the backend for your Kubernetes control plane, ensuring that your cluster's state is stored reliably and consistently across multiple nodes.




## 6. Setting up the Kubernetes Control Plane

In an air-gapped environment, setting up the Kubernetes Control Plane requires using local resources and repositories. We'll use the local package repository and container registry we set up earlier.

### 6.1 Preparing Control Plane Nodes

On each control plane node, perform the following steps:

1. Configure the local package repository:

```bash
sudo tee /etc/yum.repos.d/local-kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://<your-local-repo-ip>/kubernetes/
enabled=1
gpgcheck=0
EOF
```

2. Install kubeadm, kubelet, and kubectl:

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

### 6.2 Initializing the Control Plane

On the first control plane node:

1. Create a kubeadm configuration file:

```bash
cat << EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.23.0
controlPlaneEndpoint: "k8s.<load-balancer-ip>.sslip.io:6443"
etcd:
  external:
    endpoints:
      - https://10.0.0.1:2379
      - https://10.0.0.2:2379
      - https://10.0.0.3:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/etcd-client.pem
    keyFile: /etc/etcd/etcd-client-key.pem
networking:
  podSubnet: 192.168.0.0/16
apiServer:
  certSANs:
    - "k8s.<load-balancer-ip>.sslip.io"
imageRepository: <your-local-registry-ip>:5000
EOF
```

Replace `<load-balancer-ip>`, `<your-local-registry-ip>`, and the ETCD endpoints with your actual values.

2. Initialize the control plane:

```bash
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
```

3. Set up kubeconfig for the root user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

4. Install a CNI plugin (e.g., Calico) using the local registry:

```bash
kubectl apply -f http://<your-local-repo-ip>/kubernetes/calico.yaml
```

### 6.3 Joining Additional Control Plane Nodes

On the additional control plane nodes, join the cluster using the command provided by the `kubeadm init` output. It will look similar to this:

```bash
sudo kubeadm join k8s.<load-balancer-ip>.sslip.io:6443 --token <token>   --discovery-token-ca-cert-hash sha256:<hash>   --control-plane --certificate-key <certificate-key>
```

### 6.4 Verifying the Control Plane

After setting up all control plane nodes, verify the cluster status:

```bash
kubectl get nodes
```

All control plane nodes should be in the "Ready" state.

### 6.5 Configuring kubectl on Your Local Machine

To manage the cluster from your local machine:

1. Copy the kubeconfig file from the first control plane node:

```bash
scp root@<first-control-plane-ip>:/etc/kubernetes/admin.conf ~/.kube/config
```

2. Modify the server address in the kubeconfig file to use the load balancer address:

```bash
sed -i 's/127.0.0.1/<load-balancer-ip>/g' ~/.kube/config
```

Now you can manage your cluster from your local machine using kubectl.




## 7. Adding Worker Nodes

In this section, we'll add worker nodes to our Kubernetes cluster in the air-gapped environment. We'll use the local package repository and container registry we set up earlier.

### 7.1 Preparing Worker Nodes

On each worker node, perform the following steps:

1. Install the required packages:

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

2. Enable and start the kubelet service:

```bash
sudo systemctl enable --now kubelet
```

3. Configure containerd to use the local registry:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/k8s.gcr.io/<your-local-registry-ip>:5000/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

Replace `<your-local-registry-ip>` with the IP address of your local container registry.

### 7.2 Joining Worker Nodes to the Cluster

1. On the control plane node, generate a new token if the previous one has expired:

```bash
kubeadm token create --print-join-command
```

2. On each worker node, use the join command output from the previous step. It will look similar to this:

```bash
sudo kubeadm join k8s.<load-balancer-ip>.sslip.io:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 7.3 Verifying Worker Nodes

After joining all worker nodes, verify their status from the control plane node or your local machine:

```bash
kubectl get nodes
```

All worker nodes should appear in the list and eventually reach the "Ready" state.

### 7.4 Labeling Worker Nodes

You can add labels to your worker nodes to help with scheduling and resource allocation:

```bash
kubectl label node <worker-node-name> node-role.kubernetes.io/worker=worker
kubectl label node <worker-node-name> node-type=compute
```

Replace `<worker-node-name>` with the actual node name.

### 7.5 Configuring Node-specific Settings

Depending on your requirements, you might want to configure specific settings for worker nodes:

1. Adjusting kubelet configuration:

```bash
sudo vi /var/lib/kubelet/config.yaml
```

Add or modify settings as needed, then restart kubelet:

```bash
sudo systemctl restart kubelet
```

2. Setting up node-specific firewall rules:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

These ports are used for kubelet API and NodePort services, respectively.

### 7.6 Testing Worker Node Functionality

1. Create a test deployment:

```bash
kubectl create deployment nginx --image=<your-local-registry-ip>:5000/nginx
kubectl scale deployment nginx --replicas=3
```

2. Verify that pods are scheduled on worker nodes:

```bash
kubectl get pods -o wide
```

You should see pods distributed across your worker nodes.

3. Create a test service:

```bash
kubectl expose deployment nginx --type=NodePort --port=80
```

4. Get the NodePort:

```bash
kubectl get svc nginx
```

5. Test accessing the service through a worker node:

```bash
curl http://<worker-node-ip>:<node-port>
```

Replace `<worker-node-ip>` with the IP of any worker node and `<node-port>` with the port number from the previous step.

By following these steps, you have successfully added worker nodes to your Kubernetes cluster in an air-gapped environment. Your cluster now has the capacity to run workloads across multiple nodes, providing increased performance and reliability.


In an air-gapped environment, adding worker nodes to the Kubernetes cluster requires using local resources and repositories. We'll use the local package repository and container registry we set up earlier.

### 7.1 Preparing Worker Nodes

On each worker node, perform the following steps:

1. Configure the local package repository:

```bash
sudo tee /etc/yum.repos.d/local-kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=http://<your-local-repo-ip>/kubernetes/
enabled=1
gpgcheck=0
EOF
```

2. Install kubeadm, kubelet, and kubectl:

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

3. Configure containerd to use the local registry:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/k8s.gcr.io/<your-local-registry-ip>:5000/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```

### 7.2 Joining Worker Nodes to the Cluster

1. On the control plane node, generate a new token if the previous one has expired:

```bash
kubeadm token create --print-join-command
```

2. On each worker node, use the join command output from the previous step. It will look similar to this:

```bash
sudo kubeadm join k8s.<load-balancer-ip>.sslip.io:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### 7.3 Verifying Worker Nodes

After joining all worker nodes, verify their status from the control plane node or your local machine:

```bash
kubectl get nodes
```

All worker nodes should appear in the list and eventually reach the "Ready" state.

### 7.4 Labeling Worker Nodes (Optional)

You can add labels to your worker nodes to help with scheduling and resource allocation:

```bash
kubectl label node <worker-node-name> node-role.kubernetes.io/worker=worker
kubectl label node <worker-node-name> node-type=compute
```

Replace `<worker-node-name>` with the actual node name.

### 7.5 Configuring Node-specific Settings (Optional)

Depending on your requirements, you might want to configure specific settings for worker nodes, such as:

1. Adjusting kubelet configuration:

```bash
sudo vi /var/lib/kubelet/config.yaml
```

Add or modify settings as needed, then restart kubelet:

```bash
sudo systemctl restart kubelet
```

2. Setting up node-specific firewall rules:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

These ports are used for kubelet API and NodePort services, respectively.

By following these steps, you should now have a fully functional Kubernetes cluster with both control plane and worker nodes in your air-gapped environment.



## 8. Configuring Networking

In an air-gapped Kubernetes environment, configuring networking requires special considerations. We'll focus on setting up the Container Network Interface (CNI) and configuring network policies.

### 8.1 Installing the CNI Plugin

We'll use Calico as our CNI plugin. In an air-gapped environment, you need to download the Calico manifests and modify them to use your local registry.

1. Download the Calico manifests on a machine with internet access:

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```

2. Modify the manifest to use your local registry:

```bash
sed -i 's|docker.io/calico/|<your-local-registry-ip>:5000/calico/|g' calico.yaml
```

3. Transfer the modified `calico.yaml` file to your air-gapped environment.

4. Apply the Calico manifest:

```bash
kubectl apply -f calico.yaml
```

### 8.2 Verifying CNI Installation

Check the status of the Calico pods:

```bash
kubectl get pods -n kube-system | grep calico
```

All Calico pods should be in the "Running" state.

### 8.3 Configuring Network Policies

Network policies allow you to control traffic flow between pods. Here's an example of a basic network policy:

1. Create a file named `default-deny-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

2. Apply the network policy:

```bash
kubectl apply -f default-deny-ingress.yaml
```

This policy will deny all ingress traffic to pods in the default namespace unless explicitly allowed by another policy.

### 8.4 Configuring DNS

In an air-gapped environment, you might need to configure custom DNS settings:

1. Edit the CoreDNS ConfigMap:

```bash
kubectl edit configmap coredns -n kube-system
```

2. Add your custom DNS server under the `forward` section:

```yaml
forward . <your-dns-server-ip>
```

3. Save the changes and restart the CoreDNS pods:

```bash
kubectl delete pod -l k8s-app=kube-dns -n kube-system
```

### 8.5 Verifying Cluster Networking

To verify that networking is set up correctly:

1. Create a test deployment:

```bash
kubectl create deployment nginx --image=<your-local-registry-ip>:5000/nginx
kubectl expose deployment nginx --port=80
```

2. Create a test pod:

```bash
kubectl run busybox --image=<your-local-registry-ip>:5000/busybox -- sleep 3600
```

3. Test network connectivity:

```bash
kubectl exec busybox -- wget -O- http://nginx
```

If you see the nginx welcome page, your cluster networking is working correctly.

### 8.6 Troubleshooting Networking Issues

If you encounter networking issues:

1. Check pod status:
   ```bash
   kubectl get pods --all-namespaces
   ```

2. Check CNI plugin logs:
   ```bash
   kubectl logs -n kube-system calico-node-xxxxx
   ```

3. Verify node network interface configuration:
   ```bash
   ip addr show
   ```

4. Check iptables rules:
   ```bash
   sudo iptables -L
   ```

By following these steps, you should have a properly configured network for your air-gapped Kubernetes cluster.



## 9. Setting up Dev/Staging/Production Environments

In an air-gapped Kubernetes cluster, setting up separate environments for development, staging, and production requires careful planning and configuration. We'll use namespaces, resource quotas, and other Kubernetes features to create isolated environments.

### 9.1 Creating Namespaces

Create separate namespaces for each environment:

```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

### 9.2 Configuring Resource Quotas

Set resource quotas for each namespace to limit resource usage:

1. Create a file named `resource-quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    requests.nvidia.com/gpu: 1
```

2. Apply the resource quota to each namespace:

```bash
kubectl apply -f resource-quota.yaml -n development
kubectl apply -f resource-quota.yaml -n staging
kubectl apply -f resource-quota.yaml -n production
```

Adjust the values according to your cluster's capacity and requirements.

### 9.3 Implementing Network Policies

Create network policies to isolate environments:

1. Create a file named `isolate-namespace.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: same-namespace
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: same-namespace
```

2. Apply the network policy to each namespace:

```bash
kubectl apply -f isolate-namespace.yaml -n development
kubectl apply -f isolate-namespace.yaml -n staging
kubectl apply -f isolate-namespace.yaml -n production
```

### 9.4 Setting Up Role-Based Access Control (RBAC)

Create roles and role bindings for each environment:

1. Create a file named `environment-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: environment-user
rules:
- apiGroups: ["", "apps", "batch", "extensions"]
  resources: ["deployments", "replicasets", "pods", "jobs", "cronjobs", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

2. Apply the role to each namespace:

```bash
kubectl apply -f environment-role.yaml -n development
kubectl apply -f environment-role.yaml -n staging
kubectl apply -f environment-role.yaml -n production
```

3. Create role bindings for users or groups:

```bash
kubectl create rolebinding dev-user-binding --role=environment-user --user=dev-user --namespace=development
kubectl create rolebinding staging-user-binding --role=environment-user --user=staging-user --namespace=staging
kubectl create rolebinding prod-user-binding --role=environment-user --user=prod-user --namespace=production
```

### 9.5 Configuring Environment-specific Resources

1. Create environment-specific ConfigMaps and Secrets:

```bash
kubectl create configmap app-config --from-file=config.properties -n development
kubectl create secret generic app-secrets --from-file=secrets.properties -n development
```

Repeat for staging and production namespaces with appropriate configurations.

2. Use namespace-specific Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: development
spec:
  rules:
  - host: dev.app.<load-balancer-ip>.sslip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port: 
              number: 80
```

Create similar Ingress resources for staging and production, using different hostnames.

### 9.6 Implementing CI/CD for Different Environments

1. Set up separate deployment pipelines for each environment.
2. Use environment-specific variables and configurations in your CI/CD tools.
3. Implement approval processes for promoting changes between environments.

### 9.7 Monitoring and Logging

1. Set up monitoring and logging solutions that can differentiate between environments:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

2. Configure dashboards and alerts specific to each environment.

By following these steps, you'll have separate Dev, Staging, and Production environments within your air-gapped Kubernetes cluster, each with its own resources, access controls, and configurations.



## 10. Real-time Project Examples

In this section, we'll walk through some real-time project examples that you can deploy in your air-gapped Kubernetes cluster. These examples will demonstrate how to utilize the environments we've set up and showcase various Kubernetes features.

### 10.1 Deploying a Web Application

Let's deploy a simple web application with a database backend in the development namespace.

1. Create a MySQL deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: <your-local-registry-ip>:5000/mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secrets
              key: root-password
        ports:
        - containerPort: 3306
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: development
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
```

2. Create a web application deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: development
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: <your-local-registry-ip>:5000/your-webapp:latest
        env:
        - name: DB_HOST
          value: mysql
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secrets
              key: root-password
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: development
spec:
  selector:
    app: webapp
  ports:
    - port: 80
  type: LoadBalancer
```

3. Apply the configurations:

```bash
kubectl apply -f mysql-deployment.yaml
kubectl apply -f webapp-deployment.yaml
```

### 10.2 Implementing Continuous Deployment

Set up a simple continuous deployment pipeline using GitLab CI/CD (assuming you have GitLab set up in your air-gapped environment):

1. Create a `.gitlab-ci.yml` file in your project repository:

```yaml
stages:
  - build
  - deploy

build:
  stage: build
  script:
    - docker build -t <your-local-registry-ip>:5000/your-webapp:$CI_COMMIT_SHA .
    - docker push <your-local-registry-ip>:5000/your-webapp:$CI_COMMIT_SHA

deploy_development:
  stage: deploy
  script:
    - kubectl set image deployment/webapp webapp=<your-local-registry-ip>:5000/your-webapp:$CI_COMMIT_SHA -n development
  only:
    - develop

deploy_staging:
  stage: deploy
  script:
    - kubectl set image deployment/webapp webapp=<your-local-registry-ip>:5000/your-webapp:$CI_COMMIT_SHA -n staging
  only:
    - staging

deploy_production:
  stage: deploy
  script:
    - kubectl set image deployment/webapp webapp=<your-local-registry-ip>:5000/your-webapp:$CI_COMMIT_SHA -n production
  only:
    - main
  when: manual
```

### 10.3 Implementing Autoscaling

Set up Horizontal Pod Autoscaling for your web application:

1. Create an HPA resource:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: development
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```

2. Apply the HPA:

```bash
kubectl apply -f webapp-hpa.yaml
```

### 10.4 Implementing Blue-Green Deployments

Implement a blue-green deployment strategy for your web application:

1. Create a blue deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
  namespace: development
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: blue
  template:
    metadata:
      labels:
        app: webapp
        version: blue
    spec:
      containers:
      - name: webapp
        image: <your-local-registry-ip>:5000/your-webapp:blue
        ports:
        - containerPort: 80
```

2. Create a green deployment (similar to blue, but with `version: green` and appropriate image tag).

3. Create a service that can switch between blue and green:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: development
spec:
  selector:
    app: webapp
    version: blue  # Change this to 'green' to switch versions
  ports:
    - port: 80
  type: LoadBalancer
```

4. To switch versions, update the service selector from `blue` to `green` or vice versa.

These examples demonstrate how to deploy applications, implement CI/CD, set up autoscaling, and perform blue-green deployments in your air-gapped Kubernetes cluster. Remember to adjust image names, registry addresses, and other specifics to match your environment.



## 11. Additional Kubernetes Topics

In this section, we'll cover some additional Kubernetes topics that are particularly relevant to air-gapped environments.

### 11.1 Local Image Registry Management

In an air-gapped environment, managing your local image registry is crucial:

1. Regularly update images in your local registry:

```bash
docker pull some-image:latest
docker tag some-image:latest <your-local-registry-ip>:5000/some-image:latest
docker push <your-local-registry-ip>:5000/some-image:latest
```

2. Implement a cleanup policy to manage disk space:

```bash
docker exec -it registry registry garbage-collect /etc/docker/registry/config.yml
```

3. Consider using a tool like Harbor for advanced registry management features.

### 11.2 Air-gapped Helm Usage

Helm is a package manager for Kubernetes. In an air-gapped environment:

1. Download Helm charts and their dependencies when you have internet access:

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm pull stable/mysql --version 1.6.9
```

2. Host a local Helm chart repository:

```bash
helm serve --address 0.0.0.0:8879 --repo-path ./charts
```

3. Use the local Helm repository in your air-gapped environment:

```bash
helm repo add local http://<your-helm-repo-ip>:8879
helm install my-release local/mysql
```

### 11.3 Backup and Restore in Air-gapped Environments

Implement a robust backup and restore strategy:

1. Use etcd snapshots for backing up Kubernetes cluster state:

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

2. Use Velero for backing up Kubernetes resources and persistent volumes:

```bash
velero backup create my-backup --include-cluster-resources=true
```

Ensure your backup storage is also air-gapped and secure.

### 11.4 Updating Kubernetes in Air-gapped Environments

Updating Kubernetes in an air-gapped environment requires careful planning:

1. Download required images and binaries when you have internet access.

2. Update kubeadm first:

```bash
yum update -y kubeadm-<version>
kubeadm upgrade plan
kubeadm upgrade apply v<version>
```

3. Update kubelet and kubectl on all nodes:

```bash
yum update -y kubelet-<version> kubectl-<version>
systemctl daemon-reload
systemctl restart kubelet
```

### 11.5 Custom Resource Definitions (CRDs) and Operators

CRDs and Operators can extend Kubernetes functionality:

1. Create a simple CRD:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myresources.mygroup.example.com
spec:
  group: mygroup.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                someField:
                  type: string
  scope: Namespaced
  names:
    plural: myresources
    singular: myresource
    kind: MyResource
```

2. Develop and deploy custom operators to manage these CRDs in your air-gapped environment.

### 11.6 Multi-tenancy in Kubernetes

Implement multi-tenancy for larger organizations:

1. Use namespaces for logical separation.
2. Implement ResourceQuotas and LimitRanges.
3. Use NetworkPolicies for network isolation.
4. Leverage RBAC for access control.

### 11.7 Monitoring and Logging in Air-gapped Environments

Adapt monitoring and logging solutions for air-gapped use:

1. Deploy Prometheus and Grafana for monitoring:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

2. Set up EFK (Elasticsearch, Fluentd, Kibana) stack for logging:

```bash
helm install elasticsearch elastic/elasticsearch --namespace logging --create-namespace
helm install fluentd fluent/fluentd --namespace logging
helm install kibana elastic/kibana --namespace logging
```

Ensure all required images are available in your local registry.

By understanding and implementing these additional topics, you can create a more robust, secure, and manageable Kubernetes environment in your air-gapped setup.



## 12. Troubleshooting and Best Practices

In this final section, we'll cover some common issues you might encounter in an air-gapped Kubernetes environment and provide best practices to maintain a healthy cluster.

### 12.1 Common Issues and Solutions

1. **Image Pull Errors**:
   - Symptom: Pods stuck in "ImagePullBackOff" state.
   - Solution: Ensure the image exists in your local registry and the image name in your deployment YAML is correct.
   ```bash
   docker pull <your-local-registry-ip>:5000/image-name:tag
   kubectl describe pod <pod-name>
   ```

2. **DNS Resolution Issues**:
   - Symptom: Services unable to communicate using DNS names.
   - Solution: Check CoreDNS pods and configuration.
   ```bash
   kubectl get pods -n kube-system | grep coredns
   kubectl describe configmap coredns -n kube-system
   ```

3. **Node Not Ready**:
   - Symptom: Nodes showing "NotReady" status.
   - Solution: Check kubelet status and logs.
   ```bash
   systemctl status kubelet
   journalctl -u kubelet
   ```

4. **Persistent Volume Claims Pending**:
   - Symptom: PVCs stuck in "Pending" state.
   - Solution: Verify storage class and provisioner.
   ```bash
   kubectl get sc
   kubectl describe pvc <pvc-name>
   ```

### 12.2 Best Practices

1. **Regular Backups**:
   - Implement regular etcd snapshots and Velero backups.
   - Test your restore process periodically.

2. **Update Strategy**:
   - Develop a clear strategy for updating Kubernetes components and applications.
   - Test updates in a separate environment before applying to production.

3. **Resource Management**:
   - Use ResourceQuotas and LimitRanges to prevent resource exhaustion.
   - Regularly monitor resource usage and adjust limits as needed.

4. **Security**:
   - Implement strong RBAC policies.
   - Regularly update and patch all components.
   - Use network policies to control pod-to-pod communication.

5. **Logging and Monitoring**:
   - Implement comprehensive logging and monitoring solutions.
   - Set up alerts for critical issues.

6. **Documentation**:
   - Maintain detailed documentation of your cluster setup and configurations.
   - Document all custom procedures specific to your air-gapped environment.

### 12.3 Performance Tuning

1. **Optimize etcd**:
   - Use SSD storage for etcd.
   - Regularly defragment etcd data.
   ```bash
   etcdctl defrag --endpoints=https://127.0.0.1:2379
   ```

2. **Kubernetes API Server**:
   - Adjust `--max-requests-inflight` and `--max-mutating-requests-inflight` based on cluster size.

3. **Kubelet**:
   - Optimize `--kube-api-qps` and `--kube-api-burst` settings.

4. **Container Runtime**:
   - Use containerd instead of Docker for better performance.

### 12.4 Disaster Recovery

1. Develop a comprehensive disaster recovery plan.
2. Regularly practice disaster recovery scenarios.
3. Ensure all critical data and configurations are backed up off-site.

### 12.5 Capacity Planning

1. Regularly review cluster capacity and plan for growth.
2. Monitor node resource usage and add capacity proactively.
3. Use cluster autoscaler for automatic scaling of worker nodes.


### 12.6 Implementing CI/CD with Jenkins, GitLab, and DevOps Practices

In an air-gapped environment, implementing a robust CI/CD pipeline requires careful planning and integration of various tools. Here's how to set up a comprehensive pipeline using Jenkins, GitLab, and incorporating DevOps, GitOps, and DevSecOps practices.

#### 12.6.1 Setting up Jenkins in Air-gapped Environment

1. Deploy Jenkins on Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: <your-local-registry-ip>:5000/jenkins/jenkins:lts
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
      volumes:
      - name: jenkins-home
        emptyDir: {}
```

2. Expose Jenkins service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: devops
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
```

#### 12.6.2 Integrating GitLab with Jenkins

1. Set up GitLab in your air-gapped environment.
2. Configure Jenkins to use GitLab as the source code management system.
3. Install the GitLab plugin in Jenkins.
4. Set up webhooks in GitLab to trigger Jenkins jobs on code changes.

#### 12.6.3 Implementing a Jenkins Pipeline with Quality Gates

Create a Jenkinsfile in your project repository:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t <your-local-registry-ip>:5000/myapp:${BUILD_NUMBER} .'
            }
        }
        stage('Unit Tests') {
            steps {
                sh 'docker run <your-local-registry-ip>:5000/myapp:${BUILD_NUMBER} npm test'
            }
        }
        stage('Static Code Analysis') {
            steps {
                sh 'docker run <your-local-registry-ip>:5000/sonarqube-scanner:latest'
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Security Scan') {
            steps {
                sh 'docker run <your-local-registry-ip>:5000/trivy:latest image <your-local-registry-ip>:5000/myapp:${BUILD_NUMBER}'
            }
        }
        stage('Push to Registry') {
            steps {
                sh 'docker push <your-local-registry-ip>:5000/myapp:${BUILD_NUMBER}'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl set image deployment/myapp myapp=<your-local-registry-ip>:5000/myapp:${BUILD_NUMBER}'
            }
        }
    }
    post {
        always {
            junit '**/test-results/*.xml'
        }
    }
}
```

#### 12.6.4 Implementing GitOps with ArgoCD

1. Install ArgoCD in your cluster:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Create an ArgoCD application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://<your-gitlab-ip>/myapp.git
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### 12.6.5 Implementing DevSecOps Practices

1. Integrate security scanning in your CI/CD pipeline (as shown in the Jenkinsfile above with Trivy).
2. Implement policy-as-code using Open Policy Agent (OPA):

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-labels
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels: ["owner", "environment"]
```

3. Use Vault for secrets management:

```bash
helm install vault hashicorp/vault --namespace vault --create-namespace
```

#### 12.6.6 Monitoring and Observability

1. Set up Prometheus and Grafana for monitoring:

```bash
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

2. Implement distributed tracing with Jaeger:

```bash
kubectl create namespace observability
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.28.0/jaeger-operator.yaml -n observability
```

By implementing these practices and tools, you create a comprehensive DevOps, GitOps, and DevSecOps environment in your air-gapped Kubernetes cluster. This setup ensures continuous integration, delivery, and deployment with proper security measures and observability.


By following these troubleshooting tips and best practices, you can maintain a robust and efficient air-gapped Kubernetes environment. Remember that managing an air-gapped cluster requires ongoing attention and adaptation to your specific needs and challenges.



## 13. Conclusion

This comprehensive guide has walked you through the process of setting up, managing, and optimizing a Kubernetes High Availability (HA) cluster in an air-gapped environment. We've covered a wide range of topics, from the initial setup to advanced DevOps practices. Here's a summary of the key points we've discussed:

1. **Air-gapped Environment Setup**: We detailed the specific considerations and steps needed to prepare an air-gapped environment, including setting up local repositories and registries.

2. **Kubernetes Cluster Setup**: We covered the installation and configuration of all necessary components, including the control plane, worker nodes, and supporting services like etcd and HAProxy.

3. **Networking and Security**: We discussed how to set up networking in an air-gapped environment, implement security best practices, and use tools like network policies to control traffic.

4. **Storage and Persistence**: We explored options for persistent storage in Kubernetes and how to manage it in an air-gapped setup.

5. **Monitoring and Logging**: We covered the implementation of monitoring and logging solutions adapted for air-gapped environments.

6. **Backup and Disaster Recovery**: We discussed strategies for backing up cluster data and implementing disaster recovery plans.

7. **CI/CD and DevOps Practices**: We detailed how to set up a comprehensive CI/CD pipeline using Jenkins and GitLab, and how to implement DevOps, GitOps, and DevSecOps practices in an air-gapped environment.

8. **Troubleshooting and Best Practices**: We provided guidance on common issues you might encounter and best practices for maintaining a healthy cluster.

Managing a Kubernetes cluster in an air-gapped environment presents unique challenges, but it also offers enhanced security and control. By following the practices outlined in this guide, you can create a robust, secure, and efficient Kubernetes environment that meets the stringent requirements of air-gapped operations.

Remember that Kubernetes and its ecosystem are constantly evolving. Stay informed about new developments and regularly review and update your cluster setup to ensure it remains secure, efficient, and aligned with your organization's needs.

Lastly, while this guide provides a comprehensive overview, every environment is unique. Be prepared to adapt these guidelines to your specific requirements and constraints. Continuous learning, testing, and improvement are key to successfully managing a Kubernetes cluster in any environment, especially an air-gapped one.

Thank you for following this guide. We hope it serves as a valuable resource in your Kubernetes journey!


## 14. Architectural Diagrams

To better illustrate the concepts and architectures discussed in this guide, we've included some diagrams below.

### 14.1 Overall Kubernetes HA Cluster Architecture

```
                                  +------------------------+
                                  |                        |
                                  |   Load Balancer (HAProxy)   |
                                  |                        |
                                  +------------------------+
                                    |          |          |
                            +-------+  +-------+  +-------+
                            |          |          |
                    +-------v--+ +-----v----+ +---v------+
                    | Master 1 | | Master 2 | | Master 3 |
                    |  (etcd)  | |  (etcd)  | |  (etcd)  |
                    +----------+ +----------+ +----------+
                         |            |            |
                    +----v------------v------------v----+
                    |                                   |
                    |        Kubernetes Cluster         |
                    |                                   |
                    +---+---+---+---+---+---+---+---+---+
                        |   |   |   |   |   |   |   |
                    +---v---v---v---v---v---v---v---v---+
                    |                                   |
                    |         Worker Nodes Pool         |
                    |                                   |
                    +-----------------------------------+
```

### 14.2 Air-gapped Environment Setup

```
+----------------------------------+
|                                  |
|    Air-gapped Environment        |
|                                  |
|  +----------------------------+  |
|  |                            |  |
|  |   Local Package Repository |  |
|  |                            |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |                            |  |
|  |   Local Container Registry |  |
|  |                            |  |
|  +----------------------------+  |
|                                  |
|  +----------------------------+  |
|  |                            |  |
|  |   Kubernetes Cluster       |  |
|  |                            |  |
|  +----------------------------+  |
|                                  |
+----------------------------------+
```

### 14.3 CI/CD Pipeline in Air-gapped Environment

```
+-------------+     +-------------+     +-------------+
|             |     |             |     |             |
|  Local Git  +---->+  Jenkins    +---->+  Local      |
|  (GitLab)   |     |  Pipeline   |     |  Registry   |
|             |     |             |     |             |
+-------------+     +-------------+     +------+------+
                                               |
                                               |
                                               v
                                        +------+------+
                                        |             |
                                        | Kubernetes  |
                                        | Cluster     |
                                        |             |
                                        +-------------+
```

These diagrams provide a visual representation of the key components and workflows discussed in this guide. They should help readers better understand the overall architecture and relationships between different parts of the air-gapped Kubernetes HA cluster setup.


### Additional Considerations for Air-gapped Environments

1. Certificate Management: 
   In an air-gapped environment, you might need to set up your own Certificate Authority (CA) and issue certificates for your services.

2. Updates and Patches: 
   Establish a process for regularly updating your local repositories and registries with the latest packages and images.

3. Monitoring and Logging: 
   Ensure that any monitoring or logging solutions you implement can function without internet access, or set up local alternatives.

4. Backup and Disaster Recovery: 
   Implement local backup solutions and test recovery procedures in your air-gapped environment.

Remember to replace `<load-balancer-ip>`, `<your-local-repo-ip>`, and `<your-local-registry-ip>` with the actual IP addresses in your environment.



## 4. Creating the ETCD Cluster

ETCD is a distributed key-value store that Kubernetes uses to store all its data. We'll set up a three-node ETCD cluster for high availability.

1. On each ETCD node, download and install ETCD:

```bash
ETCD_VER=v3.5.9
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
```

2. Create the ETCD configuration file on each node:

```bash
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo nano /etc/etcd/etcd.conf
```

Add the following content, adjusting the IP addresses and names as needed:

```
name: etcd-1
data-dir: /var/lib/etcd
initial-cluster-state: new
initial-cluster-token: etcd-cluster-1
initial-cluster: etcd-1=https://<etcd-1-ip>:2380,etcd-2=https://<etcd-2-ip>:2380,etcd-3=https://<etcd-3-ip>:2380
initial-advertise-peer-urls: https://<etcd-1-ip>:2380
listen-peer-urls: https://<etcd-1-ip>:2380
listen-client-urls: https://<etcd-1-ip>:2379,https://127.0.0.1:2379
advertise-client-urls: https://<etcd-1-ip>:2379
```

3. Create the ETCD systemd service file:

```bash
sudo nano /etc/systemd/system/etcd.service
```

Add the following content:

```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd/etcd.conf
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

4. Start and enable the ETCD service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

5. Verify that ETCD is running:

```bash
sudo systemctl status etcd
```

Repeat these steps for each ETCD node, adjusting the configuration as needed.

## 5. Setting up the Kubernetes Control Plane

Now that we have our load balancer and ETCD cluster set up, we can proceed with setting up the Kubernetes control plane.

1. On the first control plane node, initialize the Kubernetes cluster:

```bash
sudo kubeadm init --control-plane-endpoint "<load-balancer-dns>:6443" --upload-certs --pod-network-cidr=192.168.0.0/16
```

2. After the initialization is complete, follow the instructions provided in the output to set up your kubeconfig file:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

3. Install a CNI (Container Network Interface) plugin. For this guide, we'll use Calico:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

4. Join the other control plane nodes to the cluster using the command provided in the `kubeadm init` output:

```bash
sudo kubeadm join <load-balancer-dns>:6443 --token <token> --discovery-token-ca-cert-hash <hash> --control-plane --certificate-key <certificate-key>
```

5. Verify that all control plane nodes are ready:

```bash
kubectl get nodes
```

## 6. Adding Worker Nodes

1. On each worker node, run the join command provided in the `kubeadm init` output:

```bash
sudo kubeadm join <load-balancer-dns>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

2. Verify that the worker nodes have joined the cluster:

```bash
kubectl get nodes
```

## 7. Configuring Networking

Ensure that your network is properly configured for Kubernetes. This includes:

- Opening necessary ports (6443 for API server, 2379-2380 for etcd, etc.)
- Configuring firewalls to allow traffic between nodes
- Setting up DNS for service discovery


## 8. Setting up Dev/Staging/Production Environments

When setting up multiple environments (Development, Staging, and Production), it's important to maintain consistency while allowing for environment-specific configurations. Here's how you can set up these environments:

### 8.1 Namespace Separation

Use Kubernetes namespaces to logically separate your environments:

```bash
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production
```

### 8.2 Resource Quotas

Set resource quotas for each namespace to prevent one environment from consuming all cluster resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
```

Apply similar quotas for staging and production, adjusting the values as needed.

### 8.3 Network Policies

Implement network policies to control traffic between namespaces:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: production
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
```

### 8.4 Configuration Management

Use ConfigMaps and Secrets to manage environment-specific configurations:

```bash
kubectl create configmap app-config --from-file=config.properties -n development
kubectl create secret generic app-secrets --from-file=secrets.properties -n development
```

Repeat for staging and production with appropriate values.

### 8.5 Continuous Deployment

Set up a CI/CD pipeline that deploys to the appropriate namespace based on the branch or tag:

- `develop` branch → Development namespace
- `staging` branch → Staging namespace
- `main` branch → Production namespace

## 9. Real-time Project Examples

Let's walk through setting up a simple web application in our Kubernetes cluster.

### 9.1 Deploying a Web Application

1. Create a deployment for a sample web application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: development
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:latest
        ports:
        - containerPort: 80
```

2. Create a service to expose the application:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: development
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

3. Apply these configurations:

```bash
kubectl apply -f web-app-deployment.yaml
kubectl apply -f web-app-service.yaml
```

### 9.2 Scaling the Application

To scale the application, update the `replicas` field in the deployment:

```bash
kubectl scale deployment web-app --replicas=5 -n development
```

### 9.3 Updating the Application

To update the application, change the image in the deployment:

```bash
kubectl set image deployment/web-app web-app=nginx:1.19 -n development
```

## 10. Additional Kubernetes Topics

### 10.1 Persistent Storage

Set up persistent storage using StorageClass and PersistentVolumeClaims:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
```

### 10.2 Monitoring and Logging

Deploy Prometheus and Grafana for monitoring:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

### 10.3 Autoscaling

Implement Horizontal Pod Autoscaler:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: development
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```

## 11. Troubleshooting and Best Practices

### 11.1 Common Issues

- Pods stuck in `Pending` state: Check node resources and events
- Service not accessible: Verify service and pod selectors match
- Persistent volumes not mounting: Check storage provisioner and PV/PVC status

### 11.2 Best Practices

- Use liveness and readiness probes for all containers
- Implement resource requests and limits for all pods
- Regularly update Kubernetes and all components
- Use RBAC for fine-grained access control
- Implement proper logging and monitoring
- Use Helm charts for complex application deployments
- Regularly backup etcd data

### 11.3 Useful Commands for Troubleshooting

```bash
# Get detailed information about a pod
kubectl describe pod <pod-name> -n <namespace>

# Check logs of a container
kubectl logs <pod-name> -c <container-name> -n <namespace>

# Execute a command in a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Check cluster events
kubectl get events --sort-by=.metadata.creationTimestamp -n <namespace>

# Check node resource usage
kubectl top nodes

# Check pod resource usage
kubectl top pods -n <namespace>
```



## 12. Security Considerations

### 12.1 Role-Based Access Control (RBAC)

Implement RBAC to control access to cluster resources:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### 12.2 Pod Security Policies

Enable and configure Pod Security Policies to control security-sensitive aspects of pod specifications:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
```

### 12.3 Network Policies

Implement network policies to control traffic between pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

## 13. Backup and Disaster Recovery

### 13.1 Etcd Backup

Regularly backup etcd data:

```bash
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key   snapshot save /backup/etcd-snapshot-$(date +%Y-%m-%d_%H:%M:%S).db
```

### 13.2 Velero for Cluster Backup

Install and configure Velero for cluster-wide backups:

```bash
velero install     --provider aws     --plugins velero/velero-plugin-for-aws:v1.2.0     --bucket kubernetesbucket     --secret-file ./credentials-velero     --use-volume-snapshots=false     --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.velero.svc:9000
```

## 14. Cluster Upgrades

### 14.1 Upgrading Control Plane Nodes

Upgrade the kubeadm binary:

```bash
apt-get update && apt-get install -y --allow-change-held-packages kubeadm=1.22.x-00
```

Plan the upgrade:

```bash
kubeadm upgrade plan
```

Apply the upgrade:

```bash
kubeadm upgrade apply v1.22.x
```

### 14.2 Upgrading Worker Nodes

Drain the node:

```bash
kubectl drain <node-name> --ignore-daemonsets
```

Upgrade kubelet and kubectl:

```bash
apt-get update && apt-get install -y --allow-change-held-packages kubelet=1.22.x-00 kubectl=1.22.x-00
```

Restart the kubelet:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Uncordon the node:

```bash
kubectl uncordon <node-name>
```

## 15. Performance Tuning

### 15.1 Resource Limits and Requests

Set appropriate resource limits and requests for all containers:

```yaml
resources:
  limits:
    cpu: "1"
    memory: 1Gi
  requests:
    cpu: "0.5"
    memory: 512Mi
```

### 15.2 Horizontal Pod Autoscaler (HPA)

Implement HPA for automatic scaling based on metrics:

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 50
```

### 15.3 Cluster Autoscaler

Enable cluster autoscaler for automatic node scaling (cloud-specific example):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
    spec:
      containers:
        - image: k8s.gcr.io/cluster-autoscaler:v1.21.0
          name: cluster-autoscaler
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
```


## 16. Advanced Networking

### 16.1 Service Mesh with Istio

Implement Istio for advanced traffic management, security, and observability:

1. Install Istio:

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.11.0
export PATH=$PWD/bin:$PATH
istioctl install --set profile=demo -y
```

2. Enable Istio injection for a namespace:

```bash
kubectl label namespace default istio-injection=enabled
```

3. Deploy a sample application with Istio:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

### 16.2 Network Policies for Micro-segmentation

Implement fine-grained network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - protocol: TCP
      port: 8080
```

## 17. GitOps with ArgoCD

Implement GitOps practices using ArgoCD:

1. Install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Access the ArgoCD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

3. Configure an application in ArgoCD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

## 18. Advanced Monitoring and Logging

### 18.1 Prometheus Operator and Grafana

Enhance monitoring with Prometheus Operator and Grafana:

1. Install Prometheus Operator:

```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring
```

2. Access Grafana:

```bash
kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring
```

### 18.2 EFK Stack for Logging

Implement centralized logging with Elasticsearch, Fluentd, and Kibana:

1. Install Elasticsearch:

```bash
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch -n logging
```

2. Install Fluentd:

```bash
kubectl create -f https://raw.githubusercontent.com/fluent/fluentd-kubernetes-daemonset/master/fluentd-daemonset-elasticsearch.yaml
```

3. Install Kibana:

```bash
helm install kibana elastic/kibana -n logging
```

## 19. Stateful Applications

### 19.1 StatefulSets for Databases

Deploy a stateful application (e.g., PostgreSQL) using StatefulSets:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres 
spec:
  serviceName: "postgres"
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
```

### 19.2 Operators for Complex Applications

Use Operators for managing complex applications. Example: Deploy MongoDB using the MongoDB Operator:

1. Install the MongoDB Operator:

```bash
kubectl create namespace mongodb
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes-operator/master/config/crd/bases/mongodbcommunity.mongodb.com_mongodbcommunity.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes-operator/master/config/rbac/role.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes-operator/master/config/rbac/role_binding.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes-operator/master/config/rbac/service_account.yaml
kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-kubernetes-operator/master/config/manager/manager.yaml
```

2. Deploy a MongoDB instance:

```yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: example-mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "4.4.0"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: my-user
      db: admin
      roles:
        - name: clusterAdmin
          db: admin
        - name: userAdminAnyDatabase
          db: admin
      scramCredentialsSecretName: my-scram
```

## 20. Multi-cluster Management

### 20.1 Cluster Federation with KubeFed

Implement cluster federation for managing multiple Kubernetes clusters:

1. Install KubeFed:

```bash
helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts
helm install kubefed kubefed-charts/kubefed --namespace kube-federation-system --create-namespace
```

2. Join clusters to the federation:

```bash
kubefedctl join cluster1 --cluster-context cluster1 --host-cluster-context host-cluster --v=2
kubefedctl join cluster2 --cluster-context cluster2 --host-cluster-context host-cluster --v=2
```

### 20.2 Multi-cluster Service Mesh with Istio

Extend Istio across multiple clusters:

1. Install Istio on both clusters
2. Establish cross-cluster communication
3. Configure multi-cluster service mesh

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio
spec:
  values:
    global:
      multiCluster:
        clusterName: cluster1
      network: network1
```

## Conclusion

This comprehensive guide now covers advanced topics crucial for running a production-grade Kubernetes HA cluster at scale. From sophisticated networking setups and GitOps practices to advanced monitoring, logging, and multi-cluster management, you have the knowledge to build and maintain a robust, scalable, and efficient Kubernetes environment.

Remember that the Kubernetes ecosystem is constantly evolving. Stay informed about new features, best practices, and security updates. Regularly review and update your cluster configuration, and continuously monitor and optimize your setup to ensure it meets your organization's evolving needs.

As you implement these advanced features, always consider the specific requirements of your applications and organization. Start with a solid foundation and gradually introduce more complex components as needed.

Happy Kubernetes clustering, and may your containers always be healthy and your clusters forever scalable!



