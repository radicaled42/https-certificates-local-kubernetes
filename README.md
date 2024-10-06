# **HTTPS Certificates in Local Kubernetes – The "Let's-Encrypt-but-Not-Quite" Adventure**

So, one day, a wild challenge appeared: Dockerize a Python app, slap on a valid certificate, deploy it in Kubernetes, and make it accessible. Oh, and the cherry on top? The certificate *has* to be trusted, so no funny business with ignoring SSL validation. Easy, right? (Spoiler: It's Kubernetes. It's never that easy.)

> The client has to call the server by its fancy alternative DNS names using private SSL certification. You can use OpenSSL, Let's Encrypt, Certbot, or whatever flavor of cryptographic pain you enjoy, as long as it’s trusted. Skipping validation? Pfft, amateurs.

### Enter **Pebble**, the Let's Encrypt Simulator

Could you generate a simple CA with OpenSSL and call it a day? Sure, but where's the fun in that? We're in Kubernetes, after all. If you aren't complicating things with DNS, Ingress controllers, and containers that just won’t die, are you even DevOps-ing?

Instead of the vanilla approach, I opted for Pebble, a *miniature* version of [Boulder](https://github.com/letsencrypt/boulder). Pebble is like Boulder’s little sibling who doesn’t handle production but is great for messing around in your local cluster. Thanks to [this guide](https://blog.manabie.io/2021/11/simulate-https-certificates-acme-k8s/), I embarked on this glorious quest to simulate SSL certificates with the grace of a cat knocking over your coffee cup.

Here’s the plan:

1. Spin up a Kind cluster (or Minikube if you're feeling adventurous).
2. Install Nginx Ingress (because we like things simple—right?).
3. Hack CoreDNS (because why not).
4. Install Pebble (the highlight of our story).
5. Install your sample app (yes, you need an actual app).
6. Pull your CA certificates (the most boring part).
7. Test (hope for the best).
   
Easy peasy.
So lets get going.

## Step 1: Install the Kind Cluster

We start with **Kind**, you could have many other option but we will be doing with this one. 
Anyway, follow the [Kind installation instructions](https://kind.sigs.k8s.io/docs/user/quick-start/#installation).

For the **Linux folks**, here’s the magic spell:

**For linux**:
```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Now that we’ve summoned Kind, let’s make a **kind-config** file. It's like Kubernetes' shopping list, where you tell it what you want: ports 80 and 443 exposed, and a node-label because Nginx you need to be able to create the pods somewhere.

For example:
```yaml
kind: Cluster
name: https-cluster
...
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
...
```

Run `kind create cluster --config=kind-config` and voilà:
```bash
main > kind create cluster --config kind-config
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

Now, Kubernetes is set up, and everything is working perfectly. But for now, let’s move on.

## Step 2: Install Nginx Ingress Controller

Why Nginx? Because kind has already develop the resources definition. 
You can download and change them or you could use some other ingress controller (like Traefik).

Use this command to install it:
```bash
main > kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

If everything went well (ha!), you should see:
```bash
main > kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
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

main > kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-52z5n        0/1     Completed   0          58s
ingress-nginx-admission-patch-qdd78         0/1     Completed   0          58s
ingress-nginx-controller-6b8cfc8d84-vvdrf   1/1     Running     0          58s
```

## Step 3: Modify CoreDNS (The Hacky Part)

Now, we’ll do something truly DevOps-y: hack CoreDNS. Think of it as the DNS version of fixing your sink with duct tape. We're making Kubernetes pretend to know what it's doing with domain resolution.
Because this domain doesn't exist anywhere, please look it here.

First, grab the IP of your Ingress controller:

```bash
main > kubectl get svc ingress-nginx-controller --no-headers -n ingress-nginx | awk '{print$3}'
10.96.94.238
```

Then, convince CoreDNS to resolve `foo.example.com` to this IP. Add the CoreDNS configfile the information of the new host:

```yaml
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
```

Apply the changes and restart CoreDNS. (Or just kill all the pods and let Kubernetes handle the mess this is development)

```bash
main > kubectl delete pod -n kube-system --wait $(kubectl get pods -n kube-system | grep coredns)
```

## Step 4: Install Pebble (The Cool Part)

Finally, we get to the fun bit: installing Pebble. But first, you need **cert-manager**. Let's install it with [**Helm**](https://helm.sh/docs/intro/install/) because why make things easy?

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
```

Next, deploy Pebble.
I have attached the Pebble resources to the **pebble** directory on this repo. 
Otherwise you can download them from the [original] (https://github.com/manabie-com/manabie-com.github.io/blob/main/content/posts/simulate-https-certificates-acme-k8s/examples/pebble.yaml)
Or you can create them its not difficult.
I have added the **15000** service, because with the latest version of the docker image **sh** is not available.

```bash
cd ./pebble
kubectl apply -f pebble.yaml
kubectl get pod
```

If all went well (don’t hold your breath), **Pebble** will be up and running, looking very smug with its fake certificates.

```bash
~/Documents/https-certificates-local-kubernetes main > kubectl get pods -n cert-manager
NAME                                     READY   STATUS    RESTARTS   AGE
cert-manager-7b9875fbcc-b2mmv            1/1     Running   0          88s
cert-manager-cainjector-948d47c6-6sjbj   1/1     Running   0          88s
cert-manager-webhook-78bd84d46b-bxt29    1/1     Running   0          88s

~/Documents/https-certificates-local-kubernetes main > kubectl get pods -n default | grep pebble
pebble-585fd5f6c-m4n4j   1/1     Running   0          54s
```

The first thing you'll need to configure after you've installed **cert-manager** and **Pebble** is an Issuer or a ClusterIssuer. These are resources that represent certificate authorities (CAs) able to sign certificates in response to certificate signing requests.

Install pebble cluster-issuer

```bash
cd ./pebble
kubectl apply -f ./pebble-issuer.yaml

main > kubectl get clusterissuers.cert-manager.io --all-namespaces
NAME            READY   AGE
pebble-issuer   True    6s
```

## Step 5: Install the Sample Application

Now for the app—something small and simple. In this case, we're deploying a barebones app and adding an **Ingress** with TLS enabled. I'm going to use the kind [application example](https://kind.sigs.k8s.io/docs/user/ingress/).

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

Also, my **ingress** changed. I have added a host (foo.example.com) and a TLS section has been enabled.
I have also added the **cluster-issuer** annotation with the name of of our pebble issuer.

```yaml
  tls:
    - hosts:
        - foo.example.com
      secretName: foo-tls-secret
```

Apply the changes, cross your fingers, and let Kubernetes do its thing.

```bash
main > kubectl apply -f example-app.yaml
```

You will get this output:
```bash
main > kubectl apply -f example-app.yaml
pod/foo-app created
service/foo-service created
ingress.networking.k8s.io/foo-ingress created
```

Also, if you check the CRDs, you will find changes on **certificaterequests**, **certificates** and **orders**:
```bash
main > kubectl get certificaterequests --all-namespaces
NAMESPACE   NAME               APPROVED   DENIED   READY   ISSUER          REQUESTOR                                         AGE
default     foo-tls-secret-1   True                True    pebble-issuer   system:serviceaccount:cert-manager:cert-manager   5m40s

main > kubectl get certificates --all-namespaces
NAMESPACE   NAME             READY   SECRET           AGE
default     foo-tls-secret   True    foo-tls-secret   6m7s

main > kubectl get orders --all-namespaces
NAMESPACE   NAME                          STATE   AGE
default     foo-tls-secret-1-3991384144   valid   6m37s
```

### First Test

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

## Step 6: Get Your CA Certificates

Now comes the fun part: pulling the **intermediate** and **root** certificates from Pebble, because no one trusts your internal CA.

You can find pebble information [here](https://github.com/letsencrypt/pebble?tab=readme-ov-file#ca-root-and-intermediate-certificates) on the location of the **intermediate** and **root** certs.

Depending on the pebble version you will be able to get the certificates directly with the instructions below. 
Another option is to change the service for pebble-mgmt from **ClusterIP** to **NodePort** and access directly or you could start a **pod helper** with (`kubectl run my-shell --rm -i --tty --image ubuntu -- bash`) and access them from the cluster network.
There are many ways, you are grown up test them.

Get the Intermediate and root certificates

```bash
kubectl exec deploy/pebble -- sh -c "apk add curl > /dev/null; curl -ksS https://localhost:15000/intermediates/0" > pebble.intermediate.pem.crt

kubectl exec deploy/pebble -- sh -c "apk add curl > /dev/null; curl -ksS https://localhost:15000/roots/0" > pebble.root.pem.crt
```

Combine the certificates like a pro:

```bash
cat pebble.intermediate.pem.crt pebble.root.pem.crt > pebble.root.crt
```

Now, import the certificates into your browser or your OS's CA store. You’re almost there!

## Step 7: Test and Rejoice (Maybe)

Finally, test your app! Use `curl` with the `--cacert` flag to verify the certificate.

If all goes well, you’ll see something like:

```bash
/tmp > curl --cacert /etc/ssl/certs/ca-certificates.crt  https://foo.example.com
NOW: 2024-10-06 15:54:09.135725422 +0000 UTC m=+2747.085528173%   
```

So, this doesn't say much, basically we can access the url. 
But if you pay attention, I'm accessing it using **https** so its going to request the certificate and it is trusted.

Below you can see the same request using the -v (verbose) in curl to get more information.
There is a successful **TLS** handshare and the certificate is up to date and `SL certificate verify ok`.

```bash
main > curl -v --cacert /etc/ssl/certs/ca-certificates.crt  https://foo.example.com
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

If not, well, Kubernetes can be fickle. Double-check your CoreDNS settings, cert-manager, and Pebble configuration. Or, as is tradition, blame DNS.

## Conclusion

There are a million ways to simulate SSL in Kubernetes. This one is just one more. The best part? You’ve basically recreated a production-like environment with cert-manager, Ingress, and a Let's-Encrypt-but-not-quite cert issuance system. And hey, it only took a few hours of troubleshooting, head-scratching, and maybe a little crying.
