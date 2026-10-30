# 实验 2 记录：第 18 组

评分时依据本文件和 `runs/` 中的原始数据核对。本文件中的每一个数值，都须能在某个 `runs/<名字>/` 的文件中找到：表格用 `scripts/table.sh` 生成后粘贴，其他数值注明出处（例如“`runs/r1-pool4/red.csv` 第 6 阶段”）。不要只贴截图。将 `<填写>` 替换为本组的内容。

第 4.1、4.4、5.1、5.3 节的预测须在对应的测量之前写好，写好后不要修改。预测错误不扣分，与实测不一致之处在分析中讨论。

## 1. 基本信息

| 项 | 内容 |
|---|---|
| 小组 | 第 18 组 |
| 仓库地址 | <https://github.com/xXiao-T/ops-lab2-18> |
| 主机 | 华为云 ECS，`x1.2u.2g`（2 vCPU / 2 GiB），可用区 `cn-east-3c`，Ubuntu 20.04 server 64bit 镜像 |
| CPU | Model name：`General Purpose Processor`；2 个 vCPU；`lscpu -e` 显示 CPU 0 与 CPU 1 的 CORE 均为 0（同一物理核的两个超线程，无法分到不同 CORE） |
| 内存 | `free -m` total = 1778 MiB |
| 数据目录所在的盘 | `findmnt -T data`：`/dev/vda1`，`ext4`；40 GiB 云硬盘（`lsblk` 为 40G，ROTA=1；实例内未暴露云盘规格名） |
| 操作系统与内核 | `Ubuntu 20.04.6 LTS`；`5.4.0-216-generic` |
| Go、Prometheus、node_exporter | `go1.26.9 linux/amd64`；`prometheus, version 3.13.4`；`node_exporter, version 1.12.1` |
| 分核 | enrolld：1；loadgen、Prometheus、node_exporter：0 |

## 2. 任务 0：基线

`bin/enrolld -bench-fsync 2000` 连续运行三次的输出：

```text
===== run 1 =====
fsync 次数 2000，总耗时 2.37 s，数据目录 data
fsync 耗时（毫秒）：mean 1.179  p50 1.121  p90 1.348  p99 2.259  max 4.374
按平均值推算，一个写者串行 fsync 每秒最多约 848 次（= 1000 / 1.179）
===== run 2 =====
fsync 次数 2000，总耗时 2.41 s，数据目录 data
fsync 耗时（毫秒）：mean 1.187  p50 1.122  p90 1.332  p99 2.365  max 6.575
按平均值推算，一个写者串行 fsync 每秒最多约 843 次（= 1000 / 1.187）
===== run 3 =====
fsync 次数 2000，总耗时 2.38 s，数据目录 data
fsync 耗时（毫秒）：mean 1.182  p50 1.122  p90 1.354  p99 2.247  max 3.314
按平均值推算，一个写者串行 fsync 每秒最多约 846 次（= 1000 / 1.182）
```

按平均值计算，单个写者串行 fsync 每秒最多约 **846** 次（三次均值的平均为 1.1827 ms，$1000/1.1827 \approx 845.5$）。三次之间相差 0.008 ms（1.179～1.187 ms，约 0.7%），说明这块云盘的 fsync 延迟很稳定。

空载和运行 fsync 测试时各工具的读数（各观察 10 秒以上，填写典型值；均忽略工具输出的第一行“开机以来平均值”）：

| 工具与列 | 空载 | 运行 `-bench-fsync` 时 | 该列的含义 |
|---|---|---|---|
| `vmstat -w 1`：r、b | r=0，b=0（偶见 b=1） | r=0–1，b=1 | r：可运行（等待 CPU）的进程数；b：处于不可中断睡眠、通常在等 I/O 的进程数 |
| `vmstat -w 1`：us、sy、id、wa、st | 0–1、0–1、99–100、0–2、0（%） | 0–1、2–4、50–52、45–47、0（%） | us 用户态、sy 内核态、id 空闲、wa 等 I/O、st 被宿主机偷走的时间占比 |
| `iostat -dx 1 <盘>`：w/s、wkB/s | 0–2、0–108（偶有后台写） | 2439–2559、13280–13980 | 每秒完成的写次数；每秒写入的 KiB 数 |
| `iostat -dx 1 <盘>`：w_await、f/s、f_await | 0、-、-（无写） | 0.37–0.38 ms、-、- | 每次写的平均等待时间；每秒 flush 次数；每次 flush 的平均耗时（本机 sysstat 12.2.0 与 5.4 内核不输出 f/s、f_await） |
| `iostat -dx 1 <盘>`：aqu-sz、%util | 0.00、0–3.6% | 0.00、97.6–100% | 平均请求队列长度；设备处于忙碌状态的时间比例 |

