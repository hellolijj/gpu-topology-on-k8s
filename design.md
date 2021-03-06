---
title: "Gpu Topolgy Scheduler Design"
date: 2019-06-27T21:00:04+08:00
draft: false
---

gsoc 设计文档更新版。

## 目标

当一个机器学习任务使用到 n 块 gpu 卡时，可根据 gpu 之间的拓扑，选择亲和性强的 gpu 完成任务。

## 非目标

节点内部的 亲和性除了 gpu 与 gpu 之间的亲和性外，还有 gpu 与 cpu 之间的亲和性。gpu 与 cpu 的亲和性不在此次考虑范围内。

## 约定

当只有一块gpu的时候，不考虑 gpu 拓扑结构。也就是说不存在 `map[0][0]` 这种情况

## 思路

### 1. device 上报 gpu topology

#### 1.1 nvml 获取 gpu topology 底层信息汇总

在 linux 机器中，可通过  `nvidia-smi topo -m` 获取机器的 gpu 拓扑信息。

![gpu topology on machine](./imgs/gpu_topology_on_machine.png)

缩写的对应关系如下

| P2PLinkType | P2PLinkTypeDesc | 缩写 | 带宽权值 |
| --- | --- | --- | --- |
| sdfP2PLinkCrossCPU | Cross CPU socket | SYS |  n |
| sdP2PLinkSameCPU | Same CPU socket | NODE | n |
| P2PLinkHostBridge | Host PCI bridge | PHB | n |
| P2PLinkMultiSwitch | Multiple PCI switches | PXB | n |
| P2PLinkSingleSwitch | Single PCI switch | PIX | n |
| P2PLinkSameBoard | Same board | PSB | n |
| SingleNVLINKLink |  | NV1 |  |
| TwoNVLINKLinks |  | NV2 | |
| ThreeNVLINKLinks |  | NV3 | |
| FourNVLINKLinks |  | NV4 | |


TODO: 这里带宽权重的设置需要找到依据。

参考文档：

