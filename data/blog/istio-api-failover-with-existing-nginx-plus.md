---
title: 'API Failover Across Independent Istio Clusters with an Existing NGINX Plus Ingress'
date: '2026-11-18'
lastmod: '2026-11-18'
tags: ['Istio', 'Kubernetes', 'NGINX', 'Infra']
draft: false
summary: 'Reuse F5 NGINX Plus Ingress for per-API health checks and primary/backup routing across two independent Istio meshes, with HTTPS configuration and a practical rollout plan.'
images:
  [
    'https://dev.azure.com/zslyvain/pic_blog/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?api-version=7.1&path=%2Fcc869f7e1686d6a573ab799fcb2c7b5191e28b4171c85241ef7138312f84f2bd.png&versionDescriptor.versionType=commit&versionDescriptor.version=ab660d166506057521b41d822ff9ec09947fc11f&download=false&%24format=octetStream',
  ]
layout: PostLayout
---

Running the same API in two Kubernetes clusters gives us a backup deployment. It still leaves an important question: who decides when a browser request should use that backup?

Consider two independent Istio meshes. Each has an Istio ingress gateway and several APIs, while a separate F5 NGINX Plus Ingress Controller serves the frontend. The browser currently calls the API gateway in Cluster A directly.

The solution explored here is to put the **existing NGINX Plus ingress in the API request path**. It probes each API through both gateways, prefers Cluster A, and uses Cluster B when that API's primary becomes unavailable. Istio continues routing requests inside each cluster.

