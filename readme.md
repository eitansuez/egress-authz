# Egress with external authorization

## Preface

The following instructions are a variation of the [Egress Gateways with TLS Origination](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway-tls-origination/) task from the Istio documentation.

We configure calls to an external service, api.github.com, via an egress gateway.  Traffic from the workload to the gateway will use Istio Mutual TLS, and the egress gateway will perform TLS origination to the target service/endpoint.

Once routing is configured, we then use an [external authorizer](https://istio.io/latest/docs/tasks/security/authorization/authz-custom/) to control access to external services.

## Prerequisites

### A Kubernetes cluster

For example, here is a command that provisions a local k8s cluster with [k3d](https://k3d.io/v5.6.3/):

```shell
k3d cluster create my-istio-cluster \
  --api-port 6443 \
  --k3s-arg "--disable=traefik@server:0" \
  --port 80:80@loadbalancer \
  --port 443:443@loadbalancer
```

Verify with:

```shell
kubectl config get-contexts
```

### Istio

Download a copy of the Istio distribution (version 1.22.1) with:

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.22.1 sh -
```

Capture the path to the Istio distribution directory:

```shell
export ISTIO_DIR="$PWD/istio-1.22.1"
```

Put the istio CLI, `istioctl` in your PATH environment variable.

Deploy Istio with:

```shell
istioctl install --skip-confirmation
```

Verify with:

```shell
istioctl version
```

## Deploy an egress gateway

```shell
istioctl install -f egress-install-manifest.yaml --skip-confirmation
```

Verify that an egress gateway pod is running in `istio-system` with:

```shell
kubectl get pod -n istio-system
```

## Configure telemetry

The following command will ensure that envoy logs get sent to stdout:

```shell
kubectl apply -f telemetry.yaml
```

## Define a service entry

Enable both HTTP and HTTPS access to the target service:

```shell
kubectl apply -f service-entry.yaml
```

## Deploy a mesh workload

Label the `default` namespace for sidecar injection:

```shell
kubectl label ns default istio-injection=enabled
```

Deploy the `sleep` sample:

```shell
kubectl apply -f $ISTIO_DIR/samples/sleep/sleep.yaml
```

Verify that the `sleep` pod has two containers:

```shell
kubectl get pod
```

Send a test request to the external service:

```shell
kubectl exec deploy/sleep -- curl -s https://api.github.com/users/kennethreitz
```

## Configure the gateway

Configure the gateway to receive Istio Mutual TLS requests from inside the mesh on port 80:

```shell
kubectl apply -f egress-gateway.yaml
```

## Configure traffic policy to the gateway

Configure mesh clients to call out to the external service through the egress gateway to use mutual TLS:

```shell
kubectl apply -f dr-internal.yaml
```

## Configure routing

The first `match` clause is for traffic to the egress gateway.  The second one is for the TLS origination to the target service:

```shell
kubectl apply -f egress-routes.yaml
```

## Configure traffic policy from the gateway

From the gateway, the call to the external service is with simple TLS:

```shell
kubectl apply -f dr-external.yaml
```

## Verify calls route through the egress gateway

At this point we should be able to confirm that calls from `sleep` to `api.github.com` are routed through the egress gateway and succeed.

In one terminal, tail the logs of the egress gateway:

```shell
kubectl logs --follow -n istio-system -l istio=egressgateway
```

In another shell, send a call to `api.github.com`:

```shell
kubectl exec deploy/sleep -- curl -s http://api.github.com/users/kennethreitz
```

In the logs, you should see evidence

```console
[2024-06-27T15:55:43.880Z] "GET /users/kennethreitz HTTP/2" 200 - via_upstream - "-" 0 1518 39 39 "10.42.0.9" "curl/8.8.0" "9062d7c1-6896-4ce0-9082-2c52aeeb2f6f" "api.github.com" "140.82.113.5:443" outbound|443||api.github.com 10.42.0.10:46398 10.42.0.10:8080 10.42.0.9:59654 api.github.com -
```

## Deploy an external Authorizer

Here we use the sample external authorizer published as a sample with the Istio distribution:

```shell
kubectl apply -f "$ISTIO_DIR/samples/extauthz/ext-authz.yaml"
```

It allows calls containing the header `x-ext-authz` with a value of `allow`, and otherwise denies requests.

Verify that a pod for the `ext-authz` deployment is running in the `default` namespace:

```shell
kubectl get pod
```

## Configure Istio with the ext-authz extension provider

Edit the configmap `istio`:

```shell
kubectl edit configmap istio -n istio-system
```

Add the `envoyExtAuthzHttp` section, as shown below:

```yaml
data:
  mesh: |-
    # Add the following content to define the external authorizers.
    extensionProviders:
    - name: sample-ext-authz-http
      envoyExtAuthzHttp:
        service: ext-authz.default.svc.cluster.local
        port: 8000
        includeRequestHeadersInCheck:
        - x-ext-authz
```

Save the modified configmap and exit the editor.

### Notes

When configuring the [External Authorization Provider](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-EnvoyExternalAuthorizationHttpProvider), note the field `headersToUpstreamOnAllow` can be used to configure sending extra headers to the upstream service.


## Configure external authorization

We want the external authorizer to govern whether calls to api.github.com are allowed.

Apply the below resource, which associates the authorizer you deployed with the egress gateway through a workload selector:

```shell
kubectl apply -f authz-external.yaml
```

## Test an unauthorized call

In one terminal, tail the logs of the authorizer:

```shell
kubectl logs --follow deploy/ext-authz
```

In a separate terminal, make an outbound call from `sleep`:

```shell
kubectl exec deploy/sleep -- curl -s http://api.github.com/users/kennethreitz
```

The response indicates that the request was denied.

Also, in the logs you will see evidence of a denied request:

```console
2024/06/27 16:10:15 [HTTP][denied]: GET api.github.com/users/kennethreitz, headers: map[Content-Length:[0] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/default/sa/default;Hash=8648d3c79c9f71355a952e9517591aa10ddd83ecb6834a927e780478a4acd400;Subject="";URI=spiffe://cluster.local/ns/istio-system/sa/istio-egressgateway-service-account] X-Forwarded-Proto:[https] X-Request-Id:[72bd1983-fe52-415b-a3de-5576f0720903]], body: []
```

## Test an authorized call

On the other hand, the following call will succeed:

```shell
kubectl exec deploy/sleep -- curl -s -H "x-ext-authz: allow" http://api.github.com/users/kennethreitz
```

The authorizer logs should also indicate that the request was allowed:

```console
2024/06/27 16:11:40 [HTTP][allowed]: GET api.github.com/users/kennethreitz, headers: map[Content-Length:[0] X-Ext-Authz:[allow] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/default/sa/default;Hash=8648d3c79c9f71355a952e9517591aa10ddd83ecb6834a927e780478a4acd400;Subject="";URI=spiffe://cluster.local/ns/istio-system/sa/istio-egressgateway-service-account] X-Forwarded-Proto:[https] X-Request-Id:[e5c6079d-2165-46ca-b482-fa42fbcdb4fd]], body: []
```

