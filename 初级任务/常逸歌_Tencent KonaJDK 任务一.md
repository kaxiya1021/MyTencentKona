# **Tencent** KonaJDK 任务一

## 任务要求

写一个测试用例，通过不同的GC参数（Serial GC，Parallel Scavenge，G1GC，ZGC，Shenandoah GC），通过打印GC日志完整的展示GC的各个阶段 (比如，统一的 –Xmx –Xms)

展示：GC暂停时间，测试完成时间，GC吞吐率

对于这些concurrent GC来说，会存在GC的回收无法赶上分配的阶段

- G1GC: To-space exhausted -> Full GC

- ZGC: Allocation Stall

- Shenandoah GC: Pacing -> Degenerated GC -> Full GC



## JDK编译和构建

使用Ubuntu22.04虚拟机进行测试：内存8G，处理器为8。

![](https://jsd.onmicrosoft.cn/gh/kaxiya1021/tuchuang@main/img/image-20240821222854646.png)

下载**[TencentKona-17.0.11](https://github.com/Tencent/TencentKona-17/releases/tag/TencentKona-17.0.11)**源码进行编译，`bash configure` 后可以看到 `JVM feature` 部分默认开启 `shenandoahgc`

![](https://jsd.onmicrosoft.cn/gh/kaxiya1021/tuchuang@main/img/image-20240823135820178.png)

执行 `make images` 构建镜像

![](https://jsd.onmicrosoft.cn/gh/kaxiya1021/tuchuang@main/img/image-20240823141303718.png)

查看镜像版本，为TencentKona-17.0.11，证明镜像构建成功

![](https://jsd.onmicrosoft.cn/gh/kaxiya1021/tuchuang@main/img/image-20240823141413891.png)

`make clean` 后重新进行 `bash configure`，不过这次禁用 `ShenandoahGC`

```shell
bash configure --with-jvm-features="-shenandoahgc"
```

编译完成后查看 `JVM feature` ，可以看到此时 `shenandoahgc`没有开启

![](https://jsd.onmicrosoft.cn/gh/kaxiya1021/tuchuang@main/img/image-20240823141527078.png)

## 测试用例和日志分析

### 一、代码程序

下面为测试 java 程序，它会频繁的进行对象分配，进而产生 GC

```java
import java.util.ArrayList;
import java.util.List;

public class GCTest {
        public static void main(String[] args) {
            List<Object> list = new ArrayList<>();

            try {
                while (true) {
                    byte[] array = new byte[10 * 1024 * 1024];
                    list.add(array);
                    Thread.sleep(10);
                }
            } catch (OutOfMemoryError e) {
                System.out.println("OutOfMemoryError: " + e.getMessage());
            } catch (InterruptedException e) {
                System.out.println("InterruptedException: " + e.getMessage());
            }
        }
}
```



### 二、参数设置

jvm参数设置如下

```
-Xms512m 
-Xmx512m 
-Xlog:gc*:file=gc.log 
```



### 三、日志分析

#### 1、Serial

**内存分配**

| **分代**                 | **分配内存** | **最大值** |
| :----------------------- | :----------- | :--------- |
| Young Generation         | 153.56 mb    | 146.5 mb   |
| Old Generation           | 341.38 mb    | 340.86 mb  |
| Meta Space               | 896 kb       | 781 kb     |
| Young + Old + Meta space | 494.88 mb    | 487.76 mb  |

**CPU时间**

| CPU 时间 | 670 毫秒 |
| :------- | -------- |
| 用户时间 | 240 毫秒 |
| 系统时间 | 430 毫秒 |

**吞吐量和GC停顿时间**

| **吞吐量**     | 58.142% |
| -------------- | ------- |
| 平均GC停顿时间 | 113 ms  |
| 最大GC停顿时间 | 240 ms  |

**GC原因**

| **Cause**                    | **Count** | **Avg Time** | **Max Time** | **Total Time** |
| :--------------------------- | :-------- | :----------- | :----------- | :------------- |
| Allocation Failure           | 3         | 143 ms       | 240 ms       | 430 ms         |
| Full GC - Allocation Failure | 3         | 83.3 ms      | 190 ms       | 250 ms         |



#### 2、Parallel

**内存分配**

| **分代**                 | **分配内存** | **最大值** |
| :----------------------- | :----------- | :--------- |
| Young Generation         | 149.5 mb     | 143.42 mb  |
| Old Generation           | 341.5 mb     | 340.49 mb  |
| Meta Space               | 896 kb       | 754 kb     |
| Young + Old + Meta space | 491.88 mb    | 473.74 mb  |

**CPU时间**

| CPU 时间 | 960 毫秒 |
| :------- | -------- |
| 用户时间 | 450 毫秒 |
| 系统时间 | 510 毫秒 |

**吞吐量和GC停顿时间**

| **吞吐量**     | 79.428% |
| -------------- | ------- |
| 平均GC停顿时间 | 38.3ms  |
| 最大GC停顿时间 | 60.0ms  |

**GC原因**

| **Cause**                    | **Count** | **Avg Time** | **Max Time** | **Total Time** |
| :--------------------------- | :-------- | :----------- | :----------- | :------------- |
| Allocation Failure           | 3         | 43.3 ms      | 50.0 ms      | 130 ms         |
| Ergonomics                   | 2         | 45.0 ms      | 60.0 ms      | 90.0 ms        |
| Full GC - Allocation Failure | 1         | 10.0 ms      | 10.0 ms      | 10.0 ms        |



#### 3、G1

**内存分配**

| **分代**                 | **分配**  | **最大** |
| :----------------------- | :-------- | :------- |
| Young Generation         | 133 mb    | 3 mb     |
| Old Generation           | 379 mb    | 2 mb     |
| Humongous                | n/a       | 506 mb   |
| Meta Space               | 1.19 mb   | 1,022 kb |
| Young + Old + Meta space | 513.19 mb | 509 mb   |

**CPU时间**

| CPU 时间 | 130 毫秒 |
| :------- | -------- |
| 用户时间 | 50 毫秒  |
| 系统时间 | 80 毫秒  |

**吞吐量和GC停顿时间**

| **吞吐量**     | 87.568% |
| -------------- | ------- |
| 平均GC停顿时间 | 2.57 ms |
| 最大GC停顿时间 | 10.0 ms |

**停顿时间**

| 平均     | 2.57 毫秒   |
| -------- | ----------- |
| 标准偏差 | 2.75 毫秒   |
| 最短     | 0.0200 毫秒 |
| 最长     | 10.0 毫秒   |

**并发时间**

| 平均     | 23.1 毫秒 |
| -------- | --------- |
| 标准偏差 | 4.36 毫秒 |
| 最短     | 9.06 毫秒 |
| 最长     | 30.7      |

**GC原因**

| **Cause**               | **Count** | **Avg Time** | **Max Time** | **Total Time** |
| :---------------------- | :-------- | :----------- | :----------- | :------------- |
| G1 Humongous Allocation | 28        | 4.20 ms      | 10.0 ms      | 117 ms         |
| G1 Compaction Pause     | 2         | 10.0 ms      | 10.0 ms      | 20.0 ms        |



#### 4、ZGC

**日志节选**

```
[1.444s][info][gc,start    ] GC(7) Garbage Collection (Allocation Stall)
[1.444s][info][gc,ref      ] GC(7) Clearing All SoftReferences
[1.444s][info][gc,task     ] GC(7) Using 2 workers
[1.444s][info][gc,phases   ] GC(7) Pause Mark Start 0.009ms
[1.448s][info][gc,phases   ] GC(7) Concurrent Mark 3.571ms
[1.448s][info][gc,phases   ] GC(7) Pause Mark End 0.014ms
[1.448s][info][gc,phases   ] GC(7) Concurrent Mark Free 0.001ms
[1.449s][info][gc,phases   ] GC(7) Concurrent Process Non-Strong References 1.624ms
[1.449s][info][gc,phases   ] GC(7) Concurrent Reset Relocation Set 0.001ms
[1.452s][info][gc,phases   ] GC(7) Concurrent Select Relocation Set 2.355ms
[1.452s][info][gc,phases   ] GC(7) Pause Relocate Start 0.010ms
[1.452s][info][gc,phases   ] GC(7) Concurrent Relocate 0.250ms
[1.452s][info][gc,load     ] GC(7) Load: 0.55/0.65/0.52
[1.452s][info][gc,mmu      ] GC(7) MMU: 2ms/98.9%, 5ms/99.3%, 10ms/99.5%, 20ms/99.8%, 50ms/99.9%, 100ms/99.9%
[1.452s][info][gc,marking  ] GC(7) Mark: 2 stripe(s), 1 proactive flush(es), 1 terminate flush(es), 0 completion(s), 0 continuation(s) 
[1.452s][info][gc,marking  ] GC(7) Mark Stack Usage: 32M
[1.452s][info][gc,nmethod  ] GC(7) NMethods: 190 registered, 0 unregistered
[1.452s][info][gc,metaspace] GC(7) Metaspace: 0M used, 0M committed, 1088M reserved
[1.452s][info][gc,ref      ] GC(7) Soft: 45 encountered, 43 discovered, 25 enqueued
[1.452s][info][gc,ref      ] GC(7) Weak: 99 encountered, 0 discovered, 0 enqueued
[1.452s][info][gc,ref      ] GC(7) Final: 0 encountered, 0 discovered, 0 enqueued
[1.452s][info][gc,ref      ] GC(7) Phantom: 18 encountered, 10 discovered, 0 enqueued
[1.452s][info][gc,reloc    ] GC(7) Small Pages: 1 / 2M, Empty: 0M, Relocated: 0M, In-Place: 0
[1.452s][info][gc,reloc    ] GC(7) Medium Pages: 0 / 0M, Empty: 0M, Relocated: 0M, In-Place: 0
[1.452s][info][gc,reloc    ] GC(7) Large Pages: 42 / 504M, Empty: 0M, Relocated: 0M, In-Place: 0
[1.452s][info][gc,reloc    ] GC(7) Forwarding Usage: 0M
[1.452s][info][gc,heap     ] GC(7) Min Capacity: 512M(100%)
[1.452s][info][gc,heap     ] GC(7) Max Capacity: 512M(100%)
[1.452s][info][gc,heap     ] GC(7) Soft Max Capacity: 512M(100%)
[1.452s][info][gc,heap     ] GC(7)                Mark Start          Mark End        Relocate Start      Relocate End           High               Low         
[1.452s][info][gc,heap     ] GC(7)  Capacity:      512M (100%)        512M (100%)        512M (100%)        512M (100%)        512M (100%)        512M (100%)   
[1.452s][info][gc,heap     ] GC(7)      Free:        6M (1%)            6M (1%)            6M (1%)            6M (1%)            6M (1%)            6M (1%)     
[1.452s][info][gc,heap     ] GC(7)      Used:      506M (99%)         506M (99%)         506M (99%)         506M (99%)         506M (99%)         506M (99%)    
[1.452s][info][gc,heap     ] GC(7)      Live:         -               505M (99%)         505M (99%)         505M (99%)            -                  -          
[1.452s][info][gc,heap     ] GC(7) Allocated:         -                 0M (0%)            0M (0%)            0M (0%)             -                  -          
[1.452s][info][gc,heap     ] GC(7)   Garbage:         -                 0M (0%)            0M (0%)            0M (0%)             -                  -          
[1.452s][info][gc,heap     ] GC(7) Reclaimed:         -                  -                 0M (0%)            0M (0%)             -                  -          
[1.452s][info][gc          ] GC(7) Garbage Collection (Allocation Stall) 506M(99%)->506M(99%)
[1.453s][info][gc          ] Allocation Stall (main) 8.945ms
[1.453s][info][gc          ] Out Of Memory (main)
[1.460s][info][gc,heap,exit] Heap
[1.460s][info][gc,heap,exit]  ZHeap           used 510M, capacity 512M, max capacity 512M
[1.460s][info][gc,heap,exit]  Metaspace       used 758K, committed 896K, reserved 1114112K
[1.460s][info][gc,heap,exit]   class space    used 58K, committed 128K, reserved 1048576K
```

在ZGC的日志中，可以看到一次由于分配延迟（Allocation Stall）触发的垃圾回收事件。

**日志分析**

日志 `[1.452s][info][gc,heap] GC(7) Free: 6M (1%)` 显示在垃圾收集开始时，堆上只有6M的空闲内存，占总容量的1%，这表明内存资源极其紧张。`[1.452s][info][gc,heap] GC(7) Used: 506M (99%)` 进一步确认了几乎所有的堆空间都已被占用。`GC(7) Pause Mark Start` 和 `Pause Mark End` 这两个短暂的停顿阶段用于标记阶段的开始和结束，耗时非常短，GC的停顿时间被有效控制。`Concurrent Mark` 和 `Concurrent Relocate` 等并发阶段表示ZGC尝试在应用线程运行时并发地完成垃圾回收任务。然而，即便是这些努力，最终在 `Allocation Stall (main) 8.945ms` 中显示，应用还是遭受了一次明显的延迟。在整个GC过程结束时，`Used: 506M (99%)` 到 `Live: 505M (99%)` 的数据几乎没有变化，表明此次GC没有能有效回收多少内存。

**原因分析**

- **堆空间不足**：ZGC堆的配置大小为512M，几乎全部被占用，留下极少的空间来处理新的内存分配请求。这种高占用率和低空闲率是造成分配延迟的直接原因。

- **内存分配速度与回收速度不匹配**：应用可能在短时间内产生大量内存分配请求，而GC线程未能及时释放足够的内存来满足这些需求，导致必须进行停顿以等待内存空间的释放。

- **大对象或频繁分配**：如果应用频繁地创建大量对象，尤其是生命周期较长的大对象，可能会迅速耗尽可用空间，增加GC压力。



**吞吐量和GC停顿时间**

| **吞吐量**     | 99.997%   |
| -------------- | --------- |
| 平均GC停顿时间 | 0.0139 ms |
| 最大GC停顿时间 | 0.0220 ms |

**时间统计**

|              | **并发标记** | **并发选择重定位集** | **并发重定位** | **并发处理非强引用** | **暂停标记结束** | **暂停标记开始** | **暂停重定位开始** | **并发重置重定位集** |
| ------------ | ------------ | -------------------- | -------------- | -------------------- | ---------------- | ---------------- | ------------------ | -------------------- |
| **总时间**   | 34.6 ms      | 27.5 ms              | 13.7 ms        | 10.4 ms              | 0.134 ms         | 0.116 ms         | 0.0840 ms          | 0.0100 ms            |
| **平均**     | 2.16 ms      | 3.43 ms              | 1.72 ms        | 1.31 ms              | 0.0168 ms        | 0.0145 ms        | 0.0105 ms          | 0.00125 ms           |
| **标准偏差** | 2.19 ms      | 3.16 ms              | 1.22 ms        | 0.370 ms             | 0.00244 ms       | 0.00283 ms       | 0.00150 ms         | 0.000661 ms          |
| **最短**     | 0.00100 ms   | 1.77 ms              | 0.250 ms       | 0.603 ms             | 0.0140 ms        | 0.00900 ms       | 0.00800 ms         | 0.00100 ms           |
| **最长**     | 5.04 ms      | 11.7 ms              | 3.37 ms        | 1.97 ms              | 0.0220 ms        | 0.0190 ms        | 0.0130 ms          | 0.00300 ms           |
| **次数**     | 16           | 8                    | 8              | 8                    | 8                | 8                | 8                  | 8                    |

**暂停时间**

| **总时间**   | 0.334 ms   |
| ------------ | ---------- |
| **平均**     | 0.0139 ms  |
| **标准偏差** | 0.00348 ms |
| **最短**     | 0.00800 ms |
| **最长**     | 0.0220 ms  |

**并发时间**

| **总时间**   | 86.2 ms |
| ------------ | ------- |
| **平均**     | 10.8 ms |
| **标准偏差** | 3.38 ms |
| **最短**     | 7.80 ms |
| **最长**     | 19.1 ms |

**Allocation stall指标**

| **总时间**   | 8.94 ms |
| ------------ | ------- |
| **平均**     | 8.94 ms |
| **标准偏差** | 0       |
| **最短**     | 8.94 ms |
| **最长**     | 8.94 ms |

**GC原因**

| **Cause**        | **Count** | **Avg Time** | **Max Time** | **Total Time** |
| :--------------- | :-------- | :----------- | :----------- | :------------- |
| Allocation Rate  | 4         | 0.0440 ms    | 0.0480 ms    | 0.176 ms       |
| Warmup           | 3         | 0.0417 ms    | 0.0430 ms    | 0.125 ms       |
| Allocation Stall | 1         | 0.0330 ms    | 0.0330 ms    | 0.0330 ms      |



#### 5、Shenandoah

**日志节选**

```
[1.616s][info][gc,stats       ] Allocation pacing accrued:
[1.616s][info][gc,stats       ]       0 of    18 ms (  0.0%): <total>
[1.616s][info][gc,stats       ]       0 of    18 ms (  0.0%): <average total>
[1.616s][info][gc,stats       ] 
[1.616s][info][gc,metaspace   ] Metaspace: 833K(960K)->840K(960K) NonClass: 768K(832K)->775K(832K) Class: 64K(128K)->64K(128K)
[1.616s][info][gc,ergo        ] Pacer for Idle. Initial: 10485K, Alloc Tax Rate: 1.0x
[1.618s][info][gc             ] Trigger: Handle Allocation Failure
[1.618s][info][gc,ergo        ] Free: 3417K, Max: 256K regular, 2560K humongous, Frag: 24% external, 16% internal; Reserve: 26368K, Max: 256K
[1.618s][info][gc,start       ] GC(12) Pause Degenerated GC (Outside of Cycle)
[1.618s][info][gc,task        ] GC(12) Using 4 of 4 workers for stw degenerated gc
[1.618s][info][gc,ref         ] GC(12) Clearing All SoftReferences
[1.620s][info][gc,ref         ] GC(12) Encountered references: Soft: 42, Weak: 105, Final: 0, Phantom: 18
[1.620s][info][gc,ref         ] GC(12) Discovered  references: Soft: 40, Weak: 87, Final: 0, Phantom: 16
[1.620s][info][gc,ref         ] GC(12) Enqueued    references: Soft: 0, Weak: 0, Final: 0, Phantom: 0
[1.620s][info][gc,ergo        ] GC(12) Adaptive CSet Selection. Target Free: 74274K, Actual Free: 29696K, Max CSet: 21845K, Min Garbage: 44578K
[1.620s][info][gc,ergo        ] GC(12) Collectable Garbage: 12672B (100%), Immediate: 0B (0%), CSet: 12672B (100%)
[1.623s][info][gc,ergo        ] GC(12) Bad progress for free space: 3072K, need 5242K
[1.624s][info][gc             ] GC(12) Cancelling GC: Upgrade To Full GC

```

从日志中可以看到 Shenandoah GC 出现了从 Pacing 到 Degenerated GC 再到 Full GC 的过程：

**Pacing**

- 日志 `[1.616s][info][gc,stats] Allocation pacing accrued: 0 of 18 ms (0.0%): <total>` 指出在特定时段内没有进行额外的空间预留。这意味着在该时间内没有因为空间压力而触发额外的垃圾回收。

**Degenerated GC**

- 触发条件：`[1.618s][info][gc] Trigger: Handle Allocation Failure` 表明是因为分配失败（可能是内存不足）触发了垃圾收集。
- 这种GC通常发生在Shenandoah尝试进行常规的并发垃圾回收，但由于某些原因（如并发回收无法及时释放足够内存或内部阈值触发）不得不中断并发过程，转而执行一个STW（stop-the-world）收集。
- 日志中的 `Pause Degenerated GC (Outside of Cycle)` 指这是一个在常规GC周期之外的降级收集。

**Upgrade To Full GC**

- 日志中的 `[1.624s][info][gc] GC(12) Cancelling GC: Upgrade To Full GC` 表示Shenandoah GC在尝试处理降级GC失败后，不得不升级到一个完全的GC。
- 原因可能包括：
  - **空间不足**：`Bad progress for free space: 3072K, need 5242K` 显示尽管进行了回收尝试，但可用空间进展不佳，未能达到所需的空间需求。
  - **内部碎片**：`Frag: 24% external, 16% internal;` 高内部和外部碎片率可能导致内存分配效率低下，进而触发更激进的垃圾收集策略。



**吞吐量和GC停顿时间**

| **吞吐量**     | **85.291**% |
| -------------- | ----------- |
| 平均GC停顿时间 | 6.91 ms     |
| 最大GC停顿时间 | 174 ms      |

**时间统计**

|                | **暂停退化GC** | **并发标记** | **并发更新** | **暂停最终标记** | **并发清理** | **暂停初始标记** | **并发整理** | **暂停最终更新** | **暂停初始化更新** |
| -------------- | -------------- | ------------ | ------------ | ---------------- | ------------ | ---------------- | ------------ | ---------------- | ------------------ |
| **总时间**     | 227 ms         | 52.2 ms      | 30.3 ms      | 7.67 ms          | 5.76 ms      | 4.29 ms          | 2.51 ms      | 1.50 ms          | 0.331 ms           |
| **平均时间**   | 28.4 ms        | 5.22 ms      | 5.04 ms      | 0.852 ms         | 0.822 ms     | 0.429 ms         | 0.278 ms     | 0.250 ms         | 0.0552 ms          |
| **标准差时间** | 55.4 ms        | 4.74 ms      | 6.24 ms      | 0.323 ms         | 0.458 ms     | 0.291 ms         | 0.128 ms     | 0.0468 ms        | 0.0204 ms          |
| **最小时间**   | 1.79 ms        | 1.68 ms      | 0.589 ms     | 0.513 ms         | 0.259 ms     | 0.203 ms         | 0.135 ms     | 0.212 ms         | 0.0320 ms          |
| **最大时间**   | 174 ms         | 15.5 ms      | 15.8 ms      | 1.71 ms          | 1.44 ms      | 1.26 ms          | 0.497 ms     | 0.352 ms         | 0.0940 ms          |
| **计数**       | 8              | 10           | 6            | 9                | 7            | 10               | 9            | 6                | 6                  |

**暂停时间**

| **总时间**   | 241 ms    |
| ------------ | --------- |
| **平均**     | 6.18 ms   |
| **标准偏差** | 27.5 ms   |
| **最短**     | 0.0320 ms |
| **最长**     | 174 ms    |

**并发时间**

| **总时间**   | 90.7 ms |
| ------------ | ------- |
| **平均**     | 9.07 ms |
| **标准偏差** | 6.39 ms |
| **最短**     | 1.82 ms |
| **最长**     | 19.1 ms |

**GC原因**

| **Cause**                       | **Count** | **Avg Time** | **Max Time** | **Total Time** |
| :------------------------------ | :-------- | :----------- | :----------- | :------------- |
| Allocation Failure              | 4         | 53.0 ms      | 174 ms       | 212 ms         |
| Free memory below min threshold | 10        | 10.9 ms      | 21.4 ms      | 109 ms         |



#### 总结

G1 GC在各种条件下都显示出最优的性能，具有最高的吞吐量和最低的GC停顿时间。

Parallel GC在处理高内存负载时比Serial GC表现更好，具有更高的吞吐量和较低的停顿时间。

ZGC和Shenandoah在内存非常紧张的情况下，虽然能够保持低停顿时间，但在压力极大时仍可能面临性能退化。