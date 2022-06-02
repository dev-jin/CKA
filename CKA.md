# Cluster Architecture, Installation & Configuration (25%)

## Manage role-based access control (RBAC)

Doc: https://kubernetes.io/docs/reference/access-authn-authz/rbac/

## Use KubeADM to install a basic cluster

<details><summary>Solution</summary>
<p>

If you don't have cluster nodes yet, check the terraform deployment from below: [Provision underlying infrastructure to deploy a Kubernetes cluster](https://github.com/alijahnas/CKA-practice-exercises/blob/CKA-v1.20/cluster-architecture-installation-configuration.md#provision-underlying-infrastructure-to-deploy-a-kubernetes-cluster)

Installation from [scratch](https://github.com/kelseyhightower/kubernetes-the-hard-way/) is too time consuming. We will be using KubeADM (v1.20) to install the Kubernetes cluster.

### Install container runtime

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/setup/production-environment/container-runtimes/

Do this on all three nodes:

```bash
# containerd preinstall configuration
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Install containerd
## Set up the repository
### Install packages to allow apt to use a repository over HTTPS
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

## Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

## Add Docker apt repository.
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

## Install packages
sudo apt-get update
sudo apt-get install -y \
  containerd.io

# Configure containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
```

</p>
</details>

### Install kubeadm, kubelet and kubectl

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Do this on all three nodes:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet=1.20.7-00 kubeadm=1.20.7-00 kubectl=1.20.7-00
sudo apt-mark hold kubelet kubeadm kubectl
```

</p>
</details>

### Create a cluster with KubeADM

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

Make sure the nodes have different hostnames.

On controlplane node:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///run/containerd/containerd.sock
```

Run the output of the init command on worker nodes:
```bash
sudo kubeadm join 172.16.1.11:6443 --token h8vno9.7eroqaei7v1isdpn \
    --discovery-token-ca-cert-hash sha256:44f1def2a041f116bc024f7e57cdc0cdcc8d8f36f0b942bdd27c7f864f645407 --cri-socket unix:///run/containerd/containerd.sock
```

On master node again:
```bash
# Configure kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Deploy Flannel as a network plugin
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

</p>
</details>

### Check that your nodes are running and ready

<details><summary>Solution</summary>
<p>

```bash
kubectl get nodes
NAME               STATUS   ROLES                  AGE     VERSION
k8s-controlplane   Ready    control-plane,master   2m35s   v1.20.7
k8s-node-1         Ready    <none>                 2m7s    v1.20.7
k8s-node-2         Ready    <none>                 117s    v1.20.7
```

</p>
</details>

</p>
</details>

## Provision underlying infrastructure to deploy a Kubernetes cluster

<details><summary>Solution</summary>
<p>

You can use any cloud provider (AWS, Azure, GCP, OpenStack, etc.) and multiple tools to provision nodes for your Kubernetes cluster.

We will deploy a three node cluster, with one master node and two worker nodes.

Three Libvirt/KVM nodes (or any cloud provider you are using):
- k8s-controlplane: 2 vCPUs, 4GB RAM, 40GB Disk, 172.16.1.11/24
- k8s-node-1: 2 vCPUs, 2GB RAM, 40GB Disk, 172.16.1.21/24
- k8s-node-2: 2 vCPUs, 2GB RAM, 40GB Disk, 172.16.1.22/24

OS description:

```bash
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 20.04.4 LTS
Release:	    20.04
Codename:	    focal
```

We will use a local libvirt/KVM baremetal node with terraform (v0.15.3) to provision the three node cluster described above.

```bash
mkdir terraform
cd terraform
wget https://raw.githubusercontent.com/alijahnas/CKA-practice-exercises/master/terraform/cluster-infra.tf
terraform init
terraform plan
terraform apply
```
w
</p>
</details>

## Perform a version upgrade on a Kubernetes cluster using KubeADM

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/

After installing Kubernetes v1.20 here: [install](https://github.com/alijahnas/CKA-practice-exercises/blob/CKA-v1.20/cluster-architecture-installation-configuration.md#use-kubeadm-to-install-a-basic-cluster)

We will now upgrade the cluster to v1.21.

On controlplane node:

```bash
# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.21.1-00
sudo apt-mark hold kubeadm

# Upgrade controlplane node
kubectl drain k8s-controlplane --ignore-daemonsets
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.21.1

# Update Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.21.1-00 kubectl=1.21.1-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Make master node reschedulable
kubectl uncordon k8s-controlplane
```

On worker nodes:

```bash
# Upgrade kubeadm
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.21.1-00
sudo apt-mark hold kubeadm

# Upgrade worker node
kubectl drain k8s-node-1 --ignore-daemonsets
sudo kubeadm upgrade node

# Upgrade kubelet and kubectl
sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet=1.21.1-00 kubectl=1.21.1-00
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload
sudo systemctl restart kubelet

# Make worker node reschedulable
kubectl uncordon k8s-node-1
```

Verify that the nodes are upgraded to v1.21:

```bash
kubectl get nodes
NAME               STATUS   ROLES                  AGE   VERSION
k8s-controlplane   Ready    control-plane,master   21m   v1.21.1
k8s-node-1         Ready    <none>                 21m   v1.21.1
k8s-node-2         Ready    <none>                 21m   v1.21.1
```

</p>
</details>

### Facilitate operating system upgrades

<details><summary>Solution</summary>
<p>

When having a one master node in you cluster, you cannot upgrade the OS system (with reboot) without loosing temporarily access to your cluster.

Here we will upgrade our worker nodes:

```bash
# Hold kubernetes from upgrading
sudo apt-mark hold kubeadm kubelet kubectl

# Upgrade node
kubectl drain k8s-node-1 --ignore-daemonsets
sudo apt update && sudo apt upgrade -y # Be careful about container runtime (e.g., docker) upgrade.

# Reboot node if necessary
sudo reboot

# Make worker node reschedulable
kubectl uncordon k8s-node-1
```

</p>
</details>

## Implement etcd backup and restore

<details><summary>Solution</summary>
<p>

### Backup etcd cluster

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster

Check the version of your etcd cluster depending on how you installed it.

```bash
kubectl exec -it -n kube-system etcd-k8s-controlplane -- etcd --version
etcd Version: 3.4.13
Git SHA: ae9734ed2
Go Version: go1.12.17
Go OS/Arch: linux/amd64
```

```bash
# Download etcd client
wget https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz
tar xzvf etcd-v3.4.13-linux-amd64.tar.gz
sudo mv etcd-v3.4.13-linux-amd64/etcdctl /usr/local/bin

# save etcd snapshot
sudo ETCDCTL_API=3 etcdctl snapshot save --endpoints 172.16.1.11:2379 snapshot.db --cacert /etc/kubernetes/pki/etcd/server.crt --cert /etc/kubernetes/pki/etcd/ca.crt --key /etc/kubernetes/pki/etcd/ca.key

# View the snapshot
ETCDCTL_API=3 sudo etcdctl --write-out=table snapshot status snapshot.db 
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 4056f9fc |    18821 |        809 |     4.1 MB |
+----------+----------+------------+------------+
```

</p>
</details>

### Restore an etcd cluster from a snapshot

<details><summary>Solution</summary>
<p>

Doc: https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster

</p>
</details>

</p>
</details>

# Workloads & Scheduling (15%)

## Understand deployments and how to perform rolling update and rollbacks

Doc: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Questions:
- Create a deployment named `nginx-deploy` in the `ngx` namespace using nginx image version 1.19 with three replicas. Check that the deployment rolled out and show running pods.

<details><summary>Solution</summary>
<p>

```bash
# Create the template from kubectl
kubectl -n ngx create deployment nginx-deploy --replicas=3 --image=nginx:1.19 --dry-run=client -o yaml > nginx-deploy.yaml

# Create the namespace first
kubectl create ns ngx
kubectl apply -f nginx-deploy.yaml
```

Check that the deployment has rolled out and that it is running:

```bash
kubectl -n ngx rollout status deployment/nginx-deploy
deployment "nginx-deploy" successfully rolled out

kubectl -n ngx get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   3/3     3            3           44s
```

Check the pods from the deployment:

```bash
kubectl -n ngx get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-57767fb8cf-fjtls   1/1     Running   0          29s
nginx-deploy-57767fb8cf-krp4m   1/1     Running   0          29s
nginx-deploy-57767fb8cf-xvz8l   1/1     Running   0          29s
```

</p>
</details>

Questions:
- Scale the deployment to 5 replicas and check the status again.
- Then change the image tag of nginx container from 1.19 to 1.20.

<details><summary>Solution</summary>
<p>

```bash
kubectl -n ngx scale deployment nginx-deploy --replicas=5

kubectl -n ngx rollout status deployment nginx-deploy
deployment "nginx-deploy" successfully rolled out

kubectl -n ngx get deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   5/5     5            5           73s

kubectl -n ngx get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-57767fb8cf-fjtls   1/1     Running   0          89s
nginx-deploy-57767fb8cf-krp4m   1/1     Running   0          89s
nginx-deploy-57767fb8cf-rlvwt   1/1     Running   0          26s
nginx-deploy-57767fb8cf-wdxt7   1/1     Running   0          26s
nginx-deploy-57767fb8cf-xvz8l   1/1     Running   0          89s
```

Change the image tag to 1.20:

```bash
kubectl -n ngx edit deployment/nginx-deploy
...
    spec:
      containers:
      - image: nginx:1.20
        imagePullPolicy: IfNotPresent
...
```

Check that new replicaset was created and new pods were deployed:

```bash
kubectl -n ngx get replicaset
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-57767fb8cf   0         0         0       2m54s
nginx-deploy-7bbd8545f9   5         5         5       17s

kubectl -n ngx get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-7bbd8545f9-588mj   1/1     Running   0          30s
nginx-deploy-7bbd8545f9-djql7   1/1     Running   0          30s
nginx-deploy-7bbd8545f9-l77vm   1/1     Running   0          24s
nginx-deploy-7bbd8545f9-p46lm   1/1     Running   0          30s
nginx-deploy-7bbd8545f9-sxn4d   1/1     Running   0          22s
```

</p>
</details>

Questions:
- Check the history of the deployment and rollback to previous revision.
- Then check that the nginx image was reverted to 1.19.

<details><summary>Solution</summary>
<p>

```bash
kubectl -n ngx rollout history deployment nginx-deploy
kubectl -n ngx rollout undo deployment nginx-deploy
deployment.apps/nginx-deploy rolled back

kubectl -n ngx get replicaset
NAME                      DESIRED   CURRENT   READY   AGE
nginx-deploy-57767fb8cf   5         5         5       3m53s
nginx-deploy-7bbd8545f9   0         0         0       76s

kubectl -n ngx get pods
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-57767fb8cf-6mxpd   1/1     Running   0          29s
nginx-deploy-57767fb8cf-7xwls   1/1     Running   0          28s
nginx-deploy-57767fb8cf-dzbkr   1/1     Running   0          28s
nginx-deploy-57767fb8cf-tw7pr   1/1     Running   0          29s
nginx-deploy-57767fb8cf-zklv4   1/1     Running   0          29s

kubectl -n ngx get pods nginx-deploy-57767fb8cf-zklv4 -o jsonpath='{.spec.containers[0].image}'
nginx:1.19

```
</p>
</details>

## Use ConfigMaps and Secrets to configure applications

### Environment variables

Doc: https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/

Questions:
- Create a pod with the latest busybox image running a sleep for 1 hour, and give it an environment variable named `PLANET` with the value `blue`.
- Then exec a command in the container to show that it has the configured environment variable.

<details><summary>Solution</summary>
<p>

The pod yaml `envvar.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: envvar
  name: envvar
spec:
  containers:
  - image: busybox:latest
    name: envvar
    args:
      - sleep
      - "3600"
    env:
      - name: PLANET
        value: "blue"
```

Run and check:

```bash
# Run the pod:
kubectl apply -f envvar.yaml

# Check the env variable:
kubectl exec envvar -- env | grep PLANET
PLANET=blue
```

</p>
</details>

### ConfigMaps

Doc: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/

Questions:
- Create a configmap named `space` with two values `planet=blue` and `moon=white`.
- Create a pod similar to the previous where you have two environment variables taken from the above configmap and show them in the container.

<details><summary>Solution</summary>
<p>

The configmap:
```bash
kubectl create configmap space --from-literal=planet=blue --from-literal=moon=white
configmap/space created
```

The pod yaml `configenvvar.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: configenvvar
  name: configenvvar
spec:
  containers:
  - image: busybox:latest
    name: configenvvar
    args:
      - sleep
      - "3600"
    env:
      - name: PLANET
        valueFrom:
          configMapKeyRef:
            name: space
            key: planet
      - name: MOON
        valueFrom:
         configMapKeyRef:
            name: space
            key: moon
```

Create pod and show variables:

```bash
kubectl apply -f configenvvar.yaml
kubectl exec configenvvar -- env | grep -E "PLANET|MOON"
PLANET=blue
MOON=white
```

</p>
</details>

Questions:
- Create a configmap named `space-system` that contains a file named `system.conf` with the values `planet=blue` and `moon=white`.
- Mount the configmap to a pod and display it from the container through the path `/etc/system.conf`

<details><summary>Solution</summary>
<p>

```bash
cat << EOF > system.conf
planet=blue
moon=white
EOF

kubectl create configmap space-system --from-file=system.conf
```

The pod yaml `confvolume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: confvolume
  name: confvolume
spec:
  containers:
  - image: busybox:latest
    name: confvolume
    args:
      - sleep
      - "3600"
    volumeMounts:
      - name: system
        mountPath: /etc/system.conf
        subPath: system.conf
    resources: {}
  volumes:
  - name: system
    configMap:
      name: space-system
```

Create pod and show file:

```bash
kubectl apply -f confvolume.yaml

kubectl exec confvolume -- cat /etc/system.conf
planet=blue
moon=white
```

</p>
</details>

### Secrets

Doc: https://kubernetes.io/docs/concepts/configuration/secret/

Questions:
- Create a secret from files containing a username and a password.
- Use the secrets to define environment variables and display them.
- Mount the secret to a pod to `admin-cred` folder and display it.

<details><summary>Solution</summary>
<p>

Create secret.

```bash
echo -n 'admin' > username
echo -n 'admin-pass' > password

kubectl create secret generic admin-cred --from-file=username --from-file=password
```

Use secret as environment variables.

secretenv.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: secretenv
  name: secretenv
spec:
  containers:
  - image: busybox:latest
    name: secretenv
    args:
      - sleep
      - "3600"
    env:
      - name: USERNAME
        valueFrom:
          secretKeyRef:
            name: admin-cred
            key: username
      - name: PASSWORD
        valueFrom:
          secretKeyRef:
            name: admin-cred
            key: password

```

```bash
kubectl apply -f secretenv.yaml

kubectl exec secretenv -- env | grep -E "USERNAME|PASSWORD"
USERNAME=admin
PASSWORD=admin-pass
```

Mount a secret to pod as a volume.

secretvolume.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: secretvolume
  name: secretvolume
spec:
  containers:
  - image: busybox:latest
    name: secretvolume
    args:
      - sleep
      - "3600"
    volumeMounts:
      - name: admincred
        mountPath: /etc/admin-cred
        readOnly: true
  volumes:
  - name: admincred
    secret:
      secretName: admin-cred

```

```bash
kubectl apply -f secretvolume.yaml

kubectl exec secretvolume -- ls /etc/admin-cred
password
username

kubectl exec secretvolume -- cat /etc/admin-cred/username
admin

kubectl exec secretvolume -- cat /etc/admin-cred/password
admin-pass
```

</p>
</details>

## Know how to scale applications

Docs:
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

Questions:
- Create a deployment with the latest nginx image and scale the deployment to 4 replicas.

<details><summary>Solution</summary>
<p>

```bash
kubectl create deployment scalable --image=nginx:latest
kubectl scale deployment scalable --replicas=4
kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
scalable-6bbdb8895b-2fp5k   1/1     Running   0          6s
scalable-6bbdb8895b-2lww8   1/1     Running   0          6s
scalable-6bbdb8895b-l6ctd   1/1     Running   0          16s
scalable-6bbdb8895b-rh8cz   1/1     Running   0          6s
```

</p>
</details>

Questions:
- Autoscale a deployment to have a minimum of two pods and a maximum of 6 pods and that transitions when cpu usage goes above 70%.

<details><summary>Solution</summary>
<p>

In order to use Horizontal Pod Autoscaling, you need to have the metrics server installed in you cluster.

```bash
# Configure metrics server
git clone https://github.com/kubernetes-sigs/metrics-server
# Add --kubelet-insecure-tls to metrics-server/manifests/base/deployment.yaml if necessary
...
      containers:
      - name: metrics-server
        image: gcr.io/k8s-staging-metrics-server/metrics-server:master
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=443
          - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
          - --kubelet-use-node-status-port
          - --metric-resolution=15s
          - --kubelet-insecure-tls
...

# Deploy the metrics server
kubectl apply -k metrics-server/manifests/base/

# Autoscale a deployment
kubectl create deployment autoscalable --image=nginx:latest
kubectl autoscale deployment autoscalable --min=2 --max=6 --cpu-percent=70
kubectl get hpa
kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
autoscalable-6cdbc9b4c9-2c4kh   1/1     Running   0          28s
autoscalable-6cdbc9b4c9-2vdqj   1/1     Running   0          6s
```

</p>
</details>

## Understand the primitives used to create robust, self-healing, application deployments

Docs:
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/

<details><summary>Solution</summary>
<p>

A deployment uses a replicaset object to maintain the right number of desired replicas of a pod.
See section "Understand Deployments and how to perform rolling updates and rollbacks" above to see how deployments handle replicaset for updating.

</p>
</details>

### Understand the role of DaemonSets

Doc: https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/

Questions:
- Create a DaemonSet with the latest busybox image and see that it runs on all nodes.

<details><summary>Solution</summary>
<p>

daemonset.yaml
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    type: daemon
  name: daemontest
spec:
  selector:
    matchLabels:
      run: daemon
  template:
    metadata:
      labels:
        run: daemon
      name: daemonpod
    spec:
      containers:
      - image: busybox:latest
        name: daemonpod
        args:
          - sleep
          - "3600"
```

```bash
kubectl apply -f daemonset.yaml

kubectl get pods -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
daemontest-qc6sb   1/1     Running   0          14s   10.244.2.22   k8s-node-2   <none>           <none>
daemontest-st9wn   1/1     Running   0          14s   10.244.1.23   k8s-node-1   <none>           <none>
```

If you want the daemonset to run on the controlplane node, it needs to tolerate the controlnode taints, for example node-role.kubernetes.io/master:NoSchedule.

</p>
</details>

## Understand how to resource limits can affect Pod scheduling

Doc: https://kubernetes.io/docs/concepts/policy/resource-quotas/

Questions:
- Create a pod with a busybox container that requests 1G of memory and half a CPU, and has limits at 2G of memory and a whole CPU.

<details><summary>Solution</summary>
<p>

podquota.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podquota
  name: podquota
spec:
  containers:
  - image: busybox:latest
    name: podquota
    args:
      - sleep
      - "3600"
    resources:
      requests:
        memory: "1Gi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1"

```

```bash
kubectl apply -f podquota.yaml

kubectl describe pod podquota
...
    Limits:
      cpu:     1
      memory:  2Gi
    Requests:
      cpu:        500m
      memory:     1Gi
...
```

</p>
</details>

### Use label selectors to schedule Pods

Doc: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

Questions:
- Label a node with `kind=special` and schedule a pod to that node.

<details><summary>Solution</summary>
<p>

pod-selector.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podsel
  name: podsel
spec:
  containers:
  - image: busybox:latest
    name: podsel
    args:
      - sleep
      - "3600"
  nodeSelector:
    kind: special
```

```bash
kubectl label nodes k8s-node-1 kind=special
kubectl apply -f pod-selector.yaml

kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
podsel   1/1     Running   0          4s    10.244.1.24   k8s-node-1   <none>           <none>

```

</p>
</details>

Questions:
- Use antiaffinity to launch a pod to a different node than the pod where the first one was scheduled.

<details><summary>Solution</summary>
<p>

pod-antiaffinity.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podaff
  name: podaff
spec:
  containers:
  - image: busybox:latest
    name: podaff
    args:
      - sleep
      - "3600"
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: run
                operator: In
                values:
                  - podsel
          topologyKey: kubernetes.io/hostname
```

```bash
kubectl apply -f pod-antiaffinity.yaml

kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE    IP            NODE         NOMINATED NODE   READINESS GATES
podaff   1/1     Running   0          7s     10.244.2.24   k8s-node-2   <none>           <none>
podsel   1/1     Running   0          2m3s   10.244.1.24   k8s-node-1   <none>           <none>

```

</p>
</details>

Questions:
- Taint a node with `type=special:NoSchedule`, make the other node unschedulable, and create a pod to tolerate this taint.

<details><summary>Solution</summary>
<p>

pod-toleration.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podtol
  name: podtol
spec:
  containers:
  - image: busybox:latest
    name: podtol
    args:
      - sleep
      - "3600"
  tolerations:
  - key: "type"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"

```

```bash
kubectl taint node k8s-node-1 type=special:NoSchedule
kubectl cordon k8s-node-2 #to force the scheduler to choose k8s-node-1
kubectl apply -f pod-toleration.yaml

kubectl get pods -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
podtol   1/1     Running   0          6s    10.244.1.26   k8s-node-1   <none>           <none>

# uncordon and remove taint
kubectl uncordon k8s-node-2
kubectl taint node k8s-node-1 type=special:NoSchedule- 
```

</p>
</details>

### Understand how to run multiple schedulers and how to configure Pods to use them

Doc: https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/

### Manually schedule a Pod without a scheduler

Doc: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodename

Questions:
- Force a pod to be on a specific node with using the scheduler, and show that it was assigned to it.

<details><summary>Solution</summary>
<p>

pod-node.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podnode
  name: podnode
spec:
  containers:
  - image: busybox:latest
    name: podnode
    args:
      - sleep
      - "3600"
  nodeName: k8s-node-2
```

```bash
kubectl apply -f pod-node.yaml

kubectl get pods -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
podnode   1/1     Running   0          14s   10.244.2.25   k8s-node-2   <none>           <none>
```

</p>
</details>

### Display scheduler events

Doc: https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/#verifying-that-the-pods-were-scheduled-using-the-desired-schedulers

Questions:
- Check the scheduler events.

<details><summary>Solution</summary>
<p>

```bash
kubectl get events
kubectl get events --all-namespaces
```

</p>
</details>

### Know how to configure the Kubernetes scheduler

Doc: https://kubernetes.io/docs/concepts/scheduling/scheduler-perf-tuning/

## Awareness of manifest management and common templating tools

You can use either Helm or Kustomize to make Kubernetes templates:
- Helm: https://helm.sh/docs/intro/quickstart/
- Kustomize: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/

# Services & Networking (20%)

## Understand host networking configuration on the cluster nodes

Doc: https://kubernetes.io/docs/concepts/cluster-administration/networking/

## Understand connectivity between Pods

Doc: https://kubernetes.io/docs/concepts/cluster-administration/networking/

## Understand ClusterIP, NodePort, LoadBalancer service types and endpoints

Doc: https://kubernetes.io/docs/concepts/services-networking/service/

Questions:
- Create a deployment with the latest nginx image and two replicas.
- Expose it's port 80 through a service of type NodePort.
- Show all elements, including the endpoints.
- Get the nginx index page through the NodePort.

<details><summary>Solution</summary>
<p>

```bash
kubectl create deployment nginx --image=nginx:latest
kubectl scale deployment nginx --replicas=2
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
kubectl describe svc nginx
Name:                     nginx
Namespace:                default
Labels:                   app=nginx
Annotations:              <none>
Selector:                 app=nginx
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.104.222.43
IPs:                      10.104.222.43
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32740/TCP
Endpoints:                10.244.1.27:80,10.244.2.26:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

kubectl get pods -l app=nginx -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx-55649fd747-6xvlq   1/1     Running   0          35s   10.244.2.26   k8s-node-2   <none>           <none>
nginx-55649fd747-vnbjz   1/1     Running   0          35s   10.244.1.27   k8s-node-1   <none>           <none>

# We are getting the page through IP address of the controlplane node and the port allocated by the NodePort service
curl http://172.16.1.11:32740
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
...

```

</p>
</details>


### Deploy and configure network load balancer

Doc: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/

Questions:
- Do the same exercice as in the previous section but use a Load Balancer service type rather than a NodePort.

Hint: If you are not running your cluster on a cloud providing a load balancer service, you can use [MetalLB](https://metallb.universe.tf/installation/)

<details><summary>Solution</summary>
<p>

```bash
# We will deploy MetalLB first to provide Load Balancer service type
mkdir metallb
cd metallb
wget https://raw.githubusercontent.com/google/metallb/v0.9.6/manifests/namespace.yaml
wget https://raw.githubusercontent.com/google/metallb/v0.9.6/manifests/metallb.yaml

# We are giving MetalLB an IP range from our cluster infra to allocate from
cat << EOF > metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol:
      addresses:
      - 172.16.1.101-172.16.1.150
EOF

# Apply the manifests
kubectl apply -f namespace.yaml
kubectl apply -f metallb-config.yaml
kubectl apply -f metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"

# Now we create the deployment with a Load Balancer service type
kubectl create deployment nginx-lb --image=nginx:latest
kubectl scale deployment nginx-lb --replicas=2
kubectl expose deployment nginx-lb --port=80 --target-port=80 --type=LoadBalancer
kubectl describe svc nginx-lb
Name:                     nginx-lb
Namespace:                default
Labels:                   app=nginx-lb
Annotations:              <none>
Selector:                 app=nginx-lb
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.102.200.223
IPs:                      10.102.200.223
LoadBalancer Ingress:     172.16.1.101
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32193/TCP
Endpoints:                10.244.1.28:80,10.244.2.30:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age   From                Message
  ----    ------        ----  ----                -------
  Normal  IPAllocated   46s   metallb-controller  Assigned IP "172.16.1.101"
  Normal  nodeAssigned  46s   metallb-speaker     announcing from node "k8s-node-2"

# We are getting the page through the IP address allocated by MetalLB from the pool we provided
curl http://172.16.1.101:80
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
...
 
```

</p>
</details>

## Know how to use Ingress controllers and Ingress resources

Doc: https://kubernetes.io/docs/concepts/services-networking/ingress/

Questions:
- Keep the previous deployment of nginx and add a new deployment using the image `bitnami/apache` with two replicas.
- Expose its port 8080 through a service and query it.
- Deploy nginx ingress controller
- Create an ingress service that redirects /nginx to the nginx service and /apache to the apache service.

<details><summary>Solution</summary>
<p>

```bash
kubectl create deployment apache-lb --image=bitnami/apache:latest
kubectl scale deployment apache-lb --replicas=2
kubectl expose deployment apache-lb --port=8080 --target-port=8080 --type=LoadBalancer # Replace by NodePort if you don't have a LoadBalancer provider
kubectl describe svc apache-lb
Name:                     apache-lb
Namespace:                default
Labels:                   app=apache-lb
Annotations:              <none>
Selector:                 app=apache-lb
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.101.245.9
IPs:                      10.101.245.9
LoadBalancer Ingress:     172.16.1.101
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30174/TCP
Endpoints:                10.244.1.32:8080,10.244.2.35:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age              From                Message
  ----    ------        ----             ----                -------
  Normal  IPAllocated   4s               metallb-controller  Assigned IP "172.16.1.101"
  Normal  nodeAssigned  3s (x2 over 3s)  metallb-speaker     announcing from node "k8s-node-1"

curl http://172.16.1.101:8080
<html><body><h1>It works!</h1></body></html>
```

web-ingress.yaml:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: nginx-or-apache.com
    http:
      paths:
      - pathType: Prefix
        path: /nginx
        backend:
          service:
            name: nginx-lb
            port:
              number: 80
      - pathType: Prefix
        path: /apache
        backend:
          service:
            name: apache-lb
            port:
              number: 8080
```

Deploy nginx ingress controller:
```bash
# If using metallb or cloud deployment
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/cloud/deploy.yaml
# If using NodePort
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/baremetal/deploy.yaml

kubectl -n ingress-nginx get svc
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.97.187.125    172.16.1.103   80:31416/TCP,443:32749/TCP   15s
ingress-nginx-controller-admission   ClusterIP      10.107.225.180   <none>         443/TCP                      15s
```

Deploy web-ingress.yaml:
```bash
kubectl apply -f web-ingress.yaml
kubectl describe ingress web-ingress
Name:             web-ingress
Namespace:        default
Address:          172.16.1.103
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  nginx-or-apache.com
                       /nginx    nginx-lb:80 (10.244.1.30:80,10.244.2.32:80)
                       /apache   apache-lb:8080 (10.244.1.32:8080,10.244.2.35:8080)
Annotations:           <none>
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    24s (x3 over 3m21s)  nginx-ingress-controller  Scheduled for sync

```

</p>
</details>

## Know how to configure and use CoreDNS

Doc: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
Doc: https://kubernetes.io/docs/tasks/administer-cluster/coredns/

Questions:
- Create a busybox pod and resolve the nginx and apache services created earlier from within the pod.

<details><summary>Solution</summary>
<p>

```bash
kubectl run busybox --image=busybox --rm -it --restart=Never -- sh
If you don't see a command prompt, try pressing enter.
# nslookup apache-lb
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	apache-lb.default.svc.cluster.local
Address: 10.101.245.9

# nslookup nginx-lb
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	nginx-lb.default.svc.cluster.local
Address: 10.108.72.239

```

</p>
</details>

## Choose an appropriate container network interface plugin

<details><summary>Solution</summary>
<p>

Docs:
- https://kubernetes.io/docs/concepts/cluster-administration/networking/
- https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

We installed flannel in [Provision underlying infrastructure to deploy a Kubernetes cluster](https://github.com/alijahnas/CKA-practice-exercises/blob/CKA-v1.20/cluster-architecture-installation-configuration.md#create-a-cluster-with-kubeadm)

</p>
</details>

# Storage (10%)

## Understand storage classes, persistent volumes

Doc: https://kubernetes.io/docs/concepts/storage/storage-classes/
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

## Understand volume mode, access modes and reclaim policies for volumes

Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#reclaiming
Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#volume-mode

## Understand persistent volume claims primitive

Doc: https://kubernetes.io/docs/tasks/administer-cluster/change-pv-reclaim-policy/

## Know how to configure applications with persistent storage

Doc: https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Questions:
- Create a pod and mount a volume with hostPath directory.
- Check that the contents of the directory are accessible through the pod.

<details><summary>Solution</summary>
<p>

pv-pod.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pv-pod
  name: pv-pod
spec:
  containers:
  - image: busybox:latest
    name: pv-pod
    args:
      - sleep
      - "3600"
    volumeMounts:
    - name: data
      mountPath: "/data"
  volumes:
  - name: data
    hostPath:
      path: "/home/ubuntu/data/"
```

```bash
# Create directory and file inside it on worker nodes
mkdir /home/ubuntu/data
touch data/file

kubectl apply -f pv-pod.yaml
kubectl exec pv-pod -- ls /data
file
```

</p>
</details>

Questions:
- Create a persistent volume from hostPath and a persistent volume claim corresponding tothat PV. Create a pod that uses the PVC and check that the volume is mounted in the pod.
- Create a file from the pod in the volume then delete it and create a new pod with the same volume and show the created file by the first pod.

<details><summary>Solution</summary>
<p>

pv-data.yaml:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-data
spec:
  storageClassName: "local"
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/home/ubuntu/data"

```

pvc-data.yaml:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-data
spec:
  storageClassName: "local"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

pvc-pod.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pvc-pod
  name: pvc-pod
spec:
  containers:
  - image: busybox:latest
    name: pvc-pod
    args:
      - sleep
      - "3600"
    volumeMounts:
    - name: data
      mountPath: "/data"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-data

```

Create a pod with the PVC. Create a file on volume. Delete the pod and create a new one with the same volume. Check that the file has persisted.

```bash
kubectl apply -f pv-data.yaml
kubectl apply -f pvc-data.yaml
kubectl apply -f pvc-pod.yaml

kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
pv-data   1Gi        RWO            Retain           Bound    default/pvc-data   local                   20m

kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-data   Bound    pv-data   1Gi        RWO            local          20m

# Check that the volume has been mounted
kubectl exec pvc-pod -- ls /data/
file

# Create a new file
kubectl exec pvc-pod -- touch /data/file2

# Delete the pod
kubectl delete -f pvc-pod.yaml

# Copy the pvc-pod.yaml and change the name of the pod to pvc-pod-2
kubectl apply -f pvc-pod-2.yaml

# Check that the file from previous pod has persisted on volume
kubectl exec pvc-pod-2 -- ls /data/
file
file2
```

</p>
</details>

# Troubleshooting (30%)

## Evaluate cluster and node logging

Questions:
- Get cluster components logs.

<details><summary>Solution</summary>
<p>

Logs depend on how your cluster was deployed.

For our deployment done in [Cluster Architecture, Installation & Configuration](https://github.com/alijahnas/CKA-practice-exercises/blob/CKA-v1.20/cluster-architecture-installation-configuration.md) here is how to get logs.

```bash
# Kubelet on all nodes
sudo journalctl -u kubelet

# API server
kubectl -n kube-system logs kube-apiserver-k8s-controlplane

# Controller Manager
kubectl -n kube-system logs kube-controller-manager-k8s-controlplane

# Scheduler
kubectl -n kube-system logs kube-scheduler-k8s-controlplane

```

</p>
</details>

## Understand how to monitor applications

Doc: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

Questions:
- Create an nginx pod with a liveness and a readiness probe for the port 80.

<details><summary>Solution</summary>
<p>

pod-ness.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    run: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
```

```bash
kubectl apply -f pod-ness.yaml
kubectl describe pods nginx
...
    Liveness:       http-get http://:80/ delay=0s timeout=1s period=10s #success=1 #failure=3
    Readiness:      http-get http://:80/ delay=5s timeout=1s period=5s #success=1 #failure=3
...

```

</p>
</details>

### Understand how to monitor all cluster components

Doc: https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/

Questions:
- Install the metrics server and show metrics for nodes and for pods in `kube-system` namespace.

<details><summary>Solution</summary>
<p>

```bash
git clone https://github.com/kubernetes-sigs/metrics-server
# Add --kubelet-insecure-tls to metrics-server/manifests/base/deployment.yaml if necessary
...
      containers:
      - name: metrics-server
        image: gcr.io/k8s-staging-metrics-server/metrics-server:master
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=443
          - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
          - --kubelet-use-node-status-port
          - --metric-resolution=15s
          - --kubelet-insecure-tls
...

# Deploy the metrics server
kubectl apply -k metrics-server/manifests/base/

# Wait for the server to get metrics and show them
kubectl top nodes
NAME               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
k8s-controlplane   271m         13%    1075Mi          28%
k8s-node-1         115m         5%     636Mi           33%
k8s-node-2         97m          4%     564Mi           29%

kubectl top pods -n kube-system
NAME                                       CPU(cores)   MEMORY(bytes)
coredns-558bd4d5db-6cdkr                   6m           11Mi
coredns-558bd4d5db-k9qxs                   5m           19Mi
etcd-k8s-controlplane                      27m          71Mi
kube-apiserver-k8s-controlplane            112m         312Mi
kube-controller-manager-k8s-controlplane   34m          56Mi
kube-flannel-ds-nr5ms                      4m           11Mi
kube-flannel-ds-vl79c                      5m           13Mi
kube-flannel-ds-xvp8z                      7m           14Mi
kube-proxy-jjvc9                           2m           20Mi
kube-proxy-mwwnn                           1m           17Mi
kube-proxy-wr4v7                           1m           21Mi
kube-scheduler-k8s-controlplane            8m           18Mi
metrics-server-ffc48cc6c-g92v8             6m           16Mi
```

</p>
</details>

## Manage container stdout & stderr logs

Doc: https://kubernetes.io/docs/concepts/cluster-administration/logging/

Questions:
- Get logs from the nginx pod deployed earlier and redirect them to a file.

<details><summary>Solution</summary>
<p>

```bash
kubectl logs nginx > nginx.log
```

</p>
</details>

## Troubleshoot application failure

Doc: https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/

Questions:
- Launch a pod with a busybox container that launches with the `sheep 3600` command (this command doesn't exist.
- Get the logs from the pod, then correct the error to make it launch `sleep 3600`.

<details><summary>Solution</summary>
<p>

podfail.yaml:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: podfail
  name: podfail
spec:
  containers:
  - image: busybox:latest
    name: podfail
    args:
      - sheep
      - "3600"
```

```bash
kubectl apply -f podfail.yaml

kubectl describe pods podfail
...
Warning  Failed     5s (x2 over 6s)  kubelet            Error: failed to create containerd task: OCI runtime create failed: container_linux.go:367: starting container process caused: exec: "sheep": executable file not found in $PATH: unknown
...

kubectl delete -f podfail.yaml
# Change sheep to sleep
kubectl apply -f podfail.yaml
...
Normal  Started    4s    kubelet            Started container podfail #Not failing anymore
...
```

</p>
</details>

## Troubleshoot cluster component failure

### Troubleshoot control plane failure

Doc: https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/

Questions:
- Get logs from the control plane in the `kube-system` namespace.

<details><summary>Solution</summary>
<p>

```bash
# API server
kubectl -n kube-system logs kube-apiserver-k8s-controlplane

# Controller Manager
kubectl -n kube-system logs kube-controller-manager-k8s-controlplane

# Scheduler
kubectl -n kube-system logs kube-scheduler-k8s-controlplane
```

</p>
</details>

### Troubleshoot worker node failure

Doc: https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/

Questions:
- Check the node status and the system logs for kubelet on the failing node.

<details><summary>Solution</summary>
<p>

```bash
kubectl describe node k8s-node-1

# From k8s-node-1 if reachable
sudo journalctl -u kubelet | grep -i error
```

</p>
</details>

## Troubleshoot networking

Doc: https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/

Questions:
- Check the `kube-dns` service running in the `kube-system` namespace and check the endpoints behind the service. Check the pods that serve the endpoints.

<details><summary>Solution</summary>
<p>

```bash
kubectl -n kube-system describe svc kube-dns
Name:              kube-dns
Namespace:         kube-system
Labels:            k8s-app=kube-dns
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=CoreDNS
Annotations:       prometheus.io/port: 9153
                   prometheus.io/scrape: true
Selector:          k8s-app=kube-dns
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.96.0.10
IPs:               10.96.0.10
Port:              dns  53/UDP
TargetPort:        53/UDP
Endpoints:         10.244.1.7:53,10.244.1.8:53
Port:              dns-tcp  53/TCP
TargetPort:        53/TCP
Endpoints:         10.244.1.7:53,10.244.1.8:53
Port:              metrics  9153/TCP
TargetPort:        9153/TCP
Endpoints:         10.244.1.7:9153,10.244.1.8:9153
Session Affinity:  None
Events:            <none>

kubectl -n kube-system describe ep kube-dns
Name:         kube-dns
Namespace:    kube-system
Labels:       k8s-app=kube-dns
              kubernetes.io/cluster-service=true
              kubernetes.io/name=CoreDNS
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-05-19T08:39:25Z
Subsets:
  Addresses:          10.244.1.7,10.244.1.8
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    dns-tcp  53    TCP
    dns      53    UDP
    metrics  9153  TCP

Events:  <none>

kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
NAME                       READY   STATUS    RESTARTS   AGE    IP           NODE         NOMINATED NODE   READINESS GATES
coredns-558bd4d5db-6cdkr   1/1     Running   1          5d3h   10.244.1.8   k8s-node-1   <none>           <none>
coredns-558bd4d5db-k9qxs   1/1     Running   1          5d3h   10.244.1.7   k8s-node-1   <none>           <none>
```

</p>
</details>
