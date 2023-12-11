# AuthZ w/ AuthService, AuthorizationPolicy, and RequestAuthentication POC

- [Docs](https://github.com/istio-ecosystem/authservice/tree/master/bookinfo-example#further-protect-via-requestauthentication-and-authorization-policy)


## Demo 

### Keycloak Setup 

Install Keycloak

```bash
kubectl create ns keycloak 
helm install uds-kc oci://registry-1.docker.io/bitnamicharts/keycloak -n keycloak 
```

Get the keycloak admin password

```bash
PW=$(k get secret -n keycloak  uds-kc-keycloak   -ojsonpath='{.data.admin-password}' | base64 -d)
echo $PW | pbcopy
```

Login [console](http://localhost:3000) with user `user` and password `$PW`, create a new realm `uds-prod`

```bash
k port-forward svc/uds-kc-keycloak 3000:80 -n keycloak
```     

Create a new client with `Client ID` `bookinfo`, `Client Type` `OpenID Connect`, `Authentication Flow` `standard flow and Direct access` and `Valid Redirect URIs` `https://localhost:8443/*`. Make sure `Client authentication` is On.

Go to the `Credentials` Tab and copy the client secret

```bash
export OIDC_CLIENT_ID=bookinfo
export OIDC_CLIENT_SECRET="<your-client-secret>"
```

### Istio/AuthService Setup

Install istio and label default ns `istio-injection=enabled`

```bash
istioctl install -y
kubectl label namespace default istio-injection=enabled --overwrite
```

Install and Enable Authservice
```bash
bash ./authservice/bookinfo-example/scripts/generate-self-signed-certs-for-ingress-gateway.sh
```


Deploy the external authorizer and verify it is up.

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/extauthz/ext-authz.yaml

 kubectl logs "$(kubectl get pod -l app=ext-authz  -o jsonpath={.items..metadata.name})"  -c ext-authz
```     

Define the external authorizer in the `extensionProviders` section of istio `cm`

```bash
kubectl get cm istio -n istio-system -oyaml | pbcopy
```

Edit the `cm` adding the `extensionProviders` section

```yaml
kubectl replace -f -<<EOF
apiVersion: v1
data:
  mesh: |-
    # Add the following content to define the external authorizers.
    extensionProviders:
    - name: "authservice-grpc"
        envoyExtAuthzGrpc:
        service: authservice.default.svc.cluster.local
        port: "10003"
    - name: "sample-ext-authz-grpc"
      envoyExtAuthzGrpc:
        service: "ext-authz.default.svc.cluster.local"
        port: "9000"
    - name: "sample-ext-authz-http"
      envoyExtAuthzHttp:
        service: "ext-authz.default.svc.cluster.local"
        port: "8000"
        includeRequestHeadersInCheck: ["x-ext-authz"]
    defaultConfig:
      discoveryAddress: istiod.istio-system.svc:15012
      proxyMetadata: {}
      tracing:
        zipkin:
          address: zipkin.istio-system:9411
    defaultProviders:
      metrics:
      - prometheus
    enablePrometheusMerge: true
    rootNamespace: istio-system
    trustDomain: cluster.local
  meshNetworks: 'networks: {}'
kind: ConfigMap
metadata:
  labels:
    install.operator.istio.io/owning-resource: installed-state
    install.operator.istio.io/owning-resource-namespace: istio-system
    istio.io/rev: default
    operator.istio.io/component: Pilot
    operator.istio.io/managed: Reconcile
    operator.istio.io/version: 1.20.0
    release: istio
  name: istio
  namespace: istio-system
EOF
```


Install autservice via helm 

```bash
cd authservice/bookinfo-example

# make sure the OIDC_CLIENT_ID and OIDC_CLIENT_SECRET are set
helm template authservice \
   --set oidc.clientID=${OIDC_CLIENT_ID} \
   --set oidc.clientSecret=${OIDC_CLIENT_SECRET} \
   | kubectl apply --dry-run=client -oyaml -f - | egrep "client_"


helm template authservice \
   --set oidc.clientID=${OIDC_CLIENT_ID} \
   --set oidc.clientSecret=${OIDC_CLIENT_SECRET} \
   | kubectl apply -f -
```

### Deploy Bookinfo 

```bash
kubectl create -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
```

```bash