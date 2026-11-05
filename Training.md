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

6 Test routing before configuring Windows/WSL

- Deploy the ingress using git push. Argocd would deploy it using gitops.
- You should receive an HTTP response from your Amazon Clone application rather than Traefik's default 404 page.

```bash
kubectl get ingress -n amazon-clone

kubectl describe ingress amazon-clone-ingress \
  -n amazon-clone

kubectl get applications -n argocd

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

## Used to install Argocd via helm and set up as Gitops

```bash
helm install argocd-platform argo/argo-cd \
  --version 7.8.28 \
  --namespace argocd \
  --create-namespace \
  --values kubernetes/argocd-platform/values.yaml \
  --wait \
  --timeout 10m

## Command to list out if sync services
kubectl get application argocd-platform -n argocd -o jsonpath='{range .status.resources[*]}{.kind}{" / "}{.name}{" | Sync: "}{.status}{" | Health: "}{.health.status}{"\n"}{end}'
```

## Setting up HTTPS with cert-manager for the amazon-clone application

1 Create a vertificate.yml file and issuer.yml file to be deployed in name space where you deployed your application

Certificate.yml

```bash
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: amazon-app-cert
  namespace: amazon-clone
spec:
  commonName: amazon.test
  dnsNames:
    - amazon.test
  secretName: amazon-app-tls
  issuerRef:
    name: selfsigned-cluster-issuer
    kind: ClusterIssuer
```

- Issuer.yml

```bash
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
```

2 Update your Ingress file to include  TLS section

- Your existing Ingress will retain its routing configuration, with a TLS section added:

```bash
  tls:
    - hosts:
        - amazon.test
      secretName: amazon-app-tls
```

- This configures Traefik to serve HTTPS for amazon.test using the certificate stored in amazon-app-tls.

3 Install cert-manager

- This installs the cert-manager components and their CRDs. The official installation documentation recommends this approach for getting started.
- Would be installed in namespace/cert-manager.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.3/cert-manager.yaml

kubectl apply -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.16.5/cert-manager.yaml                                        | Compatible version

kubectl delete -f \
  https://github.com/cert-manager/cert-manager/releases/download/v1.21.3/cert-manager.yaml \
  --ignore-not-found                                                                                                | Used to delete the installed cert manager

kubectl wait \
  --for=condition=Available \
  deployment \
  --all \
  -n cert-manager \
  --timeout=180s
```

4 Validate the installation and Confirm the API resources exist

```bash
kubectl get pods -n cert-manager
kubectl get crd certificates.cert-manager.io
kubectl get crd clusterissuers.cert-manager.io

kubectl wait \
  --for=condition=Available \
  deployment \
  --all \
  -n cert-manager \
  --timeout=180s
```

- Update helm/traefik/values.yaml
- Traefik currently exposes HTTP only. Even if cert-manager successfully generates both certificates, HTTPS traffic cannot reach Traefik.

```bash
ports:
  web:
    port: 8000
    exposedPort: 80
    expose:
      default: true
    nodePort: 30080
    protocol: TCP

  websecure:
    port: 8443
    exposedPort: 443
    expose:
      default: true
    nodePort: 30443
    protocol: TCP

  traefik:
    expose:
      default: false


helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --version 41.7.1 \
  --values helm/traefik/values.yaml
```

This gives us:

Protocol      NodePort          Purpose
HTTP          30080             Existing HTTP access
HTTPS         30443             New HTTPS access

- NodePort 30443 means external clients normally access HTTPS using :30443. To achieve your preferred URLs without a port number, we'll also need to establish how port 443 reaches the Kubernetes node.

## Final CA Architecture

Namespace: cert-manager

1. local-ca-bootstrap — self-signed bootstrap ClusterIssuer.
2. local-root-ca — Certificate that creates the CA.
3. local-ca-key-pair — Secret containing the CA certificate and private key.
4. local-ca-issuer — ClusterIssuer that signs application certificates.

Namespace: amazon-clone

- Certificate: amazon-app-cert
- TLS Secret: amazon-app-tls
- DNS: amazon.test

Namespace: argocd

- Certificate: argocd-server-certificate
- TLS Secret: argocd-server-tls
- DNS: argocd.test

- The root CA will be generated by cert-manager. Its private key will never be committed to Git.

## Configuring application to use only URLs

