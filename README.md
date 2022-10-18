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

### Introducing: Pods

TYPICALLY, 1 pod contains 1 container. However, if the containers cannot live without each other, then it may be wiser to put them into the same pod. *But c'mon man, encapsulation principals?*

These pods are shortlived and *ephemeral*. Imagine Java methods. Each time it is supposed to run, it'll be called, executed and left alone. Similarly for pods, it'll be **CREATED**, execute what it needs to do, and finally "die" or *evicted*.
