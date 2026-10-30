# enrolld 三行九格看板（O3）

按 L-08 第 29 页的“三行九格”搭建。每格使用的 PromQL 与 Grafana 面板一一对应；
`deploy/grafana-dashboard.json` 可直接导入 Grafana（数据源选 Prometheus）。

| 行 | 格 1 | 格 2 | 格 3 |
|---|---|---|---|
| **业务：学生是否选课成功** | 选课成功率：`sum(rate(http_requests_total{route="/api/enroll",code=~"2.."}[5m])) / sum(rate(http_requests_total{route="/api/enroll"}[5m]))` | 读路径 SLI（≤50 ms 的成功占比）：`sum(rate(http_request_duration_seconds_bucket{route="/api/courses/{id}",code="200",le="0.05"}[5m])) / sum(rate(http_request_duration_seconds_count{route="/api/courses/{id}"}[5m]))` | 在途请求与 goroutine：`sum(http_inflight_requests)`、`sum(go_goroutines{job="enrolld"})` |
| **服务：谁在等待（RED）** | 按接口 QPS：`sum by (route) (rate(http_requests_total[5m]))` | 按接口 5xx 比例：`sum by (route) (rate(http_requests_total{code=~"5.."}[5m])) / sum by (route) (rate(http_requests_total[5m]))` | 按接口 p50/p99：`histogram_quantile(0.5, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))`、`histogram_quantile(0.99, ...)` |
| **资源：在等待什么（USE）** | CPU：`sum(rate(node_cpu_seconds_total{mode!~"idle|iowait",cpu=~"1"}[5m])) / count(node_cpu_seconds_total{mode="idle",cpu=~"1"})`，进程 `rate(process_cpu_seconds_total{job="enrolld"}[5m])` | 内存：`node_memory_MemAvailable_bytes/1048576`、`process_resident_memory_bytes/1048576` | 磁盘：`rate(node_disk_io_time_seconds_total{device="vda"}[5m])`、`1000*rate(node_disk_write_time_seconds_total{device="vda"}[5m])/rate(node_disk_writes_completed_total{device="vda"}[5m])`、`rate(node_disk_writes_completed_total{device="vda"}[5m])` |

排障顺序自上而下：先看业务成功率是否下降，再看哪个接口在排队（RED、inflight/连接池），最后看 CPU、内存、磁盘哪个先饱和（USE）。容量规划则自下而上：先算资源上限，再换算能承载的业务量。