- We could expose Traefik directly using a Kubernetes LoadBalancer service with MetalLB.
- We're building a Kubernetes GitOps lab with Argo CD, Helm, cert-manager, and Traefik. Having MetalLB assign an IP to Traefik would make the architecture cleaner and more Kubernetes-native.

Advantages:

- Entire application ingress architecture stays inside Kubernetes.
- No Windows port forwarding.
- More representative of production Kubernetes environments.
- Additional applications automatically use the same ingress infrastructure.
- Can eventually expose services through standard ports 80 and 443.

Disadvantages:

- Requires MetalLB installation and configuration.
- Requires an available IP range on your home LAN.
- Requires ensuring the cluster networking supports Layer 2 announcements.

- We'll use MetalLB's native installation manifest. We don't need FRR or BGP because we're using Layer 2 mode. The official documentation supports native mode for this use case.
MetalLB+1

1. Install the pinned release

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml

- This creates the metallb-system namespace, CRDs, controller, speaker DaemonSet, service accounts, and RBAC resources.


kubectl rollout status deployment/controller \
  -n metallb-system --timeout=180s

kubectl rollout status daemonset/speaker \
  -n metallb-system --timeout=180s
```

3.Inspect the installation

```bash
kubectl get pods -n metallb-system -o wide
kubectl get daemonsets,deployments -n metallb-system
kubectl get crds | grep metallb
```

Step 6 — Configure the MetalLB address pool

- Create kubernetes/metallb/ipaddresspool.yaml:

```bash
mkdir -p kubernetes/metallb

cat > kubernetes/metallb/ipaddresspool.yaml <<'EOF'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: host-only-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.3.0.100-10.3.0.110
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: host-only-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - host-only-pool
  interfaces:
    - enp0s9
EOF
```

- Why these settings?

Setting                           Purpose

host-only-pool                    Defines the available LoadBalancer IP addresses
10.3.0.100–110                    Our reserved address range
autoAssign: true                  Allows MetalLB to allocate addresses automatically
host-only-advertisement           Enables Layer 2 announcements
enp0s9                            Restricts announcements to your host-only interface

- This means MetalLB won't advertise service IPs through your NAT or home Wi-Fi interfaces.

7.Validate, apply and verify the ipaddresspool.yaml file

```bash
kubectl apply --dry-run=server -f kubernetes/metallb/ipaddresspool.yaml                               | Would Validate
kubectl apply -f kubernetes/metallb/ipaddresspool.yaml                                                | Would apply

kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
kubectl describe ipaddresspool host-only-pool -n metallb-system
kubectl get pods -n metallb-system -o wide
kubectl logs -n metallb-system daemonset/speaker --tail=50                                            | Would show logs 
kubectl get events -n metallb-system --sort-by=.lastTimestamp                                         | Would sort logs by timestamp
kubectl describe l2advertisement host-only-advertisement -n metallb-system                            | Used for verification
```

## Update the traefik ingress with the loadbalancer service porttype

```bash
service:
  enabled: true
  single: true
  annotations:
    metallb.io/loadBalancerIPs: 10.3.0.100
  spec:
    type: LoadBalancer


Upgrade the helm installation for the traefik ingress using below command:

helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --version 41.7.1 \
  --values helm/traefik/values.yaml \
  --dry-run                                                                                                           | Used for dry run

helm upgrade traefik traefik/traefik \
  --namespace traefik \
  --version 41.7.1 \
  --values helm/traefik/values.yaml                                                                                   | Used for actual update/upgrade
```

## Testing the loadbalancer addr from windows

```bash
Test-NetConnection 10.3.0.100 -Port 80
Test-NetConnection 10.3.0.100 -Port 443
curl.exe -I --ssl-no-revoke --resolve amazon.test:443:10.3.0.100 https://amazon.test
curl.exe -I --ssl-no-revoke --resolve argocd.test:443:10.3.0.100 https://argocd.test

- The --resolve option temporarily maps each hostname to the MetalLB IP for that request. It doesn't modify your Windows hosts file.
```

## Next: Configure Windows hostname resolution

```bash
notepad C:\Windows\System32\drivers\etc\hosts

Add: 

10.3.0.100 amazon.test
10.3.0.100 argocd.test
```

## What would make them publicly accessible?

That would require additional infrastructure, such as:

- A public IP address and internet-facing router configuration.
- Port forwarding or a cloud load balancer.
- Public DNS records.
- Properly trusted public TLS certificates.
- Firewall and security controls.
