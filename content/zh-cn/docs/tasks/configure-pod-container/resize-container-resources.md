---
title: 调整分配给容器的 CPU 和内存资源
content_type: task
weight: 30
min-kubernetes-server-version: 1.27
---
<!--
title: Resize CPU and Memory Resources assigned to Containers
content_type: task
weight: 30
min-kubernetes-server-version: 1.27
-->

<!-- overview -->

{{< feature-state feature_gate_name="InPlacePodVerticalScaling" >}}

<!--
This page assumes that you are familiar with [Quality of Service](/docs/tasks/configure-pod-container/quality-service-pod/)
for Kubernetes Pods.

This page shows how to resize CPU and memory resources assigned to containers
of a running pod without restarting the pod or its containers. A Kubernetes node
allocates resources for a pod based on its `requests`, and restricts the pod's
resource usage based on the `limits` specified in the pod's containers.
-->
本页假定你已经熟悉了 Kubernetes Pod
的[服务质量](/zh-cn/docs/tasks/configure-pod-container/quality-service-pod/)。

本页说明如何在不重启 Pod 或其容器的情况下调整分配给运行中 Pod 容器的 CPU 和内存资源。
Kubernetes 节点会基于 Pod 的 `requests` 为 Pod 分配资源，
并基于 Pod 的容器中指定的 `limits` 限制 Pod 的资源使用。

<!--
Changing the resource allocation for a running Pod requires the
`InPlacePodVerticalScaling` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/)
to be enabled. The alternative is to delete the Pod and let the
[workload controller](/docs/concepts/workloads/controllers/) make a replacement Pod
that has a different resource requirement.
-->
要为正在运行的 Pod 更改资源分配量，需要启用 `InPlacePodVerticalScaling`
[特性门控](/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)。
并让[工作负载控制器](/zh-cn/docs/concepts/workloads/controllers/)创建一个具有不同资源需求的新 Pod。

<!--
A resize request is made through the pod `/resize` subresource, which takes the full updated pod for
an update request, or a patch on the pod object for a patch request.
-->
resize 请求是通过 Pod 的 `/resize` 子资源处理的。
对于 update 请求，请求体是更新后的完整 Pod；
对于 patch 请求，请求体是对 Pod 对象的补丁。

<!--
For in-place resize of pod resources:
- A container's resource `requests` and `limits` are _mutable_ for CPU
  and memory resources. These fields represent the _desired_ resources for the container.
- The `resources` field in `containerStatuses` of the Pod's status reflects the resources
  _allocated_ to the pod's containers. For running containers, this reflects the actual resource
  `requests` and `limits` that are configured as reported by the container runtime. For non-running
  containers, these are the resources allocated for the container when it starts.
-->
对于原地调整 Pod 资源而言：

- 针对 CPU 和内存资源的容器的 `requests` 和 `limits` 是**可变更的**。
  这些字段表示容器**所预期**使用的资源。
- Pod 状态中 `containerStatuses` 的 `resources` 字段反映了分配给 Pod 容器的资源。
  对于正运行的容器，这个字段反映了实际配置的资源 `requests` 和 `limits`，
  字段值是容器运行时所报告的。对于未运行的容器，字段值是容器启动时为其分配的资源。

<!--
- The `resize` field in the Pod's status shows the status of the last requested
  pending resize. It can have the following values:
  - `Proposed`: This value indicates that a pod was resized, but the Kubelet has not yet processed
    the resize.
  - `InProgress`: This value indicates that the node has accepted the resize
    request and is in the process of applying it to the pod's containers.
  - `Deferred`: This value means that the requested resize cannot be granted at
    this time, and the node will keep retrying. The resize may be granted when
    other pods are removed and free up node resources.
  - `Infeasible`: is a signal that the node cannot accommodate the requested
    resize. This can happen if the requested resize exceeds the maximum
    resources the node can ever allocate for a pod.
  - `""`: An empty or unset value indicates that the last resize completed. This should only be the
    case if the resources in the container spec match the resources in the container status.
