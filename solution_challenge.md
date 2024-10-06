
# FeverUp Challenge Solution

The idea of the challenge was to dockerize, build, and deploy an application in Kubernetes.

## Multinode Cluster

To run a Kind cluster with multiple workers, you need to add new roles, such as **worker**, to the Kind configuration.

### Start Kind Cluster

Run the following command to create the cluster:

```bash
kind create cluster --config=kind-config
```

## Docker Application

As part of bonus task 1 of the challenge, you need to **perform a request every 5 seconds**. Some changes to the **client** application have been made. Additionally, the `/` route has been added to let the application listen, because with the original configuration, the client pod runs once and then restarts.

### Build Docker Application

1. Navigate to the client or server directory and run the following commands to build the Docker images:

```bash
docker build -t client-challenge:v0.0.1 -f ./Dockerfile .
docker build -t server-challenge:v0.0.1 -f ./Dockerfile .
```

2. Once the images are built, load them into the cluster environment:

```bash
kind load docker-image client-challenge:v0.0.1 --name fever-challenge
kind load docker-image server-challenge:v0.0.1 --name fever-challenge
```

3. Verify that the images have been loaded:

```bash
docker exec test-ingress-control-plane ctr --namespace=k8s.io images ls | grep 'server'
docker exec test-ingress-control-plane ctr --namespace=k8s.io images ls | grep 'client'
```

## Deploy Nginx as Ingress Controller

1. Download the Nginx Ingress controller for Kind from the following URL:

