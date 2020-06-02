# k8s-basics

In this simple demo, we will cover the following:

* Deploy a GKE cluster
* Deploy a Pod
* Deploy a Deployment
* Expose a Deployment Using a Node Port
* Expose a Deployment Using a LB
* Exec into a container

### Create GKE cluster

This is pretty easy with GKE:

```bash
gcloud container clusters create "demo" --zone "us-east4-c"
```

### Deploy a Pod

Download the git repo with all the simple examples:

```bash
git clone https://github.com/elatovg/k8s-basics
```

Now let’s create a single pod:

```bash
> k apply -f pod.yaml
pod/nginx created
```

Now you should see the pod up:

```bash
> k get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          26s
```

Let’s go inside (**exec** into) the pod and see what’s running:

```bash
> k exec -it nginx -- /bin/sh
/ $ ps
PID   USER     TIME  COMMAND
    1 nginx     0:00 nginx: master process nginx -g daemon off;
    6 nginx     0:00 nginx: worker process
   43 nginx     0:00 /bin/sh
   48 nginx     0:00 ps
/ $ netstat -alnt
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN
```

Let’s see if we can check out what the container is serving:

```bash
/ $ wget -q -O - http://localhost:8080 | grep title
<title>Hello World</title>
```

Now let’s delete the pod:

```bash
/ $ exit
command terminated with exit code 127
> k delete -f pod.yaml
pod "nginx" deleted
```

### Deploy a Deployment

Next let’s create a deployment

```bash
> k apply -f deploy.yaml
deployment.apps/nginx-deployment created
```

By default it has 3 replicas, so we should see 3 pods running:

```bash
> k get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-756d9fd5f9-55fnw   1/1     Running   0          40s
nginx-deployment-756d9fd5f9-bxcbp   1/1     Running   0          40s
nginx-deployment-756d9fd5f9-vrx6f   1/1     Running   0          40s
```

We will also see a *replicaset* for the deployment:

```bash
 > k get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-756d9fd5f9   3         3         3       44s
```

And of course the deployment it self:

```bash
> k get deploy
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           48s
```

### Expose Deployment with a ClusterIP Service
Now that we have a deployment, let’s expose it using a simple **ClusterIP** service:

```bash
> k apply -f svc-ci.yaml
service/nginx-service created
```

Let’s confirm the **service** is created:

```bash
> k get svc
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes      ClusterIP   10.63.240.1     <none>        443/TCP   64m
nginx-service   ClusterIP   10.63.245.157   <none>        80/TCP    91s
```

And let’s make sure it has **endpoints** associated with it:

```bash
> k get ep
NAME            ENDPOINTS                                AGE
kubernetes      34.86.255.196:443                        65m
nginx-service   10.60.1.4:80,10.60.2.6:80,10.60.2.7:80   2m5s
```

You will notice there are 3 endpoints (1 per pod/replica). We can also confirm those are the IPs of our Pods:

```bash
> k get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP          NODE                                  NOMINATED NODE   READINESS GATES
nginx-deployment-756d9fd5f9-55fnw   1/1     Running   0          12m   10.60.2.6   gke-demo-default-pool-fa4eea5d-1d2m   <none>           <none>
nginx-deployment-756d9fd5f9-bxcbp   1/1     Running   0          12m   10.60.1.4   gke-demo-default-pool-fa4eea5d-bpzq   <none>           <none>
nginx-deployment-756d9fd5f9-vrx6f   1/1     Running   0          12m   10.60.2.7   gke-demo-default-pool-fa4eea5d-1d2m   <none>           <none>
```

Since **ClusterIP** is internal, we have a couple of options to reach it, one is to use **port-forwarding**:

```bash
> k port-forward service/nginx-service 1080:80
Forwarding from 127.0.0.1:1080 -> 80
Forwarding from [::1]:1080 -> 80
```

Then from another terminal:

```bash
> curl -s http://localhost:1080 | grep -i title
<title>Welcome to nginx!</title>
```

Or you could deploy another pod:

```bash
> k apply -f curl-deploy.yaml
deployment.apps/curl created
```

Now you will have a new **curl** pod running along side your nginx pods:

