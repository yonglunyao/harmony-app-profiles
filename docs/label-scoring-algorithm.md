# HarmonyOS 应用画像标签评分算法

> 版本：v1.0
> 更新日期：2025-03-17
> 用途：应用风险评分计算模型

---

## 1. 评分算法概述

### 1.1 设计目标

- **量化风险**：将标签映射为0-100的风险分数
- **多维评估**：综合应用基本信息、行为、舆情、开发者画像
- **动态更新**：支持实时/定期更新评分
- **场景适配**：支持检测决策、用户提示等多种场景

### 1.2 评分范围

```
分数区间      风险等级      建议操作
─────────────────────────────────────────
0-25分       低风险        正常上架
26-50分      中低风险      延迟上架，需复核
51-75分      中高风险      拒绝上架，需整改
76-100分     高风险        拒绝上架，列入黑名单
```

---

## 2. 评分因子

### 2.1 基础评分因子

| 因子 | 说明 | 权重 |
|------|------|------|
| `W_label` | 标签权重 | 1-10 |
| `R_level` | 风险等级系数 | critical=4, high=3, medium=2, low=1 |
| `C_conf` | 置信度 | 0.5-1.0 |
| `E_expert` | 专家审核系数 | 已审核=1.2, 未审核=1.0 |

### 2.2 维度评分因子

| 维度 | 权重 | 说明 |
|------|------|------|
| `α_basic` | 0.20 | 应用基本信息画像 |
| `α_behavior` | 0.40 | 应用行为画像 |
| `α_public` | 0.25 | 应用舆情画像 |
| `α_developer` | 0.15 | 开发者画像 |

---

## 3. 评分算法公式

### 3.1 单标签评分

```
S_label = W_label × R_level × C_conf × E_expert
```

其中：
- `S_label ∈ [0, 48]` （最大值：10×4×1.2=48）
- 实际最大值归一化到 0-100

**示例计算**：
```
标签：恶意订阅检测 (BH_001)
- W_label = 10
- R_level = 4 (critical)
- C_conf = 0.95 (AI分析置信度)
- E_expert = 1.2 (专家已审核)

S_label = 10 × 4 × 0.95 × 1.2 = 45.6
```

### 3.2 维度评分

```
S_dim = (Σ S_label) × N_factor × V_factor
```

其中：
- `Σ S_label`：该维度下所有标签评分之和
- `N_factor`：标签数量惩罚因子
- `V_factor`：标签多样性因子

**N_factor（标签数量惩罚）**：
```
N_factor = 1 + (n_critical × 0.5 + n_high × 0.2 + n_medium × 0.1)

其中：
- n_critical：Critical标签数量
- n_high：High标签数量
- n_medium：Medium标签数量
```

**V_factor（多样性惩罚）**：
```
V_factor = 1 + (unique_categories / total_categories) × 0.3

其中：
- unique_categories：该维度涉及的二级类别数
- total_categories：该维度总二级类别数
```

### 3.3 综合评分

```
S_total = α_basic × S_basic + α_behavior × S_behavior
        + α_public × S_public + α_developer × S_developer
```

归一化到 0-100：
```
S_final = min(100, max(0, S_total × normalization_factor))
```

---

## 4. 评分算法实现

### 4.1 伪代码实现

```
function calculate_app_score(app_labels, label_definitions):
    # 初始化维度评分
    dimension_scores = {
        "basic": 0,
        "behavior": 0,
        "public": 0,
        "developer": 0
    }

    # 统计各维度标签
    dimension_labels = group_labels_by_dimension(app_labels)

    # 计算每个维度的评分
    for dim, labels in dimension_labels:
        score = 0
        n_critical = 0
        n_high = 0
        n_medium = 0

        # 计算单标签评分总和
        for label in labels:
            def = label_definitions[label.id]
            label_score = def.weight × get_risk_coefficient(def.risk_level)
                          × label.confidence × get_expert_coefficient(label.expert_reviewed)
            score += label_score

            # 统计风险等级
            if def.risk_level == "critical":
                n_critical += 1
            elif def.risk_level == "high":
                n_high += 1
            elif def.risk_level == "medium":
                n_medium += 1

        # 计算惩罚因子
        n_factor = 1 + (n_critical × 0.5 + n_high × 0.2 + n_medium × 0.1)

        # 计算多样性因子
        categories = get_unique_subcategories(labels)
        v_factor = 1 + (len(categories) / get_total_subcategories(dim)) × 0.3

        # 维度评分
        dimension_scores[dim] = score × n_factor × v_factor

    # 综合评分
    weights = {"basic": 0.20, "behavior": 0.40, "public": 0.25, "developer": 0.15}
    total_score = sum(dimension_scores[dim] × weights[dim] for dim in dimension_scores)

    # 归一化到 0-100
    final_score = normalize_score(total_score)

    return final_score
```