-->
- Pod 状态中 `resize` 字段显示上次请求待处理的调整状态。此字段可以具有以下值：
  - `Proposed`：此值表示 Pod 大小的规约已被调整，但 kubelet 还未处理此调整请求。
  - `InProgress`：此值表示节点已接受调整请求，并正在将其应用于 Pod 的容器。
  - `Deferred`：此值意味着在此时无法批准请求的调整，节点将继续重试。
    当其他 Pod 被移除并释放节点资源时，调整可能会被真正实施。
  - `Infeasible`：此值是一种信号，表示节点无法承接所请求的调整值。
    如果所请求的调整超过节点可分配给 Pod 的最大资源，则可能会发生这种情况。
  - `""`：留空或不设置，表示上一次调整已完成。
    只有容器规约中的资源与容器状态中的资源相匹配时，才会是这种情况。

<!--
If a node has pods with an incomplete resize, the scheduler will compute the pod requests from the
maximum of a container's desired resource requests, and it's actual requests reported in the status.
-->
如果节点有一些 Pod 还未完成调整，调度器将基于容器的预期资源请求的最大值来计算 Pod 请求，
它是状态中所报告的实际请求。

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

<!--
The `InPlacePodVerticalScaling` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) must be enabled
for your control plane and for all nodes in your cluster.
-->
你必须在控制平面和集群中的所有节点上启用 `InPlacePodVerticalScaling`
[特性门控](/zh-cn/docs/reference/command-line-tools-reference/feature-gates/)。

<!--
## Container Resize Policies

Resize policies allow for a more fine-grained control over how pod's containers
are resized for CPU and memory resources. For example, the container's
application may be able to handle CPU resources resized without being restarted,
but resizing memory may require that the application hence the containers be restarted.
-->
## 容器调整策略   {#container-resize-policies}

调整策略允许更精细地控制 Pod 中的容器如何针对 CPU 和内存资源进行调整。
例如，容器的应用程序可以处理 CPU 资源的调整而不必重启，
但是调整内存可能需要应用程序重启，因此容器也必须重启。

<!--
To enable this, the Container specification allows users to specify a `resizePolicy`.
The following restart policies can be specified for resizing CPU and memory:
* `NotRequired`: Resize the container's resources while it is running.
* `RestartContainer`: Restart the container and apply new resources upon restart.

If `resizePolicy[*].restartPolicy` is not specified, it defaults to `NotRequired`.
-->
为了实现这一点，容器规约允许用户指定 `resizePolicy`。
针对调整 CPU 和内存可以设置以下重启策略：

- `NotRequired`：在运行时调整容器的资源。
- `RestartContainer`：重启容器并在重启后应用新资源。

如果未指定 `resizePolicy[*].restartPolicy`，则默认为 `NotRequired`。

{{< note >}}
<!--
If the Pod's `restartPolicy` is `Never`, container's resize restart policy must be
set to `NotRequired` for all Containers in the Pod.
-->
如果 Pod 的 `restartPolicy` 为 `Never`，则 Pod 中所有容器的调整重启策略必须被设置为 `NotRequired`。
{{< /note >}}

<!--
Below example shows a Pod whose Container's CPU can be resized without restart, but
resizing memory requires the container to be restarted.
-->
下面的示例显示了一个 Pod，其中 CPU 可以在不重启容器的情况下进行调整，但是内存调整需要重启容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-5
  namespace: qos-example
spec:
  containers:
    - name: qos-demo-ctr-5
      image: nginx
      resizePolicy:
        - resourceName: cpu
          restartPolicy: NotRequired
        - resourceName: memory
          restartPolicy: RestartContainer
      resources:
        limits:
          memory: "200Mi"
          cpu: "700m"
        requests:
          memory: "200Mi"
          cpu: "700m"
