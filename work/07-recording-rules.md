# 07 – Recording rules của mixin

Tài liệu này mô tả chi tiết 22 recording rule trong [rules/resource-review.libsonnet](../rules/resource-review.libsonnet), dạng render sẵn nằm ở [generated/prometheus-rules.yaml](../generated/prometheus-rules.yaml) và [manifests/prometheus-rule.yaml](../manifests/prometheus-rule.yaml). Cách dashboard dùng các rule này được mô tả ở [08-dashboard.md](08-dashboard.md).

## 1. Tổng quan

| Thuộc tính | Giá trị |
|---|---|
| Số group | 1, tên `resource-review` |
| Chu kỳ tính | `interval: 1m` |
| Số rule | 22 recording rule, **không có alert** |
| Tiền tố tên | `resource_review:` |
| Nguồn dữ liệu | kube-state-metrics (`job="kube-state-metrics"`) và cAdvisor (`job="kubelet", metrics_path="/metrics/cadvisor"`) |

Hai selector nguồn lấy từ `kubeStateMetricsSelector` và `cadvisorSelector` trong [config.libsonnet](../config.libsonnet). Nhãn cluster là `clusterLabel` (mặc định `cluster`).

**Vì sao tất cả rule nằm trong một group?** Prometheus tính các rule **trong cùng một group** theo thứ tự, tại cùng một timestamp. Rule danh tính (identity) đứng đầu, nên các rule bằng chứng phía sau luôn thấy danh tính của đúng vòng tính đó. Nếu tách ra nhiều group, các group chạy độc lập và có thể lệch nhau một chu kỳ.

### Quy ước đặt tên

Theo chuẩn của Prometheus, tên có dạng `level:metric:operations`:

- `resource_review:container_cpu_usage_cores:rate5m`: cấp container, metric là CPU tính bằng core, phép tính là rate trên cửa sổ 5 phút.
- Tên không có phần `:operations` (ví dụ `resource_review:container_memory_usage_bytes`) là giá trị tức thời (gauge) đã được gắn danh tính.

### Hai bộ nhãn chuẩn

Mọi rule đều trả về một trong hai bộ nhãn sau, để dashboard luôn ghép được dữ liệu với nhau.

| Bộ nhãn | Các nhãn | Dùng cho |
|---|---|---|
| **identityLabels** (cấp Pod) | `cluster, namespace, workload_type, workload, revision, revision_source, pod, uid, node` | Danh tính Pod, thời điểm tạo Pod, số restart |
| **containerLabels** (cấp container) | identityLabels + `container, image, image_id` | Mọi bằng chứng CPU, memory, cấu hình và OOM |

Ý nghĩa của các nhãn danh tính:

| Nhãn | Ý nghĩa | Ví dụ |
|---|---|---|
| `workload_type` | Loại workload | `deployment`, `statefulset`, `daemonset`, `CronJob`, `job`, `replicaset`, `staticpod`, `pod`, hoặc tên kind của controller khác (`Rollout`...) |
| `workload` | Tên workload ở cấp cao nhất | `checkout-api` |
| `revision` | Đơn vị "phiên bản" để so sánh | Tên ReplicaSet `checkout-api-7f9cb4f86b`, tên Job, hoặc Pod UID |
| `revision_source` | Revision được lấy từ đâu | `replicaset`, `replicaset_name`, `job_name`, `pod_uid` |
| `pod`, `uid` | Tên Pod và **UID của lần chạy Pod đó** | Tên Pod có thể được dùng lại, UID thì không |
| `node` | Node chạy Pod | |
| `container`, `image`, `image_id` | Container, image theo tag, image theo digest bất biến | `image_id` chứa `sha256:...` |

Trong cluster đơn, các series thường không có nhãn `cluster`. Khi đó `max by (cluster, ...)` vẫn chạy bình thường, chỉ là kết quả không có nhãn đó.

## 2. Luồng phụ thuộc

