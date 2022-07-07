# sh /usr/local/bin/k3s-uninstall.sh
# curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.21.9+k3s1" sh -
# cat /etc/rancher/k3s/k3s.yaml

curl -sfL https://get.k3s.io | sh -
sudo snap install helm --classic
sudo chmod -R +644 /etc/rancher/
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown -R root:root ~/.kube
chmod +644 ~/.kube/config
alias k=kubectl

kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.5.1
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook

helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
# helm repo add rancher-alpha https://releases.rancher.com/server-charts/alpha
# helm search repo --versions
helm repo update
kubectl create namespace cattle-system

# LetsEncrypt - latest version
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher-demo.sheldonloanjoe.com --set replicas=1 --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=sw.loanjoe@gmail.com

# LetsEncrypt - latest development version (RC's etc.)
# helm install rancher rancher-latest/rancher --devel --namespace cattle-system --set hostname=rancher-rc.sheldonloanjoe.com --set replicas=1 --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=sw.loanjoe@gmail.com

# Self-signed certs
# helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher-hp.sheldonloanjoe.site --set bootstrapPassword=welcome01 --set replicas=1 --set tls=external

kubectl -n cattle-system rollout status deploy/rancher
