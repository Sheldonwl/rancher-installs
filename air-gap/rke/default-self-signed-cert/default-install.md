# Nodes
- 1x Registry *E.g. (10.133.0.5:5000 will be used during this guide)*  
- 1x Rancher Server  
- Downstream cluster(s)  

## Node pre-requisites 
- Make sure the necessary ports are open: https://rancher.com/docs/rancher/v2.x/en/installation/requirements/ 

# In environment with internet access
## Find assets required for your Rancher version 
Go to our releases page, find the Rancher v2.x.x release that you want to install, and click Assets. Note: Don’t use releases marked rc or Pre-release, as they are not stable for production environments.
https://github.com/rancher/rancher/releases

From the release’s Assets section, download the following files, which are required to install Rancher in an air gap environment:  

**rancher-images.txt**  
This file contains a list of images needed to install Rancher, provision clusters and user Rancher tools.  

**rancher-save-images.sh**  
This script pulls all the images in the rancher-images.txt from Docker Hub and saves all of the images as rancher-images.tar.gz.  

**rancher-load-images.sh**  
This script loads images from the rancher-images.tar.gz file and pushes them to your private registry.

Run the following to get the necessary scripts and text file:  
(Make sure to change the version, to a more recent release)
```
wget https://github.com/rancher/rancher/releases/download/v2.4.5/rancher-images.txt
wget https://github.com/rancher/rancher/releases/download/v2.4.5/rancher-load-images.sh
wget https://github.com/rancher/rancher/releases/download/v2.4.5/rancher-save-images.sh
```

Sort the images and pull and compress them all in one archive: 
```
sort -u rancher-images.txt -o rancher-images.txt
./rancher-save-images.sh --image-list ./rancher-images.txt
```

## Tools
### Install Helm: 
```
wget -O helm.tar.gz https://get.helm.sh/helm-v3.0.2-linux-amd64.tar.gz
tar -zxf helm.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
sudo chmod +x /usr/local/bin/helm
rm -rf linux-amd64
rm -f helm.tar.gz
helm version --client
```

### Install RKE 
*Go to the Rancher release page and for your specific version check under Tools what RKE version should be used.*  
```
wget https://github.com/rancher/rke/releases/download/v1.1.3/rke_linux-amd64
mv rke_linux-amd64 rke
chmod +x rke
sudo mv rke /usr/local/bin/rke
```

### Install kubectl 
```
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### Render Helm Charts
If you use the default self-signed cert option, you will need to install cert-manager and then Rancher with the following command:   
```
helm repo add jetstack https://charts.jetstack.io
helm repo update
# Fetch the cert-manager chart. This will pull down the chart and save it in the current directory as a .tgz file.
helm fetch jetstack/cert-manager --version v0.12.0

# Render the cert-manager template with the options you would like to use to install the chart. Remember to set the image.repository option to pull the image from your private registry. This will create a cert-manager directory with the Kubernetes manifest files.
helm template cert-manager ./cert-manager-v0.12.0.tgz --output-dir . \
   --namespace cert-manager \
   --set image.repository=10.133.0.5:5000/quay.io/jetstack/cert-manager-controller \
   --set webhook.image.repository=10.133.0.5:5000/quay.io/jetstack/cert-manager-webhook \
   --set cainjector.image.repository=10.133.0.5:5000/quay.io/jetstack/cert-manager-cainjector

curl -L -o cert-manager/cert-manager-crd.yaml https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml
```

Now let's fetch the latest Rancher chart. This will pull down the chart and save it in the current directory as a .tgz file.
```
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
helm fetch rancher-latest/rancher
```

Render Rancher template. This will create a directory named *rancher* containing all Kubernetes manifest files.  
```
helm template rancher ./rancher-2.4.5.tgz --output-dir . \
 --namespace cattle-system \
 --set hostname=air-gap.eternaltechjourney.com \
 --set certmanager.version=v0.12.0 \
 --set rancherImage=10.133.0.5:5000/rancher/rancher \
 --set systemDefaultRegistry=10.133.0.5:5000 \
 --set useBundledSystemChart=true
```

# In the air-gapped environment
## Images
After moving the created **rancher-images.tar.gz** archive from your host with internet access to the air-gapped environment, run the **rancher-load-images.sh** script to push all the necessary images to the private registry. (this can also be done from the initial node, if it has access to the internal registry as well) 
```
./rancher-load-images.sh --image-list ./rancher-images.txt --registry 10.133.0.5:5000
```

### Install Rancher 
Either from the machine that has internet or after moving the *rancher* and *cert-manager* directories to the air-gapped environment, run the following in the directory containing the manifests.  
  
Go to your rendered *cert-manager* directory and run the following: 
```
kubectl create namespace cert-manager               # Create namespace
kubectl apply -f cert-manager/cert-manager-crd.yaml # Create the cert-manager CustomResourceDefinitions (CRDs)
kubectl apply -R -f ./cert-manager                  # Launch cert-manager
```
Now go to your rendered *rancher* directory and install Rancher: 
```
kubectl create namespace cattle-system              # Create namespace
kubectl -n cattle-system apply -R -f ./rancher      # Launch Rancher
```

# Downstream clusters
- Create custom cluster - Add private repo from the UI.
- Create RKE cluster, with private repo and import into Rancher. 

# Registry
Add private registry, to docker insecure registries if needed. E.g.: 
```
vi /etc/docker/daemon.json

{
  "insecure-registries": [
    "10.x.x.x:5000"
  ]
}
```

# Create RKE cluster with private registry. E.g:
```
nodes:
- address: 10.133.0.4
  port: "22"
  role:
  - controlplane
  - worker
  - etcd
  hostname_override: ""
  user: root
  docker_socket: /var/run/docker.sock
  ssh_key: ""
  ssh_key_path: /home/id_rsa
private_registries:
- url: 10.133.0.5:5000 # private registry url
  user: test
  password: test
  is_default: true
```