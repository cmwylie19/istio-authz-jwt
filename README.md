# AuthZ w/ AuthService, AuthorizationPolicy, and RequestAuthentication POC

- [AuthService Docs](https://github.com/istio-ecosystem/authservice/tree/master/bookinfo-example#further-protect-via-requestauthentication-and-authorization-policy)
- [Keycloak Downloads](https://www.keycloak.org/downloads)

----
**STATUS** Seems like a CA-CERT issue, it authservice trusts [google](#trusts-google) but not [KeyCloak](#not-trusting-keycloak) .

Relevant [code](https://github.com/istio-ecosystem/authservice/blob/02b9101ad15451accfb9e572f037160056d41c1f/src/common/http/http.cc#L610)


The problem is with KeyCloak's `jwks_uri` endpoint.

```bash
curl -k -X GET -i https://www.googleapis.com/oauth2/v3/certs             
HTTP/2 200 
server: scaffolding on HTTPServer2
x-xss-protection: 0
x-frame-options: SAMEORIGIN
x-content-type-options: nosniff
date: Tue, 12 Dec 2023 21:43:14 GMT
expires: Wed, 13 Dec 2023 02:59:41 GMT
cache-control: public, max-age=18987, must-revalidate, no-transform
content-type: application/json; charset=UTF-8
age: 52
alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
accept-ranges: none
vary: Origin,X-Origin,Referer,Accept-Encoding

{
  "keys": [
    {
      "e": "AQAB",
      "kty": "RSA",
      "use": "sig",
      "kid": "e4adfb436b9e197e2e1106af2c842284e4986aff",
      "n": "psply8S991RswM0JQJwv51fooFFvZUtYdL8avyKObshyzj7oJuJD8vkf5DKJJF1XOGi6Wv2D-U4b3htgrVXeOjAvaKTYtrQVUG_Txwjebdm2EvBJ4R6UaOULjavcSkb8VzW4l4AmP_yWoidkHq8n6vfHt9alDAONILi7jPDzRC7NvnHQ_x0hkRVh_OAmOJCpkgb0gx9-U8zSBSmowQmvw15AZ1I0buYZSSugY7jwNS2U716oujAiqtRkC7kg4gPouW_SxMleeo8PyRsHpYCfBME66m-P8Zr9Fh1Qgmqg4cWdy_6wUuNc1cbVY_7w1BpHZtZCNeQ56AHUgUFmo2LAQQ",
      "alg": "RS256"
    },
    {
      "alg": "RS256",
      "use": "sig",
      "kid": "0ad1fec78504f447bae65bcf5afaedb65eec9e81",
      "e": "AQAB",
      "kty": "RSA",
      "n": "sm72oBH-R2Rqt4hkjp66tz5qCtq42TMnVgZg2Pdm_zs7_-EoFyNs9sD1MKsZAFaBPXBHDiWywyaHhLgwETLN9hlJIZPzGCEtV3mXJFSYG-8L6t3kyKi9X1lUTZzbmNpE0tf-eMW-3gs3VQSBJQOcQnuiANxbSXwS3PFmi173C_5fDSuC1RoYGT6X3JqLc3DWUmBGucuQjPaUF0w6LMqEIy0W_WYbW7HImwANT6dT52T72md0JWZuAKsRRnRr_bvaUX8_e3K8Pb1K_t3dD6WSLvtmEfUnGQgLynVl3aV5sRYC0Hy_IkRgoxl2fd8AaZT1X_rdPexYpx152Pl_CHJ79Q"
    }
  ]
}

 > curl -k  -X GET -i https://localhost/realms/uds/protocol/openid-connect/certs 
HTTP/2 200 
content-length: 2901
cache-control: no-cache
content-type: application/json;charset=UTF-8

{"keys":[{"kid":"COyMcu1srOaSWwYdFm4ply8nMVhYdTv4FJDC6bNyHFk","kty":"RSA","alg":"RS256","use":"sig","n":"8XsiNnMqUbRoPSJT9GE6U11aCg-rbJ18BpxEDRlSXOGfZejLCdP1d0T3SIPOBOSt8BkjoPGZvgrAK54FDiY3zXZD5XsxyBaHQanb0RmJWtJRGexUOQjyX_Iet08tRx353tdckvjQ5P6Zdha2tc5Zny9xpAp3p8tmonXJuokoX5qfr6dJlNgbB1BGas29pblbFdoho4tLfrrANBgtFLPQVJWDfm8SgeipPOpeXdQ8i5eJ9h_pz-N4xaaoyWaBzd6R0KtX9Rp4ZGqlbdlum1Y2zEX0RWsrlqW5vmHbs5_7__QVmhyN9KH8HAVHxG5N0e-LURU0Y_h0jeWEtMJJ8-VqMQ","e":"AQAB","x5c":["MIIClTCCAX0CBgGMW8pePjANBgkqhkiG9w0BAQsFADAOMQwwCgYDVQQDDAN1ZHMwHhcNMjMxMjEyMDIwODU4WhcNMzMxMjEyMDIxMDM4WjAOMQwwCgYDVQQDDAN1ZHMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDxeyI2cypRtGg9IlP0YTpTXVoKD6tsnXwGnEQNGVJc4Z9l6MsJ0/V3RPdIg84E5K3wGSOg8Zm+CsArngUOJjfNdkPlezHIFodBqdvRGYla0lEZ7FQ5CPJf8h63Ty1HHfne11yS+NDk/pl2Fra1zlmfL3GkCneny2aidcm6iShfmp+vp0mU2BsHUEZqzb2luVsV2iGji0t+usA0GC0Us9BUlYN+bxKB6Kk86l5d1DyLl4n2H+nP43jFpqjJZoHN3pHQq1f1GnhkaqVt2W6bVjbMRfRFayuWpbm+Yduzn/v/9BWaHI30ofwcBUfEbk3R74tRFTRj+HSN5YS0wknz5WoxAgMBAAEwDQYJKoZIhvcNAQELBQADggEBABm7FLL6S8KHsnGK+c7pTRGGtLm72uFuRMIq5Oij7vhUSfd/9vvFFDC59jUVpykNJwsQizTBK8Fii7VtnzW95MUk6Smc2Mu/nmTswTkVJMBI51l6NCZKId4fGwLaT30KkiQew3cigKO8gw/zhR8pQvKOEqUQrxP6mgn/lGWgqrOpa1hiyK5Ox0LDrixZ/9BIw4XAe+gydXZb4Grg0KZZhKD4SKK9MFcwHkA4aFK4BzczAddAmAVtQFF6Z1Pjyg1/gHah3BKFViOdX79J0HvZlCg5ly2TDVuGCkGIi14qjP27Y/bHhLjZrQEvt7JWTocYhQ9D1/j4ejb0MtIqD8N/H0w="],"x5t":"Z0pZP69GJOG5IZsumY0LiycQ6no","x5t#S256":"NuygL8MBbIA6oDmQA8ZNJHpxmaTKcbpqfhCxYZM_MdM"},{"kid":"H8ow2lLfIRwKanwVrDEMf4QqBvaV3yJ82SKhvTplIYE","kty":"RSA","alg":"RSA-OAEP","use":"enc","n":"rbIKVuBXtwA2yAdHv-aUuYUfy7p0tdAkw8JjBXr7HqRULRs06PY7fsrxmHG9Pz_xLELUT8ttuBllvpkM0BNohiV58swz_rJeHg6rawL3lcfl9JZWvi-2JVZD4woQEZbBjvy5hQrhdh9TJvAy2-JDX6gN1XEGIPIvOIgUmJj125Pw7jACZl3DMmF5bNEBMOgx4bwuvU8u8V4migwnZ4Il5KHux2E8-GtTss9zsXfU1OxKhtbFgl8hNiLMiCJbT4bOP6uoQhid2zc91erTQJFuBTBRCONLyo9yUkuta5rcbpETXhi82rW0zNvkHiRRlfsjlwGYiRDIaeax847zmsV5lQ","e":"AQAB","x5c":["MIIClTCCAX0CBgGMW8pfZzANBgkqhkiG9w0BAQsFADAOMQwwCgYDVQQDDAN1ZHMwHhcNMjMxMjEyMDIwODU4WhcNMzMxMjEyMDIxMDM4WjAOMQwwCgYDVQQDDAN1ZHMwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCtsgpW4Fe3ADbIB0e/5pS5hR/LunS10CTDwmMFevsepFQtGzTo9jt+yvGYcb0/P/EsQtRPy224GWW+mQzQE2iGJXnyzDP+sl4eDqtrAveVx+X0lla+L7YlVkPjChARlsGO/LmFCuF2H1Mm8DLb4kNfqA3VcQYg8i84iBSYmPXbk/DuMAJmXcMyYXls0QEw6DHhvC69Ty7xXiaKDCdngiXkoe7HYTz4a1Oyz3Oxd9TU7EqG1sWCXyE2IsyIIltPhs4/q6hCGJ3bNz3V6tNAkW4FMFEI40vKj3JSS61rmtxukRNeGLzatbTM2+QeJFGV+yOXAZiJEMhp5rHzjvOaxXmVAgMBAAEwDQYJKoZIhvcNAQELBQADggEBABe0judqVmu49JQGXe6RmV4PnQ70myJj3xZtf2ychJLp8wc8/5c1cRs6Jf/4oWtlCpTF6lRYKBCqeVJw4hz/6XDrG+AHMQchDE6w6tB1zb49AEhZLAmgxP0QEVUYszt9qxFX2wXyf4PHj5goGlMMWuqh7CUJe7zrIsZfOefbVIvyMJ8qRFFWH1xYiufxRHYeMsWZlfx1vn9AmpnoPJnbAdx6wPMFrEyNkFKXtCJS8wyXwD9Ec1CwBxV3PyFQL+TtaVOc1pOqC28GuvcktJWFjWmcUW9PiWrn6bdu+x/6W0bLA6GrhiRtWSsdBC8RZ8kyelbv/sTrFz7Lh45M8wKWfSs="],"x5t":"QBPagxCybTiYCVm64nWbJxIyyXg","x5t#S256":"PU_GKReohtxuPQFIzZhls1HcSxRo0ApMUZ7e4eZ7AgM"}]}%
```
---
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

#### trusts google
```bash
k logs -f -l app=authservice
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
                  "jwks_uri": "https://localhost/realms/uds/.well-known/openid-configuration",
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

#### not trusting keycloak
```bash
k logs -l app=authservice -f
[2023-12-12 20:38:39.284] [console] [info] Get: opening connection to 0.0.0.0:443
[2023-12-12 20:38:39.289] [console] [error] Get: unexpected exception: Connection refused [system:111]
[2023-12-12 20:38:39.289] [console] [warning] operator(): HTTP connection error
[2023-12-12 20:38:41.397] [console] [warning] onReadDone: chain:idp_filter_chain JWKS is not ready
```

I believe it is that it does not trust the Keycloak CA, the endpoint is open
### Deploy Bookinfo [not here yet]

```bash
kubectl create -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml
```

```bash
kubectl port-forward service/istio-ingressgateway 3000:443 -n istio-system
```

Visit [https://localhost:8443/productpage](https://localhost:8443/productpage) 