### 4.2 Python实现示例

```python
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class Label:
    id: str
    confidence: float  # 0.5-1.0
    expert_reviewed: bool

@dataclass
class LabelDefinition:
    id: str
    name: str
    dimension: str
    risk_level: str  # critical, high, medium, low
    weight: int  # 1-10

class AppScoringEngine:
    """应用评分引擎"""

    # 风险等级系数
    RISK_COEFFICIENT = {
        "critical": 4.0,
        "high": 3.0,
        "medium": 2.0,
        "low": 1.0
    }

    # 专家审核系数
    EXPERT_COEFFICIENT = {
        True: 1.2,   # 专家已审核
        False: 1.0   # 未审核
    }

    # 维度权重
    DIMENSION_WEIGHTS = {
        "basic": 0.20,
        "behavior": 0.40,
        "public": 0.25,
        "developer": 0.15
    }

    # 归一化因子（基于历史数据统计）
    NORMALIZATION_FACTOR = 0.15

    def __init__(self, label_definitions: Dict[str, LabelDefinition]):
        self.label_definitions = label_definitions

    def calculate_label_score(self, label: Label) -> float:
        """计算单个标签评分"""
        definition = self.label_definitions[label.id]

        base_score = (
            definition.weight *
            self.RISK_COEFFICIENT[definition.risk_level] *
            label.confidence *
            self.EXPERT_COEFFICIENT[label.expert_reviewed]
        )

        return base_score

    def calculate_dimension_score(self, labels: List[Label]) -> float:
        """计算维度评分"""
        if not labels:
            return 0.0

        # 计算基础评分
        base_score = sum(self.calculate_label_score(label) for label in labels)

        # 统计风险等级数量
        risk_counts = {"critical": 0, "high": 0, "medium": 0, "low": 0}
        for label in labels:
            definition = self.label_definitions[label.id]
            risk_counts[definition.risk_level] += 1

        # 计算数量惩罚因子
        n_factor = (
            1.0 +
            risk_counts["critical"] * 0.5 +
            risk_counts["high"] * 0.2 +
            risk_counts["medium"] * 0.1
        )

        # 计算多样性因子
        subcategories = set(
            self.label_definitions[label.id].dimension
            for label in labels
        )
        # 假设每个维度最多6个二级类别
        v_factor = 1.0 + (len(subcategories) / 6.0) * 0.3

        return base_score * n_factor * v_factor

    def calculate_app_score(self, app_labels: List[Label]) -> Dict:
        """计算应用综合评分"""
        # 按维度分组
        dimension_labels = {
            "basic": [],
            "behavior": [],
            "public": [],
            "developer": []
        }

        for label in app_labels:
            definition = self.label_definitions.get(label.id)
            if definition:
                dimension_labels[definition.dimension].append(label)

        # 计算各维度评分
        dimension_scores = {}
        for dim, labels in dimension_labels.items():
            dimension_scores[dim] = self.calculate_dimension_score(labels)

        # 综合评分
        total_score = sum(
            dimension_scores[dim] * self.DIMENSION_WEIGHTS[dim]
            for dim in dimension_scores
        )

        # 归一化到 0-100
        final_score = min(100.0, max(0.0, total_score * self.NORMALIZATION_FACTOR))

        # 确定风险等级
        risk_level = self.get_risk_level(final_score)

        return {
            "score": round(final_score, 2),
            "risk_level": risk_level,
            "dimension_scores": {
                dim: round(score * self.NORMALIZATION_FACTOR, 2)
                for dim, score in dimension_scores.items()
            },
            "label_count": len(app_labels),
            "critical_count": sum(
                1 for label in app_labels
                if self.label_definitions.get(label.id).risk_level == "critical"
            )
        }

    def get_risk_level(self, score: float) -> str:
        """根据分数确定风险等级"""
        if score <= 25:
            return "低风险"
        elif score <= 50:
            return "中低风险"
        elif score <= 75:
            return "中高风险"
        else:
            return "高风险"

# 使用示例
if __name__ == "__main__":
    # 定义标签库
    label_definitions = {
        "BH_001": LabelDefinition("BH_001", "恶意订阅检测", "behavior", "critical", 10),
        "APP_003": LabelDefinition("APP_003", "签名信息缺失", "basic", "critical", 10),
        "REP_002": LabelDefinition("REP_002", "威胁情报标记", "public", "critical", 10),
        # ... 其他标签定义
    }

    # 创建评分引擎
    engine = AppScoringEngine(label_definitions)

    # 应用标签数据
    app_labels = [
        Label("BH_001", confidence=0.95, expert_reviewed=True),
        Label("APP_003", confidence=1.0, expert_reviewed=True),
        Label("REP_002", confidence=0.9, expert_reviewed=False),
    ]

    # 计算评分
    result = engine.calculate_app_score(app_labels)

    print(f"应用评分: {result['score']}")
    print(f"风险等级: {result['risk_level']}")
    print(f"维度评分: {result['dimension_scores']}")
    print(f"标签数量: {result['label_count']}")
    print(f"Critical标签: {result['critical_count']}")
```