运行 fsync 测试时，**CPU 并不繁忙**：`us+sy` 只有约 2～5%，`id` 仍有约 50%。等待体现在 `vmstat` 的 **b=1** 和 **wa≈45%～47%**，以及 `iostat` 的 **%util≈100%**；这说明进程被磁盘 fsync 阻塞，而不是在等 CPU。`iostat` 的 `aqu-sz` 在这台虚拟机上始终为 0，`f/s`、`f_await` 也不显示——虚拟化层没有把这些字段完整暴露出来，因此本报告用 `vmstat` 的 `b/wa` 与 `iostat` 的 `w/s、w_await、%util` 判断磁盘饱和，flush 次数在任务 3 用服务端 `fsync_per_s` 代替。

## 3. 任务 1：指标流水线

两个抓取目标的状态（`curl -s 127.0.0.1:9090/api/v1/targets` 中两处 `"health"` 的值）：enrolld = `up`，node = `up`。

本组编写的四条查询（与 `deploy/queries-red.txt` 一致）：

```text
server_rps          sum(rate(http_requests_total{route="${route}"}[${window}]))
server_error_rate   sum(rate(http_requests_total{route="${route}",code=~"5.."}[${window}])) / sum(rate(http_requests_total{route="${route}"}[${window}])) or vector(0)
server_p99_ms       1000 * histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket{route="${route}"}[${window}])))
server_slo_good     sum(rate(http_request_duration_seconds_bucket{route="${route}",code="200",le="${slo}"}[${window}])) / sum(rate(http_request_duration_seconds_count{route="${route}"}[${window}]))
```

`runs/t1-light` 的数据（`scripts/table.sh runs/t1-light ok_rps mean_ms p99_ms server_rps server_mean_ms server_p99_ms cpu_ms_per_req db_query_mean_ms`）：

| step | target_rps | ok_rps | mean_ms | p99_ms | server_rps | server_mean_ms | server_p99_ms | cpu_ms_per_req | db_query_mean_ms |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 0 | 0.0 | - | - | 0 | NaN | NaN | +Inf | NaN |
| 2 | 50 | 50.5 | 11.62 | 13.13 | 51.75 | 10.7813 | 19.9 | 0.637786 | 10.322 |
| 3 | 100 | 97.8 | 11.70 | 14.26 | 95.3333 | 10.9099 | 19.9043 | 0.595424 | 10.4233 |
| all | - | 74.2 | 11.67 | 14.10 | - | - | - | - | - |

从这次实验读出：每次查询实际占用连接 **10.4233 ms**（`db_query_mean_ms`，100 请求/秒阶段，`runs/t1-light/red.csv` 第 3 阶段）；每个请求消耗的 CPU **0.595424 ms**（`cpu_ms_per_req`，同阶段）。空载阶段的 `server_p99_ms`、`server_slo_good` 为 NaN/空，是因为该阶段没有请求，直方图没有样本，属于预期现象；50 与 100 请求/秒阶段 `server_error_rate=0`、`server_slo_good=1`。

## 4. 任务 2：读路径

### 4.1 预测（R1 之前写）

| 资源 | 容量 | 每个请求占用（数据来源） | 推算的上限（请求/秒） |
|---|---|---|---|
| 连接池 | `-pool` = 4 个连接 | 10.4233 ms（`runs/t1-light/red.csv` 第 3 阶段 `db_query_mean_ms`） | $4 / 0.0104233 \approx 384$ |
| CPU | enrolld 的 1 个核（`GOMAXPROCS=1`，绑定 CPU 1） | 0.595424 ms（同阶段 `cpu_ms_per_req`） | $1 / 0.000595424 \approx 1680$ |

先达到上限的是：**连接池**（384 请求/秒，远低于 CPU 的 1680 请求/秒）。

预计从多少请求/秒起，客户端 p99 超过 50 ms：**约 340 请求/秒（约 0.89 × 池上限）**。

