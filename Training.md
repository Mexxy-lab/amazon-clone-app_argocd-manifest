# Steps to Configure an Ingress on a Kubernetes cluster

Phase 1 — Install Traefik

- We'll install Traefik using Helm and expose it through a NodePort on the worker.
- That lets us access the application using the worker's IP without running kubectl port-forward.
- Our first access URL will be: <http://"your machine ip":30080>
- Later, we'll add a hostname and a LoadBalancer IP to make the setup cleaner.

Step 1 — Add the Traefik Helm repository

```bash
helm version                            | Would give helm version installed
helm repo list                          | Would show your repo list
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Step 2 — Inspect the chart's NodePort settings

```bash
helm show values traefik/traefik | grep -A12 -E '^service:|^ports:'
helm show values traefik/traefik | grep -A35 '^ports:$'
helm show values traefik/traefik | grep -A45 '^service:'
helm show values traefik/traefik | grep -A20 '^deployment:'
helm show values traefik/traefik | grep -A20 '^resources:'
helm show values traefik/traefik | grep -A12 '^nodeSelector:'
```

1. Create a Traefik helm values file

- This configuration sets Traefik’s HTTP entry point to port 80 externally, mapped through NodePort 30080, and schedules it on worker-node1.

```bash
mkdir -p helm/traefik
nano helm/traefik/values.yaml

Then validate that Helm can render the chart with these values:

helm template traefik traefik/traefik \
  --namespace traefik \
  --values helm/traefik/values.yaml
```

Step 3 — Install Traefik

- First create the Traefik namespace, then install the Traefik using the values file

```bash
kubectl create namespace traefik

helm install traefik traefik/traefik \
  --namespace traefik \
  --values helm/traefik/values.yaml
```

- Test the pods also using the command below:

```bash
curl -I http://192.168.1.121:30080

- A 404 Not Found response is expected at this stage because Traefik is running but has no Ingress route configured yet.
- It means Traefik is responding, but we haven't configured an application route yet.

vagrant@master-node:~$ curl -I http://192.168.1.121:30080
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Wed, 04 Nov 2026 14:24:57 GMT
Content-Length: 19

Sample error from applying / installing helm Traefik

pumej@Pumejpc:~/Projects/amazon-clone-app_argocd-manifest$ helm install traefik traefik/traefik \
  --namespace traefik \
  --values helm/traefik/values.yaml
Error: INSTALLATION FAILED: chart requires kubeVersion: >=1.30.0-0 which is incompatible with Kubernetes v1.28.15                       | chart requires kubeVersion: >=1.30.0-0 which     
is incompatible with Kubernetes v1.28.15

- I recommend using Traefik Helm chart v41.0.0 as our starting point, subject to checking its declared Kubernetes compatibility before installation. This would resolve that error.

helm show chart traefik/traefik --version 41.7.1                                    | Verify the installed version

- Reinstall Traefik using the right version

helm install traefik traefik/traefik \
  --version 41.7.1 \
  --namespace traefik \
  --create-namespace \
  --values helm/traefik/values.yaml
```

Step 4 - Verify the deployment or Install

```bash
helm list -n traefik
kubectl get pods -n traefik -o wide
kubectl get svc -n traefik
kubectl get ingressclass
kubectl get nodes
```

Step 5- 1. Create the correct Ingress manifest to provide route to the amazon-clone application

We'll use:

- Namespace: amazon-clone
- Ingress name: amazon-clone-ingress
- Ingress class: traefik
- Host: amazon.local
- Backend: amazon-clone-service
- Backend port: 80

- I recommend amazon.local rather than localhost, because it gives us a dedicated hostname for the application. Add ingress.yaml
- Why amazon.test? It's a reserved testing domain, making it a better choice for local development than .local, which can conflict with multicast DNS.

```bash
nano kubernetes/amazon-clone/ingress.yaml


apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: amazon-clone-ingress
  namespace: amazon-clone
spec:
  ingressClassName: traefik

  rules:
    - host: amazon.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: amazon-clone-service
                port:
                  number: 80

- Validate the file with a dry run test. 
- Also verify that the Service for your application has healthy backend endpoints:

kubectl apply --dry-run=client \
  -f kubernetes/amazon-clone/ingress.yaml

kubectl get endpoints amazon-clone-service \
  -n amazon-clone
```

6 Test routing before configuring Windows

- You should receive an HTTP response from your Amazon Clone application rather than Traefik's default 404 page.

```bash
curl -i \
  -H "Host: amazon.test" \
  http://192.168.1.121:30080
```

7 Configure Windows/Ubuntu/WSL hostname resolution

- Once the routing test succeeds, open PowerShell as Administrator and edit:

192.168.1.121 amazon.test

- You should then be able to access: <http://amazon.test:30080>

```bash
notepad C:\Windows\System32\drivers\etc\hosts

nano etc/hosts
```

## Setting up HTTPS with cert-manager for the amazon-clone application
