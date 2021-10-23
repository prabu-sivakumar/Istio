
## Istio

## Service Mesh
Dedicated infrastructure layer for managing service-to-service communication to make it manageable, visible and controlled. 

## Download Istio
The latest version of Istio can be downloaded from https://github.com/istio/istio/releases/tag/1.11.4 

    $ curl -L https://istio.io/downloadIstio | sh -

Add `istioctl` to the your path (Linux)

    cd istio-1.11.4
    export PATH=$PWD/bin:$PATH

To check if `istioctl` is on the path, run the `istioctl version`

    root@k8s-master:~/istio-1.11.4# istioctl version
    client version: 1.11.4
    control plane version: 1.11.4
    data plane version: 1.11.4 (3 proxies)

## Install Istio
We will use demo configuration profile for this installation. The demo profile is selected to have good set of defaults for testing, but there are other profiles for production or performance testing. The recommended profile for production deployments is `default` profile. 

    istioctl  install --set profile=demo -y

To deploy the istio operator, run

    root@k8s-master:~/istio-1.11.4# istioctl operator init
    Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.11.4
    Operator controller will watch namespaces: istio-system
    ✔ Istio operator installed                                                                                          
    ✔ Installation complete

The init command creates the istio-operator namespaces and deploys the custom resource definition, operator deployment, and other resource necessary for the operator to work. The operator is ready to use when the installation completes. 
To install istio, we have to create the `IstioOperator` resource and specify the configuration file we want to use. 

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata: 
  namespace: istio-system
  name: demo-istio-install
spec:
  profile: demo
```
Create resources using `kubectl apply` command. 

    kubectl apply -f demo-profile.yaml

To check the status of installation, we can look at the status of the Pods in the `istio-system` namespace. 

    root@k8s-master:~# kubectl get pod -n istio-system
    NAME                                    READY   STATUS    RESTARTS      AGE
    istio-egressgateway-756d4db566-hrk8t    1/1     Running   1 (21h ago)   35h
    istio-ingressgateway-8577c57fb6-qlwf6   1/1     Running   1 (21h ago)   35h
    istiod-6c86784695-h579g                 1/1     Running   0             18h
    prometheus-77b49cb997-24gjk             2/2     Running   0             18h

To check the status of the installation of the Istio Operator resource, 

    root@k8s-master:~# kubectl get iop -A
    NAMESPACE      NAME                 REVISION   STATUS    AGE
    istio-system   demo-istio-install              HEALTHY   18h

The Istio Operator has finished installing Istio when all Pods are `Running` and the Operator status is `HEALTHY`. 

## Enable sidecar injection
Serive Mesh needs sidecar proxies running alongside each application. To inject the sidecar proxy into an existing Kubernetes deployment, we can use `kube-inject` action in the `istioctl` command. 

However, we can also enable automatic sidecar injection on any Kubernetes namespace. If we label the namespace with istio-injection=enabled, Istio automatically injects the sidecars for any Kubernetes Pods we create in the namespace. 

    kubectl label namespace default istio-injection=enabled

To check if the namespace is labeled, run the following command. Only `default` namespace should be labeled with `istio-injection=enabled`

    root@k8s-master:~# kubectl get namespace -L istio-injection
    NAME              STATUS   AGE   ISTIO-INJECTION
    default           Active   36h   enabled
    istio-operator    Active   21h   disabled
    istio-system      Active   36h   
    kube-node-lease   Active   36h   
    kube-public       Active   36h   
    kube-system       Active   36h   

## Create Deployment
To verify the sidecar injection, we will create a deployment. 

    root@k8s-master:~# kubectl create deploy hello-nginx --image=nginx
    deployment.apps/hello-nginx created

If you inspect the Pods created, you will notice that there are 2 containers in the Pod. 

    root@k8s-master:~# kubectl get pods
    NAME                           READY   STATUS             RESTARTS         AGE
    hello-nginx-795c489f95-qnrzq   2/2     Running            0                86s

Describing the Pod shows Kubernetes created both nginx container and an istio-proxy container: 

    kubectl describe pod hello-nginx-795c489f95-qnrzq

The Events section shows that `nginx` and `docker.io/istio/proxyv2:1.11.4` are installed. 

    Events:
      Type    Reason     Age    From               Message
      ----    ------     ----   ----               -------
      Normal  Scheduled  2m53s  default-scheduler  Successfully assigned default/hello-nginx-795c489f95-qnrzq to k8s-worker
      Normal  Pulled     2m52s  kubelet            Container image "docker.io/istio/proxyv2:1.11.4" already present on machine
      Normal  Created    2m52s  kubelet            Created container istio-init
      Normal  Started    2m52s  kubelet            Started container istio-init
      Normal  Pulling    2m51s  kubelet            Pulling image "nginx"
      Normal  Pulled     2m49s  kubelet            Successfully pulled image "nginx" in 2.164649691s
      Normal  Created    2m48s  kubelet            Created container nginx
      Normal  Started    2m48s  kubelet            Started container nginx
      Normal  Pulled     2m48s  kubelet            Container image "docker.io/istio/proxyv2:1.11.4" already present on machine
      Normal  Created    2m48s  kubelet            Created container istio-proxy
      Normal  Started    2m48s  kubelet            Started container istio-proxy

To remove the Deployment, run the delete command. 

    root@k8s-master:~# kubectl delete deployment hello-nginx
    deployment.apps "hello-nginx" deleted

## Updating Istio Operator
To update the Istio Operator, we can use `kubectl` command and apply the updated `IstioOperator` resource. For example, in order to remove the egress gateway, we could update the IstioOperator resource like this. Create a file `iop-egress.yaml`

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata: 
  namespace: istio-system
  name: demo-installation
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: false
```

    root@k8s-master:~# kubectl apply -f iop-egress.yaml 
    istiooperator.install.istio.io/demo-installation created

If you list the IstioOperator resource, you will notice that status has changed to `RECONCILING`, and once the operator removes the egress gateway, the status changes back to `HEALTHY`.

Another option for updating the Istio installation is to create separate `IstioOperator` resources. You can have a resource for the base installation and separately apply different operators using an empty installation profile. Here is how you create a separate IstioOperator resource that only deploys an internal ingress gateway. 

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: internal-gateway-only
  namespace: istio-system
spec:
  profile: empty
  components:
    ingressGateways:
    -  namespace: default
       name: ilb-gateway
       enabled: true
       label:
         istio: ilb-gateway
       k8s: 
         serviceAnnotations: 
           networking.gke.io/load-balancer-type: "Internal"
```

## Updating and Uninstalling Istio

If we want to update the current installation or change the configuration profile, we will need to update the `IstioOperator` resource deployed earlier. 
To remove the installation, we have to delete the `IstioOperator`, 

    kubectl delete istiooperator -n istio-system demo-installation

Once operator deletes istio, we can also remove the operator by running:

    istioctl operator remove

**Note**:  Make sure to delete the `IstioOperator` resource before deleting the `Operator`. Otherwise, we might end up with stale Istio resources. 
