## **Step-by-Step Guide: ArgoCD on GKE with Ingress and ManagedCertificate**

### **1. Deploy ArgoCD**

- **Install ArgoCD on your cluster:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- **Wait for all pods to be ready:**

```bash
kubectl get pods -n argocd --watch
```


---

### **2. Expose ArgoCD via Ingress**

#### **A. Prepare Ingress YAML**

Create a file named `argocd-ingress.yaml` with the following content (update the `host` to your new domain):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    networking.gke.io/managed-certificates: argocd-cert
  labels:
    app: argocd
spec:
  rules:
  - host: your-new-domain.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```


#### **B. Apply Ingress**

```bash
kubectl apply -f argocd-ingress.yaml -n argocd
```


---

### **3. Create ManagedCertificate**

#### **A. Prepare ManagedCertificate YAML**

Create a file named `argocd-cert.yaml`:

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: argocd-cert
  namespace: argocd
spec:
  domains:
    - your-new-domain.example.com
```


#### **B. Apply ManagedCertificate**

```bash
kubectl apply -f argocd-cert.yaml -n argocd
```


---

### **4. Configure Health Checks for GKE Ingress**

#### **A. Prepare BackendConfig YAML**

Create a file named `argocd-healthcheck-config.yaml`:

```yaml
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: argocd-healthcheck-config
  namespace: argocd
spec:
  healthCheck:
    type: HTTP
    requestPath: /healthz
    port: 8080
```


#### **B. Apply BackendConfig**

```bash
kubectl apply -f argocd-healthcheck-config.yaml -n argocd
```


#### **C. Update ArgoCD Server Service**

Edit the `argocd-server` service to use the BackendConfig:

```bash
kubectl edit svc argocd-server -n argocd
```

Add or update the annotation:

```yaml
annotations:
  cloud.google.com/backend-config: '{"default": "argocd-healthcheck-config"}'
```


---

### **5. (Optional) Set ArgoCD to Insecure Mode**

If you experience redirect loops or health check failures due to HTTPS redirects:

#### **A. Edit ConfigMap**

```bash
kubectl edit configmap argocd-cmd-params-cm -n argocd
```

Add:

```yaml
data:
  server.insecure: "true"
```


#### **B. Restart ArgoCD Server**

```bash
kubectl rollout restart deployment argocd-server -n argocd
```


---

### **6. Monitor and Troubleshoot**

#### **A. Check Ingress Status**

```bash
kubectl describe ingress argocd-ingress -n argocd
```

- **Look for the `Address` field to get the external IP.**
- **Check for errors in the `Events` section.**


#### **B. Check ManagedCertificate Status**

```bash
kubectl describe managedcertificate argocd-cert -n argocd
```

- **Ensure `Certificate Status` and `Domain Status` are `Active`.**


#### **C. Check Backend Health**

```bash
kubectl describe ingress argocd-ingress -n argocd | grep backends
```

- **Ensure your backend is `HEALTHY`.**
- **If not, check pod logs, service endpoints, and health check configuration.**


#### **D. Check Firewall Rules**

- **Ensure Google health check IP ranges are allowed:**
    - `35.191.0.0/16`
    - `130.211.0.0/22`

---

### **7. Update DNS**

- **Once the Ingress has an external IP, update your DNS record for the domain to point to that IP.**
- **Verify DNS propagation:**

```bash
nslookup your-new-domain.example.com
```

or

```bash
dig your-new-domain.example.com
```


---

### **8. Test Access**

- **Open your browser and navigate to `https://your-new-domain.example.com`.**
- **If you set ArgoCD to insecure mode, you may need to use `http://` or adjust your Ingress for HTTPS redirection.**

---

## **Troubleshooting Checklist**

- **Ingress not getting an IP:**
    - Wait a few minutes for GKE to provision the load balancer.
    - Check for errors in the Ingress events.
- **Backend is UNHEALTHY:**
    - Ensure the health check path (`/healthz`) returns `200 OK`.
    - Ensure the health check port matches your pod’s target port.
    - Check firewall rules for health check traffic.
    - Consider setting ArgoCD to insecure mode if redirect loops occur.
- **ManagedCertificate not active:**
    - Check for errors in the ManagedCertificate events.
    - Ensure your domain is correctly spelled and matches the Ingress host.
