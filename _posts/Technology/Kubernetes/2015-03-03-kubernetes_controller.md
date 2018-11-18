---
layout: post
title: Kubernetes controller 组件
category: 技术
tags: Kubernetes
keywords: CoreOS Docker Kubernetes
---


## 简介

本文主要来自对[https://cloud.google.com/container-engine/docs](https://cloud.google.com/container-engine/docs "")的摘抄，有删减。

本文主要讲了replication controller（部分地方简称RC）

2018.11.18 补充，内容来自极客时间 《深入剖析Kubernetes》


## 编排的实现——控制器模型

docker是单机版的，当我们接触k8s时，天然的认为这是一个集群版的docker，再具体的说，就在在集群里给镜像找一个主机来运行容器。经过 [《深入剖析kubernetes》笔记](http://qiankunli.github.io/2018/08/26/parse_kubernetes_note.html)的学习，很明显不是这样。比调度更重要的是编排，那么编排如何实现呢？

### 有什么

controller是一系列控制器的集合，不单指RC。

	$ cd kubernetes/pkg/controller/
	$ ls -d */              
	deployment/             job/                    podautoscaler/          
	cloud/                  disruption/             namespace/              
	replicaset/             serviceaccount/         volume/
	cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
	...

### 构成

一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象（Pod 或 Volume等）的模板组成的。

### 逻辑

这些控制器之所以被统一放在 pkg/controller 目录下，就是因为它们都遵循 Kubernetes 项目中的一个通用编排模式，即：控制循环（control loop）。 （这是不是可以解释调度器 和控制器 不放在一起实现，因为两者是不同的处理逻辑，或者说编排依赖于调度）

	for {
	  实际状态 := 获取集群中对象 X 的实际状态（Actual State）
	  期望状态 := 获取集群中对象 X 的期望状态（Desired State）
	  if 实际状态 == 期望状态{
	    什么都不做
	  } else {
	    执行编排动作，将实际状态调整为期望状态
	  }
	}

实际状态往往来自于 Kubernetes 集群本身。 比如，kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息。而期望状态，一般来自于用户提交的 YAML 文件。 比如，Deployment 对象中 Replicas 字段的值，这些信息往往都保存在 Etcd 中。



Kubernetes 使用的这个“控制器模式”，跟我们平常所说的“事件驱动”，有点类似 select和epoll的区别。控制器模型更有利于幂等。



## What is a replication controller?

A replication controller ensures that a specified number of pod "replicas" are running at any one time. If there are too many, the replication controller kills some pods. If there are too few, it starts more. As opposed to just creating singleton pods or even creating pods in bulk, a replication controller replaces pods that are deleted or terminated for any reason, such as in the case of node failure. For this reason, we recommend that you use a replication controller even if your application requires only a single pod.（将Pod维持在一个确定的数量）

A replicationController is only appropriate for pods with RestartPolicy = Always.


### How does a replication controller work?

replication controller中的几个概念（与replication controller config file中都有对应），RC与pod之间的关系。



#### Pod template

A replication controller creates new pods from a template.

Rather than specifying the current desired state of all replicas, pod templates are like cookie cutters（饼干模型切割刀）. Once a cookie（饼干） has been cut, the cookie has no relationship to the cutter. Subsequent changes to the template or even a switch to a new template has no direct effect on the pods already created.


#### Labels

The population of pods that a replication controller monitors is defined with a label selector（A key:value pair）, which creates a loosely coupled relationship between the controller and the pods controlled.（replication controller有一个label，一个replication controller监控的所有pod（controller's target set）也都包含同一个label，两者以此建立联系）

So that only one replication controller controls any given pod, ensure that the label selectors of replication controllers do not target overlapping（重叠） sets.

To remove a pod from a replication controller's target set, change the pod's label（改变一个pod的label，可以将该pod从controller's target set中移除）. Use this technique to remove pods from service for debugging, data recovery, etc. Pods that are removed in this way will be replaced automatically (assuming that the number of replicas is not also changed).

Similarly, deleting a replication controller does not affect the pods it created. To delete the pods in a replication controller's target set, set the replication controller's replicas field to 0.
（一旦为pod创建replication controller，再想删除这个pod就要修改replication controller的replicas字段了）

### Common usage patterns（应用场景）

#### Rescheduling

Whether you have 1 pod you want to keep running, or 1,000, a replication controller will ensure that the specified number of pods exists, even in the event of node failure or pod termination (e.g., due to an action by another control agent).


#### Scaling

Replication controllers make it easy to scale the number of replicas up or down, either manually or by an auto-scaling control agent. Scaling is accomplished by updating the replicas field of the replication controller's configuration file.
（很轻松的改变pod的个数）

#### Rolling updates

Replication controllers are designed to facilitate（促进，帮助，使容易） rolling updates to a service by replacing pods one by one.

The recommended approach is:

- Create a new replication controller with 1 replica.
- Resize the new (+1) and old (-1) controllers one by one.
- Delete the old controller after it reaches 0 replicas.

This predictably updates the set of pods regardless of unexpected failures.

The two replication controllers need to create pods with at least one differentiating label.

（逐步更新pod(现成的命令喔)：建立两个Replication controllers，老的replicas减一个，新的replicas加一个，直到老的replicas为0，然后将老的Replication controllers删除）

#### Multiple release tracks

In addition to running multiple releases of an application while a rolling update is in progress, it's common to run multiple releases for an extended period of time, or even continuously, using multiple release tracks. The tracks in this case would be differentiated by labels.

For instance, a service might target all pods with tier in (frontend), environment in (prod). Now say you have 10 replicated pods that make up this tier. But you want to be able to 'canary' a new version of this component. You could set up a replicationController with replicas set to 9 for the bulk of the replicas, with labels tier=frontend, environment=prod, track=stable, and another replicationController with replicas set to 1 for the canary, with labels tier=frontend, environment=prod, track=canary. Now the service is covering both the canary and non-canary pods. But you can update the replicationControllers separately to test things out, monitor the results, etc.

（多版本长期共存：多个replicationController，不同的track字段值）

## Replication Controller Operations

### Creating a replication controller

    $ kubectl create -f xxx

A successful create request returns the name of the replication controller. 

#### Replication controller configuration file

When creating a replication controller, you must point to a configuration file as the value of the -f flag. The configuration file can be formatted as YAML or as JSON, and supports the following fields:

    {
      "id": string,
      "kind": "ReplicationController",
      "apiVersion": "v1beta1",
      "desiredState": {
        "replicas": int,
        "replicaSelector": {string: string},
        "podTemplate": {
          "desiredState": {
             "manifest": {
               manifest object
             }
           },
           "labels": {string: string}
          }},
      "labels": {string: string}
    }

Required fields are:

- id: The name of this replication controller. It must be an RFC1035 compatible value and be unique on this container cluster.
- kind: Always ReplicationController.
- apiVersion: Currently v1beta1.
- desiredState: The configuration for this replication controller. It must contain:

 - replicas: The number of pods to create and maintain.
 - **replicaSelector**: A key:value pair assigned to the set of pods that this replication controller is responsible for managing. This must match the key:value pair in the podTemplate's labels field.
 - podTemplate contains the container manifest that defines the container configuration. The manifest is itself contained within a desiredState object.
- labels: Arbitrary（任意的） key:value pairs used to target or group this replication controller. These labels are not associated with the replicaSelector field or the podTemplate's labels field.
(label标签是为Replication controller分组用的，跟pod没关系)

#### Manifest

Manifest部分的内容不再赘述（所包含字段，是否必须，以及其意义），可以参见文档

#### Sample file

    {
      "id": "frontend-controller",
      "kind": "ReplicationController",
      "apiVersion": "v1beta1",
      "desiredState": {
        "replicas": 2,
        "replicaSelector": {"name": "frontend"},
        "podTemplate": {
          "desiredState": {
            "manifest": {
              "version": "v1beta1",
              "id": "frontendController",
              "containers": [{
                "name": "php-redis",
                "image": "dockerfile/redis",
                "ports": [{"containerPort": 80, "hostPort": 8000}]
              }]
            }
          },
          "labels": {"name": "frontend"}
        }},
      "labels": {"name": "serving"}
    }
    
### Updating replication controller pods

Google Container Engine provides a rolling update mechanism for replication controllers. The rolling update works as follows:

- A new replication controller is created, according to the specifications in the configuration file.

- The replica count on the new and old controllers is increased/decreased by one respectively until the desired number of replicas is reached.

If the number of replicas in the new controller is greater than the original, the old controller is first resized to 0 (by increasing/decreasing replicas one at a time), then the new controller is resized to the final desired size.

If the number of replicas in the new controller is less than the original, the controllers are resized until the new one has reached its final desired replica count. Then, the original controller is resized to 0 to delete the remaining pods.

    $ kubectl rollingupdate NAME -f FILE \
        [--poll-interval DURATION] \
        [--timeout DURATION] \
        [--update-period DURATION]
        
Required fields are:

- NAME: The name of the replication controller to update.
- -f FILE: A replication controller configuration file, in either JSON or YAML format. The configuration file must specify a new top-level id value and include at least one of the existing replicaSelector key:value pairs.

Optional fields are:

- --poll-interval DURATION: The time between polling the controller status after update. Valid units are ns (nanoseconds), us or µs (microseconds), ms (milliseconds), s (seconds), m (minutes), or h (hours). Units can be combined (e.g. 1m30s). The default is 3s.
- --timeout DURATION: The maximum time to wait for the controller to update a pod before exiting. Default is 5m0s. Valid units are as described for --poll-interval above.
- --update-period DURATION: The time to wait between updating pods. Default is 1m0s. Valid units are as described for --poll-interval above.

If the timeout duration is reached during a rolling update, the operation will fail with some pods belonging to the new replication controller, and some to the original controller. In this case, you should retry using the same command, and rollingupdate will pick up where it left off.
    
(更新过程有很多时间限制，如果更新失败，下一次更新命令将继续完成上一次遗留的工作)

### Resizing a replication controller

    $ kubectl --replicas COUNT resize rc NAME \
        [--current-replicas COUNT] \
        [--resource-version VERSION]
        
Required fields are:

- NAME: The name of the replication controller to update.
- --replicas COUNT: The desired number of replicas.

Optional fields are:

- --current-replicas COUNT: A precondition for current size. If specified, the resize will only take place if the current number of replicas matches this value.
- --resource-version VERSION: A precondition for resource version. If specified, the resize will only take place if the current resource version matches this value. Resource versions are specified in a resource's labels field, as a key:value pair with a key of version. For example:

        "labels": {
          "version": "canary"
        }
        
### Viewing replication controllers

     $ kubectl get rc
     
A successful get command returns all replication controllers on the specified cluster in the specified zone (cluster names may be re-used in different zones). For example:

    CONTROLLER            CONTAINER(S)   IMAGE(S)           SELECTOR        REPLICAS
    frontend-controller   php-redis      dockerfile/redis   name=frontend   2
    
You can also use `kubectl get rc NAME` to return information about a specific replication controller.

To view detailed information about a specific replication controller, use the container kubectl describe sub-command:

    $ kubectl describe rc NAME
A successful describe request returns details about the replication controller:

    Name:        frontend-controller
    Image(s):    dockerfile/redis
    Selector:    name=frontend
    Labels:      name=serving
    Replicas:    2 current / 2 desired
    Pods Status: 2 Running / 0 Waiting / 0 Succeeded / 0 Failed
    No events.
    
### Deleting replication controllers

To delete a replication controller as well as the pods that it controls, use the container kubectl stop command:

    $ kubectl stop rc NAME
    
**The kubectl stop resizes the controller to zero before deleting it.**
To delete a replication controller without deleting its pods, use container kubectl delete:

    $ kubectl delete rc NAME
    
A successful delete request returns the name of the deleted resource.