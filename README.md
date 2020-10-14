## Ingress + Proxy Protocol Demo with In-Cluster Services on Linode

1. Create a Kubernetes cluster with Linode Kubernetes Engine and download the Kubeconfig

https://cloud.linode.com/kubernetes/create

2. Install ingress-nginx on your cluster via Helm

```bash
$ export KUBECONFIG=my-cluster-kubeconfig.yaml
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm install --generate-name ingress-nginx/ingress-nginx
```

3. Add an A record for our demo using your DNS provider.

```text
type  name       data         ttl
A     mybackend  <public-ip>  300 sec
```

* `<public-ip>` comes from the `EXTERNAL-IP` for ingress-nginx seen with `kubectl get services -A`

```bash
$ kubectl get services -A
NAMESPACE     NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default       ingress-nginx-1602681250-controller             LoadBalancer   10.128.15.49     <public-ip>   80:30003/TCP,443:30843/TCP   11m
...
```

* `mybackend` is a name that you choose for the service
* Choose a short TTL, so that we can interact with this domain in a few minutes


This will soon allow us to reach our service at mybackend.mydomain.com

4. Deploy a backend Service with an Ingress resource

We will use `httpbin` as a demo backend service which by default echoes to us our client IP address when we make a request.

Your edits at this step:

* Set the value of `host:` in the Ingress resource to the domain name that you chose above
* No other edits should be made to this manifest

```yaml
# mybackend.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: mybackend-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
    - host: mybackend.mydomain.com
      http:
        paths:
          - path: /
            backend:
              serviceName: mybackend-service
              servicePort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: mybackend-service
spec:
  selector:
    app: httpbin
  ports:
    - protocol: TCP
      port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mybackend-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin
          ports:
            - containerPort: 80
```

``` bash
$ kubectl apply -f mybackend.yaml
```

Hit http://mybackend.mydomain.com/get in a local web browser.

This is an httpbin endpoint which displays request information, including the client IP and all HTTP headers. Note that the client IP, called "origin", will either be in the Kubernetes Pod network of 10.2.0.0/16, or will be an IP address of one of the Nodes in your cluster. This is the IP of ingress-nginx using the Kubernetes Service network to reach the backend.

5. Enable Proxy Protocol for ingress-nginx

We now want our backend service to see the true client IP, so we enable Proxy Protocol for ingress-nginx.

Add the Proxy Protocol annotation to the ingress-nginx service. The name of the service is obtained with `kubectl get services -A`.

```bash
$ kubectl edit service ingress-nginx-1602681250-controller
  # Add the Proxy Protocol annotation to the annotations section
  annotations:
    ...
    service.beta.kubernetes.io/linode-loadbalancer-proxy-protocol: v2
```

Configure ingress-nginx to expect Proxy Protocol data. The name of this configmap can be found with `kubectl get configmaps -A`.

```bash
$ kubectl edit configmap ingress-nginx-1602681250-controller
# Add the data section at the root level of this yaml, which might not yet exist
data:
  use-proxy-protocol: "true"
```

Hit http://mybackend.mydomain.com/get again in a web browser.

You will see your own machine's IP in the "origin" section of httpbin's info. Proxy Protocol successfully enabled!

```
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", 
    "Accept-Encoding": "gzip, deflate", 
    "Accept-Language": "en-US,en;q=0.9,ja;q=0.8", 
    "Host": "mybackend.mydomain.com", 
    "Upgrade-Insecure-Requests": "1", 
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.75 Safari/537.36", 
    "X-Forwarded-Host": "mybackend.mydomain.com", 
    "X-Scheme": "http"
  }, 
  "origin": "68.68.68.68", 
  "url": "http://mybackend.mydomain.com/get"
}
```
6. Note that at this point in-cluster clients cannot reach the service via the public hostname.

```
$ kubectl run myshell --image busybox --command sleep 10000
$ kubectl exec -ti myshell -- sh
/ # wget http://mybackend.mydomain.com/get
Connecting to mybackend.mydomain.com (<public-ip>:80)
wget: error getting response: Resource temporarily unavailable
/ # exit
$ kubectl logs ingress-nginx-1602681250-controller-8df8684fc-5xbmd
...
2020/10/14 14:14:37 [error] 187#187: *11787 broken header:
...
```

The name of the ingress-nginx pod can be found with `kubectl get pods -A`.

These `broken header: ` messages indicate that the in-cluster client is not sending Proxy Protocol data to ingress-nginx, which now expects it.

7. Fix this problem by using the service hostname instead of the public hostname.

Instead of using the Ingress hostname, clients which are inside your cluster should use the Service hostname for the backend service. This bypasses ingress-nginx entirely, avoiding the need for the client to send Proxy Protocol headers.

```
$ kubectl run myshell --image busybox --command sleep 10000
$ kubectl exec -ti myshell -- sh
/ # wget http://mybackend-service.default.svc.cluster.local/get
/ # cat get
{
  "args": {},
  "headers": {
    "Connection": "close",
    "Host": "mybackend-service.default.svc.cluster.local",
    "User-Agent": "Wget"
  },
  "origin": "10.2.1.9",
  "url": "http://mybackend-service.default.svc.cluster.local/get"
}
```

You will see that in this case the request succeeded and that your "origin" is your Pod IP (the Pod IP of this busybox Pod) inside the cluster.

Here `svc.cluster.local` is the default domain name for the Service network in Kubernetes, you should be able to use that portion of the hostname without any modifications. Similarly, `default` is the name of the `default` namespace in Kubernetes, if your backend resides in a different namespace, then you can substitute that namespace name in this URL. The DNS records for this hostname are queried automatically via CoreDNS running in this cluster.

At this point you are able to reach the backend from both public and private clients:

* Publicly via http://mybackend.mydomain.com/
* Inside the cluster via http://mybackend-service.mybackend-namespace.svc.cluster.local/

Service hostnames should always be the preferred way to reach Services inside a Kubernetes cluster. This is the mechanism for service discovery in Kubernetes.
