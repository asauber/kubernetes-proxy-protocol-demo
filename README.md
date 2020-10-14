## Ingress + Proxy Protocol Demo with In-Cluster Services on Linode

1. Install ingress-nginx on your cluster via Helm

```
$ export KUBECONFIG=my-cluster-kubeconfig.yaml
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm install --generate-name ingress-nginx/ingress-nginx
```

2. Point a domain name at your ingress-nginx NodeBalancer

```
$ kubectl get services -A
NAMESPACE     NAME                                            TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
default       ingress-nginx-1602681250-controller             LoadBalancer   10.128.15.49     <public-ip>   80:30003/TCP,443:30843/TCP   11m
...
```

Add an A record using your DNS provider for the service

```
mybackend.mydomain.com <public-ip>
```

3. Deploy a backend Service with an Ingress resource

The following is an example backend service manifest using an ingress for that domain name. We will use `httpbin` which can echo to us our client IP address when we make a request.

```
# mybackend.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  namespace: mybackend-namespace
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
  namespace: mybackend-namespace
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
  namespace: mybackend-namespace
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

```
kubectl apply -f mybackend.yaml
```

Hit http://mybackend.mydomain.com/get in a local web browser.

This is an httpbin endpoint which displays request information, including the client IP and all HTTP headers. Note that the client IP, will be in the Kubernetes Pod network of 10.2.0.0/16, this is the address of ingress-nginx reaching the backend.

4. Enable Proxy Protocol for ingress-nginx

We now want our backend service to see the true client IP, so we enable Proxy Protocol for ingress-nginx.

Add the Proxy Protocol annotation

```
$ k edit service ingress-nginx-1602681250-controller
  # Add the Proxy Protocol annotation to the annotations section
  annotations:
    ...
    service.beta.kubernetes.io/linode-loadbalancer-proxy-protocol: v2
```

Configure ingress-nginx to expect Proxy Protocol data

```
$ k edit configmap ingress-nginx-1602681250-controller
# Add the data section at the root level, which might not yet exist
data:
  use-proxy-protocol: "true"
```

Hit http://mybackend.mydomain.com/get again in a web browser.

You will see your own machine's IP in the "origin" section of httpbin's info. Proxy Protocol successfully enabled!

5. Note that at this point in-cluster clients cannot reach the service via the public hostname.

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

These broken header messages indicate that the in-cluster client is not sending Proxy Protocol data to ingress-nginx, which now exepcts it.

6. Fix this problem by using the service hostname instead of the public hostname.

Instead of using the Ingress hostname, clients which are inside your cluster should use the Service hostname for the backend service. This bypasses ingress-nginx entirely, avoiding the need for the client to send Proxy Protocol headers.

```
$ kubectl delete pod myshell
$ kubectl run myshell --image busybox --command sleep 10000
$ kubectl exec -ti myshell -- sh
/ # wget http://mybackend-service.mybackend-namespace.svc.cluster.local/get
/ # cat get
# You will see that request succeeded and that your "origin" is your Pod IP
# inside the cluster!
```

Here `svc.cluster.local` is the default domain name for the Service network in Kubernetes, you should be able to use that portion of the hostname without any modifications.

At this point you are able to reach the backend from both public and private clients:

* Publicly via http://mybackend.mydomain.com/
* Inside the cluster via http://mybackend-service.mybackend-namespace.svc.cluster.local/