```bash
> k get pods
NAME                                READY   STATUS    RESTARTS   AGE
curl-844849bbdb-cvx99               1/1     Running   0          11s
nginx-deployment-756d9fd5f9-55fnw   1/1     Running   0          16m
nginx-deployment-756d9fd5f9-bxcbp   1/1     Running   0          16m
nginx-deployment-756d9fd5f9-vrx6f   1/1     Running   0          16m
```

Now let’s **exec** into the `curl` pod and try to reach our nginx service:

```bash
> k exec -it curl-844849bbdb-cvx99 -- /bin/sh
/bin/sh: shopt: not found
[ root@curl-844849bbdb-cvx99:/ ]$ curl -s nginx-service | grep title
<title>Welcome to nginx!</title>
```

Let’s clean up the **clusterIP** service and the **curl** pods:

```bash
> k delete -f curl-deploy.yaml
deployment.apps "curl" deleted
> k delete -f svc-ci.yaml
service "nginx-service" deleted
```

### Expose Deployment with a Node Port Service

Now let’s expose the deployment using a **NodePort**. With a Node Port you either pick a high # port to be used for your service or if you don’t specify one it will automatically pick one. I decided to use port **30080**. First let’s create the service:

```bash
> k apply -f svc-np.yaml
service/nginx-service created
```

Now confirm the service is created:

```bash
> k get svc
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes      ClusterIP   10.63.240.1    <none>        443/TCP        83m
nginx-service   NodePort    10.63.246.56   <none>        80:30080/TCP   22s
```

Notice the port port **30080** is there. Now let’s open up the firewall for that port:

```bash
> gcloud compute firewall-rules create demo-node-port --allow tcp:30080
Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/demo/global/firewalls/demo-node-port].
Creating firewall...done.
NAME            NETWORK  DIRECTION  PRIORITY  ALLOW      DENY  DISABLED
demo-node-port  default  INGRESS    1000      tcp:30080        False
```

Now let’s figure out the public IP for one of the nodes:

```bash
> k get nodes -o wide
NAME                                  STATUS   ROLES    AGE   VERSION           INTERNAL-IP   EXTERNAL-IP      OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-demo-default-pool-fa4eea5d-1d2m   Ready    <none>   83m   v1.14.10-gke.36   10.150.0.22   35.245.3.107     Container-Optimized OS from Google   4.14.138+        docker://18.9.7
gke-demo-default-pool-fa4eea5d-bpzq   Ready    <none>   83m   v1.14.10-gke.36   10.150.0.21   35.236.220.162   Container-Optimized OS from Google   4.14.138+        docker://18.9.7
gke-demo-default-pool-fa4eea5d-hck0   Ready    <none>   83m   v1.14.10-gke.36   10.150.0.26   35.245.172.62    Container-Optimized OS from Google   4.14.138+        docker://18.9.7
```

So let’s pick the first one:

```bash
> curl -s http://35.245.3.107:30080 | grep title
<title>Welcome to nginx!</title>
```

With the use of **iptables** k8s is able to forward the traffic to the node where the pod resides. Now let’s delete the service and the firewall:

```bash
> gcloud compute firewall-rules delete demo-node-port -q
Deleted [https://www.googleapis.com/compute/v1/projects/demo/global/firewalls/demo-node-port].
```

### Expose Deployment with a Load Balancer Service

The last service type we are going to cover is **LoadBalancer**. This is typically offered by a cloud provider. In my example it’s GCP. First create the service:

```bash
> k apply -f svc-lb.yaml
service/nginx-service created
```

We have to wait a couple of minutes, but when it’s ready you will see a public IP associated with the service:

```bash
> k get svc
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
kubernetes      ClusterIP      10.63.240.1     <none>           443/TCP        92m
nginx-service   LoadBalancer   10.63.240.181   35.230.162.191   80:31033/TCP   46s
```

You will notice there is also a high number port assigned to the service as well, just like with the node port :) Now confirm the LB is able to reach the deployment:

```bash
> curl -s http://35.230.162.191 | grep title
<title>Welcome to nginx!</title>
```

Let’s delete the service after we are done:

```bash
> k delete -f svc-lb.yaml
service "nginx-service" deleted
```

Lastly delete the GKE cluster:

```
> gcloud container clusters delete demo --zone us-east4-c -q
Deleting cluster demo...done.
Deleted [https://container.googleapis.com/v1/projects/demo/zones/us-east4-c/clusters/demo].
```

That’s it for now.
