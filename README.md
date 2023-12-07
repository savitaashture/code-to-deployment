# code-to-deployment

# Create a Kind cluster
```KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster --name argocd```

# Install ArgoCD
```
kubectl create ns argocd 
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/install.yaml -n argocd
```

# Install ImageUpdater
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/v0.12.2/manifests/install.yaml
```
# Update the Config maps of ArgoCD to add new user and role for image updater
```
kubectl patch cm/argocd-cm -n argocd --type=merge -p='{"data":{"accounts.image-updater":"apiKey"}}'
k patch cm/argocd-rbac-cm -n argocd --type=merge -p='{"data":{"policy.csv":"p, role:image-updater, applications, get, */*, allow\np, role:image-updater, applications, update, */*, allow\ng, image-updater, role:image-updater"}}'
```
```
kubectl port-forward -n argocd svc/argocd-server 8443:443 > /dev/null 2>&1 &
ADMIN_PASSWD=$(kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' -n argocd | base64 -d)
argocd login --username admin --password ${ADMIN_PASSWD} localhost:8443 --insecure
IMAGE_UPDATER_TOKEN=$(argocd account generate-token --account image-updater --id image-updater)
kubectl create secret generic argocd-image-updater-secret \
  --from-literal argocd.token=${IMAGE_UPDATER_TOKEN} --dry-run=client -o yaml | kubectl -n argocd apply -f - 
```
```
kubectl patch cm argocd-image-updater-config -n argocd --type=merge -p '{"data":{"log.level":"debug"}}'
```

# Install the argo application

```
kubectl create -f application_integ.yaml -n argocd
```

# Install ImageUpdater CLI for debugging
```
wget https://github.com/argoproj-labs/argocd-image-updater/releases/download/v0.12.2/argocd-image-updater-$(go env GOOS)_$(go env GOARCH) -O ~/bin/argocd-image-updater
sudo chmod +x ~/bin/argocd-image-updater
```

# Install KyVerno
```
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml -n kyverno
```

# Install KyVerno Image Signing policies

```
NA
```

# Install Sigstore Policy Controller

```
kubectl create namespace cosign-system
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update
helm install policy-controller -n cosign-system sigstore/policy-controller --devel
```

# Wait for the policy controller to be available
```
kubectl -n cosign-system wait --for=condition=Available deployment/policy-controller-webhook && \
kubectl -n cosign-system wait --for=condition=Available deployment/policy-controller-webhook
```

# Enable guestbook namespace in image validation and policy enforcement
```
kubectl create ns guestbook
kubectl label namespace guestbook policy.sigstore.dev/include=true
```

```
kubectl create ns kubeday-integ
kubectl label namespace kubeday-integ policy.sigstore.dev/include=true
```