[k8s-device-plugin nvidia包](https://github.com/NVIDIA/k8s-device-plugin/blob/master/vendor/github.com/NVIDIA/gpu-monitoring-tools/bindings/go/nvml/nvml.go#L101)

[gpu-monitoring-tools](https://github.com/NVIDIA/gpu-monitoring-tools/blob/master/bindings/go/dcgm/topology.go#L17)

[浅析GPU通信技术（中）-NVLink](https://yq.aliyun.com/articles/599183)

#### 1.2 通过 nvml 包获取 节点中 gpu topology 信息

device-plugin 在初始化的时候，通过 nvml 包获取 节点中 gpu 的信息，包括 gpu topoloy。 

设计 gpu topology 的数据结构如下：

```
type gpuTopologyType nvml.P2PLinkType

type gpuTopology map[uint]map[uint]gpuTopologyType

func (t gpuTopologyType)String() string  {
	return nvml.P2PLinkType(t).String()
}

// 例如：如下格式表示gpuTopology
map[0][0] = 0
```

#### 1.3 device-plugin 上报 gpu topology 给 node

使用 node annotation 字段表示 gpu tology

如： GPU_SYS_0_1: Cross CPU socket

> 这里的 SYS 来自于 Cross CPU socket 对应的缩写（详情参见上表）

#### 1.4 listAndWatch 上报自定义资源字段

device-plugin 在 listAndWatch 的过程 上传资源类型 "aliyun.com/gpu"

### 2. scheduler extender 根据 gpu topology 调度

#### 2.1 scheduler extender 的配置

```
{
  "kind": "Policy",
  "apiVersion": "v1",
  "extenders": [
    {
      "urlPrefix": "http://127.0.0.1:32743/gputopology-scheduler",
      "PrioritizeVerb" : "sort",
      "bindVerb":   "bind",
      "enableHttps": false,
      "nodeCacheCapable": true,
      "managedResources": [
        {
          "name": "aliyun.com/gpu",
          "ignoredByScheduler": false
        }
      ],
      "ignorable": false
    }
  ]
}
```

说明：

- scheduler extender 不需要 filter 过程，gpu topology 的 预选过程是通过，“节点中的gpu-count” 是否满足调节来预选。这个过程在默认调度器中完成。
- 新增 PrioritizeVerb 过程。此过程计算每个节点上的 满足gpu数量条件的 gpu 亲和性打分（即，计算每个节点上gpu 亲和性最好的方案对应的分数）。
- bind 过程。在符合条件，且最高分数（经过预选、优选过程）的 node 节点上，选择最优 gpu 组合方案，写入到 annotation上，完成绑定，更新pod的annotation 字段。

参考 [scheduler extender](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md) 的设计

#### 2.2 scheduler extender 打分方案

优选过程可以拆分为一下两个步骤

- 选择最佳gpu组合设计
- 计算最佳组合的分数设计


##### 2.2.1 gpu 最佳亲和性选择

gpu 最佳亲和性方案如下：

- 当 请求 aliyun.com/gpu-count 为 1 时，选择一个“远离 gpu 群”的未经使用到gpu。

算法：
```
for gpu-sets 
    if gpu not used {
      计算 该 gpu 到 所有 gpu 的距离只和
    }
    找最长的路径和
    if 有两个相同的gpu和
    计算 cpu 亲和性，选择cpu亲和性最好的那个。
endfor
```

- 当 请求 aliyun.com/gpu-count 大于等于 2 时，先选择一个路径最短的两个gpu, 然后通过图的最短生成树方案（改良过的 prim 算法）得出最佳 gpu 组合方案（device list）。

伪代码如下：

```
// req == 1, 随机返回device id
if req == 1 {
	for:  device list 列表
		if device 没有使用
			选择该 device 
			return
}

// 获取最短路径
for gputopogy 二维数组
	if gpu1 is used 
		continue;
	for gpu1 连接 的设备
		if gpu 2 is used
		    continue
		if getToploge(gpu1, gpu2) < min {
			min = getToploge(gpu1, gpu2)
			最短的两个gpu节点作为
		}
		    
将选择的两个节点加入到 ids
if req == 2 {
	return
}


// 寻找接下来的点。
for 遍历所有没有使用过，未经选择的节点
	选择出到 ids 距离和最短到节点，加入到 ids

如此循环直到找到满足请求的 gpu-count 个数为止。

```

> 此方案有个问题：当选择最短两个节点时，出现了两个距离相等的方案。默认选择后者，这时候可能在选择第三个device时，到另外一种方案的距离更短。

> 例如：有[0,1,2,3,4]5个节点。发现，最短2个device方案有[0,1] 或者 [0,2]。在选择第三个方案的时候，发现 节点3，到[0,1] 的距离更短。而这个时候可能已经确定了[0,2]作为两个device最短方案。

#### 2.2.2 计算最佳组合的分数设计

每种gpu link 关系赋值如下表：

| P2PLinkType | P2PLinkTypeDesc | mark |
| --- | --- | --- |
| sdfP2PLinkCrossCPU | Cross CPU socket | 1 |
| sdP2PLinkSameCPU | Same CPU socket | 2 |
| P2PLinkHostBridge | Host PCI bridge | 3 |
| P2PLinkMultiSwitch | Multiple PCI switches | 4 |
| P2PLinkSingleSwitch | Single PCI switch | 5 |
| P2PLinkSameBoard | Same board | 6 |

分数计算方式如下：

```
10 - 10 * sum(topology) / 6 * len(topology)
```

对所有的gpu连接路径求和 再作归一化处理，再成以最大值 10， 再被10 减去。

例如：有选择了4个device， 它们之间的拓扑关系是：1，1，1，2，3，3

```
计算结果为：10*[1-(1+1+1+2+3+3)/6*6]
```


[打分方案参照腾讯论文](./reference/gaia_gpu_topology/gaia_gpu_topology_scheduler.md)


### 2.3 scheduler bind 方案

bind 的过程 将 最佳gpu组合方案，写入到 pod 的annotation 里。 同时记录分配时间ALIYUN_COM_GPU_ASSUME_TIME、是否实际分配到节点 ALIYUN_COM_GPU_ASSIGNED。

例如：
```
ALIYUN_COM_GPU_ASSIGNED: false
ALIYUN_COM_GPU_ASSUME_TIME: 1561717704
ALIYUN_COM_GPU_GROUP: 0,1,2,3
```

同时更新 pod 的annotion， 在 device 上记录运行的 pod 信息。

## 3. device plugin allocate 设计


allocate 获取分配的device 列表后，穿入环境变量 NVIDIA_VISIBLE_DEVICES 到nvidia-docker容器里，容器的任务会根据这个变量将任务运行在指定的GPU里。

同时也会更新 pod annotion 字段信息

```
ALIYUN_COM_GPU_ASSIGNED: true
ALIYUN_COM_GPU_ASSUME_TIME: 1561718702
```


## 流程

根据以上思路，可作下图简单描述整个流程

![gpu topology on k8s](./imgs/gpu_topology_on_k8s.png)

todo: 将自己做的工作用不同颜色标记出来，且贴上原来设计图的链接