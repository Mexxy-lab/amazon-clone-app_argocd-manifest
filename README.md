# Deployment of amazon clone app

```bash
# Register the Amazon Clone application with Argo CD.
kubectl apply -f application.yaml

# Check the application status.
kubectl get applications -n argocd

# Inspect synchronization and health status.
kubectl describe application amazon-clone -n argocd
# Check the application namespace.
kubectl get namespaces

# Check the Amazon Clone deployment.
kubectl get deployments -n amazon-clone

# Check the pods.
kubectl get pods -n amazon-clone -o wide

# Check the Service.
kubectl get services -n amazon-clone

# Check the Argo CD application.
kubectl get application amazon-clone -n argocd


# Forward local port 8083 to the application's Service port 80.
kubectl port-forward -n amazon-clone \
  service/amazon-clone-service 8083:80

Then access:

http://localhost:8083

Keep that terminal open while testing.
```
