---
name: healthcare-cdss-patterns
description: 临床决策支持系统（CDSS）开发模式。涵盖药物相互作用检查、剂量验证、临床评分（NEWS2、qSOFA）、告警严重性分类，以及与 EMR 工作流的集成。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# 医疗 CDSS 开发模式

用于构建集成到 EMR 工作流的临床决策支持系统的开发模式。CDSS 模块属于患者安全关键系统——对假阴性零容忍。

## 使用时机

- 实现药物相互作用检查
- 构建剂量验证引擎
- 实现临床评分系统（NEWS2、qSOFA、APACHE、GCS）
- 设计异常临床值告警系统
- 构建带安全检查的药物医嘱录入
- 将化验结果解读与临床上下文集成

## 工作原理

CDSS 引擎是一个**无副作用的纯函数库**。输入临床数据，输出告警。这使其完全可测试。

三个核心模块：

1. **`checkInteractions(newDrug, currentMeds, allergies)`** — 根据当前用药和已知过敏史检查新药。返回按严重性排序的 `InteractionAlert[]`，使用 `DrugInteractionPair` 数据模型。
2. **`validateDose(drug, dose, route, weight, age, renalFunction)`** — 根据基于体重、年龄调整和肾功能调整的规则验证处方剂量。返回 `DoseValidationResult`。
3. **`calculateNEWS2(vitals)`** — 根据 `NEWS2Input` 计算英国国家早期预警评分 2。返回包含总分、风险等级和升级指导的 `NEWS2Result`。

```
EMR UI
  ↓ (用户录入数据)
CDSS Engine (纯函数，无副作用)
  ├── 药物相互作用检查器
  ├── 剂量验证器
  ├── 临床评分 (NEWS2, qSOFA 等)
  └── 告警分类器
  ↓ (返回告警)
EMR UI (内联显示告警，危急时阻断操作)
```

### 药物相互作用检查

```typescript
interface DrugInteractionPair {
  drugA: string;           // generic name
  drugB: string;           // generic name
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
        message: `Cross-reactivity with documented allergy: ${allergy}`,
        recommendation: 'Do not prescribe without allergy consultation' });
    }
  }
  return alerts.sort((a, b) => severityOrder(a.severity) - severityOrder(b.severity));
}
```