理由：100 请求/秒时池利用率 $\rho = 100 \times 0.0104233 / 4 \approx 0.26$，实测 `pool_wait_p99_ms` = 2.198 ms（`runs/t1-light/red.csv` 第 3 阶段）。若把等待分位数近似看作随 $\rho/(1-\rho)$ 增长，则 p99 从 2.198 ms 涨到 50 ms 需要约 22.7 倍，对应 $\rho \approx 0.89$，即 $0.89 \times 384 \approx 340$ 请求/秒。这个估计没有考虑模拟数据库的服务时间近似确定（`db-ms=10`，排队增量以 10 ms 为台阶），实际拐点可能更靠近上限；预测错误不扣分，R1 的实测会检验它。

R1 的到达率（`scripts/rates.sh 384` 的输出）：**77,154,230,269,307,346,384,422,461**（0.2～1.2 倍，共 9 个阶段）。

### 4.2 R1：`-pool 4`

`scripts/table.sh runs/r1-pool4 ok_rps error_rate p99_ms server_p99_ms server_mean_ms inflight_avg cpu_ms_per_req pool_in_use_avg pool_wait_p99_ms sched_wait_p99_ms svc_cpu_busy procs_running_avg`：

| step | target_rps | ok_rps | error_rate | p99_ms | server_p99_ms | server_mean_ms | inflight_avg | cpu_ms_per_req | pool_in_use_avg | pool_wait_p99_ms | sched_wait_p99_ms | svc_cpu_busy | procs_running_avg |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 77 | 79.0 | 0.0000 | 13.43 | 19.9 | 10.8461 | 1.12 | 0.611454 | 1.12 | 1.23286 | 0.873158 | 0.046 | 1.6 |
| 2 | 154 | 152.8 | 0.0000 | 18.32 | 19.9349 | 11.1922 | 2.16 | 0.57495 | 2.12 | 7.90726 | 0.908552 | 0.097 | 2 |
| 3 | 230 | 229.8 | 0.0000 | 22.52 | 34.5009 | 11.9839 | 2.44 | 0.54538 | 2.28 | 14.0811 | 0.9 | 0.1225 | 3 |
| 4 | 269 | 273.9 | 0.0000 | 32.37 | 46.9832 | 13.7766 | 4.04 | 0.540127 | 3 | 25.1112 | 0.900336 | 0.144 | 2.2 |
| 5 | 307 | 311.1 | 0.0000 | 46.28 | 49.3153 | 16.254 | 5.12 | 0.531406 | 3.36 | 44.992 | 0.890587 | 0.16 | 2.4 |
| 6 | 346 | 342.4 | 0.0000 | 60.59 | 85.1313 | 21.5606 | 7.88 | 0.521911 | 3.88 | 51.3765 | 0.848935 | 0.1745 | 2 |
| 7 | 384 | 386.3 | 0.0009 | 993.80 | 990.542 | 493.659 | 192.52 | 0.523684 | 4 | 990.106 | 0.892346 | 0.1915 | 2.2 |
| 8 | 422 | 377.0 | 0.0971 | 1011.83 | 1981.46 | 996.728 | 415.8 | 0.506358 | 4 | 1896.65 | 0.905379 | 0.201 | 2.4 |
| 9 | 461 | 377.2 | 0.1852 | 1012.22 | 1987.18 | 1002.47 | 464.16 | 0.499886 | 4 | 1948.41 | 0.889781 | 0.2295 | 2.4 |
| all | - | 281.1 | 0.0477 | 1011.31 | - | - | - | - | - | - | - | - | - |

最大吞吐量 **386.3 请求/秒**，出现在第 7 阶段（目标 384；该阶段已有 0.09% 的 503，之后成功吞吐稳定在约 377 请求/秒）；客户端 p99 首次超过 50 ms 是在第 6 阶段（目标 **346 请求/秒**，`ok_rps`=342.4，p99=60.59 ms）。预测的拐点约 340 请求/秒与实测 346 相差约 6 请求/秒（1.8%），说明“池利用率接近 0.89 时 p99 开始越线”的估计基本成立。

### 4.3 CPU 消耗在何处

在约 0.8 倍上限的负载下采集 20 秒 CPU 剖析，`go tool pprof -top` 的前 10 行（同时保存在 `runs/r1-pool4/pprof-top.txt`）：

