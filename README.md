# Motivation

This is a simple demonstration of running various core infrastructure services 
using kubernetes. The focus is here on how to set up and configure applications
as an end user (and not cluster administration). I use k0s as a single-node
for my setup but the manifests should work for other variants of kubernetes
too. In my experience, this approach of setting up a single node ad-hoc
cluster could prove useful in specific conditions such as
application (unit/integration/regression) testing. Also, as someone who
believes in a well-run infrastructure, I think much of the core infrastructure
that typically is hosted on a VM and run by corporate IT teams can be
modernised and shifted to run within kubernetes. Hence, I have chosen that sort
of core applications for my test bed.

In particular, the demo setup shows:

- how web and network based services can be configured
- how ingress is managed via nginx controller (or OpenELB)
- how secrets are managed using sops
- how ZeroSSL certificate manager can be used for TLS

## Pre-requisites

Before we can deploy our services, we need to prepare our environment.

## k0s

To configure k0s as a single node backend, see instructions here: https://docs.k0sproject.io/v0.9.0/k0s-single-node/.

To ensure k0s works correctly, run `sudo k0s status`

Create a namespace `core` for our services:

```
kubectl create ns core
kubectl config set-context --current --namespace=core
```

### Ingress Controller

To expose a service, we are going to compare the capabilities of nginx and 
OpenELB as a load balancer.

#### Nginx 

```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.3/deploy/static/provider/baremetal/deploy.yaml
kb apply -f deploy.yaml
```

#### OpenELB

We will be using load balancer of [VIP](https://openelb.io/docs/getting-started/usage/use-openelb-in-vip-mode/) type.
For the pool of IP Address, ensure you use IP address range from your local network.

### Certificate Manager

Install cert-manager:
```
wget https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl apply -f cert-manager.yaml
kubectl get pods -n cert-manager
```

#### ZeroSSL

We will use [ZeroSSL](https://cert-manager.io/docs/tutorials/zerossl/zerossl/) as our certificate provider.
Create an account in ZeroSSL (https://app.zerossl.com). From the developer
section of the dashboard, generate EAB credentials for ACME clients. Keep note
of the key id and hmac key for creating the secret (see `manifests/core/certs/zerossl-cluster-issuer.yaml`).

To issue certificates, I use Route53 dns for validation. This requires an AWS
key which allows route53 read/write privilege (see `manifests/core/secrets/route53-clusterissuer.yaml`).

### PGP and Sops

On linux, generate a pgp key using `gpg --generate-key`.

Install sops tool from https://github.com/getsops/sops/releases.
Copy the tool to /usr/local/bin/sops and ensure executable permission is set.

To encrypt a kubernetes secret:
```
sops --encrypt --pgp <pgp_key_id> --encrypted-regex '^(data|stringData)$' --in-place <secrets_yaml>
```

To decrypt a kubernetes secret:
```
sops --decrypt --pgp <pgp_key_id> --in-place <secrets_yaml>
```

Before applying kustomize, the secrets needs to be decrypted using sops.
