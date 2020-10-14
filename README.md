## Ingress + Proxy Protocol Demo with In-Cluster Services on Linode

1. Create a Kubernetes cluster with Linode Kubernetes Engine and downlod the Kubeconfig

https://cloud.linode.com/kubernetes/create

2. Install ingress-nginx on your cluster via Helm

```bash
$ export KUBECONFIG=my-cluster-kubeconfig.yaml
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm install --generate-name ingress-nginx/ingress-nginx
```

3. Point a domain name at your ingress-nginx NodeBalancer

```bash
$ kubectl get services -A
NAMESPACE     NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default       ingress-nginx-1602681250-controller             LoadBalancer   10.128.15.49     <public-ip>   80:30003/TCP,443:30843/TCP   11m
...
```

Add an A record using your DNS provider for the service.

Your edits at this step:

* `<public-ip>` comes from the `EXTERNAL-IP` above
* `mybackend` is a name that you choose for the service
* Choose a short TTL, so that we can interact with this domain in a few minutes

```text
type  name        data         ttl
A     mybackend   <public-ip>  300 sec
```

This will soon allow us to reach our service at mybackend.mydomain.com

4. Deploy a backend Service with an Ingress resource

The following is an example backend service manifest using an ingress for that domain name. We will use `httpbin` which can echo to us our client IP address when we make a request.

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

This is an httpbin endpoint which displays request information, including the client IP and all HTTP headers. Note that the client IP, called "origin", will be in the Kubernetes Pod network of 10.2.0.0/16. This is the address of ingress-nginx reaching the backend.

5. Enable Proxy Protocol for ingress-nginx

We now want our backend service to see the true client IP, so we enable Proxy Protocol for ingress-nginx.

Add the Proxy Protocol annotation

```bash
$ k edit service ingress-nginx-1602681250-controller
  # Add the Proxy Protocol annotation to the annotations section
  annotations:
    ...
    service.beta.kubernetes.io/linode-loadbalancer-proxy-protocol: v2
```

Configure ingress-nginx to expect Proxy Protocol data

```bash
$ k edit configmap ingress-nginx-1602681250-controller
# Add the data section at the root level of this yaml, which might not yet exist
data:
  use-proxy-protocol: "true"
```

Hit http://mybackend.mydomain.com/get again in a web browser.

You will see your own machine's IP in the "origin" section of httpbin's info. Proxy Protocol successfully enabled!

6. Note that at this point in-cluster clients cannot reach the service via the public hostname.

```bash
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

These broken header messages indicate that the in-cluster client is not sending Proxy Protocol data to ingress-nginx, which now exepcts it.

7. Fix this problem by using the service hostname instead of the public hostname.

Instead of using the Ingress hostname, clients which are inside your cluster should use the Service hostname for the backend service. This bypasses ingress-nginx entirely, avoiding the need for the client to send Proxy Protocol headers.

```bash
$ kubectl delete pod myshell
$ kubectl run myshell --image busybox --command sleep 10000
$ kubectl exec -ti myshell -- sh
/ # wget http://mybackend-service.default.svc.cluster.local/get
/ # cat get
# You will see that request succeeded and that your "origin" is your Pod IP
# inside the cluster!
```

Here `svc.cluster.local` is the default domain name for the Service network in Kubernetes, you should be able to use that portion of the hostname without any modifications. Similarly, `default` is the name of the `default` namespace in Kubernetes, if your backend resides in a different namespace, then you can substitute that namespace name in this URL.

At this point you are able to reach the backend from both public and private clients:

* Publicly via http://mybackend.mydomain.com/
* Inside the cluster via http://mybackend-service.mybackend-namespace.svc.cluster.local/