```bash
https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

2. Modify the ConfigMap to allow the use of the source IP:

```yaml
log-format-upstream: $proxy_add_x_forwarded_for - $remote_addr - $host $remote_user
  [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent
  $request_length $request_time [$proxy_upstream_name] [$proxy_alternative_upstream_name]
  $upstream_addr $upstream_response_length $upstream_response_time $upstream_status
  $req_id
real-ip-header: X-Forwarded-For
real-ip-recursive: "true"
set-real-ip-from: 0.0.0.0/0
use-forwarded-headers: "true"
```

3. Install the Ingress controller:

```bash
kubectl apply -f deploy.yaml
```

## Modify CoreDNS

Given that I used Pebble as a solution for the certificate creation, we need to add the host to the CoreDNS configmap so we can access them externally.

1. Obtain the Nginx Ingress Controller IP:

```bash
kubectl get svc ingress-nginx-controller --no-headers -n ingress-nginx | awk '{print$3}'
```

2. Add this IP to the **coredns-challenge.yaml** file.
3. Apply the CoreDNS configuration:

```bash
kubectl apply -f coredns-configmap.yaml
```

4. Restart the CoreDNS pods.

## Pebble - Let's Encrypt Simulator

1. Install **cert-manager**:

```bash
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set crds.enabled=true
```

2. Navigate to the pebble directory and install Pebble:

```bash
kubectl apply -f pebble.yaml
kubectl get pod
```

### Pebble Issuer

A `ClusterIssuer` is used to specify the configuration and provider for issuing certificates that will be available across all namespaces in the cluster.

1. Install the Pebble Issuer:

```bash
kubectl apply -f ./pebble-issuer.yaml
```

## Application Installation

### Server

1. Navigate to the **server** directory.
2. Install the **ingress.yaml** file first:

```bash
kubectl apply -f ./ingress.yaml
```

3. Wait until the server certificate is created.
4. Install the **service.yaml** and **deployment.yaml**:

```bash
kubectl apply -f ./service.yaml
kubectl apply -f ./deployment.yaml
```

### Client

1. Navigate to the **client** directory.
2. Install the **ingress.yaml** file first:

```bash
kubectl apply -f ./ingress.yaml
```

3. Wait until the client certificate is created.
4. Install the **service.yaml** and **deployment.yaml**:

```bash
kubectl apply -f ./service.yaml
kubectl apply -f ./deployment.yaml
```

## Test

Pebble is a small ACME test server not intended for production use. It is for testing only.

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

4. Modify the **/etc/hosts** file to map the URLs:

```bash
127.0.0.1 localhost
127.0.0.1 client.example.com server.example.com server.local server
```

5. Run the curl command to test:

```bash
#curl --cacert /etc/ssl/certs/ca-certificates.crt  https://client.example.com

# Single Request

~/Documents/fever/fever-challenge/pebble main > curl --cacert /etc/ssl/certs/ca-certificates.crt  https://client.example.com
Response from https://server: Domain: server, Client IP: 10.244.0.5
<br>Response from https://server.example.com: Domain: server.example.com, Client IP: 10.244.0.5
<br>Response from https://server.local: Domain: server.local, Client IP: 10.244.0.5<br>% 

# Multiple Requests

~/Documents/fever/fever-challenge/client main > curl --cacert /etc/ssl/certs/ca-certificates.crt  https://client.example.com/multiple                                                           danielbianco@eureka
Response from https://server on attempt 1: Domain: server, Client IP: 10.244.0.5
Response from https://server on attempt 2: Domain: server, Client IP: 10.244.0.5
Response from https://server on attempt 3: Domain: server, Client IP: 10.244.0.5
Response from https://server on attempt 4: Domain: server, Client IP: 10.244.0.5
Response from https://server on attempt 5: Domain: server, Client IP: 10.244.0.5
<br>Response from https://server.example.com on attempt 1: Domain: server.example.com, Client IP: 10.244.0.5
Response from https://server.example.com on attempt 2: Domain: server.example.com, Client IP: 10.244.0.5
Response from https://server.example.com on attempt 3: Domain: server.example.com, Client IP: 10.244.0.5
Response from https://server.example.com on attempt 4: Domain: server.example.com, Client IP: 10.244.0.5
Response from https://server.example.com on attempt 5: Domain: server.example.com, Client IP: 10.244.0.5
<br>Response from https://server.local on attempt 1: Domain: server.local, Client IP: 10.244.0.5
Response from https://server.local on attempt 2: Domain: server.local, Client IP: 10.244.0.5
Response from https://server.local on attempt 3: Domain: server.local, Client IP: 10.244.0.5
Response from https://server.local on attempt 4: Domain: server.local, Client IP: 10.244.0.5
Response from https://server.local on attempt 5: Domain: server.local, Client IP: 10.244.0.5<br>% 

# Server Request

#curl --cacert /etc/ssl/certs/ca-certificates.crt  https://server.example.com

~ > curl --cacert /etc/ssl/certs/ca-certificates.crt  https://server.example.com
Domain: server.example.com, Client IP: 10.244.0.5%
```

6. To test on a browser, import the **pebble.root.pem.crt** certificate.

## Extract Server Certificate for Testing
To workaround the faux cert you will need to import the server certificate in kubernetes along the root certificate.

1. Get the server certificate and save it to a `.crt` file:

```bash
kubectl get secrets server-tls-secret -o jsonpath="{.data['tls\.crt']}" | base64 -d > fullchain.server.crt
```

2. Append the root certificate to the **fullchain.server.crt**:

```bash
kubectl exec deploy/pebble -- sh -c "apk add curl > /dev/null; curl -ksS https://localhost:15000/roots/0" >> fullchain.server.crt
```

3. Get the server certificate key:

```bash
kubectl get secrets server-tls-secret -o jsonpath="{.data['tls\.key']}" | base64 -d > tls.key
```

4. Create a fullchain server secret in Kubernetes:

```bash
kubectl create secret tls fullchain-server --cert=./fullchain.server.crt --key=./tls.key
```

## Notes and Bibliography

- **Kind**
  - [Build a Kind Cluster](https://medium.com/@subhampradhan966/setting-up-a-multi-node-kubernetes-cluster-with-kind-a-comprehensive-guide-146ee5994226)
  - [Kind Ingress](https://kind.sigs.k8s.io/docs/user/ingress/)
- **Pebble**
  - [Simulating HTTPS Certificates with ACME and K8s](https://blog.manabie.io/2021/11/simulate-https-certificates-acme-k8s/)
- **Dockerize Python Application**
  - [Dockerizing Python Applications](https://www.docker.com/blog/how-to-dockerize-your-python-applications/)
- **Nginx configuration**
  - [Configmap](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap)