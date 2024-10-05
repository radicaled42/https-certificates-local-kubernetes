# https-certificates-local-kubernetes

So I came after the idea to simulate a CA in kubernetes by a work challenge.
The challenge emphazised to have a valid certificate.

>  The client has to call the server on its three alternative names using the DNS resolution with private SSL certification (you can use openssl local auth, let's encrypt with Certbot, or whatever you prefer); the certificate has to be trusted (not skipping validation).
The idea is to be able to simulate similar SSL certificate as Let's Encrypt with Pebble

One could think that using a simple CA generated with openssl would be enough, but I thought that wouldn't work on kubernetes.

I also explored the option of getting a free domain (https://www.getfreedomain.name/). But it I had many problems to work with the webhooks and cert-manager.

While looking for a solution I found this this guide, [Simulate Let’s Encrypt certificate issuing in local Kubernetes](https://blog.manabie.io/2021/11/simulate-https-certificates-acme-k8s/).
This document introduce me to Pebble. It's a miniature version of [Boulder](https://github.com/letsencrypt/boulder), Pebble is a small ACME test server not suited for use as a production CA.

So what is the idea:
1. Install a kind cluster (k3d, k3s, minukube would work)
2. Install Nginx ingress controller
3. Install Pebble
4. Modify CoreDNS config file
5. Install your sample application (Ingress included)
6. Pull your CA certificates
7. Test

Easy peasy

## Configuration

### Install kind cluster

Follow the [kind installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

**For linux**:
```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Now that we have kind installed we need to create a kind-config file.
The kind config file hold all the configuration for the a file. You could do the same thing through command line, but it would be difficult to do it.
You will need to expose port 80 and 443 to be able to access the application and you will need to add `node-labels: "ingress-ready=true"` to install nginx ingress controller.

For example:
```yaml
kind: Cluster
name: https-cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

Run the following command to create the cluster:
```bash
kind create cluster --config=kind-config
```

