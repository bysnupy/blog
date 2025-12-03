# Containers Starting and Termination Order in a Pod

By Daein Park

In this article, I will take a deeper look into the container’s startup and termination order as a part of the Pod lifecycle explained in the Kubernetes  upstream documentation. This topic is not brand-new however, it would be worth checking out the latest OpenShift release notes in the event that there are any changes worth noting.

## Why is this topic important?
If you want to configure a multi-container application that has some dependencies among themselves within a single Pod, you should control the order in which they start up or shut down as a method for preventing race conditions. For instance, if container A is required to start after container B, or container A needs to delay termination after Container B. As a classic use case, the Istio Proxy is a sidecar container that all traffic from the main containers traverses through and should be started prior to the main containers accepting traffic and should be terminated after the primary containers drain all remaining traffic.

## Each container's lifetime in a Pod

![The diagram shows us the lifetime of three container types in a pod lifecycle.](https://github.com/bysnupy/blog/blob/master/kubernetes/images/each_container_lifetime_in_a_pod.png)

A normal Pod can be organized with three container types, each having a lifetime as follows.

Be aware that the starting order is based on the creation time of each container. Conversely the termination order is based on issuing the order of the SIGTERM signal to each container. The following table describes the order in which each container type starts and stops.

Type | Starting Order | Termination Order | Description
-----|----------------|-------------------|------------
Init Containers | Sequential, one by one in the order defined | Sequential, one by one in the order defined | Executed and completed one after the other before the main containers run
Sidecar Containers | Sequential, one by one in the order defined | In the opposite order, after all main containers stop | Run along with the main containers after executed from the Initialization phase
Main Containers | Sequential without blocking the next | Random, due to the time it takes for the termination process to complete | Run main application processes here

Now, let’s look into the finer points for how each container type starts and stops.

## Init Containers
Init Containers must start up successfully before the next one starts. This specification makes the starting and termination order sequential. Typical use-cases for Init Containers are for any pre-processes that need to take place prior to running the Main Containers.

For instance, all the Init Containers in the following Pod A case would start and terminate A, B, and C in order, and each Init Container must terminate successfully as a one-off task before the Main Containers start. If one of the Init Containers fails, the next one in the list does not start, and the Pod enters an error state, such as Init: Error and Init: CrashLoopBackOff. Look at the Pod B case in the following diagram where this situation is depicted. Init Container C was not started after Init Container B failed. In addition, the Main Container was also not started.

![This diagram shows the init containers starting sequentially and running to completion.](https://github.com/bysnupy/blog/blob/master/kubernetes/images/init_container_starting_order.png)

## Sidecar Containers
This container type has been added in recent versions of Kubernetes as of 1.29 (OpenShift 4.16). Both Init Containers and Sidecar Containers have similar configurations. However, a key difference between the two is that Sidecar Containers will set restartPolicy: Always so that the container will keep running after the Main Containers start. This behavior is very useful for supporting sidecar based architectures. The starting order is sequential, one by one, and if one has not started successfully, the next container will not start as was shown in the prior Init Containers section. Even if you mix other Init Containers together, they start in order specified in the spec.initContainers array in the Pod manifest.

For your understanding, look at the following diagram. Sidecar Containers A, B and C are each started one by one and are then run along with the Main Containers. If any Sidecar Container or any Init Container fails, the Main Containers do not start. 

![This diagram shows the sidecar containers starting order.](https://github.com/bysnupy/blog/blob/master/kubernetes/images/sidecar_container_starting_order.png)

Let's look at the termination phase, it shows us the value of the Sidecar Containers.

![This diagram shows the termination order of sidecar containers.](https://github.com/bysnupy/blog/blob/master/kubernetes/images/sidecar_container_termination_order.png)

All Sidecar Containers terminate in the opposite order they started after terminating all the Main Containers.
This is very helpful in resolving issues if there are any dependencies between Sidecar Containers and Main Containers. For example, if a logging agent must be terminated after the main application so that application logs are processed without any loss. If you’re interested in how this termination order is implemented, refer to the following code sample.

## Main Containers
The primary content of your applications are run as the Main Containers within a Pod, and it starts sequentially in order of spec. containers array in the Pod Template. In greater detail, the kubelet starts all Main Containers without checking the status of each container. This is a different behavior than Init and Sidecar Containers as mentioned previously. In other words, the next Main Container launches regardless of the any of the previously started containers’ starting status. 
This logic is quickly processed, so it can appear that all Main Containers started at the same time. If you want to start all Main Containers one by one like Init Containers, this approach can be implemented using the postStart lifecycle hook that blocks the start of the next container until the hook is completed.

This following diagram illustrates this process.



One interesting point to note is that all the Main Containers are terminated randomly in parallel. As depicted in this code sample, the termination is invoked concurrently through the go routine for each of the containers. If you want to implement a specific termination order in the Main Containers, you can consider configuring preStop lifecycle hooks.

You can defer the time to issue the SIGTERM signal to the container until the hook is completed. Keep in mind that all the preStop hooks are triggered immediately in parallel as soon as the pod is in Terminating status, not one by one for each of the containers. The related code is here.

## TerminationGracePeriodSeconds
One additional consideration, there is an exception for the termination order when the Pod expires as a result of the terminationGracePeriodSeconds. This timeout starts to count down as soon as the Pod enters the Terminating status.

If any of the Main Containers were not terminated within this timeframe, the kubelet forcibly kills the container with the SIGKILL signal..

Consider the situation where the Sidecar Containers are running together at that time. What will occur? Then, the Sidecar containers will terminate randomly in parallel after terminating all Main Containers without keeping the expected reverse order. If any of the Sidecar Container does not terminate immediately, the kubelet also sends the container kill signal again after waiting an additional 2 seconds. You need to modify the terminationGracePeriodSeconds for keeping the expected termination in order to suit this use case. Refer to the Termination of Pods and Pod shutdown and sidecar containers for more details.

This following diagram illustrates the process described above.

## Summary
As mentioned at the outset, while the functionality surrounding the lifecycle of a Pod has been included as a core capability of Kubernetes for many releases, understanding how it functions enables you greater insight and the ability to fine tune the configurations of your applications and your use cases. The examples provided included real world use cases which can be used within your own environment to harness the true power of OpenShift and Kubernetes. I hope I have covered enough details to get you tested for your own environment.