```
kube-state-metrics                                   cAdvisor (kubelet)
  kube_pod_owner, kube_replicaset_owner,               container_cpu_*, container_memory_*,
  kube_job_owner, kube_pod_info                        container_pressure_*, container_oom_events_total
        │                                                      │
        ▼                                                      │
 ① workload_revision_pod_info  (cấp Pod)                       │
        │                                                      │
        ├──► ③ workload_revision_pod_created_timestamp_seconds │
        ├──► ⑰ container_restarts_total                        │
        ▼                                                      │
 ② workload_revision_pod_container_info  (cấp container) ◄─────┘ (join theo namespace, pod, container)
        │
        ├──► ④–⑫ bằng chứng CPU/memory từ cAdvisor   (joinCadvisorIdentity)
        ├──► ⑬–⑯ request/limit từ kube-state-metrics  (joinKsmIdentity)
        ├──► ⑱ ⑲ ㉑ ㉒ trạng thái kết thúc và OOM
        └──► ⑳ QoS class
```

Số trong vòng tròn là thứ tự rule trong file, cũng là thứ tự Prometheus tính.

## 3. Hai kiểu join gắn danh tính

Đây là phần quan trọng nhất của mixin. Metric gốc không biết workload hay revision nào sở hữu nó, nên mỗi mẫu phải được "gắn danh tính" bằng phép join với rule ②.

### joinCadvisorIdentity: cho metric của cAdvisor

```promql
max by (<containerLabels>) (
  <metric cAdvisor đã gom theo cluster, namespace, pod, container>
  * on (cluster, namespace, pod, container)
    group_left(workload_type, workload, revision, revision_source, uid, node, image, image_id)
    resource_review:workload_revision_pod_container_info
)
```

- cAdvisor **không có Pod UID**, nên chỉ join được theo `namespace, pod, container`.
- Danh tính chỉ được gắn **khi Pod tương ứng còn tồn tại** ở thời điểm tính. Series ghi ra mang luôn UID, revision và image của lúc đó. Nhờ vậy, khi xem lại lịch sử, dashboard không bao giờ ghép một counter cũ với một Pod mới trùng tên.
- Phép nhân với `info` (giá trị 1) không làm đổi giá trị của metric.

### joinKsmIdentity: cho metric của kube-state-metrics

```promql
max by (<containerLabels>) (
  <metric KSM gom theo cluster, namespace, pod, uid, container>
  * on (cluster, namespace, pod, uid, container)
    group_left(workload_type, workload, revision, revision_source, node, image, image_id)
    resource_review:workload_revision_pod_container_info
)
```

kube-state-metrics có `uid` trên metric container, nên join có thêm `uid`. Pod StatefulSet hay static Pod được tạo lại với **cùng tên** sẽ không thừa hưởng dữ liệu của Pod cũ.

### Bộ lọc dùng chung cho cAdvisor

Mọi rule đọc cAdvisor đều lọc:

- `container!=""`: bỏ series cấp Pod hoặc cgroup (series không có tên container).
- `container!="POD"`: bỏ container pause (sandbox).
- `image!=""`: bỏ series không gắn image. Riêng PSI và OOM events không lọc điều kiện này.

### Mẫu `rate(...) and <series gốc>`

Các rule dạng rate đều viết theo mẫu:

```promql
rate(counter[5m]) and counter
```

`rate()` vẫn trả giá trị trong khoảng 5 phút sau khi container cũ đã dừng (series đã stale). Vế `and counter` giữ lại một giá trị chỉ khi series gốc **vẫn còn ở thời điểm tính**. Nhờ vậy, container đã chết không còn "kéo dài" CPU thêm 5 phút trên biểu đồ (xem commit `846782f fix: exclude stale container CPU rates`).

## 4. Chi tiết từng rule

### Nhóm A – Danh tính (3 rule)

#### ① `resource_review:workload_revision_pod_info`

| | |
|---|---|
| Nhãn | identityLabels |
| Giá trị | Luôn bằng 1 (series info) |
| Nguồn | `kube_pod_owner`, `kube_replicaset_owner`, `kube_job_owner`, `kube_pod_info` |
| Dashboard dùng | Gián tiếp, qua rule ② và ③ |

Mỗi Pod đang tồn tại sinh ra **đúng một series**. Series cho biết Pod thuộc workload nào, revision nào, chạy trên node nào. Biểu thức là phép `or` của **10 nhánh**, mỗi nhánh xử lý một kiểu chủ sở hữu (owner):

