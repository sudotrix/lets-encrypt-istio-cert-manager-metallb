# lets-encrypt-istio-cert-manager-metallb
# lets-encrypt+istio+cert-manager+k8s
#kubernetes

Main documentation from medium:
[Istio + cert-manager + Let’s Encrypt demystified - gregoireW - Medium](https://medium.com/@gregoire.waymel/istio-cert-manager-lets-encrypt-demystified-c1cbed011d67)
# This example shows how to install and configure letsencrypt  on k8s env with its, cert-manager, metallb.
Install metallb:
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml

You need to specify inside configmap available IP's form the router for your LB sac’s
```
ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```


Tools set up:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

Now you can initialize the helm server part. This will create a service and a deployment named `tiller-deploy`. Helm will use this to manage releases:
 helm init —service-account tiller —history-max 200

**Istio setup**
```
helm repo add istio.io https://storage.googleapis.com/istio-release/releases/1.1.2/charts/ 
helm repo update
helm install istio.io/istio-init —name istio-init —namespace istio-system
```

```
helm install istio.io/istio \
       —name istio \
       —namespace istio-system \
       —set gateways.istio-ingressgateway.sds.enabled=true
```

**Cert-manager**
```
 kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.7/deploy/manifests/00-crds.yaml
```

Label the cert-manager namespace to disable resource validation
```
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
```

 Add the Jetstack Helm repository
```
helm repo add jetstack  [https://charts.jetstack.io](https://charts.jetstack.io/) 
```

```
helm repo update
```

```
helm install \
  —name cert-manager \
  —namespace cert-manager \
  —version v0.9.0 \
  jetstack/cert-manager
Install metal
```

DNS

You need to define domain for your istioingressgateway LB sac IP. You need to create a record for that, associate IP address with that domain, which need to be public. (Lets encrypt need to resolve them).

**Prerequisite for certificate generation**

You need to create issuer or cluster issuer. Difference it issuer works pier namespace, cluster issuer works wildly.
Issuer:

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-prod
  namespace: istio-system
spec:
  acme:
    email: romapereverziev@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: example-issuer-account-key
    http01: {}
```

As we know this issuer will create ingress object, you need to create the Istio gateway that will do the transform. It is a simple gateway with a specific name (“istio-autogenerated-k8s-ingress”) listening everything on port 80

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-autogenerated-k8s-ingress
  namespace: istio-system
  labels:
    app: ingressgateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      protocol: HTTP2
      name: http
    hosts:
    - “*”
```
All is ready to create some certificates!

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: test-certificate
  namespace: istio-system
spec:
  secretName: test-certificate
  issuerRef:
    name: letsencrypt-prod
  commonName: hello.sudotrix.com
  dnsNames:
  - hello.sudotrix.com
  - httpbin.sudotrix.com
  acme:
    config:
    - http01:
        ingressClass: istio
      domains:
      - hello.sudotrix.com
      - httpbin.sudotrix.com
```

**Creating a gateway for https**

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: test-gateway
  namespace: istio-system
  labels:
    app: ingressgateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      protocol: HTTPS
      name: https-default
    tls:
      mode: SIMPLE
      serverCertificate: “sds”
      privateKey: “sds”
      credentialName: “test-certificate”
    hosts:
    - “*”
```


```
openssl s_client -connect httpbin.sudotrix.com:443
```

**Deploying httpbin sample**
```
kubectl create ns httpbin
kubectl label ns httpbin istio-injection=enabled
kubectl apply -n httpbin -f https://raw.githubusercontent.com/istio/istio/1.1.2/samples/httpbin/httpbin.yaml
```


```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: httpbin
spec:
  hosts:
  - “httpbin.a.b.c.d.xip.io”
  gateways:
  - istio-system/test-gateway
  http:
  - match:
    route:
    - destination:
        host: httpbin
        port:
          number: 8000
```

```
https://httpbin.sudotrix.com/status/418 
```


```
since you have used the letsencrypt-prod issuer, and haven’t done anything special/non-standard, the certificate renewal process will be completely automatic for you.
By default the letsencrypt certificates are valid fro 90-days, and renewed automatically every 30-days. If you don’t have some strict requirements to use purchased certificates, or use some other specific Certificate Authority, this is a great option to use.
If you still have doubts then you can do the following to see for yourself. First decode the current certificates secret data and inspect the certificate contents with the openssl command. You’ll be able to see the certificate expiry date, and make a note of that. Now if you subtract 59-days from that expiry date that should give you roughly the date that cert-manager will attempt to renew the certificate on. I add an extra day just to be safe we aren’t too early. Then on that date repeat this process again; decoding the certificate secret, inspecting the certificate with the openssl command, and checking the certificate expiry date. You’ll notice the expiry date for the certificate is different than before, hence it’s was automatically renewed as we expected.

```




















