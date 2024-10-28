# Istio Gateway with Simple Demo app:


## Installation Steps
In this section, you’ll find the steps to install the Istio gateway in your cluster, using NGINX as an example and Self Singed SSL (no need for certmanager).
### 1-  Install the Gateway API Custom Resource Definitions (CRDs)

```shell

ISTIO_VERSION="v1.2.0"
kubectl apply -k "github.com/kubernetes-sigs/gateway-api/config/crd/standard?ref=${ISTIO_VERSION}"

```

### 2- Enable Gateway API in Istio
Add Istioctl to your environment
```shell curl -L https://istio.io/downloadIstio | sh -
export PATH="$PATH:/path/to/istioctl"

# Install Istio with Gateway API Support:
istioctl install --set profile=default --set values.gateways.istio-ingressgateway.type=LoadBalancer -y
 ```
Enable Gateway API for Istio:
```shell
istioctl install --set values.pilot.env.PILOT_ENABLE_GATEWAY_API=true -y
 ```
Define a GatewayClass for Istio:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: istio
spec:
  controllerName: istio.io/gateway-controller

```
### 3- Prepare SSL for our Sample App
in this example we are using self signed for local machine
Create a Self-Signed Certificate with OpenSSL
```shell

# Set your IP address as a variable
YOUR_IP_ADDRESS=$(hostname -I | awk '{print $1}')

openssl genrsa -out nginx-selfsigned.key 2048

openssl req -new -x509 -key nginx-selfsigned.key -out nginx-selfsigned.crt -days 365 \
    -subj "/C=US/ST=State/L=City/O=Organization/OU=Department/CN=localhost" \
    -addext "subjectAltName=IP:$YOUR_IP_ADDRESS"

kubectl create secret tls nginx-selfsigned --cert=nginx-selfsigned.crt --key=nginx-selfsigned.key -n nginx-app

```


### 4- Create our sample app:
Create nameSpace ( opptional )
```shell
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-app

```

<b> Define an NGINX Deployment and Service </b>
```shell

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx-app
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP

Create a Gateway:
kind: Gateway
metadata:
  name: nginx-gateway
  namespace: nginx-app
spec:
  gatewayClassName: istio
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: nginx-tls
  - name: http
    protocol: HTTP
    port: 80

```
<b> Define an HTTPRoute: <b/>

```shell


apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: nginx-route
  namespace: nginx-app
spec:
  parentRefs:
  - name: nginx-gateway
  hostnames:
  - "localhost"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: nginx-service
      port: 80


```




<b> Verify the Deployment </b>
```shell
kubectl get pods -n nginx-app
kubectl get svc -n nginx-app
kubectl get ingress -n nginx-app
kubectl get gateways -n nginx-app
```

<b> Test your Sample App: </b>

```shell

 kubectl port-forward service/nginx-gateway-istio -n nginx-app --address 0.0.0.0 8443:443

```