---

## 5. 评分策略

### 5.1 一票否决策略

某些标签一旦存在，直接判定为高风险：

| 标签ID | 标签名称 | 处理策略 |
|--------|----------|----------|
| CS_006-CS_009 | 恶意代码类 | 直接拒绝 |
| BH_001, BH_002 | 恶意扣费类 | 直接拒绝 |
| BH_021 | 静默安装应用 | 直接拒绝 |
| BH_025 | 色情/赌博内容推广 | 直接拒绝 |
| COMP_001-COMP_005 | 涉黄/赌/政/暴/诈 | 直接拒绝 |
| REP_002-REP_003 | 威胁情报标记 | 直接拒绝 |

```python
def check_veto_labels(app_labels: List[Label]) -> bool:
    """检查是否有一票否决标签"""
    VETO_LABELS = {
        "CS_006", "CS_007", "CS_008", "CS_009",  # 恶意代码
        "BH_001", "BH_002", "BH_021", "BH_025",   # 恶意行为
        "COMP_001", "COMP_002", "COMP_003", "COMP_004", "COMP_005",  # 违规内容
        "REP_002", "REP_003",  # 威胁情报
    }

    for label in app_labels:
        if label.id in VETO_LABELS and label.confidence >= 0.8:
            return True
    return False
```

### 5.2 累积风险策略

同一类别的标签累积到一定数量后，触发升级：

| 类别 | 阈值 | 触发动作 |
|------|------|----------|
| Critical标签 | 1个 | 直接高风险 |
| High标签 | 3个 | 升级为高风险 |
| Medium标签 | 5个 | 升级为中高风险 |
| 同一二级类别 | 3个+ | 触发专家审核 |

### 5.3 时间衰减策略

标签评分随时间衰减：

```python
def calculate_time_decay(label_timestamp, current_timestamp):
    """计算时间衰减系数"""
    days_passed = (current_timestamp - label_timestamp).days

    # 不同时间区间的衰减系数
    if days_passed <= 7:
        return 1.0      # 7天内：100%
    elif days_passed <= 30:
        return 0.8      # 30天内：80%
    elif days_passed <= 90:
        return 0.6      # 90天内：60%
    else:
        return 0.4      # 90天后：40%
```

---

## 6. 评分输出

### 6.1 评分报告结构

```json
{
  "app_id": "com.example.app",
  "app_name": "示例应用",
  "version": "1.0.0",
  "score": 78.5,
  "risk_level": "中高风险",
  "recommendation": "拒绝上架，需整改",

  "dimension_scores": {
    "basic": {
      "score": 15.2,
      "label_count": 3,
      "critical_count": 1,
      "top_labels": [
        {"id": "APP_003", "name": "签名信息缺失", "score": 48.0}
      ]
    },
    "behavior": {
      "score": 45.8,
      "label_count": 5,
      "critical_count": 2,
      "top_labels": [
        {"id": "BH_001", "name": "恶意订阅检测", "score": 45.6},
        {"id": "BH_006", "name": "通讯录窃取", "score": 43.2}
      ]
    },
    "public": {
      "score": 10.5,
      "label_count": 2,
      "critical_count": 0,
      "top_labels": []
    },
    "developer": {
      "score": 7.0,
      "label_count": 1,
      "critical_count": 0,
      "top_labels": []
    }
  },

  "labels": [
    {
      "id": "BH_001",
      "name": "恶意订阅检测",
      "dimension": "应用行为画像",
      "risk_level": "critical",
      "score": 45.6,
      "confidence": 0.95,
      "detected_at": "2025-03-17T10:00:00Z",
      "data_sources": ["DS_03"],
      "expert_reviewed": true
    }
    // ... 其他标签
  ],

  "calculation_metadata": {
    "calculated_at": "2025-03-17T15:30:00Z",
    "algorithm_version": "v1.0",
    "total_labels": 145,
    "applied_labels": 11,
    "normalization_factor": 0.15
  }
}
```

