# Rancher on Hetzner
This is a step by step guide on how to setup k8s on Hetzner. This guide is based on the following articles:

- https://medium.com/@jmrobles/how-to-create-a-kubernetes-cluster-with-rancher-on-hetzner-3b2f7f0c037a
- https://medium.com/@jmrobles/how-to-setup-hetzner-load-balancer-on-a-kubernetes-cluster-2ce79ca4a27b

## 1. Project setup in Hetzner
- Create project
- Add you ssh key
- Add two read / write API keys
     - One named `cluster`
     - One named `rancher`
- Create floating IP
- Setup domain to floating IP (we assume `k8s.example.com`)
- Setup new network (default settings suffice)

## 2. Create server for Rancher
- Create server (e.g., cx21-ceph), select
    - ubuntu 20.04
    - your network
    - your ssh key
    - user data as follows
    
```
#cloud-config

package_update: true
package_upgrade: true
package_reboot_if_required: true

packages:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg-agent
  - software-properties-common
  - snapd
  - git

runcmd:
  - snap install kubectl --classic
  - snap install helm --classic
  - curl -sfL https://get.rke2.io | sh -
  - 'echo "network:\n  version: 2\n  ethernets:\n    eth0:\n      addresses:\n      - FLOATINGIP/32" > /etc/netplan/60-floating-ip.yaml'
  - netplan apply
  - systemctl enable rke2-server.service
  - systemctl start rke2-server.service
```

Replace FLOATINGIP above to your floating IP.

After a while, you have an RKE2 single node cluster up and running.

## 3. Install Rancher on RKE2 cluster
- assign the floating IP to the server
- ssh to your new server and execute the following commands to install cert-manager to create the https certificate and rancher.

**Note: If you are playing around, replace letsEncrypt with rancher so as to not get rate limited with letsEncrypt**

```
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.0.4

helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=k8s.example.com \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=YOUREMAIL \
  --set replicas=1
```
Replace YOURMAIL with your email and `k8s.example.com` with your domain for the floating IP.

If you don't have a dns name, use the `xip.io` service.

## 4. Setup Rancher Node Driver
- login to `https://yourdomain`
- Navigate to `tools -> drivers -> node drivers`
- click on `Add Node Driver`
- set the following

Download URL: `https://github.com/JonasProgrammer/docker-machine-driver-hetzner/releases/download/3.0.0/docker-machine-driver-hetzner_3.0.0_linux_amd64.tar.gz`

Custom UI URL: `https://storage.googleapis.com/hcloud-rancher-v2-ui-driver/component.js`

- click on `Add Domain`

Whitelist Domains: `storage.googleapis.com`

## 5. Node Template(s)
- In Rancher, under your user icon (top right), click on `Node Templates -> Add Template`
- Select `Hetzner` and add your `rancher` api token
- Configure:
    - Region (as you used before)
    - Ubuntu 20.04
    - Size (4GB memory recommended)
    - your network
    - check `Use private network`
    - set name
    - Cloud-init as follows
    
```
#cloud-config

ssh_authorized_keys:
  - YOURKEY

packages:
  - open-iscsi
```

Replace `YOURKEY` with your ssh public key.

Replace Template steps as wished.

## 6. Add Cluster
- Go to clusters and click `Add Cluster`
- Set
    - Clustername as wished
    - Count 3
    - template as wished
    - check etcd, control plane, worker
    - In-Tree Cloud Provider, check `External`
    - Then use `Edit as YAML`
    - Under `rancher_kubernetes_engine_config` paste:
    
```
  addons: |-
    ---
    apiVersion: v1
    stringData:
      token: APITOKEN
    kind: Secret
    metadata:
      name: hcloud
      namespace: kube-system
  addons_include:
    - https://raw.githubusercontent.com/hetznercloud/hcloud-cloud-controller-manager/master/deploy/dev-ccm.yaml
```

Replace `APITOKEN` with the hetzner cluster api token.

Now click create, and wait until the cluster is ready, then select the cluster and download the kubeconfig.

## 7. Setup Hetzner CSI
- setup token as secret in k8s:

```
kubectl apply -f -  <<"EOF"
# secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: hcloud-csi
  namespace: kube-system
stringData:
  token: YOURTOKEN
EOF
```

Replace `YOURTOKEN` with the cluster token.

- Add the Hetzner CSI:

```
kubectl apply -f https://raw.githubusercontent.com/hetznercloud/csi-driver/v1.5.1/deploy/kubernetes/hcloud-csi.yml
```

You can now use the storage class `hcloud-volumes`

## 8. Setup Ingress Controller
- In Hetzner, create a load balancer, select default options, remember the balancer's name
- Create a file `hetzner-lb-options.yaml`:

```
controller:
  service:
    annotations:
      load-balancer.hetzner.cloud/name: "LBNAME"
```

Replace `LBNAME` with the name of your load balancer.

- Deploy nginx ingress controller:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install -n ingress-nginx -f hetzner-lb-options.yaml ingress-nginx ingress-nginx/ingress-nginx
```

Wait until the healthcheck of the Load Balancer is green (will flap a little bit first).

**FIN** You now have a cluster with storage, ingress controller and load balancer.

# Other Examples
## 1. busybox www service
Example service, if you see 404 that are not from nginx, it works.
```
kubectl apply -f - <<"EOF"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: busybox
  namespace: default
  labels:
    app: busybox
spec:
  replicas: 3
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      containers:
      - name: busybox
        image: busybox
        imagePullPolicy: Always
        command: ["httpd", "-fvvp", "8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: busybox
  namespace: default
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: busybox
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: busybox-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: k8s.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: busybox
          servicePort: 8080
EOF
```

Check with `httpie`: `http LBIP Host:k8s.example.com`

# 2. Hetzner volume storage
```
kubectl apply -f - <<"EOF"
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ubuntu
spec:
  selector:
    matchLabels:
      app: ubuntu
  serviceName: "ubuntu"
  replicas: 1
  template:
    metadata:
      labels:
        app: ubuntu
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: ubuntu
        image: ubuntu
        command: ["sh", "-c", "sleep infinity"]
        volumeMounts:
        - name: ubuntu 
          mountPath: /mnt/ubuntu
  volumeClaimTemplates:
  - metadata:
      name: ubuntu
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "hcloud-volumes"
      resources:
        requests:
          storage: 20Gi
EOF
```

## 3. Install longhorn
Under cluster go to `Cluster Explorer -> Apps`, select `Longhorn`, then click on `Install`

Now you can use storage class "longhorn", it will then use the node's local storage and replicate the data across the nodes.