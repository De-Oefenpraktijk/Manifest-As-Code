az aks create -n ofp -g RGR-DeOefenpraktijk --node-count 3 --node-vm-size Standard_B2s --node-osdisk-size 30 --enable-addons monitoring --generate-ssh-keys --attach-acr OefenpraktijkRegistry --network-plugin azure --enable-managed-identity

NAMESPACE=ingress-basic

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace $NAMESPACE \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

