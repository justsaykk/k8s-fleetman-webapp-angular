# Kubernetes Learnings

## What is "Kubernetes" a.k.a K8s?

- It is a system to orchestrate containers.
- We can think of containers as applications.
- Each container is deployed as instructed by an image.
- The image would contain all the relevant information that allows for the application to function.
- Each Container is like a virtual machine. In the sense that it is running a software/code.

> Basically an entire application condensed into a single file

Of course, with a growing number of containers, it becomes difficult to manage each and every container by itself. Therefore, K8s was born.

With some nifty engineering, K8s can do a lot of stuff. One such feature is the benefit of continous deployment.

Basically, within the K8s architecture, there can be 2 or more versions of a container running. When the time comes, it can immediately switch from v1 to v2 with practically no downtime. This is not achievable in the traditional way for writing applications as the whole app needs to be taken down and redeployed to achieve v1 to v2.

## How does K8s achieve this?

TL;DR: Services acts as controllers do things with pods. You'll probably never handle the pods yourself.

### Introducing: Pods

TYPICALLY, 1 pod contains 1 container. However, if the containers cannot live without each other, then it may be wiser to put them into the same pod. *But c'mon man, encapsulation principals?*

These pods are shortlived and *ephemeral*. Imagine Java methods. Each time it is supposed to run, it'll be called, executed and left alone. Similarly for pods, it'll be **CREATED**, execute what it needs to do, and finally "die" or *evicted*.

There are a few ways to get a pod running. All we have been doing is creating a pod manually.
Manually creating a pod means that we also have to manually manage the pod.

### Introducing: Services

Pods themselves are not exposed to the internet. A user will not be able to just connect to a pod directly via IP address. The user will need to connect to a service and the service will be the bridge between the pod and the outside world. Their definition will look like this:

```yaml
apiVersion: v1
kind: Service
metadata:
  # This is super important because it is the official name used 
  # by other services as a reference point
  name: fleetman-webapp
spec:
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      nodePort: 30080 # nodePorts must be greater than 30k
  # NodePort is a type of service that exposes a port to the internet
  # There are other types of services. This includes "loadBalancer" & "ClusterIP"
  type: NodePort
```

According to this definition, the Pod's port is 80, and it can be accessed using `localhost:30080`. Notice that `nodePort` is used as the endpoint that users connect to whilst the service *"redirects"* users to the Pod's port: 80.

### Intoducing: ReplicaSets

A ReplicaSet's purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.

A ReplicaSet is defined with the following information:

1. A selector that specifies how to identify Pods.
2. A number of replicas indicating the quantity of Pods it should maintain.
3. Pod template.

The ReplicaSet will try to achieve the number of replicas by deleting or creating Pods. Pods that are created will follow the Pod template.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # modify replicas according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template: # Definition for a Pod
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

Applying the ReplicaSet will yield the following Pods:

```xml
$ kubectl get pods

NAME             READY   STATUS    RESTARTS   AGE
frontend-b2zdv   1/1     Running   0          6m36s
frontend-vcmts   1/1     Running   0          6m36s
frontend-wtsmm   1/1     Running   0          6m36s
```

A ReplicaSet ensures that a specified number of pod replicas are always running at any given time. However, a Deployment is a high-level concept that manages ReplicaSets and provides declarative updates to Pods alongside other useful features.

Therefore, it is recommended to use Deployments instead of ReplicaSets unless there is a requirement for custom update orchestration or the Pods do not require updates at all.

### Introducing: Deployments

We can think of Deployments as a ReplicaSet manager. It deploys ReplicaSets which in turn, manages Pods. The Deployment .yaml file actually looks the same as the ReplicaSet .yaml file. However, there are extra features to fine-tune the deployment such as delaying the termination of a Pod for *x* seconds. Apart from the extra customizability, deployments also allows developers to "rollback" a deployment.

#### How to use deployments?

Remember the `kind: ReplicaSet`? Just change `ReplicaSet` to `Deployment` and reapply the yaml file.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```

Looking at the `$ kubectl get all` after the deployment, we see the following:

```plaintext
NAME                          READY   STATUS    RESTARTS        AGE
pod/webapp-86cbfbbbf7-5gvlm   1/1     Running   0               8m13s
pod/webapp-86cbfbbbf7-fbtlz   1/1     Running   0               7m42s

<Services portion>

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/webapp   2/2     2            2           23m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/webapp-5454bb45b    0         0         0       3m50s
replicaset.apps/webapp-6f78788c87   0         0         0       21m
replicaset.apps/webapp-86cbfbbbf7   2         2         2       14m
```

Notice that there is one deployment. This deployment created a ReplicaSet (`replicaset.apps/webapp-86cbfbbbf7`) and that ReplicaSet created 2 Pods:

1. `pod/webapp-86cbfbbbf7-5gvlm`
2. `pod/webapp-86cbfbbbf7-fbtlz`

Notice that the Pods inherits the ReplicaSet's name and appends a random ID to the back.

Also notice the other replicaSets with `0 DESIRED`, `0 CURRENT` and `0 READY`. These are replicaSets that was created and had since been obsoleted by a newer deployment. Those are the replicaSets that we could rollback to.

Some useful commands:

#### To see deployment status

```plaintext
$ kubectl rollout status deployment/webapp
> deployment "webapp" successfully rolled out
```

#### To see revisions of deployments

```plaintext
$ kubectl rollout history deployment/webapp

## alternatively:

$ kubectl rollout history deployment/webapp --revision=2
```

#### Rolling back to previous deployments

```plaintext
$ kubectl rollout undo deployment/webapp

## alternatively, rollback to other previous revisions:

$ kubectl rollout undo deployment/webapp --to-revision=2
```

> This is a dangerous feature because the .yaml file will not match the currently deployed version of the application after a rollback. It is only to be used in an emergency.