```text
      flat  flat%   sum%        cum   cum%
     2.38s 86.23% 86.23%      2.38s 86.23%  main.spin
     0.13s  4.71% 90.94%      0.13s  4.71%  internal/runtime/syscall/linux.Syscall6
     0.02s  0.72% 91.67%      0.02s  0.72%  internal/poll.convertErr (inline)
     0.01s  0.36% 92.03%      0.02s  0.72%  encoding/json.structEncoder.encode
     0.01s  0.36% 92.39%      0.06s  2.17%  net/http.(*conn).readRequest
     0.01s  0.36% 92.75%      0.07s  2.54%  runtime.mcall
     0.01s  0.36% 93.12%      0.03s  1.09%  runtime.newobject
     0.01s  0.36% 93.48%      0.06s  2.17%  runtime.park_m
         0     0% 93.48%      0.05s  1.81%  bufio.(*Reader).Peek
         0     0% 93.48%      0.05s  1.81%  bufio.(*Reader).fill
```

`main.spin` 占 **86.23%**（本次剖析总采样 2.76 s，约占一个核的 13.8%）。按 R1 第 5 阶段（0.8 × 上限，307 请求/秒）的 `cpu_ms_per_req` = 0.531406 ms 计算，每个请求约 **0.531 ms**，其中 spin 约 **0.458 ms**（$0.531406 \times 0.8623$），其余约 **0.073 ms** 消耗在 HTTP 处理、系统调用（`Syscall6` 4.71%）、JSON 编码、Go 运行时调度/GC 与网络轮询上。

### 4.4 R2：调大连接池（预测在 R2 之前写）

| 项 | 值与算法 |
|---|---|
| 每个请求占用的 CPU（取 R1 哪一阶段的 `cpu_ms_per_req`，选取理由） | 0.531406 ms（R1 第 5 阶段，目标 307 请求/秒）：这是尚未出现 503、池占用 3.36/4、p99 46.28 ms 的最高阶段，CPU 数据受连接池排队影响最小 |
| 推算的 CPU 上限 C | $1 / 0.000531406 \approx 1882$ 请求/秒 |
| R2 的池大小 P2 = ceil(2 × C × 每次查询的秒数) | $\lceil 2 \times 1882 \times 0.0105062 \rceil = \lceil 39.54 \rceil = 40$ 个连接（池上限约 $40/0.0105062 \approx 3807$ 请求/秒，是 CPU 上限的 2 倍） |
| R2 的到达率（`scripts/rates.sh 1882 0.3 0.6 0.8 0.9 1.0 1.1 1.25 1.5`） | 565,1129,1506,1694,1882,2070,2353,2823 |
| 预测：R2 中先达到上限的是 | CPU（1882 请求/秒；连接池上限约 3807 请求/秒） |

`scripts/table.sh runs/r2-pool40 ok_rps error_rate p99_ms server_p99_ms inflight_avg cpu_cores cpu_ms_per_req cpu_wait_threads sched_wait_p99_ms pool_in_use_avg svc_cpu_busy procs_running_avg cpu_psi_some`：