| # | Kiểu owner | Điều kiện | `workload` | `revision` | `revision_source` | `workload_type` |
|---|---|---|---|---|---|---|
| 1 | Deployment | Pod ← ReplicaSet ← Deployment | Tên Deployment | Tên ReplicaSet | `replicaset` | `deployment` |
| 2 | ReplicaSet của controller khác (Argo Rollout...) | Pod ← ReplicaSet ← owner không phải Deployment | Tên owner | Tên ReplicaSet | `replicaset` | Kind của owner, ví dụ `Rollout` |
| 3 | ReplicaSet đứng riêng | Pod ← ReplicaSet không có owner | Tên ReplicaSet | Tên ReplicaSet | `replicaset_name` | `replicaset` |
| 4 | StatefulSet | Pod ← StatefulSet | Tên StatefulSet | **Pod UID** | `pod_uid` | `statefulset` |
| 5 | DaemonSet | Pod ← DaemonSet | Tên DaemonSet | **Pod UID** | `pod_uid` | `daemonset` |
| 6 | Job do controller tạo (CronJob...) | Pod ← Job ← CronJob | Tên CronJob | Tên Job (mỗi lần chạy) | `job_name` | `CronJob` |
| 7 | Job đứng riêng | Pod ← Job không có owner | Tên Job | **Pod UID** | `pod_uid` | `job` |
| 8 | Static Pod | Pod ← Node | Tên Pod | **Pod UID** | `pod_uid` | `staticpod` |
| 9 | Controller khác | Pod ← kind khác các loại trên | Tên owner | **Pod UID** | `pod_uid` | Kind của owner |
| 10 | Pod trần | Không có owner controller | Tên Pod | **Pod UID** | `pod_uid` | `pod` |

Cách đọc:

- **Deployment và Rollout** có revision đúng nghĩa: mỗi lần đổi template sinh ra một ReplicaSet mới. Revision cũ và revision mới vì thế được so sánh tách biệt.
- **CronJob** coi mỗi Job là một revision. Các Pod chạy song song trong cùng một Job được gom lại, còn các lần chạy theo lịch khác nhau thì không trộn vào nhau.
- **StatefulSet, DaemonSet, Job đứng riêng, static Pod, Pod trần** không có đơn vị revision ổn định, nên mixin **dùng Pod UID làm revision**. Mỗi Pod là một "revision" riêng. Dashboard đánh dấu trường hợp này bằng cảnh báo "Revision falls back to Pod UID".
- Mọi nhánh đều join `kube_pod_info` theo `namespace, pod` để lấy `uid` và `node`.
- Mọi nhánh chỉ dùng owner có `owner_is_controller="true"`.

#### ② `resource_review:workload_revision_pod_container_info`

| | |
|---|---|
| Nhãn | containerLabels |
| Giá trị | 1 |
| Nguồn | `kube_pod_container_info{container!=""}` × rule ① (join theo `namespace, pod, uid`) |
| Dashboard dùng | Mọi biến lọc, các stat, các bảng, findings |

Bản mở rộng của rule ① xuống cấp container, có thêm `image` và `image_id`. Đây là **bảng danh tính trung tâm**:
- Mọi rule bằng chứng phía sau đều join vào rule này.
- Mọi biến lọc trên dashboard (namespace, workload, revision, pod, container) đều lấy giá trị từ rule này.

Nếu rule này rỗng thì toàn bộ dashboard rỗng.

#### ③ `resource_review:workload_revision_pod_created_timestamp_seconds`

| | |
|---|---|
| Nhãn | identityLabels |
| Giá trị | Unix timestamp (giây) lúc Pod được tạo |
| Nguồn | `kube_pod_created` × rule ① |
| Dashboard dùng | Cột "Revision age" của bảng 9: `time() - min(created)` |

### Nhóm B – CPU (4 rule, nguồn cAdvisor)

#### ④ `resource_review:container_cpu_usage_cores:rate5m`

| | |
|---|---|
| Đơn vị | Core (0.25 = 250m) |
| Nguồn | `rate(container_cpu_usage_seconds_total[5m])` |
| Dashboard dùng | Biểu đồ CPU, cột Avg/p95/Max, Shortest history, CPU coverage, findings |

