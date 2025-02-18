### 性能工程手册<Badge type="tip" text="linux" />

##### 最大延迟预算

多个服务串联时，每个服务都有最大延迟预算，总和不能超过总目标延迟。

##### 所需知识和技能

![image-20241127095644630](/image-20241127095644630.png)

##### 性能问题解决思路

![image-20241127100416195](/image-20241127100416195.png)

##### 统计知识

贝叶斯定理

置信区间

点估计理论

数据分布模型

排队理论

##### 如何分析性能数据？

数据的因果关系

数据的大小和趋势

数据有效性

性能数据之间的关系

##### 如何展示性能数据？

确定展示的目的和对象

根据展示目的，确定展示所需要的图形

##### 性能数据估计

**磁盘**

![image-20241128093159731](/image-20241128093159731.png)

**缓存**

![image-20241128094644718](/image-20241128094644718.png)

**网络**

![image-20241128095436132](/image-20241128095436132.png)

##### 性能测试的种类

**根据流量大小划分**

![image-20241128100418449](/image-20241128100418449.png)

##### 如何规划性能测试？

![image-20241128101259130](/image-20241128101259130.png)

1. 搞清楚测试对象
2. 确定性能指标，常见的有响应时间，吞吐量，利用率。
3. 决定测试指标度量，平均值，百分位数等。
4. 确定性能测试的期望结果
5. 性能测试规划
   1. 负载如何注入
   2. 黑盒还是白盒
   3. 测试工具选择
   4. 测试环境搭建

6. 执行测试
   1. 分步进行
   2. 先短时间测试，再长时间测试
   3. 模拟测试，进行预先检查工具正确性
7. 分析测试结果

##### 测试工具选择

![image-20241129100505769](/image-20241129100505769.png)

##### 如何保证测试的可靠性？

测试规划

测试进行

测试分析

##### 如何将性能测试集成到开发流程中？

![image-20241202133051405](/image-20241202133051405.png)

##### 性能分析概述

主要指标

- 服务延迟
- 吞吐率
- 资源使用率

主要性能瓶颈

![image-20241203104132017](/image-20241203104132017.png)

##### CPU相关性能分析

![image-20241203114325360](/image-20241203114325360.png)

内核进程调度负责分配CPU资源，调度优先级关系是硬件中断>内核进程>用户进程。

##### 内存相关性能分析

| **概念**               | **对性能的影响**                                                                                                   | **操作系统如何进行管理和优化**                                                                                                  |
|------------------------|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| **缓存命中率**          | 高缓存命中率意味着 CPU 能更快地从缓存中获取数据，减少访问主内存的延迟，提高处理速度。低缓存命中率则导致频繁的内存访问，影响性能。  | 操作系统通过智能的缓存管理策略（如 LRU 算法）来优化缓存的使用，操作系统和硬件协同工作以提高缓存命中率。                         |
| **缓存一致性**          | 在多核 CPU 中，缓存一致性确保各个核心的缓存中存储的数据是一致的，避免不同核心读取到不同版本的数据。若缓存一致性机制不完善，会导致数据不一致、错误或性能瓶颈。  | 操作系统配合硬件的缓存一致性协议（如 MESI 协议）管理缓存更新，确保各核心缓存的数据一致性，减少同步开销。                          |
| **内存带宽**            | 内存带宽越大，CPU 和内存之间的数据传输速度越快，能更快速地访问大量数据。低内存带宽限制了数据传输速度，可能成为性能瓶颈。        | 操作系统通过优化内存访问模式，减少频繁的内存访问冲突。硬件层面，如 DDR4、GDDR 等内存标准的选择直接影响内存带宽。                    |
| **内存延迟**            | 内存延迟影响 CPU 从内存获取数据的速度，低延迟可以加快数据访问，提升计算效率。高延迟则导致 CPU 等待数据返回，浪费宝贵的计算时间。  | 操作系统通过内存访问优化、数据预取等技术降低内存延迟。同时，操作系统可调整内存调度策略，优先访问“热”数据，减少延迟。               |
| **内存使用大小及碎片**   | 内存使用过大或内存碎片过多可能导致系统内存不足，频繁交换数据到硬盘，影响性能。内存碎片化还可能导致内存浪费，降低可用内存。    | 操作系统通过内存分配算法（如伙伴系统、区块分配）管理内存，避免碎片化。通过垃圾回收、内存压缩等方式优化内存使用。                   |
| **内存的分配和回收速度** | 内存分配和回收速度影响程序的启动、运行及资源管理效率。分配慢会导致程序启动延迟，回收慢会造成内存泄漏或过多的内存占用。       | 操作系统通过内存管理策略（如虚拟内存、堆栈管理、内存池）进行内存分配和回收。高效的内存分配算法（如 SLAB、Tcmalloc）可以优化这一过程。|

##### 存储系统相关性能分析

主要指标

- IOPS
- 访问延迟
- 吞吐率/带宽

![image-20241203134357720](/image-20241203134357720.png)

##### 网络性能分析

主要指标

- 可用性
- 响应时间
- 网络带宽
- 网络吞吐量
- 网络利用率

![image-20241203135811934](/image-20241203135811934.png)

##### 调优原则

![image-20241203143429054](/image-20241203143429054.png)

![image-20241203143544699](/image-20241203143544699.png)

##### 如何进行容量测试？