| step | target_rps | ok_rps | error_rate | p99_ms | server_p99_ms | inflight_avg | cpu_cores | cpu_ms_per_req | cpu_wait_threads | sched_wait_p99_ms | pool_in_use_avg | svc_cpu_busy | procs_running_avg | cpu_psi_some |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 565 | 566.5 | 0.0000 | 13.32 | 19.9 | 5.8 | 0.277914 | 0.492355 | 0.016261 | 0.849754 | 5.8 | 0.27 | 3.2 | 0.0144679 |
| 2 | 1129 | 1134.5 | 0.0000 | 13.91 | 19.9018 | 11.32 | 0.500963 | 0.443249 | 0.0116262 | 1.27236 | 11.32 | 0.493 | 3.2 | 0.01301 |
| 3 | 1506 | 1504.1 | 0.0000 | 14.69 | 19.9 | 14.48 | 0.639502 | 0.42707 | 0.0064383 | 1.82885 | 14.48 | 0.6315 | 2.8 | 0.00697765 |
| 4 | 1694 | 1686.4 | 0.0000 | 15.73 | 19.9024 | 16.2 | 0.715567 | 0.423267 | 0.00445227 | 2.34073 | 16.2 | 0.713 | 3 | 0.0101774 |
| 5 | 1882 | 1875.6 | 0.0000 | 16.67 | 19.9 | 18.28 | 0.785762 | 0.41911 | 0.00313495 | 3.81274 | 18.28 | 0.783 | 3.2 | 0.00545965 |
| 6 | 2070 | 2065.1 | 0.0000 | 21.05 | 19.9074 | 21.2 | 0.857719 | 0.41663 | 0.00304767 | 5.58546 | 21.2 | 0.851 | 3.6 | 0.00551815 |
| 7 | 2353 | 2297.2 | 0.0269 | 1121.18 | 962.776 | 287.24 | 0.975775 | 0.397478 | 0.00267573 | 38.0306 | 27.16 | 0.973 | 3.4 | 0.0072297 |
| 8 | 2823 | 1561.6 | 0.4462 | 1955.56 | 138.876 | 64.25 | 0.172948 | 0.426905 | 0.00203156 | 1000 | 33.5 | 1 | 4 | 0.0176418 |
| all | - | 1586.4 | 0.0943 | 1321.20 | - | - | - | - | - | - | - | - | - | - |

最大吞吐量 **2297.2 请求/秒**（第 7 阶段，目标 2353，错误率 2.69%）；与预测的 C=1882 请求/秒相差 **+415 请求/秒（+22%）**。预测偏低的原因：`cpu_ms_per_req` 从低负载的 0.53 ms 降到高负载的约 0.40–0.42 ms（缓存变热、每次请求的固定开销被摊薄），而预测用的是 0.531 ms；此外 2 个 vCPU 是同一物理核的两个超线程，loadgen 与 enrolld 共享执行单元，实际可用 CPU 与“1 个核”的假设也有出入。

需要说明第 8 阶段的异常：客户端 `ok_rps`=1561.6、错误率 44.6%，而 `red.csv` 的 `server_rps` 只有 405、`server_p99_ms` 只有 138.9 ms——这是因为 CPU 已 100% 饱和（`use.csv` 的 `svc_cpu_busy`=1.0），enrolld 没有 CPU 及时响应 Prometheus 每秒一次的抓取，`/metrics` 抓取出现缺口，按窗口求 rate 的结果失真。判断饱和应以客户端的 `ok_rps/error_rate` 和 node_exporter 的 `svc_cpu_busy`、`sched_wait_p99_ms` 为准，不能只看 `red.csv` 的末阶段。第 7 阶段 `pool_in_use_avg`=27.16 < 40，连接池未满，也证明此时瓶颈已转移到 CPU。

## 5. 任务 3：写路径

### 5.1 预测：`-sync each`（W1 之前写）

| 项 | 值与算法 |
|---|---|
| fsync 平均耗时（任务 0） | 1.1827 ms（三次 1.179/1.187/1.182 的平均） |
| 推算的最大 TPS | $1000 / 1.1827 \approx 846$ 请求/秒；单个写者串行执行 fsync 时，吞吐由一次 fsync 的往返时间决定 |
| 达到上限时 enrolld 大约使用几个核（写路径没有 spin，按 4.3 节拆分出的“spin 以外的开销”估计每个写请求的 CPU） | 约 **0.06 个核**：4.3 节拆出的 spin 以外开销约 0.073 ms/请求，$846 \times 0.000073 \approx 0.062$；即使写路径的系统调用开销是读路径的 2–3 倍，也远小于 1 个核，因此**先达到上限的应是磁盘 fsync，而不是 CPU** |
| W1 的到达率（`scripts/rates.sh 846 0.2 0.4 0.6 0.8 0.9 1.0 1.1 1.2`） | 169,338,508,677,761,846,931,1015 |

### 5.2 W1：`-sync each`

`scripts/table.sh runs/w1-<…> ok_rps error_rate p99_ms server_p99_ms cpu_cores fsync_per_s fsync_mean_ms batch_mean svc_cpu_busy cpu_iowait disk_w_per_s disk_w_await_ms disk_f_per_s disk_f_await_ms disk_aqu_sz disk_util`：

<粘贴表格>

### 5.3 预测：`-sync group`（W2 之前写）

在 W1 达到上限时的到达率下，改为组提交后：吞吐量 <填写>；平均每次 fsync 处理 <填写> 条记录；客户端 p99 约 <填写>。理由：<填写>

