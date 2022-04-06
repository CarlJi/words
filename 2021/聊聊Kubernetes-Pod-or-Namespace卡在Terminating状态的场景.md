这个话题，想必玩过kubernetes的同学当不陌生，我会分Pod和Namespace分别来谈。

## 开门见山，为什么Pod会卡在Terminationg状态？

一句话，本质是API Server虽然标记了对象的删除，但是作为实际清理的控制器kubelet， 并不能关停Pod或相关资源, 因而没能通知API Server做实际对象的清理。

原因何在？要解开这个原因，我们先来看Pod Terminating的基本流程:

1. 客户端(比如kubectl)提交删除请求到API Server
	* 可选传递 --grace-period 参数
2. API Server接受到请求之后，做 Graceful Deletion 检查
	* 若需要 graceful 删除时，则更新对象的 metadata.deletionGracePeriodSeconds和metadata.deletionTimestamp字段。这时候describe查看对象的话，会发现其已经变成Terminating状态了
3. Pod所在的节点，kubelet检测到Pod处于Terminating状态时，就会开启Pod的真正删除流程
	* 如果Pod中的容器有定义preStop hook事件，那kubelet会先执行这些容器的hook事件
	* 之后，kubelet就会Trigger容器运行时发起`TERM`signal 给该Pod中的每个容器
4. 在Kubelet开启Graceful Shutdown的同时，Control Plane也会从目标Service的Endpoints中摘除要关闭的Pod。ReplicaSet和其他的workload服务也会认定这个Pod不是个有效副本了。同时，Kube-proxy 也会摘除这个Pod的Endpoint，这样即使Pod关闭很慢，也不会有流量再打到它上面。
5. 如果容器正常关闭那很好，但如果在grace period 时间内，容器仍然运行，kubelet会开始强制shutdown。容器运行时会发送`SIGKILL`信号给Pod中所有运行的进程进行强制关闭
6. 注意在开启Pod删除的同时，kubelet的其它控制器也会处理Pod相关的其他资源的清理动作，比如Volume。**而待一切都清理干净之后**，Kubelet才通过把Pod的grace period时间设为0来通知API Server强制删除Pod对象。

> 参考链接: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination

只有执行完第六步，Pod的API对象才会被真正删除。那怎样才认为是**"一切都清理干净了"**呢？我们来看源码:

``` go
// PodResourcesAreReclaimed returns true if all required node-level resources that a pod was consuming have
// been reclaimed by the kubelet.  Reclaiming resources is a prerequisite to deleting a pod from theAPI Server.
func (kl *Kubelet) PodResourcesAreReclaimed(pod *v1.Pod, status v1.PodStatus) bool {
	if kl.podWorkers.CouldHaveRunningContainers(pod.UID) {
		// We shouldn't delete pods that still have running containers
		klog.V(3).InfoS("Pod is terminated, but some containers are still running", "pod", klog.KObj(pod))
		return false
	}
	if count := countRunningContainerStatus(status); count > 0 {
		// We shouldn't delete pods until the reported pod status contains no more running containers (the previous
		// check ensures no more status can be generated, this check verifies we have seen enough of the status)
		klog.V(3).InfoS("Pod is terminated, but some container status has not yet been reported", "pod", klog.KObj(pod), "running", count)
		return false
	}
	if kl.podVolumesExist(pod.UID) && !kl.keepTerminatedPodVolumes {
		// We shouldn't delete pods whose volumes have not been cleaned up if we are not keeping terminated pod volumes
		klog.V(3).InfoS("Pod is terminated, but some volumes have not been cleaned up", "pod", klog.KObj(pod))
		return false
	}
	if kl.kubeletConfiguration.CgroupsPerQOS {
		pcm := kl.containerManager.NewPodContainerManager()
		if pcm.Exists(pod) {
			klog.V(3).InfoS("Pod is terminated, but pod cgroup sandbox has not been cleaned up", "pod", klog.KObj(pod))
			return false
		}
	}

	// Note: we leave pod containers to be reclaimed in the background since dockershim requires the
	// container for retrieving logs and we want to make sure logs are available until the pod is
	// physically deleted.

	klog.V(3).InfoS("Pod is terminated and all resources are reclaimed", "pod", klog.KObj(pod))
	return true
}
```
> 源码位置: https://github.com/kubernetes/kubernetes/blob/1f2813368eb0eb17140caa354ccbb0e72dcd6a69/pkg/kubelet/kubelet_pods.go#L923

是不是很清晰？总结下来就三个原因：

