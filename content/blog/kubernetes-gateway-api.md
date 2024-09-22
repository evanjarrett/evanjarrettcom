---
title: "Kubernetes Gateway Api"
date: 2023-08-14T00:01:09-05:00
draft: false
author: Evan Jarrett
tags:
 - kubernetes
 - devops
image:
description:
toc:
---

With my latest iteration of my homelab I created a Kubernetes cluster on some Raspberry Pis. In researching various ways to handle traffic coming into the cluster, I stumbled upon the new Kubernetes Gateway API and the NKG [(nginx-kubernetes-gateway)](https://github.com/nginxinc/nginx-kubernetes-gateway) project that implements it. In the past, I have used something like traefik as an ingress, but that was too confusing for me to implement. I wanted something simple and customizable. I didn't feel like messing with configmaps, or having to modify dozens of variables in a helm chart in order to set up my routing. I wanted a "k8s way" of dealing with it through abstraction.

For my use cases I needed a few things for my cluster to achieve. 
- Load balance traffic using a virtual IP across the cluster (MetalLB).
- Automatically generate Let's Encrypt certificates for my domains (cert-manager).
- Proxy traffic to services outside my cluster (services and endpoints).
- Support websockets for my Home Assistant instance.

For the purposes of this article, I'm going to skip some additional details, like setting up a metallb as a load balancer or cert-manager to manage let's encrypt certificates. This is because depending on where you host, you may have a different load balancer. And frankly, cert-manager is very simple to configure with a single annotation in our Gateway manifest.

The [kubernetes gateway api](https://gateway-api.sigs.k8s.io/) is early in its development, but is designed to simplify the complexity of load balancers and ingresses and services. With this API, you only need to manage two Custom Resource Definitions (CRDs): Gateways and HTTPRoutes.Technically there are more route types for TLS, TCP, UDP etc, but for the current implementation of NKG, there is only HTTPRoutes.

To begin, install NKG. I used Helm for this:

`helm install nkg oci://ghcr.io/nginxinc/charts/nginx-kubernetes-gateway --create-namespace --wait -n nginx-gateway --version 0.0.0-edge`

Give it a minute to begin deploying and then check to make sure it received an external IP from your loadbalancer
```
k -n nginx-gateway get svc
NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
nkg-nginx-kubernetes-gateway   LoadBalancer   10.43.224.206   192.168.3.1   80:30768/TCP,443:31470/TCP   6d2h
```

Now we need to define our Gateway manifest:

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: jarrett-net
  namespace: nginx-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
    - name: jarrett-net
      hostname: "*.jarrett.net"
      port: 443nginx-kubernetes-gateway
      protocol: HTTPS
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - name: jarrett-net-tls
```

There are a couple of key points to note here. First, gatewayClassName: nginx corresponds to the name provided by NKG upon installation:

```
k get gatewayclass
NAME    CONTROLLER                                       ACCEPTED   AGE
nginx   k8s-gateway.nginx.org/nginx-gateway-controller   True       6d6h
```

Next, for cert-manager, we employ the annotation for the cluster-issuer to facilitate certificate creation. This is then referenced in our Gateway through the jarrett-net-tls name.
Lastly, the example establishes a wildcard entry for *.jarrett.net. Note that this doesn't match jarrett.net, which would be handled in a separate listener.

With that addressed, we can proceed to develop our application and define HTTPRoutes like the following:

```
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: whoami-tls-route
spec:
  parentRefs:
  - name: jarrett-net
    namespace: nginx-gateway
  hostnames:
  - "whoami.jarrett.net"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: whoami
      namespace: whoami
      port: 80
```

The rules section of this manifest provides considerable flexibility, although for now, we're simply instructing it to match any path starting with '/'.
Additionally, we can match based on headers, methods, and query parameters. However, I'll delve into that in another article.
The backendRef points to our Service object, and from there, we can define our Service and Deployments according to the needs of our application.
What's remarkable about the Gateway API is that each application developer can control their own routing within their namespace. There isn't a need to touch the Gateway unless there isn't a listener that is supported. For instance, when adding additional domains. 

There was one slight problem though, and something I eluded to above. I still needed a way to handle websockets. Unfortunetly as of NKG v0.5.0, and the time of writing article, websockets aren't supported.

Beyond the realm of routing based on attributes such as paths and headers, the HTTPRoute mechanism also introduces filters for refining requests and altering headers. To illustrate, consider the example of proxying websocket connections: by employing filters, you can dynamically append headers to requests, akin to the following:

```
filters:
- type: RequestHeaderModifier
  requestHeaderModifier:
    add:
      - name: Upgrade
        value: websocket
```

This translates to the nginx configuration as:

`proxy_set_header Upgrade "websocket";`

But with nginx, you would normally write the following to handle websockets.
```
proxy_set_header Upgrade $http_upgrade
proxy_set_header Connection "upgrade"
```

You see, the problem is that `$http_upgrade` variable. There is not a way to write that in the CRD. So I needed my own solution.
Under the hood NKG generates standard nginx config files from Go templates based on the Gateways and HTTPRoutes. With a little change to that template I am able to modify the code to create the appropriate headers I need to proxy websockets to Home Assistant.

```
+++ b/internal/mode/static/nginx/config/servers_template.go
@@ -48,7 +48,12 @@ server {
 
         {{- if $l.ProxyPass -}}
             {{ range $h := $l.ProxySetHeaders }}
+                {{- if eq $h.Name "Upgrade" -}}
+        proxy_set_header Upgrade $http_upgrade;
+        proxy_set_header Connection "upgrade";
+                {{- else }}
         proxy_set_header {{ $h.Name }} "{{ $h.Value }}";
+                {{- end }}
             {{- end }}
         proxy_set_header Host $gw_api_compliant_host;
         proxy_pass {{ $l.ProxyPass }}$request_uri;
```


And with that complete, I was able to fulfill all my current use cases.
I'm excited for the futre of nginx-kubernetes-gateway and the Gateway API in general. I think it solves a need to simplyfy networking and ingress for kubernetes in a way that is flexible and easy to understand.