- **DNS not resolving:**
    - Double-check your DNS record and wait for propagation.

---

## **Summary Table**

| Step | Command/Action | Notes |
| :-- | :-- | :-- |
| Deploy ArgoCD | `kubectl apply -n argocd -f ...` | Use official manifests |
| Create Ingress | `kubectl apply -f argocd-ingress.yaml -n argocd` | Update host and labels |
| Create ManagedCert | `kubectl apply -f argocd-cert.yaml -n argocd` | Update domain |
| Create BackendConfig | `kubectl apply -f argocd-healthcheck-config.yaml -n argocd` | Ensure port and path are correct |
| Update Service | `kubectl edit svc argocd-server -n argocd` | Add BackendConfig annotation |
| Set Insecure (opt.) | `kubectl edit configmap argocd-cmd-params-cm -n argocd` | Add `server.insecure: "true"` |
| Restart ArgoCD (opt.) | `kubectl rollout restart deployment argocd-server -n argocd` | Apply insecure config |
| Check Ingress | `kubectl describe ingress argocd-ingress -n argocd` | Look for IP and errors |
| Check ManagedCert | `kubectl describe managedcertificate argocd-cert -n argocd` | Look for Active status |
| Check Backend Health | `kubectl describe ingress argocd-ingress -n argocd \| grep backends` | Ensure HEALTHY |
| Update DNS | Update DNS record to Ingress IP | Use nslookup/dig to verify |
| Test Access | Open browser to `https://your-new-domain.example.com` | Use http if insecure mode enabled |


---

## **References**

- **ArgoCD Official Docs:** [Getting Started][^84_1], [Installation][^84_6]
- **GKE and ArgoCD Guides:** [GitLab’s GKE + ArgoCD][^84_2], [GKE Multi-Cluster with ArgoCD][^84_4]
- **Troubleshooting:** See events and logs for errors, check health check and firewall settings.

---

[^84_1]: https://argo-cd.readthedocs.io/en/stable/getting_started/

[^84_2]: https://about.gitlab.com/blog/2024/01/31/quick-setup-of-a-gke-cluster-with-argocd-pre-installed-using-terraform/

[^84_3]: https://www.youtube.com/watch?v=W6gjekyu5xg

[^84_4]: https://cloud.google.com/blog/products/containers-kubernetes/empower-your-teams-with-self-service-kubernetes-using-gke-fleets-and-argo-cd

[^84_5]: https://merlin.microworka.com/setting-up-argocd-on-private-google-kubernetes-engine-cluster-for-gitops-deployment

[^84_6]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/

[^84_7]: https://www.digitalocean.com/community/tutorials/how-to-deploy-to-kubernetes-using-argo-cd-and-gitops

[^84_8]: https://www.solo.io/blog/istio-ambient-argo-cd-gke-15-minutes

# Argo CD Example Apps

This repository contains example applications for demoing Argo CD functionality. Feel free
to register this repository to your ArgoCD instance, or fork this repo and push your own commits
to explore Argo CD and GitOps!