Đây là mức dùng CPU trung bình trong 5 phút. Rule này còn được dùng làm **thước đo độ phủ dữ liệu**: rule chạy mỗi phút nên một Pod được quan sát đầy đủ sẽ có 60 mẫu mỗi giờ. Đếm số mẫu (`count_over_time`) cho biết Pod đã được quan sát bao lâu (History) và có bị hụt mẫu không (Coverage).

#### ⑤ `resource_review:container_cpu_throttled_periods:ratio_rate5m`

| | |
|---|---|
| Đơn vị | Tỉ lệ từ 0 đến 1 |
| Nguồn | `rate(container_cpu_cfs_throttled_periods_total[5m]) / rate(container_cpu_cfs_periods_total[5m])` |
| Dashboard dùng | Biểu đồ "CPU throttled periods", cột Throttled periods, finding `cpu-limit-constrained` |

Tỉ lệ chu kỳ CFS (mặc định 100 ms) bị throttle. Mẫu số có điều kiện `> 0`: container **không đặt CPU limit** thì không có chu kỳ CFS, nên rule không ra giá trị, thay vì ra 0 hoặc NaN. Giá trị cao chứng minh limit **đang được thực thi**. Nó chưa đủ để chứng minh ứng dụng bị chậm.

#### ⑥ `resource_review:container_cpu_throttled_seconds:rate5m`

| | |
|---|---|
| Đơn vị | Giây bị throttle trên mỗi giây thực |
| Nguồn | `rate(container_cpu_cfs_throttled_seconds_total[5m])` |
| Dashboard dùng | Biểu đồ "CPU throttled time", cột Throttled time |

Bổ sung cho ⑤: ⑤ cho biết **có bị** throttle hay không, còn ⑥ cho biết **bị nặng đến đâu**. Với ứng dụng nhiều luồng, giá trị có thể lớn hơn 1.

kube-prometheus-stack mặc định **bỏ** metric gốc này. File [kube-prometheus-stack-values.yaml](../kube-prometheus-stack-values.yaml) giữ lại nó (xem 05 §4 bước 2).

#### ⑦ `resource_review:container_cpu_pressure_waiting:ratio_rate5m`

| | |
|---|---|
| Đơn vị | Tỉ lệ thời gian từ 0 đến 1 |
| Nguồn | `rate(container_pressure_cpu_waiting_seconds_total[5m])` (PSI) |
| Dashboard dùng | Biểu đồ "CPU pressure (PSI)", cột CPU pressure |

Tỉ lệ thời gian có ít nhất một tác vụ sẵn sàng chạy nhưng phải chờ CPU. Metric này **tuỳ chọn**: cần kernel có PSI và cAdvisor đủ mới. Không có thì dashboard hiện "không có dữ liệu", không hiện 0. PSI không cho biết nguyên nhân chờ là do limit hay do node đang tranh chấp CPU.

### Nhóm C – Memory (5 rule, nguồn cAdvisor)

Bốn rule gauge đều lấy `max by (cluster, namespace, pod, container)` rồi gắn danh tính. Rule thứ năm (PSI) dùng `rate`.

| # | Rule | Metric gốc | Ý nghĩa |
|---|---|---|---|
| ⑧ | `container_memory_usage_bytes` | `container_memory_usage_bytes` | **Tổng** memory cgroup bị tính, gồm cả page cache và bộ nhớ kernel |
| ⑨ | `container_memory_working_set_bytes` | `container_memory_working_set_bytes` | Tổng trừ đi inactive file cache. Đây là con số kubelet dùng khi quyết định eviction |
| ⑩ | `container_memory_rss_bytes` | `container_memory_rss` | Chủ yếu là bộ nhớ anonymous đang nằm trong RAM (heap, stack) |
| ⑪ | `container_memory_cache_bytes` | `container_memory_cache` | Nội dung file đang được giữ trong page cache |
| ⑫ | `container_memory_pressure_waiting:ratio_rate5m` | `rate(container_pressure_memory_waiting_seconds_total[5m])` | PSI memory: tỉ lệ thời gian tác vụ bị kẹt vì kernel đang thu hồi bộ nhớ. Tuỳ chọn |

Các con số này **chồng lấn nhau**, không được cộng dồn. Dashboard vẽ chúng thành các đường riêng, không xếp chồng. Mixin dùng:
- ⑧ (tổng) cho "Memory maximum" và finding `memory-near-limit`, vì OOM killer xét theo tổng.
- ⑨ (working set) cho p95 và finding `memory-usage-above-request`, `memory-trending-to-limit`.