1. Pod里没有Running的容器
2. Pod的Volume也清理干净了
3. Pod的cgroup设置也没了

如是而已。

自然，其反向对应的就是各个异常场景了。我们来细看：

* 容器停不掉 - 这种属于CRI范畴，常见的一般使用docker作为容器运行时。笔者就曾经遇到过个场景，用`docker ps` 能看到目标容器是`Up`状态，但是执行`docker stop or rm` 却没有任何反应，而执行`docker exec`，会报`no such container`的错误。也就是说此时这个容器的状态是错乱的，docker自己都没法清理这个容器，可想而知kubelet更是无能无力。workaround恢复操作也简单，此时我只是简单的重启了下docker，目标容器就消失了，Pod的卡住状态也很快恢复了。当然，若要深究，就需要看看docker侧，为何这个容器的状态错乱了。
	* 更常见的情况是出现了僵尸进程，对应容器清理不了，Pod自然也会卡在Terminating状态。此时要想恢复，可能就只能重启机器了。
* Volume清理不了 - 我们知道在PV的"两阶段处理流程中"，Attach&Dettach由Volume Controller负责，而Mount&Unmount则是kubelet要参与负责。笔者在日常中有看到一些因为自定义CSI的不完善，导致kubelet不能Unmount Volume，从而让Pod卡住的场景。所以我们在日常开发和测试自定义CSI时，要小心这一点。
* cgroups没删除 - 启用QoS功能来管理Pod的服务质量时，kubelet需要为Pod设置合适的cgroup level，而这是需要在相应的位置写入合适配置文件的。自然，这个配置也需要在Pod删除时清理掉。笔者日常到是没有碰到过cgroups清理不了的场景，所以此处暂且不表。

现实中导致Pod卡住的细分场景可能还有很多，但不用担心，其实多数情况下通过查看kubelet日志都能很快定位出来的。之后顺藤摸瓜，恢复方案也大多不难。

> 当然还有一些系统级或者基础设施级异常，比如kubelet挂了，节点访问不了API Server了，甚至节点宕机等等，已经超过了kubelet的能力范畴，不在此讨论范围之类。

> 还有个注意点，如果你发现kubelet里面的日志有效信息很少，要注意看是不是Log Level等级过低了。从源码看，很多更具体的信息，是需要大于等于3级别才输出的。

## 那Namespace卡在Terminating状态的原因是啥？

显而易见，删除Namespace意味着要删除其下的所有资源，而如果其中Pod删除卡住了，那Namespace必然也会卡在Terminating状态。

除此之外，结合日常使用，笔者发现CRD资源发生删不掉的情况也比较高。这是为什么呢？至此，那就不得不聊聊 Finalizers机制了。

官方有篇博客专门讲到了这个，里面有个实验挺有意思。随便给一个configmap，加上个finalizers字段之后，然后使用`kubectl delete`删除它就会发现，直接是卡住的，kubernetes自身永远也删不了它。

> 参考: https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/#understanding-finalizers

原因何在？

原来Finalizers在设计上就是个pre-delete的钩子，其目的是让相关控制器有机会做自定义的清理动作。通常控制器在清理完资源后，会将对象的finalizers字段清空，然后kubernetes才能接着删除对象。而像上面的实验，没有相关控制器能处理我们随意添加的finalizers字段,那对象当然会一直卡在Terminating状态了。

自己开发CRD及Controller，因成熟度等因素，发生问题的概率自然比较大。除此之外，引入webhook(mutatingwebhookconfigurations/validatingwebhookconfigurations)出问题的概率也比较大，日常也要比较注意。

综合来看，遇Namespace删除卡住的场景，笔者认为，基本可以按以下思路排查：

1. `kubectl get ns $NAMESPACE -o yaml`， 查看`conditions`字段，看看是否有相关信息
2. 如果上面不明显，那就可以具体分析空间下，还遗留哪些资源，然后做更针对性处理
   *  参考命令: `kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n $NAMESPACE
`

找准了问题原因，然后做相应处理，kubernetes自然能够清理对应的ns对象。不建议直接清空ns的finalizers字段做强制删除，这会引入不可控风险。

> 参考: https://github.com/kubernetes/kubernetes/issues/60807#issuecomment-524772920

### 相关阅读

前同事也有几篇关于kubernetes资源删除的文章，写的非常好，推荐大家读读：

* https://zhuanlan.zhihu.com/p/164601470
* https://zhuanlan.zhihu.com/p/161072336
		


