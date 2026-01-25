# SKDH 项目概览（聚焦睡眠模块）

## 1. 项目定位
Scikit Digital Health（SKDH）是一个用于可穿戴惯性传感器（IMU/加速度计）数据读取、预处理与算法分析的 Python 包。项目覆盖了从数据读取、特征计算到下游算法模块的完整流程，包含睡眠、活动、步态、坐站转换等多个子模块。

模块一览（与文档一致）：
- `skdh.io`：数据读取
- `skdh.preprocessing`：预处理
- `skdh.context`：上下文/状态识别
- `skdh.gait`、`skdh.sit2stand`、`skdh.activity`、`skdh.sleep`
- `skdh.features`、`skdh.utility`

## 2. 数据与流水线约定
项目对输入数据有统一约定（见文档 `docs/src/data.rst`）：
- `time`：Unix 时间戳（秒）。支持“本地/无时区”或“UTC + tz_name”的两种方式。
- `accel`：三轴加速度，单位 g。
- `fs`：采样频率（Hz）。
- 管线（Pipeline）中的参数会被逐层传递；`Sleep` 等最终算法也支持 `tz_name` 以进行时区转换。

## 3. 睡眠模块总体流程（`skdh.sleep`）
### 3.1 输入与输出
核心入口是 `Sleep` 类（`src/skdh/sleep/sleep.py`）：
- **输入**：`time`, `accel`，可选 `temperature`、`fs`、`wear`（佩戴区间索引）。
- **输出**：
  - 睡眠指标字典（每个日窗口一行）
  - 睡眠窗口索引（`{"sleep": sleep_idx}`）
- 可选：按分钟保存睡眠/休息预测（`save_per_minute_results=True`）。

### 3.2 关键步骤（按实现逻辑）
1. **下采样**：可选将数据重采样到 20 Hz（默认开启）。
2. **日窗口切分**：默认 `day_window=(12, 24)`，即“中午到中午”的 24h 窗口，适合夜间睡眠场景。
3. **佩戴时间处理**：
   - 若未提供 `wear`，默认全程佩戴。
   - 与日窗口取交集后进入后续计算。
4. **TSO 检测（Total Sleep Opportunity）**：寻找“可能入睡的大休息窗口”。
5. **活动指数（Activity Index）**：基于 60s 窗口的加速度高通特征。
6. **睡眠/觉醒分类**：使用 Cole-Kripke 规则（含 Webster 复评分）。
7. **睡眠指标计算**：在 TSO 内执行运行长度编码（RLE）后计算各类指标。

流程示意：
```
raw accel/time
  -> (downsample to 20Hz)
  -> day windowing
  -> wear mask
  -> TSO detection
  -> activity index (60s epochs)
  -> sleep/wake classification
  -> endpoints (metrics)
```

## 4. TSO（Total Sleep Opportunity）检测细节
实现位置：`src/skdh/sleep/tso.py`。
主要逻辑：
- 以 5 秒窗口计算加速度滚动中位数。
- 计算 z-angle，再取 5 秒滚动平均并求相邻差值的绝对值。
- 对差值做 5 分钟滚动中位数。
- 阈值由“分位数 * 系数”计算，并被限制在 `[tso_min_thresh, tso_max_thresh]` 范围内。
- 结合佩戴时间（外部/内部），去除过短的休息块与过短的活动块。
- 取最长连续休息段作为 TSO。

内部佩戴检测：
- 如果有温度数据，可用 `internal_wear_temp_thresh` 过滤非佩戴段。
- 也可基于 30 分钟运动标准差阈值 `internal_wear_move_thresh` 过滤。

## 5. 睡眠/觉醒分类细节
实现位置：`src/skdh/sleep/sleep_classification.py`。
- 基于 Activity Index（60s 窗口）进行 Cole-Kripke 评分。
- 使用固定卷积核与阈值（`scores < 0.5` 判定为睡眠）。
- 默认启用 Webster 复评分规则（规则 a/b/c/d）以提升特异性。
- 可通过 `sf`（缩放系数）调节敏感度，默认 `0.243`。

## 6. 睡眠指标（Endpoints）
实现位置：`src/skdh/sleep/endpoints.py`。

**基础/常用指标**
- TotalSleepTime：TSO 内总睡眠时长（分钟）。
- PercentTimeAsleep：TSO 内睡眠占比。
- NumberWakeBouts：睡眠期间觉醒段次数（排除开头/末尾）。
- SleepOnsetLatency：入睡潜伏期（从 TSO 开始到首次睡眠）。
- WakeAfterSleepOnset：入睡后清醒时间（WASO）。

**睡眠片段化/稳定性指标**
- AverageSleepDuration / AverageWakeDuration：平均睡眠/觉醒段长度。
- SleepWakeTransitionProbability / WakeSleepTransitionProbability：睡-醒与醒-睡转移概率。
- SleepGiniIndex / WakeGiniIndex：睡眠/觉醒段长度分布的离散度。
- SleepAverageHazard / WakeAverageHazard：转移风险（hazard）指标。
- SleepPowerLawDistribution / WakePowerLawDistribution：片段长度的幂律分布参数。

## 7. 与其他模块的关系
- `skdh.activity` 可接收 `sleep` 窗口，计算“睡眠时段的活动相关指标”。
- 管线（Pipeline）机制支持将 `sleep` 结果传递给后续模块。

## 8. 面向手表算法开发的可调参数要点
以下参数在项目中已支持，适合根据设备与人群重新标定：
- **采样频率与下采样**：默认目标 20 Hz（`downsample=True`）。
- **日窗口**：`day_window=(12, 24)`，若有日间睡眠可改为 `(0, 24)`。
- **TSO 阈值**：`tso_perc`, `tso_factor`, `tso_min_thresh`, `tso_max_thresh`。
- **休息/活动块约束**：`min_rest_block`, `max_activity_break`。
- **内部佩戴判断**：`internal_wear_temp_thresh`, `internal_wear_move_thresh`。
- **睡眠分类敏感度**：`sf`（Cole-Kripke 缩放因子）。
- **有效天数门槛**：`min_day_hours`, `min_wear_time`。

> 重要实现事实：当前分类与指标都基于 **1 分钟 epoch**；若要更高时间分辨率，需要调整 Activity Index 与分类窗口设计。

## 9. 关键代码位置索引
- `src/skdh/sleep/sleep.py`：睡眠主流程与结果组织
- `src/skdh/sleep/tso.py`：TSO 检测算法
- `src/skdh/sleep/sleep_classification.py`：Cole-Kripke + Webster 复评分
- `src/skdh/sleep/utility.py`：Activity Index、z-angle 等基础函数
- `src/skdh/sleep/endpoints.py`：睡眠指标定义
- `tests/sleep/`：测试用例（TSO、分类、指标、工具函数）

## 10. 参考文献（来源于代码注释）
睡眠窗口与分类主要基于以下工作：
- van Hees 等人的夜间睡眠检测与角度法（2014/2015）
- Cole-Kripke 睡眠/觉醒识别（1992）
- Activity Index（Bai et al., 2016）
- 相关扩展工作见 `sleep.py` 注释中的参考列表
