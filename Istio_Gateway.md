## Istio Gateway

In this example, we will deploy a Hello World application to the cluster. We will then deploy a `Gateway` resource and a `VirtualService` that binds to the Gateway to expose the application on the external IP address. 

## Gateway
Let us start by creating a `Gateway` resource. We will set the `hosts` field to * to access the ingress gateway directly from an external IP address. 

If we wanted to access the ingress gateway through a domain name, such as prabusivakumar.com, we would set the hosts' value to a domain name and add the external IP address to an A record for the domain. 

```yaml 
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port: 
      number: 80
      name: http
      protocol: HTTP
    hosts:
    -  '*'  
```

    kubectl apply -f gateway.yaml

If we try to access the ingress gateway's external IP address, we will get back an HTTP 404 because there aren't any VirtualServices bound to the Gateway. The ingress proxy doesn't know where to route the traffic as we haven't defined any routes yet. 

To get the ingress gateway's external IP address, run the command below and look at the `EXTERNAL-IP` column value. 

    kubectl get svc -l=istio=ingressgateway -n istio-system

## External IP 
If the External IP value is `<none>` or perpetually `<pending>`, your environment does not provide an external load balancer for the ingress gateway. In this case, you can access the gateway using the Service's NodePort. 

Run the following command to verify the Ingress Host IP address:

    kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

To use the NodePort, 

    root@k8s-master:~# kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}'
    31681

## Deployment 
Create a Hello World `Deployment` and `Service`.  The Hello World application is a simple web application that renders "Hello World". 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      -  image: gcr.io/tetratelabs/hello-world:1.0.0
         imagePullPolicy: Always
         name: svc
         ports:
         -  containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    name: http
    targetPort: 3000
```
Create the Deployment and Service using,

    kubectl apply hello-world.yaml

If we notice the Pods running, we will notice two containers running. One is the Envoy sidecar proxy, and the second one is the application. We have also created a Kubernetes Service called `hello-world`:

    root@k8s-master:~# kubectl get pod,svc -l=app=hello-world
    NAME                              READY   STATUS    RESTARTS   AGE
    pod/hello-world-85c8685dd-gdrd7   2/2     Running   0          4m36s
    
    NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    service/hello-world   ClusterIP   10.101.103.20   <none>        80/TCP    4m36s

## Virtual Service
The next step is to create a `VirtualService` for the `hello-world` service and bind it to the Gateway resource. 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-world
spec:
  hosts:
  - '*'
  gateways:
  - gateway
  http:
  - route:
    - destination:
        host: hello-world.default.svc.cluster.local
        port:
          number: 80
```
We use * in the hosts field, just like we did in the Gateway. We have also added the Gateway resource we created earlier to the `gateways` array.  Finally, we specify a single route with a destination that points to the Kubernetes Service `hello-world.default.svc.cluster.local`

Save the above YAML to vs-hello-world.yaml and create the VirtualService using 

    kubectl apply -f vs-hello-world.yaml

To list the deployed VirtualService, 

    root@k8s-master:~# kubectl get vs
    NAME          GATEWAYS      HOSTS   AGE
    hello-world   ["gateway"]   ["*"]   4s

If we run the cURL command, we will get a response of `Hello World`.

    root@k8s-master:~# curl -v http://35.198.206.47:31681
    *   Trying 10.107.46.88:80...
    * Connected to 10.107.46.88 (10.107.46.88) port 80 (#0)
    > GET / HTTP/1.1
    > Host: 10.107.46.88
    > User-Agent: curl/7.74.0
    > Accept: */*
    > 
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < date: Sat, 23 Oct 2021 14:13:31 GMT
    < content-length: 11
    < content-type: text/plain; charset=utf-8
    < x-envoy-upstream-service-time: 29
    < server: istio-envoy
    < 
    * Connection #0 to host 10.107.46.88 left intact
    Hello World 

Also, notice that server header set to `istio-envoy` telling us that the request went through the Envoy proxy. 

## Cleanup Resources
Delete the Deployment, Service, VirtualService and Gateway. 

    kubectl delete deploy hello-world
    kubectl delete service hello-world
    kubectl delete vs hello-world
    kubectl delete gateway gateway

