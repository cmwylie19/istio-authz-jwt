# AuthZ w/ AuthService, AuthorizationPolicy, and RequestAuthentication POC

- [AuthService Docs](https://github.com/istio-ecosystem/authservice/tree/master/bookinfo-example#further-protect-via-requestauthentication-and-authorization-policy)
- [Keycloak Downloads](https://www.keycloak.org/downloads)

## Demo 

### Keycloak Setup 

Install Keycloak

```bash
helm install uds-kc oci://registry-1.docker.io/bitnamicharts/keycloak

# or locally because authservice required https
./bin/kc.sh start-dev --https-certificate-file=cert.pem --https-certificate-key-file=key.pem --https-port 443
```

Get the keycloak password (for the kubernetes version, local you create your own user and password)

```bash
PW=$(k get secret   uds-kc-keycloak   -ojsonpath='{.data.admin-password}' | base64 -d)
echo $PW | pbcopy
```

Login [console](http://localhost:3000) for kubernetes keycloak with user `user` and password `$PW`, use this link for [local](https://0.0.0.0:8443), create a new realm `uds-prod`

```bash
k port-forward svc/uds-kc-keycloak 3000:80 -n keycloak
```     

Create a new client with `Client ID` `bookinfo`, `Client Type` `OpenID Connect`, `Authentication Flow` `standard flow and Direct access` and `Valid Redirect URIs` `https://localhost:8443/productpage/oauth/callback`. Make sure `Client authentication` is On.

Go to the `Credentials` Tab and copy the client secret

```bash
export OIDC_CLIENT_ID=bookinfo
export OIDC_CLIENT_SECRET="<your-client-secret"
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


<!-- Deploy the external authorizer and verify it is up.

NOT NECESSARY

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/extauthz/ext-authz.yaml

kubectl logs "$(kubectl get pod -l app=ext-authz  -o jsonpath={.items..metadata.name})"  -c ext-authz
```      -->

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
    # - name: "sample-ext-authz-grpc"
    #   envoyExtAuthzGrpc:
    #     service: "ext-authz.default.svc.cluster.local"
    #     port: "9000"
    # - name: "sample-ext-authz-http"
    #   envoyExtAuthzHttp:
    #     service: "ext-authz.default.svc.cluster.local"
    #     port: "8000"
    #     # includeRequestHeadersInCheck: ["x-ext-authz"]
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


Install autservice via helm, notice there is a helm [`values.yaml`](./authservice/bookinfo-example/authservice/values.yaml) file that sets the `oidc.clientID` and `oidc.clientSecret`.

Values for these can be found in `realmSettings.endpoints.OpenID_EndpointConfig`, follow this [link](https://0.0.0.0:8443/realms/uds/.well-known/openid-configuration) for more openid-configuration.

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

Before we go on, check the logs for the auth service pod, we need to update a configmap because it is chatting with googleapis.com:443..

```bash
┌─[cmwylie19@Cases-MacBook-Pro] - [~/istio-authz-jwt/authservice/bookinfo-example] - [2023-12-12 03:30:16]
└─[0] <git:(main 04e5b5b✱) > k logs -f -l app=authservice
[2023-12-12 20:29:59.784] [console] [trace] Get: closing connection, response payload size 1033
[2023-12-12 20:29:59.790] [console] [debug] status for jwks parsing: parseJwks, OK
[2023-12-12 20:30:09.793] [console] [trace] Get
[2023-12-12 20:30:09.855] [console] [info] Get: opening connection to www.googleapis.com:443
[2023-12-12 20:30:09.910] [console] [trace] Get: closing connection, response payload size 1033
[2023-12-12 20:30:09.917] [console] [debug] status for jwks parsing: parseJwks, OK
[2023-12-12 20:30:19.917] [console] [trace] Get
```
Get the configmap and make edits

```bash
 k get cm authservice -oyaml | pbcopy
```

Edits, make sure they match up with the .well-known/openid-configuration


```yaml
k replace -f -<<EOF
apiVersion: v1
data:
  config.json: |
    {
      "listen_address": "0.0.0.0",
      "listen_port": "10003",
      "log_level": "trace",
      "threads": 8,
      "allow_unmatched_requests": "false",
      "chains": [
        {
          "name": "idp_filter_chain",
          "match": {
            "header": ":authority",
            "prefix": "localhost",
          },
          "filters": [
          {
            "oidc":
              {
                "authorization_uri": "https://0.0.0.0/realms/uds/protocol/openid-connect/auth",
                "token_uri": "https://0.0.0.0/realms/uds/protocol/openid-connect/token",
                "callback_uri": "https://localhost/productpage/oauth/callback",
                "jwks_fetcher": {
                  "jwks_uri": "https://0.0.0.0/realms/uds/protocol/openid-connect/certs",
                  "periodic_fetch_interval_sec": 10
                },
                "client_id": "bookinfo",
                "client_secret": "16kcHDKJngKNjOFgXRlOYoMjkQwYApGr",
                "scopes": [],
                "cookie_name_prefix": "productpage",
                "id_token": {
                  "preamble": "Bearer",
                  "header": "Authorization"
                },
                "logout": {
                  "path": "/authservice_logout",
                  "redirect_uri": "https://localhost/some/logout/path"
                }
              }
            }
          ]
        }
      ]
    }
kind: ConfigMap
metadata:
  name: authservice
  namespace: default
EOF
```

Restart the pod and listen again

```bash
k delete po -l app=authservice --force
```

[Listen to pod logs]
```bash
k logs -f -l app=authservice
```

In this case I get an error everytime, no matter the port

```bash
[2023-12-12 20:38:39.284] [console] [info] Get: opening connection to 0.0.0.0:443
[2023-12-12 20:38:39.289] [console] [error] Get: unexpected exception: Connection refused [system:111]
[2023-12-12 20:38:39.289] [console] [warning] operator(): HTTP connection error
[2023-12-12 20:38:41.397] [console] [warning] onReadDone: chain:idp_filter_chain JWKS is not ready
```

I believe it is that it does not trust the Keycloak CA
### Deploy Bookinfo 

```bash
kubectl create -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
```

```bash
kubectl port-forward service/istio-ingressgateway 3000:443 -n istio-system
```

Visit [https://localhost:8443/productpage](https://localhost:8443/productpage) 