### Nhóm D – Cấu hình request/limit (4 rule, nguồn kube-state-metrics)

| # | Rule | Metric gốc | Đơn vị |
|---|---|---|---|
| ⑬ | `container_cpu_request_cores` | `kube_pod_container_resource_requests{resource="cpu", unit="core"}` | core |
| ⑭ | `container_cpu_limit_cores` | `kube_pod_container_resource_limits{resource="cpu", unit="core"}` | core |
| ⑮ | `container_memory_request_bytes` | `kube_pod_container_resource_requests{resource="memory", unit="byte"}` | byte |
| ⑯ | `container_memory_limit_bytes` | `kube_pod_container_resource_limits{resource="memory", unit="byte"}` | byte |

Cả bốn đều dùng joinKsmIdentity, nên có join theo `uid`. **Không cấu hình nghĩa là không có series**, không phải series bằng 0. Dashboard tận dụng điều này để phân biệt "chưa cấu hình" với "đặt bằng 0":
- Không có series → cảnh báo "... not configured".
- Có series bằng 0 → cảnh báo "... is zero; ratio omitted".

### Nhóm E – Restart, OOM và trạng thái (6 rule)

#### ⑰ `resource_review:container_restarts_total`

| | |
|---|---|
| Nhãn | identityLabels + `container`, **không có image** |
| Nguồn | `kube_pod_container_status_restarts_total` × rule ① |
| Dashboard dùng | Biểu đồ OOM/restart, cột Restarts, finding `restarts-without-oom` |

Kubernetes đếm số restart theo cặp Pod UID và container, **xuyên qua cả các lần đổi image**. Rule này cố ý không mang nhãn image, để counter không bị tách làm hai và tính sai `increase()` (xem commit `827e371 fix: preserve restart counts across image changes`). Phía dashboard có thêm `sum without (image, image_id)` để vẫn đọc được các series cũ có nhãn image.

#### ⑱ `resource_review:container_last_termination_oomkilled`

| | |
|---|---|
| Giá trị | 1 nếu lần kết thúc gần nhất của container có lý do `OOMKilled`, ngược lại 0 |
| Nguồn | `kube_pod_container_status_last_terminated_reason` |
| Dashboard dùng | Cột OOMKilled, finding `memory-oom-killed` |

Vế `or ... 0 * ...` đảm bảo container đã từng kết thúc vì lý do khác vẫn có series bằng 0, thay vì không có series nào.

**Giới hạn đã biết:** metric gốc chỉ phản ánh lần kết thúc **trước đó** của container đã restart. Pod của Job dùng `restartPolicy: Never` bị OOMKilled sẽ không được ghi nhận, vì trạng thái đó nằm ở `kube_pod_container_status_terminated_reason` (xem 04, mục "Phát hiện ngoài phạm vi").

#### ⑲ `resource_review:container_oom_events_total`

| | |
|---|---|
| Giá trị | Counter số lần OOM theo cAdvisor |
| Nguồn | `container_oom_events_total` |
| Dashboard dùng | Biểu đồ OOM/restart, cột OOM events, finding `memory-oom-killed` |

Đây là nguồn OOM thứ hai, độc lập với ⑱, và bắt được cả OOM không làm container restart. Trên một số môi trường (ví dụ Docker Desktop), counter này luôn bằng 0.

#### ⑳ `resource_review:workload_revision_pod_container_qos_class`

| | |
|---|---|
| Nhãn | containerLabels + `qos_class` |
| Nguồn | `kube_pod_status_qos_class == 1` × rule ② |
| Dashboard dùng | Finding `best-effort` |

QoS (`Guaranteed`, `Burstable`, `BestEffort`) là thuộc tính của **Pod**. Rule gán giá trị này cho từng dòng container của Pod đó, để giữ nguyên cấp chi tiết của các bảng.

#### ㉑ `resource_review:container_last_terminated_reason` và ㉒ `resource_review:container_last_terminated_exit_code`

