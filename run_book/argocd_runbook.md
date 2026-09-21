kubectl apply \
  -f argocd/projects/histdata.yaml
  
  The AppProject restricts Argo CD to:

The HistData deployment repository
The histdata namespace
The in-cluster Kubernetes API
Required namespaced resources
Namespace and ClusterSecretStore cluster resources


6. Create the application

kubectl apply \
  -f argocd/applications/histdata-dev.yaml
  
 7. Monitor deployment 
  
 kubectl get application histdata-dev \
  --namespace argocd \
  --watch
  
  8. Access the Argo CD UI
  
  kubectl port-forward \
  service/argocd-server \
  --namespace argocd \
  8081:443
  
  https://localhost:8081
  
 User name admin
 
Get the temporary password locally:

kubectl get secret argocd-initial-admin-secret \
  --namespace argocd \
  --output jsonpath='{.data.password}' |
base64 --decode

echo