相互作用对必须是**双向的**：如果药物 A 与药物 B 存在相互作用，则药物 B 与药物 A 也存在相互作用。

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
  if (!rules) return { valid: true, message: 'No validation rules available', suggestedRange: null, factors: [] };
  const factors: string[] = [];

  // SAFETY: if rules require weight but weight missing, BLOCK (not pass)
  if (rules.weightBased) {
    if (!patientWeight || patientWeight <= 0) {
      return { valid: false, message: `Weight required for ${drug} (mg/kg drug)`,
        suggestedRange: null, factors: ['weight_missing'] };
    }
    factors.push('weight');
    const maxDose = rules.maxPerKg * patientWeight;
    if (dose > maxDose) {
      return { valid: false, message: `Dose exceeds max for ${patientWeight}kg`,
        suggestedRange: { min: rules.minPerKg * patientWeight, max: maxDose, unit: rules.unit }, factors };
    }
  }

  // Age-based adjustment (when rules define age brackets and age is provided)
  if (rules.ageAdjusted && patientAge !== undefined) {
    factors.push('age');
    const ageMax = rules.getAgeAdjustedMax(patientAge);
    if (dose > ageMax) {
      return { valid: false, message: `Exceeds age-adjusted max for ${patientAge}yr`,
        suggestedRange: { min: rules.typicalMin, max: ageMax, unit: rules.unit }, factors };
    }
  }

  // Renal adjustment (when rules define eGFR brackets and eGFR is provided)
  if (rules.renalAdjusted && renalFunction !== undefined) {
    factors.push('renal');
    const renalMax = rules.getRenalAdjustedMax(renalFunction);
    if (dose > renalMax) {
      return { valid: false, message: `Exceeds renal-adjusted max for eGFR ${renalFunction}`,
        suggestedRange: { min: rules.typicalMin, max: renalMax, unit: rules.unit }, factors };
    }
  }

  // Absolute max
  if (dose > rules.absoluteMax) {
    return { valid: false, message: `Exceeds absolute max ${rules.absoluteMax}${rules.unit}`,
      suggestedRange: { min: rules.typicalMin, max: rules.typicalMax, unit: rules.unit },
      factors: [...factors, 'absolute_max'] };
  }
  return { valid: true, message: 'Within range',
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

评分表必须严格按照英国皇家内科医师学会规范实现。

### 告警严重性与 UI 行为

| 严重性 | UI 行为 | 临床医生操作要求 |
|--------|---------|----------------|
| Critical（危急） | 阻断操作。不可关闭的模态框。红色。 | 必须记录覆盖理由方可继续 |
| Major（重要） | 内联警告横幅。橙色。 | 确认后方可继续 |
| Minor（次要） | 内联信息提示。黄色。 | 仅供知晓，无需操作 |

危急告警绝对不能自动关闭，也不能以 toast 通知形式实现。覆盖理由必须存储在审计追踪中。

### CDSS 测试（对假阴性零容忍）

```typescript
describe('CDSS — Patient Safety', () => {
  INTERACTION_PAIRS.forEach(({ drugA, drugB, severity }) => {
    it(`detects ${drugA} + ${drugB} (${severity})`, () => {
      const alerts = checkInteractions(drugA, [drugB], []);
      expect(alerts.length).toBeGreaterThan(0);
      expect(alerts[0].severity).toBe(severity);
    });
    it(`detects ${drugB} + ${drugA} (reverse)`, () => {
      const alerts = checkInteractions(drugB, [drugA], []);
      expect(alerts.length).toBeGreaterThan(0);
    });
  });
  it('blocks mg/kg drug when weight is missing', () => {
    const result = validateDose('gentamicin', 300, 'iv');
    expect(result.valid).toBe(false);
    expect(result.factors).toContain('weight_missing');
  });
  it('handles malformed drug data gracefully', () => {
    expect(() => checkInteractions('', [], [])).not.toThrow();
  });
});
```

通过标准：100%。任何一个漏检的相互作用都是患者安全事件。

### 反模式

- 将 CDSS 检查设为可选或无需记录理由即可跳过
- 以 toast 通知实现相互作用检查
- 对药物或临床数据使用 `any` 类型
- 硬编码相互作用对而非使用可维护的数据结构
- 在 CDSS 引擎中静默捕获错误（必须显式暴露故障）
- 体重缺失时跳过基于体重的剂量验证（必须阻断，而非通过）

## 示例

### 示例 1：药物相互作用检查

```typescript
const alerts = checkInteractions('warfarin', ['aspirin', 'metformin'], ['penicillin']);
// [{ severity: 'critical', pair: ['warfarin', 'aspirin'],
//    message: 'Increased bleeding risk', recommendation: 'Avoid combination' }]
```

### 示例 2：剂量验证

```typescript
const ok = validateDose('paracetamol', 1000, 'oral', 70, 45);
// { valid: true, suggestedRange: { min: 500, max: 4000, unit: 'mg' } }

const bad = validateDose('paracetamol', 5000, 'oral', 70, 45);
// { valid: false, message: 'Exceeds absolute max 4000mg' }

const noWeight = validateDose('gentamicin', 300, 'iv');
// { valid: false, factors: ['weight_missing'] }
```

### 示例 3：NEWS2 评分

```typescript
const result = calculateNEWS2({
  respiratoryRate: 24, oxygenSaturation: 93, supplementalOxygen: true,
  temperature: 38.5, systolicBP: 100, heartRate: 110, consciousness: 'voice'
});
// { total: 13, risk: 'high', escalation: 'Urgent clinical review. Consider ICU.' }
```