```

{{< note >}}
<!--
In the above example, if desired requests or limits for both CPU _and_ memory
have changed, the container will be restarted in order to resize its memory.
-->
在上述示例中，如果所需的 CPU 和内存请求或限制已更改，则容器将被重启以调整其内存。
{{< /note >}}

<!-- steps -->

<!--
## Limitations

In-place resize of pod resources currently has the following limitations:
-->
## 限制   {#limitations}

目前就地调整 Pod 资源大小存在以下限制：

<!--
- Only CPU and memory resources can be changed.
- Pod QoS Class cannot change. This means that requests must continue to equal limits for Guaranteed
  pods, Burstable pods cannot set requests and limits to be equal for both CPU & memory, and you
  cannot add resource requirements to Best Effort pods.
- Init containers and Ephemeral Containers cannot be resized.
- Resource requests and limits cannot be removed once set.
- A container's memory limit may not be reduced below its usage. If a request puts a container in
  this state, the resize status will remain in `InProgress` until the desired memory limit becomes
  feasible.
- Windows pods cannot be resized.
-->
- 只能更改 CPU 和内存资源。
- Pod QoS 类不能更改。这意味着 Guaranteed Pod 的 requests 必须继续等于其 limits，
  Burstable Pod 对于 CPU 和内存的 requests 不能设为等于其 limits，
  并且你不能给 BestEffort Pod 添加资源要求。
- Init 容器和临时容器不能调整大小。
- 资源请求和限制一旦被设置就不能移除。
- 容器的内存限制不能降到比其当前用量更低的值。如果某请求将容器置于此状态，
  调整的状态将停留在 `InProgress`，直到预期的内存限制成为可行为止。
- Windows Pod 不能被调整大小。

<!--
## Create a pod with resource requests and limits

You can create a Guaranteed or Burstable [Quality of Service](/docs/tasks/configure-pod-container/quality-service-pod/)
class pod by specifying requests and/or limits for a pod's containers.

Consider the following manifest for a Pod that has one Container.
-->
## 创建具有资源请求和限制的 Pod    {#create-pod-with-resource-requests-and-limits}

你可以通过为 Pod 的容器指定请求和/或限制来创建 Guaranteed 或 Burstable
[服务质量](/zh-cn/docs/tasks/configure-pod-container/quality-service-pod/)类的 Pod。

考虑以下包含一个容器的 Pod 的清单。

{{% code_sample file="pods/qos/qos-pod-5.yaml" %}}

<!--
Create the pod in the `qos-example` namespace:
-->
在 `qos-example` 名字空间中创建该 Pod：

```shell
kubectl create namespace qos-example
kubectl create -f https://k8s.io/examples/pods/qos/qos-pod-5.yaml --namespace=qos-example
```

<!--
This pod is classified as a Guaranteed QoS class requesting 700m CPU and 200Mi
memory.

View detailed information about the pod:
-->
此 Pod 被分类为 Guaranteed QoS 类，请求 700m CPU 和 200Mi 内存。

查看有关 Pod 的详细信息：

```shell
kubectl get pod qos-demo-5 --output=yaml --namespace=qos-example
```

<!--
Also notice that the values of `resizePolicy[*].restartPolicy` defaulted to
`NotRequired`, indicating that CPU and memory can be resized while container
is running.
-->
另请注意，`resizePolicy[*].restartPolicy` 的值默认为 `NotRequired`，
表示可以在容器运行的情况下调整 CPU 和内存大小。

```yaml
spec:
  containers:
    ...
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: NotRequired
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
...
  containerStatuses:
...
    name: qos-demo-ctr-5
    ready: true
...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
    restartCount: 0
    started: true
...
  qosClass: Guaranteed
