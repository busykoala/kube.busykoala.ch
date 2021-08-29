# Captain Cluster

Experimental setup to learn all about k8s.

## Initial Setup

This setup installs k3s without `Traefik` but adds `Istio` later on.
Istio provides a Service Mesh incl. Traefik's features like load balancing, logging and routing.
The k3s `KUBECONFIG` is stored at: `/etc/rancher/k3s/k3s.yaml`.
It can simply be copied to the local machine (change the server section to the server IP).

**k3s**

```bash
# install k3s without traefik
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

**Istio**

Minimal server requirements! https://stackoverflow.com/a/64470097

```bash
# install istio https://istio.io/latest/docs/setup/getting-started/
curl -L https://istio.io/downloadIstio | sh -

# export in bashrc

# symlink or copy kubeconfig
ln -s /etc/rancher/k3s/k3s.yaml /home/captain/.kube/config  # chown to correct user...
# apply profile (e.g. default -> https://istio.io/latest/docs/setup/additional-setup/config-profiles/)
istioctl install --set profile=default -y

# (uninstall: https://istio.io/latest/docs/setup/install/istioctl/#uninstall-istio)
```

**cert-manager**

```bash
# install the cert manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/latest/download/cert-manager.yaml
```

## Setup Wildcard Gateway

Wildcard certificates can only be issued using a dns01 solver (http01 does not work!).
In this example the nameserver is cloudflare.

From cloudflare a token is required for the challenge.

Checkout [this cert-manager docs for cloudflare](https://cert-manager.io/docs/configuration/acme/dns01/cloudflare/)
for the cloudflare token permissions.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: 'cloudflare-token'
```

After the secret is created a clusterissuer (`kube.clusterissuer.yml`) can be created.
The clusterissuer can then be used to create a certificate (`kube.certificate.yml`).
Once the certificate is applied it will create a `CertificateRequest` which creates an `Order`
and then the order creates `Challenges` for each dnsName of the `Certificate`.  
Check the descriptions of these resources if something goes wrong.

Once there is a `Cerfificate`, it can be used by a Gateway (`kube.gateway.yml`).

Until now everything was either deployed to `istio-system` or `cert-manager` namespace.
The following recources are gonna be in a different one we create (`captain.namespace.yml`).

The gateway can be used by `Virtualservices` (captain.virtualservices.yml`).
The `VirtualService` includes rules which requests are routed to a service (e.g. route matching).

Furthermore there is a helloworld service and the deployments available in the repo.
