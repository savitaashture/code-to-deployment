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
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```
```
kubectl patch cm/argocd-cm --type=merge -p='{"data":{"accounts.image-updater":"apiKey"}}'
k patch cm/argocd-rbac-cm --type=merge -p='{"data":{"policy.csv":"p, role:image-updater, applications, get, */*, allow\np, role:image-updater, applications, update, */*, allow\ng, image-updater, role:image-updater"}}'
```
```
kubectl port-forward -n argocd svc/argocd-server 8443:443 > /dev/null 2>&1 &
ADMIN_PASSWD=$(kubectl get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' -n argocd | base64 -d)
argocd login --username admin --password ${ADMIN_PASSWD} localhost:8443
IMAGE_UPDATER_TOKEN=$(argocd account generate-token --account image-updater --id image-updater)
kubectl create secret generic argocd-image-updater-secret \
  --from-literal argocd.token=${IMAGE_UPDATER_TOKEN} --dry-run=client -o yaml |
  kubectl -n argocd apply -f - 
```

# Install KyVerno
```
```

# Install KyVerno Image Signing policies