| Status                                                                    | Application                                        | Description                                                                                                              |
| ------------------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| [![App Status][badge_sync_example_apps]][app_sync_example_apps]           | [apps](apps/)                                      | An app composed of other apps synchronized in [cd.apps.argoproj.io][app_sync_example_apps]                               |
| [![App Status][badge_blue_green]][app_blue_green]                         | [blue-green](blue-green/)                          | Demonstrates how to implement blue-green deployment using [Argo Rollouts](https://github.com/argoproj/argo-rollouts)     |
| [![App Status][badge_guestbook]][app_guestbook]                           | [guestbook](guestbook/)                            | A hello word guestbook app as plain YAML                                                                                 |
| [![App Status][badge_helm_dependency]][app_helm_dependency]               | [helm-dependency](helm-dependency/)                | Demonstrates how to customize an OTS (off-the-shelf) helm chart from an upstream repo                                    |
| [![App Status][badge_helm_guestbook]][app_helm_guestbook]                 | [helm-guestbook](helm-guestbook/)                  | The guestbook app as a Helm chart                                                                                        |
| [![App Status][badge_helm_hooks]][app_helm_hooks]                         | [helm-hooks](helm-hooks/)                          | An application with native Helm hooks                                                                                    |
| [![App Status][badge_jsonnet_guestbook]][app_jsonnet_guestbook]           | [jsonnet-guestbook](jsonnet-guestbook/)            | The guestbook app as a raw jsonnet                                                                                       |
| [![App Status][badge_jsonnet_guestbook_tla]][app_jsonnet_guestbook_tla]   | [jsonnet-guestbook-tla](jsonnet-guestbook-tla/)    | The guestbook app as a raw jsonnet with support for top level arguments                                                  |
| [![App Status][badge_kustomize_guestbook]][app_kustomize_guestbook]       | [kustomize-guestbook](kustomize-guestbook/)        | The guestbook app as a Kustomize app                                                                                     |
| [![App Status][badge_plugin_kasane]][app_plugin_kasane]                   | [plugins/kasane](plugins/kasane)                   | Apps which demonstrate config management plugins usage with [kasane](plugins/kasane/README.md)                           |
| [![App Status][badge_plugin_kustomized_helm]][app_plugin_kustomized_helm] | [plugins/kustomized-helm](plugins/kustomized-helm) | Apps which demonstrate config management plugins usage with a [kustomized helm chart](plugins/kustomized-helm/README.md) |
| [![App Status][badge_pre_post_sync]][app_pre_post_sync]                   | [pre-post-sync](pre-post-sync/)                    | Demonstrates Argo CD PreSync and PostSync hooks                                                                          |
| [![App Status][badge_sock_shop]][app_sock_shop]                           | [sock-shop](sock-shop/)                            | A microservices demo app (https://microservices-demo.github.io)                                                          |
| [![App Status][badge_sync_waves]][app_sync_waves]                         | [sync-waves](sync-waves/)                          | Demonstrates Argo CD sync waves with hooks                                                                               |

[app_sync_example_apps]: https://cd.apps.argoproj.io/applications/sync-example-apps
[badge_sync_example_apps]: https://cd.apps.argoproj.io/api/badge?revision=true&name=sync-example-apps
[app_blue_green]: https://cd.apps.argoproj.io/applications/example.blue-green
[badge_blue_green]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.blue-green
[app_guestbook]: https://cd.apps.argoproj.io/applications/example.guestbook
[badge_guestbook]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.guestbook
[app_helm_dependency]: https://cd.apps.argoproj.io/applications/example.helm-dependency
[badge_helm_dependency]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.helm-dependency
[app_helm_guestbook]: https://cd.apps.argoproj.io/applications/example.helm-guestbook
[badge_helm_guestbook]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.helm-guestbook
[app_helm_hooks]: https://cd.apps.argoproj.io/applications/example.helm-hooks
[badge_helm_hooks]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.helm-hooks
[app_jsonnet_guestbook]: https://cd.apps.argoproj.io/applications/example.jsonnet-guestbook
[badge_jsonnet_guestbook]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.jsonnet-guestbook
[app_jsonnet_guestbook_tla]: https://cd.apps.argoproj.io/applications/example.jsonnet-guestbook-tla
[badge_jsonnet_guestbook_tla]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.jsonnet-guestbook-tla
[app_kustomize_guestbook]: https://cd.apps.argoproj.io/applications/example.kustomize-guestbook
[badge_kustomize_guestbook]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.kustomize-guestbook
[app_plugin_kasane]: https://cd.apps.argoproj.io/applications/example.plugin-kasane
[badge_plugin_kasane]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.plugin-kasane
[app_plugin_kustomized_helm]: https://cd.apps.argoproj.io/applications/example.plugin-kustomized-helm
[badge_plugin_kustomized_helm]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.plugin-kustomized-helm
[app_pre_post_sync]: https://cd.apps.argoproj.io/applications/example.pre-post-sync
[badge_pre_post_sync]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.pre-post-sync
[app_sock_shop]: https://cd.apps.argoproj.io/applications/example.sock-shop
[badge_sock_shop]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.sock-shop
[app_sync_waves]: https://cd.apps.argoproj.io/applications/example.sync-waves
[badge_sync_waves]: https://cd.apps.argoproj.io/api/badge?revision=true&name=example.sync-waves
