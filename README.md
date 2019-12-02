# Hazelcast with Istio Service Mesh

Istio is said to be the next thing if you follow the kubernetes path and Service Meshes are mentioned everywhere whenever you go to a CloudNative meetup or conferences like KubeCon. However, you do not always support something for the sake of popularity. Hazelcast Istio support discussion was started by a user via raising [github issue](https://github.com/hazelcast/hazelcast-kubernetes/issues/118) with the title `Does hazelcast-kubernetes support istio?`. 

Until recently, we were thinking that it works fine after closing the issue above but then some other user from the community raised [another issue](https://github.com/hazelcast/hazelcast-kubernetes/issues/118#issuecomment-551876442). That grabbed better piece of our attention and we started checking deeply what is going on under the hood. Luckily enough, I had won from a raffle in [Istio Meetup at All Things Open](https://www.meetup.com/Triangle-Kubernetes-Meetup/events/264418548/Kubernetes) it was exacly what I needed during that period :  Early edition of Istio in Action by Christian E. Posta, Sandeep Parikh.

This blog posts mainly focuses on how to run Hazelcast with Istio. It does not explain what Service Meshes are and How Istio works because you will find thousands resources on the internet.  

## Requirements

### Kubernetes Cluster
In this guide, I used a Google Kubernetes Engine but you can use any Kubernetes cluster you choose.
```
$ gcloud container clusters create hazelcast-istio-k8s-cluster --cluster-version=1.14.8-gke.12 --num-nodes=4
```

### Istio 1.3.4
This code sample has been tested against Istio 1.3.4

#### Istio Installation

Firstly, let's download istio and configure path for istioctl
```
$ curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.3.4 sh -
$ export PATH=`pwd`/istio-1.3.4/bin:$PATH
```

Install Istio CRDs
```
$ cd istio-1.3.4
$ for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```

Verify that you have 23 CRDs installed.
```
$ kubectl get crds | grep 'istio.io' | wc -l
23
```

Install Istio with mTLS enabled
```
$ kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```
This installation applies mTLS between Service Proxies.

Verify both istioctl client and Istio Control Plane has the same version.
```
$ istioctl version
client version: 1.3.4
control plane version: 1.3.4
```

### Hazelcast Istio Code Sample 

Clone this repository and apply RBAC. RBAC is needed by [hazelcast-kubernetes](https://github.com/hazelcast/hazelcast-kubernetes) plugin discovery.
```
$ git clone https://github.com/hazelcast-guides/hazelcast-istio.git
$ cd hazelcast-istio
$ kubectl apply -f rbac.yaml
```

Istio Sidecar AutoInjection is done by automagically if you label default namespace with `istio-injection`. If you want to implement manual injection with the deployments then you need to use `istioctl kube-inject` 

```
$ kubectl label namespace default istio-injection=enabled
```

## Hazelcast-Istio Code Sample

This guide contains a basic Spring-Boot Microservice with both Hazelcast Embedded and Hazelcast Client Server. Those microservices demonstarates how Hazelcast can be used in Istio Environments.
Business Logic in both examples are the same to keep it simple. `put` operation puts a key-value pair to Hazelcast and `get` operation returns the value together with the Kubernetes Pod Name. PodName is used to show that the value is returned from any Pod inside the Kubernetes cluster to prove the true nature of Distributed Cache.

### Hazelcast Embedded
Switch to embedded code sample directory `cd hazelcast-embedded/`

Embedded code sample can be built and pushed to your own Docker Hub or some other registry via following command but that is optional. 
```
$ mvn compile com.google.cloud.tools:jib-maven-plugin:1.8.0:build -Dimage=YOUR-NAME/hazelcast-springboot-embedded:1.0
```

Instead, pre-built image is located [here](https://hub.docker.com/r/mesut/hazelcast-springboot-embedded) and deployments in embedded example is using that image.

If you want to build your own image then you can use [jib](https://github.com/GoogleContainerTools/jib) to do that but do not forget to update `hazelcast-embedded.yaml` with your own image

Deploy Hazelcast Embedded Sample
```
$ kubectl apply -f hazelcast-embedded.yaml
statefulset.apps/hazelcast-embedded created
service/hazelcast-embedded-headless created
service/springboot-service created
```

When you list the services used, you will see that you have two Kubernetes Service: `hazelcast-embedded-headless` and `springboot-service`. `hazelcast-embedded-headless` is used to handle hazelcast cluster discovery operation so it has no need to have an IP address. `springboot-service` is Loadbalancer that is used to receive http requests and forward it to an underlying Pod to process it.
```
$ kubectl get svc
NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
hazelcast-embedded-headless   ClusterIP   None           <none>        5701/TCP   9s
kubernetes                    ClusterIP   10.19.240.1    <none>        443/TCP    73m
springboot-service            ClusterIP   10.19.252.76   <none>        80/TCP     9s
```

Lets now put a key-value pair into hazelcast cluster through Spring Boot REST Service and then call get operation in a loop to see the value is returned from different Pods. 

Firstly, let's run a container with curl installed and set an environment variable to point to Spring Load Balancer.

```
$ kubectl run curl --rm --image=radial/busyboxplus:curl -i --tty
$ SPRINGBOOT_SERVICE=10.19.252.76
```

Put a value to the cluster
```
$ curl "${SPRINGBOOT_SERVICE}/put?key=1&value=2"
{"value":"2","podName":"hazelcast-embedded-2"}
```
Get the value from cluster in a loop and see that it is retrieved from different Pod Names.
```
$ while true; do curl "${SPRINGBOOT_SERVICE}/get?key=1"; sleep 2;echo; done
{"value":"2","podName":"hazelcast-embedded-1"}
{"value":"2","podName":"hazelcast-embedded-0"}
...
```
In this sample, you were able to deploy a spring-boot based microservice with hazelcast embedded in Istio Environment. Let's clean up deployments with the following command.

```
kubectl delete -f hazelcast-embedded.yaml
```


### Hazelcast Client Server

In the previous section, we learnt how to use hazelcast embedded with Istio. Lets now focus on Hazelcast Client Server Topology. First of all, switch to client-server folder with `cd ../hazelcast-client-server`

Client-Server code sample can be built and pushed to your own Docker Hub or some other registry via following command but that is optional. 
```
mvn compile com.google.cloud.tools:jib-maven-plugin:1.8.0:build -Dimage=YOUR-NAME/hazelcast-springboot-client:1.0
```
Instead, pre-built image is located [here](https://hub.docker.com/r/mesut/hazelcast-springboot-client) and deployments in client-server example is using that image.

If you want to build your own image then you can use [jib](https://github.com/GoogleContainerTools/jib) to do that but do not forget to update `hazelcast-springboot-client.yaml` with your own image

Deploy Hazelcast Cluster
```
kubectl apply -f hazelcast-cluster.yaml
```

You can see that 3 member cluster has been initiated with 3 pods. `2/2` in READY column means that there are 2 containers running in each POD. One is hazelcast member and the other is Envoy Proxy.

```
$ kubectl get pods
NAME                  READY   STATUS    RESTARTS   AGE
hazelcast-cluster-0   2/2     Running   0          60s
hazelcast-cluster-1   2/2     Running   0          44s
hazelcast-cluster-2   2/2     Running   0          27s
```

Deploy Spring Boot Application with Hazelcast Client
```
kubectl apply -f hazelcast-springboot-client.yaml
```
Check logs and see that spring boot service is connected to the cluster.

```
$ kubectl logs hazelcast-client-0 hazelcast-client -f
...
Members [3] {
	Member [10.16.2.14]:5701 - 51274b4d-dc7f-4647-9ceb-c32bfc922c95
	Member [10.16.1.15]:5701 - 465cfefa-9b26-472d-a204-addf3b82d40a
	Member [10.16.2.15]:5701 - 67fdf27a-e7b7-4ed7-adf1-c00f785d2325
}
...
```


let's run a container with curl installed and set an environment variable to point to Spring Load Balancer.

Let's first find the IP of Spring Boot Service.

```
$ kubectl get svc springboot-service
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
springboot-service   ClusterIP   10.19.250.127   <none>        80/TCP    3m29s

```

Launch a curl container inside kubernetes cluster and set service IP as environment variable
```
$ kubectl run curl --rm --image=radial/busyboxplus:curl -i --tty
$ SPRINGBOOT_SERVICE=10.19.250.127
```

Put a value to the cluster
```
$ curl "${SPRINGBOOT_SERVICE}/put?key=1&value=2"
{"value":"2","podName":"hazelcast-embedded-2"}
```
Get the value from cluster in a loop and see that it is retrieved from different Pod Names.
```
$ while true; do curl "${SPRINGBOOT_SERVICE}/get?key=1"; sleep 2;echo; done
{"value":"2","podName":"hazelcast-embedded-1"}
{"value":"2","podName":"hazelcast-embedded-0"}
...
```
In this sample, you were able to deploy a spring-boot based microservice with hazelcast client-server topology in Istio Environment. Let's clean up deployments with the following command.

```
$ kubectl delete -f hazelcast-springboot-client.yaml
$ kubectl delete -f hazelcast-cluster.yaml
```

## Conclusion
This guide demonstrates how to use Hazelcast Embedded and Client-Server Topology in mTLS enabled Istio environment with Automatic Sidecar Injection. Hazelcast continuously tries to support Cloud Native Technologies and verifies those environments as they evolve. 
