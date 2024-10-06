# https-certificates-local-kubernetes

So I came after the idea to simulate a CA in kubernetes by a work challenge.
It was a simple challenge, you have a python application that you need to Dockerize, create a certificate, create a Kubernetes Job and be able to access the application. But the challenge emphazised to have a valid certificate.

>  The client has to call the server on its three alternative names using the DNS resolution with private SSL certification (you can use openssl local auth, let's encrypt with Certbot, or whatever you prefer); the certificate has to be trusted (not skipping validation).
The idea is to be able to simulate similar SSL certificate as Let's Encrypt with Pebble

One could think that using a simple CA generated with openssl would be enough, but I thought that wouldn't work on kubernetes.

I also explored the option of getting a free domain (https://www.getfreedomain.name/). But it I had many problems to work with the webhooks and cert-manager.

While looking for a solution I found this this guide, [Simulate Let’s Encrypt certificate issuing in local Kubernetes](https://blog.manabie.io/2021/11/simulate-https-certificates-acme-k8s/).
This document introduce me to Pebble. It's a miniature version of [Boulder](https://github.com/letsencrypt/boulder), Pebble is a small ACME test server not suited for use as a production CA.

So what is the idea:
1. Install a kind cluster (k3d, k3s, minukube would work)
2. Install Nginx ingress controller
3. Modify CoreDNS
4. Install Pebble
5. Install your sample application (Ingress included)
6. Pull your CA certificates
7. Test

Easy peasy.
So lets get going.

## Configuration

### Install kind cluster

In this step we will install kind.

So basically, you need to follow the [kind installation](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)

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

You will get an output similar to this:
```
~/Documents/https-certificates-local-kubernetes main > kind create cluster --config kind-config
Creating cluster "https-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.31.0) 🖼 
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-https-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-https-cluster

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

## Install Nginx ingress controller

We are going to need an Ingress Controller, it can be Nginx or Traefik or whatever you use.
In this case, I'm going to use Nginx mainly because its an ingress controller for kind.

Install the Nginx Ingress controller for Kind from the following URL:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

This will be the output of the installation:
```
~/Documents/https-certificates-local-kubernetes main > kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

~/Documents/https-certificates-local-kubernetes main > kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-52z5n        0/1     Completed   0          58s
ingress-nginx-admission-patch-qdd78         0/1     Completed   0          58s
ingress-nginx-controller-6b8cfc8d84-vvdrf   1/1     Running     0          58s
```

## Modify CoreDNS

We can modify the CoreDNS config so the internal cluster can resolve to our internal IP. I call this a good hack because it’s similar to how the domain owner needs to point the domain to the server’s IP.

1. Get your ingress controller current IP:
```bash
kubectl get svc ingress-nginx-controller --no-headers -n ingress-nginx | awk '{print$3}'
10.96.94.238
```
   
2. Pull the current **coredns** confimap with:
```bash
kubectl get configmaps coredns -n kube-system -o yaml
```

3. Remove the **creationTimestamp**, **resourceVersion** and **uid** and you will get something like this:
```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

4. Add the host you want to test (foo.example.com) on the config file:
```yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
    foo.example.com {
           hosts {
            10.96.94.238 foo.example.com
            fallthrough
           }
           whoami
    }
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
```

5. Apply you new configuration
```bash
kubectl apply -f coredns -n kube-system
```

6. Restart the coredns pods, give this is a dev environment you could kill all the ports at the same time.
```bash
kubectl delete pod -n kube-system --wait $(kubectl get pods -n kube-system | grep coredns | awk '{print$1}')
```

## Install Pebble

Lets start with the cool part :P 
To install Pebble you first need the Cert-Manager CRDs. I will install the complete cert-manager along with the CRDs just to be on the safe side.

1. First install [**helm**](https://helm.sh/docs/intro/install/):
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

2. Install **cert-manager**:
```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
```

3. Pebble already offers a docker image to use (ghcr.io/letsencrypt/pebble:latest) so you only need to create the deployment resource.
```bash
cd ./pebble
kubectl apply -f pebble.yaml
kubectl get pod
```

By now you should already have a installed **cert-manager** and **pebble** and it would look something like this:
```bash
~/Documents/https-certificates-local-kubernetes main > kubectl get pods -n cert-manager
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-7b9875fbcc-b2mmv            1/1     Running   0          88s
cert-manager-cainjector-948d47c6-6sjbj   1/1     Running   0          88s
cert-manager-webhook-78bd84d46b-bxt29    1/1     Running   0          88s

~/Documents/https-certificates-local-kubernetes main > kubectl get pods -n default | grep pebble
pebble-585fd5f6c-m4n4j   1/1     Running   0          54s
```

The first thing you'll need to configure after you've installed cert-manager is an Issuer or a ClusterIssuer. These are resources that represent certificate authorities (CAs) able to sign certificates in response to certificate signing requests.

4. Install pebble cluster-issuer
```bash
cd ./pebble
kubectl apply -f ./pebble-issuer.yaml
```

```bash
~/Documents/https-certificates-local-kubernetes main > kubectl get clusterissuers.cert-manager.io --all-namespaces
NAME            READY   AGE
pebble-issuer   True    6s
```

## Install your sample application

Now comes the part for the application installation. It could be anything an nginx application or some process. But the idea is to have an ingress that we can hit.
So for this example, I'm going to use the kind [application example](https://kind.sigs.k8s.io/docs/user/ingress/).

If you notice it a little different from **kind** example. I have removed all the **bar** installation, as we don't need to test multiple host at the time.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
  - command:
    - /agnhost
    - netexec
    - --http-port
    - "8080"
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    name: foo-app
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: foo-ingress
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "pebble-issuer"
spec:
  ingressClassName: "nginx"
  rules:
    - host: foo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: foo-service
                port:
                  number: 8080
  tls:
    - hosts:
        - foo.example.com
      secretName: foo-tls-secret
```

Also, my **ingress** changed. I have added a host (foo.example.com) and a tls section to create the new certificate.
I have also added the **cluster-issuer** annotation with the name of of our pebble issuer.

```yaml
  tls:
    - hosts:
        - foo.example.com
      secretName: foo-tls-secret
```

To implement this example app:
1. Navigate to the pebble directory
2. Install the new app

```bash
kubectl apply -f example-app.yaml
```

You will get this output:
```bash
~/Documents/https-certificates-local-kubernetes main > kubectl apply -f example-app.yaml
pod/foo-app created
service/foo-service created
ingress.networking.k8s.io/foo-ingress created
```

Also, if you check the CRDs, you will find changes on **certificaterequests**, **certificates** and **orders**:
```bash
~/Documents/https-certificates-local-kubernetes main > kubectl get certificaterequests --all-namespaces
NAMESPACE   NAME               APPROVED   DENIED   READY   ISSUER          REQUESTOR                                         AGE
default     foo-tls-secret-1   True                True    pebble-issuer   system:serviceaccount:cert-manager:cert-manager   5m40s

~/Documents/https-certificates-local-kubernetes main > kubectl get certificates --all-namespaces
NAMESPACE   NAME             READY   SECRET           AGE
default     foo-tls-secret   True    foo-tls-secret   6m7s

~/Documents/https-certificates-local-kubernetes main > kubectl get orders --all-namespaces
NAMESPACE   NAME                          STATE   AGE
default     foo-tls-secret-1-3991384144   valid   6m37s
```

### Change your host file

First thing first. 
This is a dev environment and you will need to change your host file to allow **127.0.0.1** to be accessed as **foo.example.com**. Because we are not hosting it anywhere but our own computer.

**For example:**
```bash
~/Documents/https-certificates-local-kubernetes main > more /etc/                                                                        danielbianco@eureka
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost foo.example.com
```

So far so good, but if you test your application will find that the url is not trusted.

```bash
~/Documents/https-certificates-local-kubernetes main > curl https://foo.example.com/
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Let continue with the next step to solve this issue.

## Pull your CA certificates

As I mention before, this is an Internal CA, so the certificates are not trusted by anyone.
To be able to test your application, you will need to pull the **intermediate** and **root** certificates and import them to your browser or **cacerts** directory. 
You can find pebble information [here](https://github.com/letsencrypt/pebble?tab=readme-ov-file#ca-root-and-intermediate-certificates) on the location of the **intermediate** and **root** certs.

Depending on the pebble version you will be able to get the certificates directly with the instructions below. 
Another option is to change the service for pebble-mgmt from **ClusterIP** to **NodePort** and access directly or you could start a **pod helper** with (`kubectl run my-shell --rm -i --tty --image ubuntu -- bash`) and access them from the cluster network.
There are many ways, you are grown up test them.

1. Get the Intermediate Certificate:
```bash
kubectl exec deploy/pebble -- sh -c "apk add curl > /dev/null; curl -ksS https://localhost:15000/intermediates/0" > pebble.intermediate.pem.crt
```

2. Get the Root Certificate:
```bash
kubectl exec deploy/pebble -- sh -c "apk add curl > /dev/null; curl -ksS https://localhost:15000/roots/0" > pebble.root.pem.crt
```

3. Combine the chain certificates:
```bash
cat pebble.intermediate.pem.crt pebble.root.pem.crt > pebble.root.crt
```

At this point you could import the **pebble.root.crt** to any browser or you could import it to **/etc/ssl/cacerts** to test it with a terminal.

## Test

So after a few hours, unless you followed the instructions we are at the testing moment :D.

So before testing lets make sure we imported the **pebble.root.crt** to our **/etc/ssl/cacerts**
Well, there isn't much science there:

```bash
sudo cp pebble.root.crt /etc/ssl/certs/ca-certificates.crt
```

Testing it with curl is not much different, you will only need to add the **cacerts** option. Depending on your configuration you could have it by default, but never too sure.

```bash
/tmp > curl --cacert /etc/ssl/certs/ca-certificates.crt  https://foo.example.com
NOW: 2024-10-06 15:54:09.135725422 +0000 UTC m=+2747.085528173%   
```

So, this doesn't say much, basically we can access the url. 
But if you pay attention, I'm accessing it using **https** so its going to request the certificate and it is trusted.

Below you can see the same request using the -v (verbose) in curl to get more information.
There is a successful **TLS** handshare and the certificate is up to date and **ok**.
```bash
/tmp > curl -v --cacert /etc/ssl/certs/ca-certificates.crt  https://foo.example.com
*   Trying 127.0.0.1:443...
* Connected to foo.example.com (127.0.0.1) port 443 (#0)
* ALPN: offers h2,http/1.1
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/certs/ca-certificates.crt
*  CApath: none
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256
* ALPN: server accepted h2
* Server certificate:
*  subject: [NONE]
*  start date: Oct  6 14:56:38 2024 GMT
*  expire date: Oct  6 14:56:37 2029 GMT
*  subjectAltName: host "foo.example.com" matched cert's "foo.example.com"
*  issuer: CN=Pebble Intermediate CA 767be1
*  SSL certificate verify ok.
* using HTTP/2
* h2 [:method: GET]
* h2 [:scheme: https]
* h2 [:authority: foo.example.com]
* h2 [:path: /]
* h2 [user-agent: curl/8.1.2]
* h2 [accept: */*]
* Using Stream ID: 1 (easy handle 0x7fbb48814800)
> GET / HTTP/2
> Host: foo.example.com
> User-Agent: curl/8.1.2
> Accept: */*
> 
< HTTP/2 200 
< date: Sun, 06 Oct 2024 16:24:57 GMT
< content-type: text/plain; charset=utf-8
< content-length: 62
< strict-transport-security: max-age=31536000; includeSubDomains
< 
* Connection #0 to host foo.example.com left intact
NOW: 2024-10-06 16:24:57.596645011 +0000 UTC m=+4596.946651119% 
```

## In conclusion

There are many method to make this kind of test. This one is one more.
The interesting thing about this one, is that you can correlate with a production evironment. Many environment rely on cert-manager using let's encrypt with the Ingress. 