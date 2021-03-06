+++
title = "Using Ingress"
description = "Linkerd works alongside your ingress controller of choice."
+++

If you're planning on injecting Linkerd into your ingress controller's pods
there is some configuration required. Linkerd discovers services based on the
`:authority` or `Host` header. This allows Linkerd to understand what service a
request is destined for without being dependent on DNS or IPs.

When it comes to ingress, most controllers do not rewrite the
incoming header (`example.com`) to the internal service name
(`example.default.svc.cluster.local`) by default. In this example, when Linkerd
receives the outgoing request it thinks the request is destined for
`example.com` and not `example.default.svc.cluster.local`. This creates an
infinite loop that can be pretty frustrating!

Luckily, many ingress controllers allow you to either modify the `Host` header
or add a custom header to the outgoing request. Here are some instructions
for common ingress controllers:

{{% pagetoc %}}

Note: If your ingress controller is terminating HTTPS, Linkerd will only provide
TCP stats for the incoming requests because all the proxy sees is encrypted
traffic. It will provide complete stats for the outgoing requests from your
controller to the backend services as this is in plain text from the
controller to Linkerd.

## Nginx

This uses `emojivoto` as an example, take a look at
[getting started](/2/getting-started/) for a refresher on how to install it.

The sample ingress definition is:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/upstream-vhost: $service_name.$namespace.svc.cluster.local
spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: web-svc
          servicePort: 80
```

The important annotation here is:

```yaml
nginx.ingress.kubernetes.io/upstream-vhost: $service_name.$namespace.svc.cluster.local
```

This will rewrite the `Host` header to be the fully qualified service name
inside your Kubernetes cluster.

To test this, you'll want to get the external IP address for your controller. If
you installed nginx-ingress via helm, you can get that IP address by running:

```bash
kubectl get svc --all-namespaces \
  -l app=nginx-ingress,component=controller \
  -o=custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].ip
```

You can then use this IP with curl:

```bash
curl -H "Host: example.com" http://external-ip
```

{{< note >}}
It is not possible to rewrite the header in this way for the default
backend. Because of this, if you inject Linkerd into your Nginx ingress
controller's pod, the default backend will not be usable.
{{< /note >}}

## Traefik

This uses `emojivoto` as an example, take a look at
[getting started](/2/getting-started/) for a refresher on how to install it.

The sample ingress definition is:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  namespace: emojivoto
  annotations:
    kubernetes.io/ingress.class: "traefik"
    ingress.kubernetes.io/custom-request-headers: l5d-dst-override:web-svc.emojivoto.svc.cluster.local
spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: web-svc
          servicePort: 80
```

The important annotation here is:

```yaml
ingress.kubernetes.io/custom-request-headers: l5d-dst-override:web-svc.emojivoto.svc.cluster.local
```

Traefik will add a `l5d-dst-override` header to instruct Linkerd what service
the request is destined for.

To test this, you'll want to get the external IP address for your controller. If
you installed Traefik via helm, you can get that IP address by running:

```bash
kubectl get svc --all-namespaces \
  -l app=traefik \
  -o='custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].ip'
```

You can then use this IP with curl:

```bash
curl -H "Host: example.com" http://external-ip
```

{{< note >}}
This solution won't work if you're using Traefik's service weights as
Linkerd will always send requests to the service name in `l5d-dst-override`. A
workaround is to use `traefik.frontend.passHostHeader: "false"` instead. Be
aware that if you're using TLS, the connection between Traefik and the backend
service will not be encrypted. There is an
[open issue](https://github.com/linkerd/linkerd2/issues/2270) to track the
solution to this problem.
{{< /note >}}

## Ambassador

This uses `emojivoto` as an example, take a look at
[getting started](/2/getting-started/) for a refresher on how to install it.

Ambassador does not use `Ingress` resources, instead relying on `Service`. The
sample service definition is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-ambassador
  namespace: emojivoto
  annotations:
    getambassador.io/config: |
      ---
      apiVersion: ambassador/v1
      kind: Mapping
      name: web-ambassador-mapping
      service: web-ambassador.emojivoto.svc.cluster.local
      host: example.com
      prefix: /
      add_request_headers:
        l5d-dst-override: web-ambassador.emojivoto.svc.cluster.local
spec:
  selector:
    app: web-svc
  ports:
  - name: http
    port: 80
    targetPort: http
```

The important annotation here is:

```yaml
      add_request_headers:
        l5d-dst-override: web-other.emojivoto.svc.cluster.local
```

Ambassador will add a `l5d-dst-override` header to instruct Linkerd what service
the request is destined for.

To test this, you'll want to get the external IP address for your controller. If
you installed Ambassador via helm, you can get that IP address by running:

```bash
kubectl get svc --all-namespaces \
  -l "app.kubernetes.io/name=ambassador" \
  -o='custom-columns=EXTERNAL-IP:.status.loadBalancer.ingress[0].ip'
```

{{< note >}}
If you've installed the admin interface, this will return two IPs, one of which
will be `<none>`. Just ignore that one and use the actual IP address.
{{</ note >}}

You can then use this IP with curl:

```bash
curl -H "Host: example.com" http://external-ip
```