```

<!--
## Updating the pod's resources

Let's say the CPU requirements have increased, and 0.8 CPU is now desired. This may
be specified manually, or determined and programmatically applied by an entity such as
[VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#readme) (VPA).
-->
## 更新 Pod 的资源   {#updating-pod-resources}

假设要求的 CPU 需求已上升，现在需要 0.8 CPU。这可以手动指定，或由如
[VerticalPodAutoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#readme) (VPA)
这样的实体确定并可能以编程方式应用。

{{< note >}}
<!--
While you can change a Pod's requests and limits to express new desired
resources, you cannot change the QoS class in which the Pod was created.
-->
尽管你可以更改 Pod 的请求和限制以表示新的期望资源，
但无法更改 Pod 创建时所归属的 QoS 类。
{{< /note >}}

<!--
Now, patch the Pod's Container with CPU requests & limits both set to `800m`:
-->
现在对 Pod 的 Container 执行 patch 命令，将容器的 CPU 请求和限制均设置为 `800m`：

```shell
kubectl -n qos-example patch pod qos-demo-5 --subresource resize --patch '{"spec":{"containers":[{"name":"qos-demo-ctr-5", "resources":{"requests":{"cpu":"800m"}, "limits":{"cpu":"800m"}}}]}}'
```

<!--
Query the Pod's detailed information after the Pod has been patched.
-->
在 Pod 已打补丁后查询其详细信息。

```shell
kubectl get pod qos-demo-5 --output=yaml --namespace=qos-example
```

<!--
The Pod's spec below reflects the updated CPU requests and limits.
-->
以下 Pod 规约反映了更新后的 CPU 请求和限制。

```yaml
spec:
  containers:
    ...
    resources:
      limits:
        cpu: 800m
        memory: 200Mi
      requests:
        cpu: 800m
        memory: 200Mi
...
  containerStatuses:
...
    resources:
      limits:
        cpu: 800m
        memory: 200Mi
      requests:
        cpu: 800m
        memory: 200Mi
    restartCount: 0
    started: true
```

<!--
Observe that the `resources` in the `containerStatuses` have been updated to reflect the new desired
CPU requests. This indicates that node was able to accommodate the increased CPU resource needs,
and the new CPU resources have been applied. The Container's `restartCount` remains unchanged,
indicating that container's CPU resources were resized without restarting the container.
-->
观察到 `allocatedResources` 的值已更新，反映了新的预期 CPU 请求。
这表明节点能够容纳提高后的 CPU 资源需求，而且新的 CPU 资源已经被应用。
Container 的 `restartCount` 保持不变，表示已在无需重启容器的情况下调整了容器的 CPU 资源。

<!--
## Clean up

Delete your namespace:
-->
## 清理   {#clean-up}

删除你的名字空间：

```shell
kubectl delete namespace qos-example
```

## {{% heading "whatsnext" %}}

<!--
### For application developers

* [Assign Memory Resources to Containers and Pods](/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Assign CPU Resources to Containers and Pods](/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [Assign Pod-level CPU and memory resources](/docs/tasks/configure-pod-container/assign-pod-level-resources/)
-->
### 对于应用开发人员

* [为容器和 Pod 分配内存资源](/zh-cn/docs/tasks/configure-pod-container/assign-memory-resource/)

* [为容器和 Pod 分配 CPU 资源](/zh-cn/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [分配 Pod 级别的 CPU 和内存资源](/zh-cn/docs/tasks/configure-pod-container/assign-pod-level-resources/)

<!--
### For cluster administrators

* [Configure Default Memory Requests and Limits for a Namespace](/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [Configure Default CPU Requests and Limits for a Namespace](/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [Configure Minimum and Maximum Memory Constraints for a Namespace](/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [Configure Minimum and Maximum CPU Constraints for a Namespace](/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [Configure Memory and CPU Quotas for a Namespace](/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)
-->
### 对于集群管理员

* [为名字空间配置默认内存请求和限制](/zh-cn/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [为名字空间配置默认 CPU 请求和限制](/zh-cn/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [为名字空间配置最小和最大内存约束](/zh-cn/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [为名字空间配置最小和最大 CPU 约束](/zh-cn/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [为名字空间配置内存和 CPU 配额](/zh-cn/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)
