# JVM 性能调优：概念、最佳实践与 Arthas 实战

> 本文系统梳理 JVM 调优的核心概念、方法论、场景化案例与 Arthas 常用诊断手段。  
> **目录跳转**：章节使用显式 `id` 锚点，避免中文标题自动生成锚点不一致。

---

## 目录

- [1. 调优目标与基本原则](#sec-01-goals)
- [2. JVM 运行时内存结构](#sec-02-memory)
- [3. 类加载与元空间](#sec-03-classloading)
- [4. 垃圾回收基础](#sec-04-gc-basics)
- [5. 主流垃圾收集器详解](#sec-05-collectors)
- [6. GC 日志与监控指标](#sec-06-gc-metrics)
- [7. 常用 JVM 参数速查](#sec-07-jvm-params)
- [8. 调优方法论与流程](#sec-08-methodology)
- [9. 内存调优最佳实践](#sec-09-memory-tuning)
- [10. GC 调优最佳实践](#sec-10-gc-tuning)
- [11. 线程与 CPU 调优](#sec-11-thread-cpu)
- [12. 常见故障诊断](#sec-12-troubleshooting)
- [13. 场景化调优案例](#sec-13-scenarios)
- [14. 生产环境配置模板](#sec-14-templates)
- [15. Arthas 常用实践场景](#sec-15-arthas)
- [16. 调优 checklist 与避坑指南](#sec-16-checklist)

---

<h2 id="sec-01-goals">1. 调优目标与基本原则</h2>

### 1.1 调优不是目的，稳定性与 SLA 才是

JVM 调优的核心目标通常包括：

| 目标 | 典型指标 |
|------|----------|
| **低延迟** | P99 响应时间、STW 停顿时间 |
| **高吞吐** | QPS、每分钟处理事务数 |
| **资源利用率** | CPU、内存占用合理，无频繁 Full GC |
| **稳定性** | 无 OOM、无长时间 STW、无内存泄漏 |

### 1.2 调优基本原则

1. **先测量，后调参**：没有监控数据和 GC 日志，不要凭感觉改参数  
2. **一次只改一个变量**：便于归因，避免「改了一堆不知道哪个生效」  
3. **优先代码，再谈 JVM**：80% 性能问题来自代码（泄漏、大对象、锁竞争），而非参数  
4. **留有余量**：堆不要顶满物理内存，给 OS 页缓存和其他进程留空间  
5. **可回滚**：保留变更记录，灰度验证后再全量  
6. **关注业务峰值**：按 2～3 倍峰值容量规划，而非平均负载  

### 1.3 什么时候需要调优

| 信号 | 可能原因 |
|------|----------|
| 频繁 Full GC | 堆过小、老年代晋升过快、内存泄漏 |
| P99 毛刺 | Young GC 频繁、STW 过长、GC 与业务争抢 CPU |
| OOM | 堆/元空间/直接内存不足，或泄漏 |
| CPU 100% | 死循环、频繁 GC、正则/序列化热点 |
| RSS 持续增长 | 堆外内存、Native 泄漏、Metaspace 膨胀 |

---

<h2 id="sec-02-memory">2. JVM 运行时内存结构</h2>

### 2.1 整体结构（JDK 8+）

```
┌─────────────────────────────────────────────────────────┐
│                    Java 堆 (Heap)                        │
│  ┌──────────────────┐  ┌─────────────────────────────┐  │
│  │  年轻代 Young     │  │       老年代 Old             │  │
│  │ Eden | S0 | S1   │  │                             │  │
│  └──────────────────┘  └─────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│  元空间 Metaspace（本地内存，非堆）  ← 类元数据            │
├─────────────────────────────────────────────────────────┤
│  线程栈、程序计数器、本地方法栈（线程私有）                 │
├─────────────────────────────────────────────────────────┤
│  直接内存 Direct Memory（NIO、Netty 等）                  │
└─────────────────────────────────────────────────────────┘
```

### 2.2 堆内存分代

| 区域 | 存放对象 | 回收特点 |
|------|----------|----------|
| **Eden** | 新对象 | Minor GC 清理 |
| **Survivor (S0/S1)** | Minor GC 后存活对象 | 复制算法，对象年龄 +1 |
| **Old** | 长期存活、大对象 | Major/Full GC |
| **Metaspace** | 类元信息 | 类卸载时回收 |

### 2.3 对象分配与晋升

```
new 对象 → Eden
    ↓ Minor GC 存活
Survivor（年龄 < MaxTenuringThreshold，默认 15）
    ↓ 年龄达标 或 Survivor 放不下
Old 老年代
```

**大对象**：超过 `-XX:PretenureSizeThreshold`（仅 Serial、ParNew 有效）可能直接进老年代。  
**动态年龄判定**：Survivor 中相同年龄对象大小超过 Survivor 一半，≥ 该年龄的对象直接晋升。

### 2.4 堆外内存

| 类型 | 来源 | 风险 |
|------|------|------|
| 直接内存 | `ByteBuffer.allocateDirect()`、Netty | `-XX:MaxDirectMemorySize` 限制 |
| Metaspace | 类加载 | `-XX:MaxMetaspaceSize` 限制 |
| 线程栈 | 每线程默认 1MB（`-Xss`） | 线程数 × 栈大小 |
| JNI / Native | 第三方库 | 难以从 JVM 参数直接控制 |

---

<h2 id="sec-03-classloading">3. 类加载与元空间</h2>

### 3.1 类加载过程

```
加载 → 验证 → 准备 → 解析 → 初始化
```

### 3.2 类加载器层次

| 加载器 | 加载范围 |
|--------|----------|
| Bootstrap | `rt.jar` / 核心模块 |
| Extension / Platform | 扩展目录 |
| Application | `classpath` |
| 自定义 | Spring、Tomcat、插件化场景 |

**双亲委派**：子加载器先委派父加载器，保证核心类不被篡改。

### 3.3 Metaspace 调优要点

- JDK 8 起 PermGen 移除，改用 Metaspace（本地内存）
- 动态类加载场景（CGLib、Groovy、热部署、大量反射）易膨胀
- 关键参数：

```bash
-XX:MetaspaceSize=256m        # 初始阈值，达到触发 GC
-XX:MaxMetaspaceSize=512m     # 上限，防止无限增长
```

### 3.4 使用案例：Spring Boot 应用 Metaspace OOM

**现象**：运行数天后 `OutOfMemoryError: Metaspace`  
**排查**：类是否被重复加载（热部署、错误的 ClassLoader 泄漏）  
**处理**：

```bash
# 设置上限 + 开启类卸载
-XX:MaxMetaspaceSize=512m
-XX:+CMSClassUnloadingEnabled   # CMS 时代；G1 默认支持类卸载
```

根本修复：停止 ClassLoader 泄漏，避免反复生成代理类而不释放。

---

<h2 id="sec-04-gc-basics">4. 垃圾回收基础</h2>

### 4.1 如何判断对象可回收

**可达性分析**（主流）：从 GC Roots 出发，不可达则可回收。

**GC Roots 包括**：

- 虚拟机栈中引用的对象
- 方法区静态属性、常量引用的对象
- 本地方法栈 JNI 引用的对象
- 活跃线程
- JMX Bean、Class 对象等

### 4.2 引用类型（影响回收时机）

| 类型 | 说明 |
|------|------|
| 强引用 | 普通 `new`，不回收 |
| 软引用 `SoftReference` | 内存不足前回收，适合缓存 |
| 弱引用 `WeakReference` | 下次 GC 即回收 |
| 虚引用 `PhantomReference` | 跟踪对象回收，用于堆外内存管理 |

### 4.3 回收算法

| 算法 | 优点 | 缺点 | 使用场景 |
|------|------|------|----------|
| 标记-清除 | 简单 | 碎片 | CMS 老年代 |
| 标记-复制 | 无碎片、快 | 浪费空间 | Young 代 |
| 标记-整理 | 无碎片 | STW 较长 | Serial Old、G1 |

### 4.4 Minor GC / Major GC / Full GC

| 名称 | 含义 |
|------|------|
| **Minor GC** | 年轻代 GC，频率高、通常较快 |
| **Major GC** | 老年代 GC |
| **Full GC** | 整堆 + 方法区（Metaspace）等，STW 最长，应尽量避免频繁发生 |

> 日常说的「GC 卡顿」多指 Full GC 或 G1 Mixed GC 停顿过长。

---

<h2 id="sec-05-collectors">5. 主流垃圾收集器详解</h2>

### 5.1 收集器对照表

| 收集器 | 区域 | 特点 | 适用场景 |
|--------|------|------|----------|
| **Serial** | Young + Old | 单线程 STW | 客户端、小堆 |
| **ParNew** | Young | 多线程版 Serial | 配合 CMS |
| **Parallel Scavenge** | Young | 吞吐量优先 | 批处理、计算密集 |
| **Serial Old** | Old | 单线程标记整理 | 配合 PS |
| **Parallel Old** | Old | 多线程标记整理 | 配合 PS，JDK 8 默认组合之一 |
| **CMS** | Old | 并发标记，低延迟 | 已废弃（JDK 14 移除） |
| **G1** | 整堆分区 | 可预测停顿、JDK 9+ 默认 | **生产首选** |
| **ZGC** | 整堆 | 超低延迟（< 10ms 级） | 大堆、低延迟 |
| **Shenandoah** | 整堆 | 低延迟 | OpenJDK 部分发行版 |

### 5.2 G1 收集器（重点）

**Region 模型**：堆划分为多个大小相等的 Region（1～32MB），角色可动态切换（Eden / Survivor / Old / Humongous）。

**关键机制**：

- **Young GC**：回收所有 Eden + Survivor Region
- **Mixed GC**：回收部分 Old Region（根据停顿目标选择）
- **Humongous 对象**：超过 Region 50% 的对象占用连续 Humongous Region

**核心参数**：

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # 期望最大停顿（非硬性保证）
-XX:G1HeapRegionSize=16m          # Region 大小，2 的幂
-XX:InitiatingHeapOccupancyPercent=45  # 触发并发标记的堆占用阈值
-XX:G1ReservePercent=10           # 预留空间防 to-space 溢出
```

### 5.3 ZGC（JDK 15+ 生产可用，JDK 21+ 分代 ZGC）

```bash
-XX:+UseZGC
-XX:+ZGenerational                # JDK 21+ 分代 ZGC，推荐开启
-Xmx16g
```

适合：**大堆（8G+）+ 极低延迟**（游戏、交易、实时系统）。

### 5.4 如何选择收集器

```
                    ┌─ 堆 < 4G，吞吐优先 ──→ Parallel GC
                    │
JDK 8～20 默认 ──────┼─ 通用服务端 ────────→ G1（推荐）
                    │
                    └─ 大堆 + 低延迟 ─────→ ZGC / Shenandoah

JDK 21+ 默认 ───────→ G1（可切换 ZGC）
```

---

<h2 id="sec-06-gc-metrics">6. GC 日志与监控指标</h2>

### 6.1 必看指标

| 指标 | 含义 | 健康参考 |
|------|------|----------|
| **GC 频率** | 单位时间 GC 次数 | Young GC 数秒～数十秒一次；Full GC 极少 |
| **STW 停顿** | Stop-The-World 时长 | G1：< 200ms；ZGC：< 10ms |
| **吞吐量** | 业务时间 / 总时间 | 批处理 > 95%；在线服务 > 99% |
| **堆使用率** | GC 后老年代占用 | Full GC 后不应持续攀升 |
| **晋升速率** | 对象进入老年代速度 | 过快说明 Young 区偏小或对象生命周期长 |
| **Allocation Rate** | 对象分配速率 | 与 Eden 大小、Young GC 频率联动 |

### 6.2 GC 日志配置

**JDK 9+（统一日志）**：

```bash
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m
```

**JDK 8**：

```bash
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-Xloggc:/var/log/app/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=5
-XX:GCLogFileSize=50M
```

### 6.3 日志分析工具

| 工具 | 用途 |
|------|------|
| [GCeasy](https://gceasy.io/) | 在线 GC 日志分析 |
| [GCErator / GCViewer](https://github.com/chewiebug/GCViewer) | 可视化 |
| `jstat` | 实时 GC 统计 |
| Prometheus + JMX Exporter | 生产持续监控 |
| Arthas `memory` / `vmtool` | 在线诊断 |

### 6.4 jstat 常用命令

```bash
# 每 1000ms 打印一次 GC 统计，共 10 次
jstat -gcutil <pid> 1000 10

# 输出列：S0 S1 E O M CCS YGC YGCT FGC FGCT GCT
# E=Eden, O=Old, YGC=Young GC 次数, FGC=Full GC 次数, GCT=GC 总耗时
```

---

<h2 id="sec-07-jvm-params">7. 常用 JVM 参数速查</h2>

### 7.1 堆与栈

```bash
-Xms4g                    # 初始堆（建议与 Xmx 相同，避免动态扩展开销）
-Xmx4g                    # 最大堆
-Xmn1536m                 # 年轻代（G1 下不推荐手动设，用比例）
-XX:NewRatio=2             # 老年代:年轻代 = 2:1
-XX:SurvivorRatio=8        # Eden:Survivor = 8:1
-Xss512k                   # 线程栈大小
-XX:MaxDirectMemorySize=1g # 直接内存上限
```

### 7.2 元空间

```bash
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### 7.3 GC 选择

```bash
-XX:+UseG1GC
-XX:+UseZGC
-XX:+UseParallelGC          # 吞吐量场景
```

### 7.4 OOM 诊断

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/app/heapdump.hprof
-XX:OnOutOfMemoryError="sh /opt/scripts/restart.sh"
-XX:+ExitOnOutOfMemoryError   # OOM 后退出，便于 K8s 重启
```

### 7.5 JIT 与诊断

```bash
-XX:+PrintCompilation       # 打印 JIT 编译（调试用）
-XX:ReservedCodeCacheSize=256m
-XX:+UnlockDiagnosticVMOptions
-XX:+LogCompilation         # 详细编译日志（性能影响大，短期开启）
-XX:CICompilerCount=4       # 编译线程数
```

### 7.6 参数查看

```bash
java -XX:+PrintFlagsFinal -version | grep HeapSize
jinfo -flags <pid>
```

---

<h2 id="sec-08-methodology">8. 调优方法论与流程</h2>

### 8.1 标准调优流程

```
1. 明确 SLA（延迟/吞吐/可用性）
        ↓
2. 建立基线（压测 + 监控 + GC 日志）
        ↓
3. 定位瓶颈类型
   ├─ 内存（泄漏/不足/分配过快）
   ├─ GC（频繁/停顿长）
   ├─ CPU（热点/锁/死循环）
   └─ IO / 线程 / 外部依赖
        ↓
4. 优先代码优化，其次 JVM 参数
        ↓
5. 单变量变更 + 对比验证
        ↓
6. 灰度上线 + 持续监控
        ↓
7. 文档化配置与回滚方案
```

### 8.2 压测注意事项

- 使用**生产同版本 JDK**、相近数据量、相近流量模型  
- 预热足够（JIT 编译完成，缓存热起来）  
- 关注 P95/P99，而非仅平均值  
- 压测环境硬件与生产成比例（CPU 核数、内存）  

### 8.3 问题分类决策树

```
响应慢？
├─ CPU 高 → 线程栈/火焰图 → 热点方法/锁竞争
├─ CPU 正常 + GC 频繁 → 调堆/G1 停顿目标/查泄漏
├─ CPU 正常 + GC 正常 → 慢 SQL、RPC、磁盘 IO
└─ 内存涨 → heap dump → MAT 分析引用链

OOM？
├─ Java heap space → 堆大小 / 泄漏 / 大对象
├─ Metaspace → 类加载泄漏
├─ Direct buffer → Netty/NIO 未释放
├─ unable to create native thread → 线程过多或 ulimit
└─ GC overhead limit exceeded → 几乎全时间在 GC，堆太小或泄漏
```

---

<h2 id="sec-09-memory-tuning">9. 内存调优最佳实践</h2>

### 9.1 堆大小设置

**经验公式**（起点，非绝对）：

```
堆大小 ≈ 峰值活跃对象大小 × 3～5 倍
```

**容器/K8s 环境**：

```bash
# 容器内存 4G 的 Pod，JVM 堆建议 2～2.5G，留给 Metaspace、Direct、Native、OS
-Xms2500m -Xmx2500m
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=256m
```

> JDK 10+ 使用 `-XX:+UseContainerSupport`（默认开启）识别 cgroup 内存限制。

### 9.2 Xms = Xmx

- 避免堆动态扩缩带来的性能抖动
- Full GC 后不会缩小堆，Xms < Xmx 意义有限

### 9.3 年轻代 sizing

**Parallel GC 时代**：`-Xmn` 或 `-XX:NewRatio` 常用。  
**G1 时代**：让 G1 自动管理，通过 `-XX:MaxGCPauseMillis` 间接影响 Young 大小。

若 Young GC 过于频繁（< 5 秒一次）：

- 适当增大堆
- 检查是否创建过多短生命周期大对象（如大 `byte[]`、JSON 字符串）

### 9.4 内存泄漏排查

**典型泄漏场景**：

| 场景 | 原因 |
|------|------|
| ThreadLocal 未 remove | 线程池复用线程，value 常驻 |
| 静态集合只增不减 | `static Map` 缓存无淘汰 |
| 监听器未注销 | 观察者模式注册后未移除 |
| 连接/流未关闭 | 间接导致相关对象无法回收 |
| ClassLoader 泄漏 | 热部署、OSGi、自定义加载器 |

**工具链**：

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
# MAT / VisualVM 分析 Dominator Tree、GC Roots 引用链
```

### 9.5 缓存与软引用

- 业务缓存优先用 **Caffeine / Guava** 带容量与过期策略  
- 软引用不适合做精确缓存（回收时机不可控）  

---

<h2 id="sec-10-gc-tuning">10. GC 调优最佳实践</h2>

### 10.1 G1 调优思路

| 现象 | 调整方向 |
|------|----------|
| Young GC 太频繁 | 增大堆；检查分配速率 |
| Mixed GC 太频繁 | 提高 `InitiatingHeapOccupancyPercent`（如 45→55） |
| STW 超过目标 | 降低 `MaxGCPauseMillis` 期望值（100～200）；检查 Humongous 对象 |
| Full GC 仍发生 | 并发标记来不及完成；增大堆或降低 IHOP；排查内存泄漏 |
| Humongous 分配多 | 减少大数组/大字符串；调整 `G1HeapRegionSize` |

### 10.2 避免 Full GC 的关键

1. 堆足够且对象晋升速率合理  
2. G1 并发标记周期能正常完成（避免 Concurrent Mode Failure）  
3. Metaspace 有上限且类加载正常  
4. 无内存泄漏导致老年代只升不降  

### 10.3 吞吐量 vs 低延迟

| 场景 | 推荐 |
|------|------|
| 离线批处理、报表 | Parallel GC，大堆，高吞吐 |
| 微服务 API | G1，`MaxGCPauseMillis=200` |
| 核心交易、撮合 | ZGC，大堆 |
| 小内存 Spring Boot（< 2G） | G1 默认即可，少调参 |

### 10.4 GC 调优反模式

- ❌ 盲目增大堆而不查泄漏（延迟 Full GC 发生，问题更大）  
- ❌ 同时改 10 个 GC 参数  
- ❌ 生产直接 `-XX:+DisableExplicitGC` 而不评估 `System.gc()` 调用方（NIO Direct 内存依赖）  
- ❌ 禁用 GC（`-XX:+DisableExplicitGC` 与 RMI DGC 等需综合评估）  

---

<h2 id="sec-11-thread-cpu">11. 线程与 CPU 调优</h2>

### 11.1 线程数经验

| 类型 | 公式 |
|------|------|
| CPU 密集 | 线程数 ≈ CPU 核数 + 1 |
| IO 密集 | 线程数 ≈ CPU 核数 × (1 + IO等待/CPU时间) |

> 实际以压测为准；线程池应**有界**。

### 11.2 常见 CPU 飙高原因

1. **正则回溯**：复杂正则匹配长字符串  
2. **无限循环 / 忙等**：`while(true)` 未 sleep/block  
3. **频繁 GC**：GC 线程占用 CPU  
4. **序列化/反序列化**：JSON、Protobuf 大对象  
5. **锁自旋**：竞争激烈的 `synchronized` / CAS  
6. **日志刷屏**：ERROR 级别循环打日志  

### 11.3 诊断命令

```bash
top -Hp <pid>                    # 找 CPU 最高的线程
printf "%x\n" <tid>              # 转 16 进制
jstack <pid> | grep -A 30 <hex>  # 定位线程栈
```

Arthas 更高效：`thread -n 3`（见第 15 节）。

### 11.4 线程过多问题

```
unable to create new native thread
```

排查：

```bash
ulimit -u                        # 最大进程/线程数
cat /proc/<pid>/status | grep Threads
jstack <pid> | grep "^\".*\"" | wc -l
```

解决：减少线程数、增大 ulimit、`-Xss` 适当减小（谨慎）。

---

<h2 id="sec-12-troubleshooting">12. 常见故障诊断</h2>

### 12.1 OOM 类型速查

| 错误信息 | 原因 | 处理 |
|----------|------|------|
| `Java heap space` | 堆满 | dump 分析；增大 `-Xmx`；修泄漏 |
| `Metaspace` | 类元数据满 | 限制 MaxMetaspaceSize；修 ClassLoader 泄漏 |
| `Direct buffer memory` | 直接内存满 | Netty leak detection；MaxDirectMemorySize |
| `unable to create native thread` | 线程耗尽 | 减线程；调 ulimit |
| `GC overhead limit exceeded` | GC 时间 > 98% | 堆太小或严重泄漏 |

### 12.2 heap dump 分析（MAT）

1. **Leak Suspects Report**：自动可疑泄漏报告  
2. **Dominator Tree**：占用最大的对象树  
3. **Path to GC Roots**：谁持有了不该持有的引用  
4. **Histogram**：按类统计实例数与大小  

### 12.3 常用命令行工具

| 命令 | 用途 |
|------|------|
| `jps -l` | 列出 Java 进程 |
| `jinfo <pid>` | 查看/修改部分参数 |
| `jstat -gcutil <pid> 1000` | GC 实时统计 |
| `jmap -heap <pid>` | 堆配置与使用 |
| `jmap -dump:live,...` | 导出 heap dump |
| `jstack <pid>` | 线程 dump |
| `jcmd <pid> help` | 动态诊断命令（JDK 7+） |

### 12.4 jcmd 实用示例

```bash
jcmd <pid> VM.flags              # 查看 JVM 参数
jcmd <pid> GC.heap_info            # 堆信息
jcmd <pid> Thread.print            # 线程栈
jcmd <pid> VM.native_memory summary # NMT 本地内存（需 -XX:NativeMemoryTracking=summary）
```

---

<h2 id="sec-13-scenarios">13. 场景化调优案例</h2>

### 案例 1：电商订单服务 — 频繁 Young GC，P99 毛刺

**背景**：4C8G 容器，`-Xms4g -Xmx4g -XX:+UseG1GC`，QPS 3000，P99 从 50ms 升到 300ms。

**现象**：

```text
jstat -gcutil 显示 YGC 每 2～3 秒一次，YGCT 累计高
GC 日志：Allocation Rate ≈ 800MB/s
```

**分析**：

- 每笔订单创建大量临时对象（JSON 序列化、日志对象、DTO 拷贝）
- Eden 偏小导致 Young GC 过于频繁

**调优动作**：

```bash
# 1. 代码：复用对象、减少无效 toString、日志异步化
# 2. JVM：堆已是 4G，G1 下增大 MaxGCPauseMillis 让 G1 适当增大 Young
-XX:MaxGCPauseMillis=200
# 3. 若容器内存允许，堆提到 6G（需扩 Pod limit）
-Xms6g -Xmx6g
```

**结果**：YGC 间隔延长到 8～12 秒，P99 回落到 80ms。

---

### 案例 2：缓存服务 — 老年代持续增长，最终 Full GC

**背景**：本地缓存 `ConcurrentHashMap` 无淘汰，运行 12 小时后 FGC 频繁。

**现象**：

```text
jstat：O（老年代）从 30% 线性涨到 95%
Full GC 后老年代下降不明显 → 泄漏
```

**分析**：

- MAT 显示 `ConcurrentHashMap$Node[]` 占用 60% 堆
- GC Root：静态字段 `CacheManager.INSTANCE.cache`

**调优动作**：

```java
// 代码修复：改用 Caffeine，限制最大条目与过期
Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(30, TimeUnit.MINUTES)
    .build();
```

**JVM 辅助**（治标）：

```bash
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/dump/
```

**结果**：老年代稳定在 40%，FGC 接近 0。

---

### 案例 3：Netty 网关 — Direct Memory OOM

**背景**：`-Xmx2g`，高并发长连接，运行数小时 OOM。

**现象**：

```text
OutOfMemoryError: Direct buffer memory
```

**分析**：

- ByteBuf 未 release（业务 Handler 异常路径遗漏）
- 默认 MaxDirectMemorySize = Xmx，Direct 与堆争抢

**调优动作**：

```java
// 代码：确保 SimpleChannelInboundHandler 或 finally 中 release
// Netty 4.1+ 开启泄漏检测（测试/预发）
ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.PARANOID);
```

```bash
-XX:MaxDirectMemorySize=512m    # 明确上限，早失败早报警
-XX:+HeapDumpOnOutOfMemoryError
```

**结果**：修复 release 后 Direct 稳定；参数仅作保护。

---

### 案例 4：批处理任务 — 吞吐优先

**背景**：夜间 ETL，8C16G，处理 5000 万条记录，希望总时长最短。

**调优**：

```bash
-Xms12g -Xmx12g
-XX:+UseParallelGC
-XX:ParallelGCThreads=8
-XX:+UseLargePages          # 大页内存（需 OS 配置）
-XX:GCTimeRatio=99          # 吞吐优先，GC 时间占比 < 1%
```

**结果**：相比 G1 默认，总耗时降低约 15%（允许 100ms+ STW）。

---

### 案例 5：微服务启动慢 — 类加载与 JIT

**背景**：Spring Boot Fat Jar 启动 90 秒。

**分析**：

- 类数量 3 万+，Spring 扫描范围过大
- 非 GC 问题，属启动优化

**动作**：

```bash
# 1. 代码：缩小 @ComponentScan、延迟初始化
spring.main.lazy-initialization=true

# 2. JVM：AppCDS / Spring AOT（JDK 17+ Native 或 AOT 另议）
-XX:TieredStopAtLevel=1       # 仅 C1 编译，加快启动（峰值性能降，适合 Serverless 冷启动）

# 3. 容器预热：K8s readiness 前完成预热流量
```

---

### 案例 6：K8s 容器 OOMKilled（exit 137）

**背景**：Pod 内存 limit 4G，JVM `-Xmx4g`，频繁被 Kill。

**原因**：

```
JVM 总内存 ≈ 堆 + Metaspace + Direct + 线程栈 + CodeCache + Native
4G 堆 + 其他 > 4G cgroup limit → OOMKilled
```

**正确配置**：

```bash
# Pod memory limit = 4G 时
-Xms2500m -Xmx2500m
-XX:MaxMetaspaceSize=256m
-XX:MaxDirectMemorySize=256m
-XX:ReservedCodeCacheSize=128m
# 预留 ~800m 给 Native、OS、堆外
```

或使用 **JDK 11+ Container Awareness** 自动推算（仍建议显式设置）。

---

### 案例 7：RMI / System.gc() 导致周期性 STW

**现象**：每 60 分钟一次 Full GC。

**分析**：

```bash
jstat -gcutil 1 60000   # FGC 周期性
# 排查 RMI DGC：-Dsun.rmi.dgc.client.gcInterval=3600000
# 或代码中 System.gc()
```

**处理**：

```bash
-XX:+DisableExplicitGC    # 谨慎：可能影响 Direct Memory 手动 GC
# RMI 场景改为：
-Dsun.rmi.dgc.server.gcInterval=86400000
-Dsun.rmi.dgc.client.gcInterval=86400000
```

---

<h2 id="sec-14-templates">14. 生产环境配置模板</h2>

### 14.1 通用 Spring Boot 微服务（4G 容器）

```bash
JAVA_OPTS="
  -Xms2500m -Xmx2500m
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:MetaspaceSize=256m
  -XX:MaxMetaspaceSize=512m
  -XX:MaxDirectMemorySize=256m
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/data/logs/heapdump.hprof
  -XX:+ExitOnOutOfMemoryError
  -Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=50m
  -Djava.security.egd=file:/dev/./urandom
"
```

### 14.2 低延迟核心服务（8G+ 堆，JDK 21）

```bash
JAVA_OPTS="
  -Xms8g -Xmx8g
  -XX:+UseZGC
  -XX:+ZGenerational
  -XX:MaxMetaspaceSize=512m
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/data/dump/
  -Xlog:gc*:file=/data/logs/gc.log:time,tags:filecount=10,filesize=100m
"
```

### 14.3 批处理 / 大数据任务

```bash
JAVA_OPTS="
  -Xms12g -Xmx12g
  -XX:+UseParallelGC
  -XX:ParallelGCThreads=8
  -XX:GCTimeRatio=99
  -XX:+HeapDumpOnOutOfMemoryError
"
```

---

<h2 id="sec-15-arthas">15. Arthas 常用实践场景</h2>

### 15.1 Arthas 是什么

阿里开源的 Java 诊断工具，**在线 attach**，无需重启，适合生产环境。

**安装与启动**：

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar              # 选择要 attach 的 PID
# 或
java -jar arthas-boot.jar <pid>
```

### 15.2 命令总览

| 命令 | 作用 |
|------|------|
| `dashboard` | 实时面板（线程、内存、GC） |
| `thread` | 线程分析 |
| `jvm` | JVM 信息 |
| `memory` | 内存详情 |
| `heapdump` | 导出 heap dump |
| `monitor` | 方法调用统计 |
| `watch` | 观察方法入参/返回值/异常 |
| `trace` | 方法内部调用链路耗时 |
| `stack` | 方法被谁调用 |
| `tt` | 时间隧道，记录并重放调用 |
| `sc/sm` | 搜索类 / 方法 |
| `jad` | 反编译 |
| `redefine` | 热替换类（谨慎） |
| `logger` | 动态修改日志级别 |
| `ognl` | 执行 OGNL 表达式 |
| `vmtool` | 获取对象、强制 GC 等 |

---

### 场景 1：CPU 100% 定位热点线程

```bash
# 1. 查看 CPU 最高的 3 个线程
thread -n 3

# 2. 查看指定线程栈（tid 来自上一步）
thread <tid>

# 3. 若怀疑某方法，追踪内部耗时
trace com.example.OrderService createOrder

# 4. 看该方法被谁调用
stack com.example.OrderService createOrder
```

**典型发现**：正则、JSON 解析、死循环、锁竞争。

---

### 场景 2：接口 RT 变慢 — 精准追踪耗时

```bash
# 追踪方法各子调用耗时，#cost > 10ms 过滤
trace com.example.controller.OrderController submitOrder '#cost > 10'

# 只关心第一层
trace com.example.service.PaymentService pay -n 5 --skipJDKMethod false
```

**输出示例**：

```text
`---[12.5ms] com.example.service.InventoryService:deduct()
`---[85.2ms] com.example.rpc.UserClient:getUser()   ← 瓶颈
`---[3.1ms] com.example.dao.OrderMapper:insert()
```

---

### 场景 3：监控方法 QPS / 成功率 / 耗时

```bash
# 每 5 秒统计一次
monitor -c 5 com.example.service.OrderService createOrder

# 输出：timestamp, class, method, total, success, fail, avg-rt(ms), fail-rate
```

适合：灰度期间对比优化前后指标。

---

### 场景 4：观察方法入参、返回值、异常（不改代码打日志）

```bash
# 观察入参和返回值，仅前 5 次
watch com.example.service.UserService getUser '{params, returnObj}' -n 5 -x 2

# 异常时触发
watch com.example.service.UserService getUser '{params, throwExp}' -e -x 2

# 条件过滤
watch com.example.service.OrderService pay 'params[0].amount' 'params[0].amount>10000' -x 2
```

---

### 场景 5：重复偶发问题 — TimeTunnel 记录重放

```bash
# 记录某方法调用
tt -t com.example.service.PaymentService callback

# 列出记录
tt -l

# 重放第 1000 号记录（索引来自 tt -l）
tt -i 1000 -p
```

适合：支付回调、MQ 消费等难以复现的请求。

---

### 场景 6：内存占用高 — 看对象分布与 dump

```bash
# 内存概览
memory

# JVM 各区域
jvm

# 导出 heap dump（与 jmap 类似）
heapdump /tmp/arthas-dump.hprof

# 查看某类实例数（vmtool，JDK 8+）
vmtool --action getInstances --className java.util.HashMap --limit 10
```

配合 MAT 分析 Dominator Tree。

---

### 场景 7：怀疑类加载问题 / 版本冲突

```bash
# 搜索类来自哪个 JAR
sc -d com.fasterxml.jackson.databind.ObjectMapper

# 反编译查看实际运行代码
jad com.example.service.OrderService

# 查看类加载器树
classloader -t
```

**典型发现**：Fat Jar 中同名类多版本，实际加载的是旧版本。

---

### 场景 8：线上临时打开 DEBUG 日志

```bash
# 查看 logger
logger

# 动态改级别
logger --name com.example.dao --level DEBUG

# 改完记得改回
logger --name com.example.dao --level INFO
```

---

### 场景 9：Spring Bean 状态检查（OGNL）

```bash
# 获取 Spring Context 中某 Bean 属性（需 spring-context 在 classpath）
ognl '@org.springframework.web.context.ContextLoader@getCurrentWebApplicationContext().getBean("orderService").cacheSize'
```

> 生产慎用 OGNL，避免误操作。

---

### 场景 10：Full GC 频繁 — 在线看重分配与 GC

```bash
dashboard          # 看 GC 次数、内存曲线
jvm                # GC 收集器、参数
memory             # 各区域使用

# 手动触发 GC 观察（仅诊断，生产少做）
vmtool --action forceGc
```

---

### 场景 11：死锁检测

```bash
thread -b          # 检测阻塞/死锁线程
```

输出会标明死锁线程及持有/等待的锁。

---

### 场景 12：方法被谁调用 — 排查意外调用链

```bash
stack com.example.util.DateUtils format
```

当某个「不该被调用」的方法频繁出现时，快速定位调用栈。

---

### 15.3 Arthas 使用注意事项

| 注意点 | 说明 |
|--------|------|
| 生产权限 | attach 权限应受控，操作可审计 |
| watch/trace 开销 | 高频方法加条件过滤，`-n` 限制次数 |
| redefine 风险 | 热替换可能导致不一致，仅紧急修复 |
| 卸载 Arthas | `stop` 命令卸载 agent |
| 防火墙 | Web Console 默认 3658 端口，勿暴露公网 |

```bash
# 结束诊断，卸载 agent
stop
```

---

<h2 id="sec-16-checklist">16. 调优 checklist 与避坑指南</h2>

### 16.1 上线前 Checklist

- [ ] `-Xms` = `-Xmx`  
- [ ] 容器内存 > 堆 + Metaspace + Direct + 预留（≥ 25% headroom）  
- [ ] 开启 GC 日志轮转  
- [ ] 开启 `-XX:+HeapDumpOnOutOfMemoryError`  
- [ ] 收集器选择合理（默认 G1，低延迟 ZGC）  
- [ ] 压测覆盖峰值 2～3 倍  
- [ ] 监控接入：堆使用、GC 次数/耗时、线程数、CPU  
- [ ] 变更文档与回滚参数  

### 16.2 避坑指南

| 误区 | 正解 |
|------|------|
| 堆越大越好 | 过大导致 GC 回收时间长；应匹配活跃数据量 |
| 零 Full GC 才算成功 | G1 Mixed GC 可接受；关注 STW 时长 |
| 复制网上参数 | 每应用对象模型不同，必须压测验证 |
| 只调 JVM 不改代码 | 泄漏和大对象应优先修代码 |
| OOM 只加内存 | 先 dump 分析根因 |
| 生产随意 Full GC | `System.gc()` 和 jmap -dump 会 STW，低峰操作 |

### 16.3 推荐学习路径

```
JVM 内存结构 → GC 算法与收集器 → GC 日志阅读
      ↓
jstat / jmap / jstack 命令行
      ↓
MAT heap 分析 → Arthas 在线诊断
      ↓
压测 + 场景化调优 → 生产监控闭环
```

---

## 附录 A：锚点 ID 对照表

| 章节 | 锚点 ID |
|------|---------|
| 1 | `sec-01-goals` |
| 2 | `sec-02-memory` |
| 3 | `sec-03-classloading` |
| 4 | `sec-04-gc-basics` |
| 5 | `sec-05-collectors` |
| 6 | `sec-06-gc-metrics` |
| 7 | `sec-07-jvm-params` |
| 8 | `sec-08-methodology` |
| 9 | `sec-09-memory-tuning` |
| 10 | `sec-10-gc-tuning` |
| 11 | `sec-11-thread-cpu` |
| 12 | `sec-12-troubleshooting` |
| 13 | `sec-13-scenarios` |
| 14 | `sec-14-templates` |
| 15 | `sec-15-arthas` |
| 16 | `sec-16-checklist` |

## 附录 B：JDK 版本与默认 GC

| JDK | 默认 GC | 备注 |
|-----|---------|------|
| 8 | Parallel（Server） | 可改 G1 |
| 9～20 | G1 | |
| 21+ | G1 | 可 `-XX:+UseZGC -XX:+ZGenerational` |

## 附录 C：延伸阅读

- [Oracle GC Tuning Guide](https://docs.oracle.com/en/java/javase/21/gctuning/)
- [Arthas 官方文档](https://arthas.aliyun.com/doc/)
- [Eclipse MAT](https://eclipse.dev/mat/)
- 《Java 性能权威指南》《深入理解 Java 虚拟机》（周志明）

---

> 文档路径：`jvm性能调优/JVM调优概念与实践.md`  
> 目录链接格式 `[标题](#sec-xx-xxx)` 与 `<h2 id="sec-xx-xxx">` 一一对应，支持 GitHub / VS Code / Cursor 稳定跳转。