### 6.2 评分趋势

```json
{
  "app_id": "com.example.app",
  "trend": {
    "current": 78.5,
    "previous": 72.3,
    "change": "+6.2",
    "direction": "上升",
    "history": [
      {"date": "2025-03-01", "score": 65.0},
      {"date": "2025-03-08", "score": 68.5},
      {"date": "2025-03-15", "score": 72.3},
      {"date": "2025-03-17", "score": 78.5}
    ]
  }
}
```

---

## 7. 评分调优

### 7.1 权重调优方法

1. **历史数据回测**：使用历史应用数据验证评分准确性
2. **专家反馈校准**：根据专家审核结果调整权重
3. **A/B测试**：对不同权重配置进行对比测试

### 7.2 归一化因子计算

```python
def calculate_normalization_factor(historical_scores: List[float]) -> float:
    """
    基于历史数据计算归一化因子

    目标：使大部分应用评分分布在 0-100 之间
    """
    max_score = max(historical_scores)
    # 使最高分接近 100
    return 100.0 / max_score if max_score > 0 else 1.0
```

### 7.3 评分校准

```python
def calibrate_score(
    raw_score: float,
    percentile_rank: float,
    adjustment_factor: float = 1.0
) -> float:
    """
    评分校准

    Args:
        raw_score: 原始评分
        percentile_rank: 在所有应用中的百分位排名
        adjustment_factor: 人工调整因子
    """
    # 基于百分位的微调
    if percentile_rank > 95:
        adjustment_factor *= 1.1  # Top 5% 应用增加10%
    elif percentile_rank < 10:
        adjustment_factor *= 0.9  # Bottom 10% 应用减少10%

    return min(100.0, raw_score * adjustment_factor)
```

---

## 8. 性能优化

### 8.1 缓存策略

```python
from functools import lru_cache

class CachedScoringEngine(AppScoringEngine):
    """带缓存的评分引擎"""

    @lru_cache(maxsize=1000)
    def calculate_label_score(self, label_id: str, confidence: float, expert_reviewed: bool) -> float:
        """缓存单标签评分结果"""
        label = Label(label_id, confidence, expert_reviewed)
        return super().calculate_label_score(label)

    def invalidate_cache(self):
        """清除缓存"""
        self.calculate_label_score.cache_clear()
```

### 8.2 批量评分

```python
def batch_calculate_scores(
    engine: AppScoringEngine,
    app_labels_list: List[List[Label]]
) -> List[Dict]:
    """批量计算应用评分"""

    # 并行计算
    from concurrent.futures import ThreadPoolExecutor

    with ThreadPoolExecutor(max_workers=10) as executor:
        results = list(executor.map(
            engine.calculate_app_score,
            app_labels_list
        ))

    return results
```

---

## 9. 监控指标

### 9.1 评分分布监控

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| 评分均值 | 所有应用平均分 | 持续上升/下降超过10% |
| 高风险占比 | >75分应用占比 | 超过5% |
| 低风险占比 | <25分应用占比 | 低于80% |
| 评分方差 | 评分分布离散度 | 异常波动 |

### 9.2 标签命中监控

| 指标 | 说明 |
|------|------|
| Top 10 命中标签 | 最常出现的10个标签 |
| 标签命中率 | 每个标签的命中频率 |
| 新增标签趋势 | 新标签的发现趋势 |

---

## 10. 附录

### 10.1 评分系数参考表

| 风险等级 | 系数 | 说明 |
|----------|------|------|
| Critical | 4.0 | 严重威胁，直接拒绝 |
| High | 3.0 | 高风险，需重点关注 |
| Medium | 2.0 | 中等风险，需监控 |
| Low | 1.0 | 低风险，可接受 |

### 10.2 维度权重参考表

| 维度 | 权重 | 理由 |
|------|------|------|
| 应用行为画像 | 40% | 行为最能反映恶意意图 |
| 应用舆情画像 | 25% | 用户反馈和信誉重要 |
| 应用基本信息画像 | 20% | 静态特征提供基础判断 |
| 开发者画像 | 15% | 开发者历史作为参考 |

---

*本文档为HarmonyOS应用安全检测平台评分算法设计文档，内容持续更新中。*