W2 的到达率：<填写>

### 5.4 W2：`-sync group`

（列与 5.2 相同）

<粘贴表格>

### 5.5 W3：`-sync interval`

（列与 5.2 相同）

<粘贴表格>

## 6. 分析问题

每题先列出所用数据（引用上面的表或 `runs/` 中的文件），再写结论，并说明结论的成立条件。每题 300～600 字。

**问题 1：上限由什么决定。** 按 L-07 第 25 页资源账的格式，为 enrolld 的读路径编制一份资源账：资源、每个请求占用（实测）、容量、推算的上限、实测的最大吞吐量、饱和观测项。预测与实测相差多少，差异来自何处？`cpu_ms_per_req` 随负载如何变化，原因是什么？这对“在低负载下测量每请求开销，再推算上限”的做法意味着什么？`db_query_mean_ms` 为什么大于 `-db-ms`？以“p99 ≤ 50 ms”为目标，R1 能承受的最大阶段相当于上限的多少比例？与 L-07 第 6 页“容量 = 上限 × 0.7”比较，为什么尚未达到上限延迟就已上升？R2 调大连接池后，瓶颈转移到了哪个资源？

<填写>

**问题 2：利用率不等于饱和。** R1 达到上限时，enrolld 所在核的 CPU 利用率是多少？哪个数值最先表明资源已经饱和？R2 中 CPU 利用率接近 100% 时，内核层的 `procs_running_avg`、`cpu_psi_some`、`cpu_wait_threads` 和 Go 运行时层的 `sched_wait_p99_ms` 分别如何变化？为什么内核层几乎观察不到排队，排队发生在哪里？这与 L-06 第 16、17 页 Tomcat 的线程池有何相同之处？最后用 Little's Law 核对：从 R1 选取三个阶段，用 `server_rps × server_mean_ms` 计算在途请求数，并与 `inflight_avg` 比较；W 为什么必须用平均值而不能用 p99？哪个阶段不一致，原因是什么？

<填写>

**问题 3：持久性以延迟为代价。** W1 实测的最大 TPS 与 1000 ÷ fsync 平均耗时相比如何？W1 达到上限时 CPU、内存、磁盘的指标各是多少，哪几列表明磁盘饱和？应用测得的 `fsync_mean_ms` 与 `disk_w_await_ms`、`disk_f_await_ms` 为什么不同？一次 fsync 大约对应几次写和几次 flush？W1 与 W2 在 `disk_util` 相近时吞吐量相差多少，`%util` 接近 100% 能否说明磁盘已经饱和？组提交的 `batch_mean` 如何随负载变化，低负载时是否有收益，本组的预测是否正确？W3 快了多少，代价是什么？选课记录应采用哪一种方式，在什么条件下 interval 可以接受？

<填写>

**问题 4：延迟数值的测量位置与方法。** 同一阶段中，客户端与服务端的平均值、p99 各相差多少？列出只计入客户端的时间组成。找出一个服务端 p99 大于客户端 p99 的阶段，并解释原因（L-08 第 10 页）。比较服务端的 `server_slo_good` 与客户端的 `slo_good`；若存在过载阶段，二者为何相差很大？SLI 应在何处测量（L-08 第 19 页）？用 R1 的 `steps.csv`，将各阶段的 p99 求平均，与 `all` 行的 p99 比较，说明百分位数为什么不能平均，以及正确的合并方法。最后计算错误预算：SLO 为“30 天内 99% 的读请求在 50 ms 内成功返回”，若 R1 中第一个不达标阶段的负载持续下去，燃烧率是多少，30 天的预算多久耗尽（L-08 第 21 页）？

<填写>

## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 内存与 GC | <未做 / 预测、`runs/` 目录、结论> |
| O2 网络 | <未做 / 两台主机的规格与拓扑、`runs/` 或日志、结论> |
| O3 告警与看板 | <未做 / 规则文件、触发记录、看板说明> |
| O4 迁移到小组项目 | <未做 / 仓库链接、测了哪个资源、结论> |

## 8. 成员分工

每位成员一行，只填写 GitHub 用户名，无需填写姓名；行数应与小组人数一致。

| GitHub 用户名 | 负责的部分 |
|---|---|
| <用户名> | <填写> |
