---
name: healthcare-cdss-patterns
description: 临床决策支持系统（CDSS）开发模式。涵盖药物相互作用检查、剂量验证、临床评分（NEWS2、qSOFA）、告警严重级别分类，以及与 EMR 工作流的集成。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# 医疗 CDSS 开发模式（Healthcare CDSS Development Patterns）

构建集成到 EMR 工作流的临床决策支持系统（CDSS）的模式指南。CDSS 模块属于患者安全关键系统——对漏报零容忍。

## 适用场景

- 实现药物相互作用检查
- 构建剂量验证引擎
- 实现临床评分系统（NEWS2、qSOFA、APACHE、GCS）
- 设计异常临床值的告警系统
- 构建带安全检查的药物医嘱录入
- 将检验结果解读与临床背景整合

## 工作原理

CDSS 引擎是**无副作用的纯函数库**。输入临床数据，输出告警。这使其具备完整可测试性。

三个核心模块：

1. **`checkInteractions(newDrug, currentMeds, allergies)`** — 检查新药与当前用药及已知过敏的相互作用。返回按严重程度排序的 `InteractionAlert[]`。使用 `DrugInteractionPair` 数据模型。
2. **`validateDose(drug, dose, route, weight, age, renalFunction)`** — 根据体重、年龄调整和肾功能调整规则验证处方剂量。返回 `DoseValidationResult`。
3. **`calculateNEWS2(vitals)`** — 基于 `NEWS2Input` 计算国家早期预警评分 2。返回含总分、风险等级和升级指导的 `NEWS2Result`。

```
EMR UI
  ↓（用户录入数据）
CDSS 引擎（纯函数，无副作用）
  ├── 药物相互作用检查器
  ├── 剂量验证器
  ├── 临床评分（NEWS2、qSOFA 等）
  └── 告警分类器
  ↓（返回告警）
EMR UI（内联展示告警，关键告警时阻断操作）
```

### 药物相互作用检查

```typescript
interface DrugInteractionPair {
  drugA: string;           // 通用名
  drugB: string;           // 通用名
  severity: 'critical' | 'major' | 'minor';
  mechanism: string;
  clinicalEffect: string;
  recommendation: string;
}

function checkInteractions(
  newDrug: string,
  currentMedications: string[],
  allergyList: string[]
): InteractionAlert[] {
  if (!newDrug) return [];
  const alerts: InteractionAlert[] = [];
  for (const current of currentMedications) {
    const interaction = findInteraction(newDrug, current);
    if (interaction) {
      alerts.push({ severity: interaction.severity, pair: [newDrug, current],
        message: interaction.clinicalEffect, recommendation: interaction.recommendation });
    }
  }
  for (const allergy of allergyList) {
    if (isCrossReactive(newDrug, allergy)) {
      alerts.push({ severity: 'critical', pair: [newDrug, allergy],
        message: `与已记录过敏原存在交叉反应：${allergy}`,
        recommendation: '未经过敏咨询不得开具处方' });
    }
  }
  return alerts.sort((a, b) => severityOrder(a.severity) - severityOrder(b.severity));
}
```

相互作用对必须是**双向的**：若药物 A 与药物 B 存在相互作用，则药物 B 与药物 A 也存在相互作用。

### 剂量验证

```typescript
interface DoseValidationResult {
  valid: boolean;
  message: string;
  suggestedRange: { min: number; max: number; unit: string } | null;
  factors: string[];
}

function validateDose(
  drug: string,
  dose: number,
  route: 'oral' | 'iv' | 'im' | 'sc' | 'topical',
  patientWeight?: number,
  patientAge?: number,
  renalFunction?: number
): DoseValidationResult {
  const rules = getDoseRules(drug, route);
  if (!rules) return { valid: true, message: '无可用验证规则', suggestedRange: null, factors: [] };
  const factors: string[] = [];

  // 安全规则：规则要求体重但体重缺失时，必须阻断（而非放行）
  if (rules.weightBased) {
    if (!patientWeight || patientWeight <= 0) {
      return { valid: false, message: `${drug} 需要体重信息（mg/kg 药物）`,
        suggestedRange: null, factors: ['weight_missing'] };
    }
    factors.push('weight');
    const maxDose = rules.maxPerKg * patientWeight;
    if (dose > maxDose) {
      return { valid: false, message: `剂量超过 ${patientWeight}kg 患者的最大值`,
        suggestedRange: { min: rules.minPerKg * patientWeight, max: maxDose, unit: rules.unit }, factors };
    }
  }

  // 年龄调整（规则定义年龄段且提供年龄时）
  if (rules.ageAdjusted && patientAge !== undefined) {
    factors.push('age');
    const ageMax = rules.getAgeAdjustedMax(patientAge);
    if (dose > ageMax) {
      return { valid: false, message: `超过 ${patientAge} 岁患者的年龄调整最大值`,
        suggestedRange: { min: rules.typicalMin, max: ageMax, unit: rules.unit }, factors };
    }
  }

  // 肾功能调整（规则定义 eGFR 段且提供肾功能时）
  if (rules.renalAdjusted && renalFunction !== undefined) {
    factors.push('renal');
    const renalMax = rules.getRenalAdjustedMax(renalFunction);
    if (dose > renalMax) {
      return { valid: false, message: `超过 eGFR ${renalFunction} 的肾功能调整最大值`,
        suggestedRange: { min: rules.typicalMin, max: renalMax, unit: rules.unit }, factors };
    }
  }

  // 绝对最大值
  if (dose > rules.absoluteMax) {
    return { valid: false, message: `超过绝对最大值 ${rules.absoluteMax}${rules.unit}`,
      suggestedRange: { min: rules.typicalMin, max: rules.absoluteMax, unit: rules.unit },
      factors: [...factors, 'absolute_max'] };
  }
  return { valid: true, message: '在范围内',
    suggestedRange: { min: rules.typicalMin, max: rules.typicalMax, unit: rules.unit }, factors };
}
```

