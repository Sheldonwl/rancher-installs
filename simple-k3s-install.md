# Install the latest K3s version
```
curl -sfL https://get.k3s.io | sh -
```

# Install Helm 
## Install the Helm binary
For any other OS distro's, please checkout: https://helm.sh/docs/intro/install/
```
sudo snap install helm --classic
```

## Set kubeconfig path
Option 1: This will set the location K3s uses for the kubeconfig. 
```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

Option 2: You can also create the default location and copy the kubeconfig and set the correct permissions
```
sudo chmod -R +644 /etc/rancher/
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown -R root:root ~/.kube
chmod +644 ~/.kube/config
```

# Install Rancher

## Install cert-manager
Before we deploy the Rancher Helm chart, we need to first deploy cert-manager. This is not necessary when deploying Rancher with custom certs, however we won't go into that in this tutorial. I'd also recommend to check the Rancher docs for the latest supported cert-manager version: https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/#4-install-cert-manager

```
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.5.1
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
```

## Add the Rancher chart repository
For the latest version (including x.x.0 versions) (most chosen option): 
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

For the stable versions (typically this means the x.x.1 or x.x.2 version): 
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

For an alpha version: 
```
helm repo add rancher-alpha https://releases.rancher.com/server-charts/alpha
helm repo update
```

You can check which versions are available after adding the repo, by running: 
```
helm search repo --versions
```

## Deploy Rancher
Before we deploy Rancher, let's create the namespace: 
```
kubectl create namespace cattle-system
```

## Default install with self-signed certs (most chosen option)
Change the following (if needed):
- rancher-latest: change the repository if you would like a specific version (not needed for the default latest installation)
- HOSTNAME: set the hostname you will use for Rancher. you might need to configure your DNS or edit your local hosts file. 
- PASSWORD: If you would like to set your own bootstrap password, you can ad a value here. If you remove '--set bootstrapPassword=PASSWORD', Rancher will generate one for you and you will have to retrieve that first. 
- replicas=1: Only change this if you are planning on creating an HA setup. The default is 3 replicas and if that's on one node, it will only make things (like debugging) harder with 3 containers. 
```
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=HOSTNAME --set bootstrapPassword=PASSWORD --set replicas=1
kubectl -n cattle-system rollout status deploy/rancher
```


## Install with LetsEncrypt certs
For this option you will need to have a public DNS configured and a server for LetsEncrypt to reach. Follow the docs for more info: https://rancher.com/docs/rancher/v2.6/en/installation/install-rancher-on-k8s/#5-install-rancher-with-helm-and-your-chosen-certificate-option

```
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=HOSTNAME --set replicas=1 --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=EMAIL
kubectl -n cattle-system rollout status deploy/rancher
```


# Tips & Tricks
## Versioned K3s install
You can find the available versions in the K3s releases and then swap out 'v1.21.9+k3s1'. 
```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.21.9+k3s1" sh -
```

## Get K3s kubeconfig file
```
cat /etc/rancher/k3s/k3s.yaml
```

## Uninstall an existing instance & all resources on the node
If you break K3s or just want to start over, you can easily delete the entire distro, all containers and all used resources by running the uninstall script.  
Server: 
```
sh /usr/local/bin/k3s-uninstall.sh
```
Agent:
```
sh /usr/local/bin/k3s-agent-uninstall.sh
```