This is a deployment template based on the [original design discussion](https://chatgpt.com/share/6afc7c57-65ec-83ec-b312-573894d4a843) and the official references linked below. It has not been exercised against a live cluster; validate it against your installed controller and CRD versions before rollout.

## Start with the actual request path

Here is the starting point:

| Component   | Cluster A                     | Cluster B                     |
| ----------- | ----------------------------- | ----------------------------- |
| Frontend    | `https://example.a.com/app-a` | `https://example.b.com/app-a` |
| API gateway | `https://example.asm.com`     | `https://example.bsm.com`     |
| App A API   | `/api/app-a`                  | `/api/app-a`                  |
| App B API   | `/api/app-b`                  | `/api/app-b`                  |

Both API applications listen on port `8000`. For this example, their Kubernetes Services also expose port `8000`.

A frontend being served through NGINX does **not** mean its API calls pass through NGINX. When JavaScript calls `https://example.asm.com/api/app-a/users`, the browser opens a separate connection to that domain. A failover rule on the frontend ingress cannot affect that connection.

Change the API base URL to `https://api.example.com`, and point that new name at the existing NGINX Plus ingress:

![Browser requests enter the existing NGINX Plus ingress, which selects a primary or backup Istio gateway across two independent meshes.](https://dev.azure.com/zslyvain/pic_blog/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?api-version=7.1&path=%2Fcc869f7e1686d6a573ab799fcb2c7b5191e28b4171c85241ef7138312f84f2bd.png&versionDescriptor.versionType=commit&versionDescriptor.version=ab660d166506057521b41d822ff9ec09947fc11f&download=false&%24format=octetStream)

_Figure 1. The existing ingress gains an API hostname. The two Istio meshes stay independent._

NGINX connects to the gateways over ordinary HTTPS. There is no special connector between NGINX and Istio, and no cross-cluster service discovery is needed for this design.

## Give each API its own health decision

The target behavior is deliberately specific:

![Per-API failover sends App A to Cluster B during an App A failure in Cluster A, while App B stays on Cluster A; App A returns after recovery.](https://dev.azure.com/zslyvain/pic_blog/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?api-version=7.1&path=%2Faecff06e75d25277adb0b6934ad9c9b60ae1d53f444822fd3903c3e9d45f5592.png&versionDescriptor.versionType=commit&versionDescriptor.version=c8fad6c7daddd3d9c874cdfe260bb2229e9b8be7&download=false&%24format=octetStream)

_Figure 2. An App A failure changes App A's upstream selection while App B stays on Cluster A._

A successful TCP connection to a gateway says very little about an application behind it. Instead, each API should expose a readiness endpoint through its normal gateway route:

```http
GET /api/app-a/health/ready
```

Return `200` when the application can serve its core workload and `503` when it cannot. Avoid returning `200` with a JSON body that merely says `DOWN`. Keep the check inexpensive, include essential dependencies, and exclude optional dependencies whose failure should not trigger a regional move.

NGINX Plus supports periodic HTTP probes and configurable failure/recovery thresholds. Probing the real API path also exercises gateway routing. See the [NGINX HTTP health-check guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-health-check/).

Kubernetes readiness probes remain useful for removing individual unhealthy Pods from their local Service. They do not select another cluster. Also, two clusters depending on the same failed database may both become unready; traffic routing cannot repair that shared dependency.

## Prerequisites and example names

This walkthrough assumes **F5 NGINX Ingress Controller running NGINX Plus**, with custom resources enabled. It does not apply directly to the community `ingress-nginx` controller or an NGINX Open Source deployment.

Use these names consistently, replacing them with your actual resources:

| Setting                                         | Example                                |
| ----------------------------------------------- | -------------------------------------- |
| Kubernetes contexts                             | `cluster-a`, `cluster-b`               |
| NGINX resource namespace                        | `api-routing`                          |
| Existing NGINX controller namespace / ConfigMap | `nginx-ingress` / `nginx-config`       |
| Existing controller Deployment / container      | `nginx-ingress` / `nginx-ingress`      |
| IngressClass                                    | `nginx`                                |
| Istio Gateway                                   | `istio-system/api-gateway`             |
| API Service                                     | `app-a.backend.svc.cluster.local:8000` |

The NGINX Pods must be able to resolve and reach both gateway addresses on TCP `443`. Verify routing, firewalls, NetworkPolicies and any load-balancer source restrictions. A gateway address reachable from your laptop may still be unreachable from those Pods.

Check the installed CRDs before using the examples:

```bash
kubectl --context cluster-a get ingressclass nginx
kubectl --context cluster-a explain virtualserver.spec.upstreams \
  --api-version=k8s.nginx.org/v1
kubectl --context cluster-a explain policy.spec.egressMTLS \
  --api-version=k8s.nginx.org/v1
kubectl --context cluster-a create namespace api-routing
```

If the namespace already exists, skip the last command. Confirm that the existing controller watches it and that no other Ingress or VirtualServer owns `api.example.com` on the same controller.

## 1. Align the gateway address, TLS SNI and HTTP Host

There are three separate pieces of an upstream HTTPS connection:

![NGINX resolves a cluster-specific gateway address, then uses api.example.com for TLS SNI, certificate verification and the HTTP Host header.](https://dev.azure.com/zslyvain/pic_blog/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?api-version=7.1&path=%2F35a5fd7a02af132cdf3303fd2fa943ad8ba8dc7a540d622450df2b8d508d5680.png&versionDescriptor.versionType=commit&versionDescriptor.version=1aec72484cf6e1a3b284391032f4c636e5af14e6&download=false&%24format=octetStream)

_Figure 3. Resolve a cluster-specific address, but use a common TLS and HTTP identity._

For example, NGINX resolves `example.bsm.com` to reach Gateway B, while sending `api.example.com` as both TLS SNI and HTTP Host. Gateway B's certificate must cover `api.example.com`, and its routing configuration must accept that host.

Keep `example.asm.com` and `example.bsm.com` pointing directly at their respective Istio gateways. They are the upstream addresses. If either resolves back to the NGINX API frontend, the proxy can call itself in a loop.

In each cluster, add a dedicated HTTPS server for the shared hostname to the **existing** Gateway. The following is an additional `spec.servers` entry, not a replacement for its current servers:

```yaml
- port:
    number: 443
    name: https-api-shared
    protocol: HTTPS
  hosts:
    - api.example.com
  tls:
    mode: SIMPLE
    credentialName: api-shared-tls
```

Provision `api-shared-tls` using your normal certificate process in the namespace where the selected gateway workload loads credentials, typically `istio-system`. Each gateway needs a valid certificate for the shared name; the certificates need not have identical private keys. Keep the existing cluster-specific hosts and certificates intact. The [Istio Gateway reference](https://istio.io/latest/docs/reference/config/networking/gateway/) describes server, host and TLS configuration.

Update App A's existing VirtualService in Cluster A to accept the new hostname. The target shape is:

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: app-a
  namespace: backend
spec:
  hosts:
    - example.asm.com
    - api.example.com
  gateways:
    - istio-system/api-gateway
  http:
    - match:
        - uri:
            exact: /api/app-a
        - uri:
            prefix: /api/app-a/
      route:
        - destination:
            host: app-a.backend.svc.cluster.local
            port:
              number: 8000
```

In Cluster B, use `example.bsm.com` in place of `example.asm.com`. Merge these changes into existing resources, retaining their authentication, CORS, rewrites and other routes. If the Kubernetes Service exposes `80` with `targetPort: 8000`, use **80** in the destination. See the [Istio VirtualService reference](https://istio.io/latest/docs/reference/config/networking/virtual-service/).

## 2. Name the two external gateways

Save the following as `external-services.yaml` in your deployment workspace:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-primary
  namespace: api-routing
spec:
  type: ExternalName
  externalName: example.asm.com
  ports:
    - name: https
      port: 443
---
apiVersion: v1
kind: Service
metadata:
  name: istio-backup
  namespace: api-routing
spec:
  type: ExternalName
  externalName: example.bsm.com
  ports:
    - name: https
      port: 443
```

These objects create no proxy Pods. They give the controller names for the two external destinations. Every API can reuse these Services while maintaining its own upstream health state.

ExternalName resolution also requires a resolver in the controller's active ConfigMap. Merge these keys into the existing ConfigMap, preserving its other settings:

```yaml
data:
  resolver-addresses: '10.96.0.10'
  resolver-valid: '10s'
  resolver-timeout: '5s'
```

`10.96.0.10` is illustrative. Use the resolver reachable from your NGINX Pods and capable of resolving both gateway names. Confirm the active ConfigMap from the controller's deployment arguments rather than assuming its name. Refer to the [F5 ConfigMap reference](https://docs.nginx.com/nginx-ingress-controller/configuration/global-configuration/configmap-resource/).

## 3. Verify upstream certificates

There are two TLS hops: browser → NGINX, and NGINX → Istio. Provision the browser-facing certificate as `api-public-tls` in `api-routing` through your usual certificate management process.

For the second hop, put the CA bundle that trusts both gateway certificate chains into a Secret:

```bash
kubectl --context cluster-a -n api-routing \
  create secret generic istio-upstream-ca \
  --from-file=ca.crt=./trusted-ca.pem
```

Save this as `upstream-tls-policy.yaml`:

```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: istio-upstream-tls
  namespace: api-routing
spec:
  egressMTLS:
    trustedCertSecret: istio-upstream-ca
    verifyServer: true
    verifyDepth: 2
    serverName: true
    sslName: api.example.com
```

Despite the policy's name, this example uses server-authenticated TLS: no client certificate is configured. Set `verifyDepth` for your actual certificate chains. The policy enables verification and SNI using the shared hostname. See the [F5 EgressMTLS reference](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/policy-reference/#egressmtls).

## 4. Configure App A's primary and backup

Save this as `virtualserver.yaml`:

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: api-failover
  namespace: api-routing
spec:
  ingressClassName: nginx
  host: api.example.com
  tls:
    secret: api-public-tls
    redirect:
      enable: true
  policies:
    - name: istio-upstream-tls
  upstreams:
    - name: app-a
      service: istio-primary
      port: 443
      backup: istio-backup
      backupPort: 443
      lb-method: round_robin
      tls:
        enable: true
      connect-timeout: 3s
      read-timeout: 30s
      send-timeout: 30s
      next-upstream: 'off'
      healthCheck:
        enable: true
        path: /api/app-a/health/ready
        interval: 5s
        fails: 3
        passes: 3
        mandatory: true
        connect-timeout: 2s
        read-timeout: 2s
        send-timeout: 2s
        statusMatch: '200'
        headers:
          - name: Host
            value: api.example.com
  routes:
    - path: '~ ^/api/app-a(/|$)'
      action:
        proxy:
          upstream: app-a
          requestHeaders:
            set:
              - name: Host
                value: api.example.com
```

The backup is an ExternalName Service, and `round_robin` is compatible with backup selection. The route boundary excludes names such as `/api/app-admin`; no URI rewrite is configured. Probes require exactly `200`, and new upstream peers must pass checks before receiving traffic. Field definitions are in the [F5 VirtualServer reference](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/).

**Failover and retry are separate decisions.** This example explicitly disables retrying a failed request with `next-upstream: "off"`. Once health selection excludes A, subsequent requests can use B, but a request already sent to A can fail. This conservative starting point avoids proxy replays of ambiguous writes. Review retry behavior separately for your API; the [NGINX proxy retry documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream) explains its conditions and limitations.

To add App B, create another upstream named `app-b`, reuse the same two Services, set its probe to `/api/app-b/health/ready`, and add its own boundary-matched route. Sharing gateway addresses does not require sharing the upstream group.

## 5. Validate before changing DNS

Apply the Istio changes in both clusters first, then validate and apply the NGINX resources in Cluster A:

```bash
kubectl --context cluster-a apply --dry-run=server -f external-services.yaml
kubectl --context cluster-a apply --dry-run=server -f upstream-tls-policy.yaml
kubectl --context cluster-a apply --dry-run=server -f virtualserver.yaml

kubectl --context cluster-a apply -f external-services.yaml
kubectl --context cluster-a apply -f upstream-tls-policy.yaml
kubectl --context cluster-a apply -f virtualserver.yaml

kubectl --context cluster-a -n api-routing \
  get virtualserver api-failover -o yaml
kubectl --context cluster-a -n api-routing \
  describe virtualserver api-failover
```

Schema acceptance alone does not prove the controller accepted the configuration. Check status and events for the VirtualServer and Policy, then inspect the generated configuration in each controller Pod. For example, using the example Deployment and container names:

```bash
kubectl --context cluster-a -n nginx-ingress \
  exec deployment/nginx-ingress -c nginx-ingress -- nginx -T
```

Confirm that the upstream contains the primary and `backup` peers, that active probes cover both, and that the resulting locations include HTTPS, SNI, CA verification and the intended Host header. Check that the generated probe location also inherits the required TLS settings. Treat any mismatch as a rollout blocker.

Before publishing DNS, test against the ingress address directly:

```bash
# Replace the documentation address with the actual NGINX ingress IP.
NGINX_INGRESS_IP=203.0.113.10
curl --fail-with-body --resolve "api.example.com:443:${NGINX_INGRESS_IP}" \
  https://api.example.com/api/app-a/health/ready
```

Do not use `-k`: certificate verification is part of this test. After this works, point `api.example.com` at the existing ingress and update **both frontend deployments** to use it:

```javascript
const API_BASE_URL = 'https://api.example.com'
fetch(`${API_BASE_URL}/api/app-a/users`)
```

Because the frontend and API still have different origins, retain CORS support for `https://example.a.com` and `https://example.b.com`. Verify preflight requests, authorization headers, cookies and redirects. Assign CORS to one layer to avoid conflicting headers. Check that the new path still enforces the intended authentication and authorization rules.

## 6. Run a controlled failure drill

Use a staging environment and an application response header such as `X-Served-Cluster: A` or `B` to identify who actually handled each request. A `200` response alone cannot prove failover.

| Test                                                   | Expected observation                                     |
| ------------------------------------------------------ | -------------------------------------------------------- |
| Both App A deployments healthy                         | Requests prefer A                                        |
| App A becomes unready in A                             | New App A requests use B after detection                 |
| App B stays healthy in A                               | App B continues using A                                  |
| App A in A recovers                                    | New requests prefer A after recovery checks              |
| Gateway A becomes unreachable; NGINX remains reachable | APIs with healthy backups use B                          |
| App A fails in both clusters                           | App A returns an error; it does not route to another app |
| Login, OPTIONS and a write during transition           | Auth works; writes are not silently duplicated           |

With five-second probes and three failures, allow several probe cycles for detection. Timeouts, scheduling and passive failure handling affect observed timing; this is not a strict fifteen-second availability guarantee. Existing streams are not migrated, and failback applies to subsequent upstream selection.

Prefer a controlled readiness failure over blindly scaling production Deployments. Record the original configuration and coordinate any drill with HPA and GitOps reconciliation.

## Where this solution stops

If the only NGINX ingress serving `api.example.com` is inside Cluster A, a complete Cluster A outage removes the component that makes the failover decision. A healthy API in B cannot help a browser reach an unavailable ingress.

For cluster-wide resilience, configure the existing NGINX Plus controllers in **both** clusters to serve the API hostname and use an existing global load balancer or health-aware DNS service to select a reachable ingress. Browser and resolver caching still affect DNS-based recovery. Merely publishing two ingress addresses does not establish health-aware entry-point failover.

Application recovery also depends on data: shared or replicated state, compatible token validation, session availability, file storage and sufficient standby capacity. A readiness check should reflect whether B can safely handle its workload, including writes.

The useful boundary is clear: **NGINX Plus chooses a healthy cluster for each API; Istio routes to the application inside that cluster.** Reusing the existing ingress gives us a small, testable change to the request path, while leaving entry-point resilience and data recovery as explicit parts of the wider design.