### 临床评分：NEWS2

```typescript
interface NEWS2Input {
  respiratoryRate: number; oxygenSaturation: number; supplementalOxygen: boolean;
  temperature: number; systolicBP: number; heartRate: number;
  consciousness: 'alert' | 'voice' | 'pain' | 'unresponsive';
}
interface NEWS2Result {
  total: number;           // 0-20
  risk: 'low' | 'low-medium' | 'medium' | 'high';
  components: Record<string, number>;
  escalation: string;
}
```

评分表必须与英国皇家内科医学院规范完全一致。

### 告警严重级别与 UI 行为

| 严重级别 | UI 行为 | 临床医生操作要求 |
|----------|---------|----------------|
| 危急（Critical） | 阻断操作。不可关闭的弹窗。红色。 | 必须记录覆盖原因方可继续 |
| 重要（Major） | 内联警告横幅。橙色。 | 必须确认后方可继续 |
| 次要（Minor） | 内联提示信息。黄色。 | 仅供知悉，无需操作 |

危急告警绝不能自动关闭，也不能以 Toast 通知形式实现。覆盖原因必须存储在审计轨迹中。

### CDSS 测试（对漏报零容忍）

```typescript
describe('CDSS — 患者安全', () => {
  INTERACTION_PAIRS.forEach(({ drugA, drugB, severity }) => {
    it(`检测 ${drugA} + ${drugB}（${severity}）`, () => {
      const alerts = checkInteractions(drugA, [drugB], []);
      expect(alerts.length).toBeGreaterThan(0);
      expect(alerts[0].severity).toBe(severity);
    });
    it(`检测 ${drugB} + ${drugA}（反向）`, () => {
      const alerts = checkInteractions(drugB, [drugA], []);
      expect(alerts.length).toBeGreaterThan(0);
    });
  });
  it('体重缺失时阻断 mg/kg 药物', () => {
    const result = validateDose('gentamicin', 300, 'iv');
    expect(result.valid).toBe(false);
    expect(result.factors).toContain('weight_missing');
  });
  it('优雅处理格式错误的药物数据', () => {
    expect(() => checkInteractions('', [], [])).not.toThrow();
  });
});
```

通过标准：100%。漏掉任何一个相互作用即为患者安全事件。

### 反模式

- 使 CDSS 检查可跳过或可选（无需记录原因）
- 以 Toast 通知形式实现相互作用检查
- 对药物或临床数据使用 `any` 类型
- 硬编码相互作用对，而非使用可维护的数据结构
- 在 CDSS 引擎中静默捕获错误（必须显式暴露故障）
- 体重缺失时跳过基于体重的剂量验证（必须阻断，不得放行）

## 示例

### 示例 1：药物相互作用检查

```typescript
const alerts = checkInteractions('warfarin', ['aspirin', 'metformin'], ['penicillin']);
// [{ severity: 'critical', pair: ['warfarin', 'aspirin'],
//    message: '出血风险增加', recommendation: '避免联合使用' }]
```

### 示例 2：剂量验证

```typescript
const ok = validateDose('paracetamol', 1000, 'oral', 70, 45);
// { valid: true, suggestedRange: { min: 500, max: 4000, unit: 'mg' } }

const bad = validateDose('paracetamol', 5000, 'oral', 70, 45);
// { valid: false, message: '超过绝对最大值 4000mg' }

const noWeight = validateDose('gentamicin', 300, 'iv');
// { valid: false, factors: ['weight_missing'] }
```

### 示例 3：NEWS2 评分

```typescript
const result = calculateNEWS2({
  respiratoryRate: 24, oxygenSaturation: 93, supplementalOxygen: true,
  temperature: 38.5, systolicBP: 100, heartRate: 110, consciousness: 'voice'
});
// { total: 13, risk: 'high', escalation: '紧急临床复查。考虑转入 ICU。' }
```