| | ㉑ | ㉒ |
|---|---|---|
| Nhãn | containerLabels + `reason` | containerLabels |
| Giá trị | 1 cho lý do kết thúc gần nhất (`Error`, `Completed`, `OOMKilled`, `ContainerCannotRun`...) | Exit code gần nhất |
| Nguồn | `kube_pod_container_status_last_terminated_reason == 1` | `kube_pod_container_status_last_terminated_exitcode` |
| Dashboard dùng | **Chưa có panel nào dùng** | **Chưa có panel nào dùng** |

Hai rule này được ghi sẵn để tra cứu trong Explore khi gặp finding `restarts-without-oom` (finding này khuyên "check exit codes"). Ví dụ:

```promql
resource_review:container_last_terminated_exit_code{namespace="shop", workload="checkout-api"}
```

## 5. Bảng tra nhanh: rule → panel

| Rule | Panel dùng (id) |
|---|---|
| `workload_revision_pod_info` | Không dùng trực tiếp (trung gian) |
| `workload_revision_pod_container_info` | Biến lọc, 19, 12, 18, 17, 14, 15, 16, 9, 41, 51, 59 |
| `workload_revision_pod_created_timestamp_seconds` | 9 |
| `container_cpu_usage_cores:rate5m` | 14, 15, 21, 9, 41, 51, 59 |
| `container_cpu_throttled_periods:ratio_rate5m` | 23, 41, 51, 59 |
| `container_cpu_throttled_seconds:rate5m` | 24, 41, 51 |
| `container_cpu_pressure_waiting:ratio_rate5m` | 25, 41, 51 |
| `container_memory_usage_bytes` | 31, 42, 51, 59 |
| `container_memory_working_set_bytes` | 32, 42, 51, 59 |
| `container_memory_rss_bytes` | 32, 42, 51 |
| `container_memory_cache_bytes` | 32, 42, 51 |
| `container_memory_pressure_waiting:ratio_rate5m` | 36, 42, 51 |
| `container_cpu_request_cores` | 21, 9, 41, 51, 59 |
| `container_cpu_limit_cores` | 21, 9, 51 |
| `container_memory_request_bytes` | 31, 9, 51, 59 |
| `container_memory_limit_bytes` | 31, 9, 51, 59 |
| `container_restarts_total` | 37, 42, 51, 59 |
| `container_last_termination_oomkilled` | 42, 51, 59 |
| `container_oom_events_total` | 37, 42, 51, 59 |
| `workload_revision_pod_container_qos_class` | 59 |
| `container_last_terminated_reason` | Không dùng (chỉ để tra cứu) |
| `container_last_terminated_exit_code` | Không dùng (chỉ để tra cứu) |

Tên và vị trí của từng panel id có trong [08-dashboard.md](08-dashboard.md).

## 6. Chi phí series

- Rule ①, ③, ⑰ sinh khoảng **1 series cho mỗi Pod** (⑰ thì 1 series cho mỗi container).
- 18 rule còn lại sinh khoảng **1 series cho mỗi container**. Những rule không có metric nguồn tương ứng thì không sinh series nào:
  - container không đặt limit → không có ⑤, ⑭, ⑯;
  - node không có PSI → không có ⑦, ⑫;
  - container chưa từng kết thúc → không có ㉑, ㉒.

Ước tính thô: **số container đang chạy × khoảng 18 đến 20** series mới. Mỗi Pod mới (rollout, Job mới) tạo series mới, còn series cũ hết hạn theo retention. Cluster có nhiều CronJob chạy dày sẽ tốn series nhanh hơn.

Số đo thực tế trên cluster thử 12 Pod: 77 series, mỗi vòng tính mất khoảng 6 ms (xem 05 và 06).

## 7. Kiểm tra nhanh trong Prometheus

```promql
# Rule đã được nạp và ra dữ liệu chưa
count(resource_review:workload_revision_pod_info)
count(resource_review:workload_revision_pod_container_info)
count(resource_review:container_cpu_usage_cores:rate5m)

# Pod nào chưa gắn được danh tính (có CPU trong cAdvisor nhưng không có trong rule)
count by (namespace, pod) (container_cpu_usage_seconds_total{container!="", container!="POD", image!=""})
  unless on (namespace, pod) resource_review:workload_revision_pod_info

# Cách mixin nhận diện workload trong một namespace
count by (workload_type, workload, revision_source) (resource_review:workload_revision_pod_info{namespace="<ns>"})
```
