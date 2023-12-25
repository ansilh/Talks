# Into to CNCF world with Kubernetes.

This directory contains the presentation and a demo video used in AWS Community Day - Kochi on 16th Dec 2023.

[Presentation](./ppt.pdf)

[Recording](https://www.youtube.com/watch?v=XMe7vQ9Klq0)

# Notes

## Pre-requisite  

The setup is on VirtualBox and installed `Ubuntu 22.04` with minimal option
Primary network set to `host-only`
Secondary network set to `NAT`
Default user is set to `ansil`

## Set IP on HostOnly Interface
```
IP: 192.168.56.101
Mask: 255.255.255.0
```
## Install ssh and curl
```
sudo apt-get install ssh curl
systemctl enable ssh
```

## SSH into the system 
```
ssh ansil@192.168.56.101
```
## Docker installation 
```
curl https://releases.rancher.com/install-docker/20.10.sh | sh
```

## Add present user to docker group
```
sudo usermod -aG docker $USER
```

## Create staging area for rke
```
mkdir rke_setup
cd rke_setup
```

## Download RKE binary and make it executable
```
curl -L https://github.com/rancher/rke/releases/download/v1.4.11/rke_linux-amd64 -o rke
chmod +x rke
```
## Generate SSH keys
```
ssh-keygen 
```
## Copy the SSH public key to the node
```
ssh-copy-id ansil@192.168.56.101
```
## Create single node k8s cluster configuration
```
./rke config 
```
## Edit the cluster configuration and set cni to none
```
vi cluster.yml
```

```
network:
  plugin: none
```

## Make the cluster up
```
./rke up
```
## Check the state file and kubeconfig files after cluster creation
```
ls -lrt 
```
## Download kubernetes client 
```
curl -LO "https://dl.k8s.io/release/v1.26.9/bin/linux/amd64/kubectl"
chmod +x kubectl 
```

## Set configuration for Kubernetes clients
```
export KUBECONFIG=$HOME/rke_setup/kube_config_cluster.yml
```

## Download Kubernetes package manager - Helm
```
curl -L https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz | tar -zxvf - --strip-components=1 linux-amd64/helm
```

## Install Cilium CNI
```
./helm repo add cilium https://helm.cilium.io
./helm repo update
./helm install cilium cilium/cilium --set operator.replicas=1 -n kube-system
```


## Download Cilium cli

```
curl -L https://github.com/cilium/cilium-cli/releases/download/v0.15.17/cilium-linux-amd64.tar.gz | tar -zxvf -
./cilium status -n kube-system
```

## Restart pending pods to get IPs from Cilium
```
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

## Kubernetes Dashboard deployment
```
./kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
## Create an account to login to Dashboard
```
./kubectl create -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

```
./kubectl create -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF
```

## Create a token and note it down
```
./kubectl -n kubernetes-dashboard create token admin-user    
```
## Start a proxy connection to Kubernetes 
```
./kubectl proxy
```
## Access the Dashboard

```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
Now enter the toke you got from previous step to login to the dashboard!
