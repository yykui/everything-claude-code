---
name: healthcare-phi-compliance
description: 医疗应用中受保护健康信息（PHI）和个人身份信息（PII）的合规模式。涵盖数据分类、访问控制、审计追踪、加密及常见泄露途径的防护。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# 医疗 PHI/PII 合规模式

在医疗应用中保护患者数据、临床医生数据和财务数据的模式。适用于 HIPAA（美国）、DISHA（印度）、GDPR（欧盟）及通用医疗数据保护规范。

## 使用时机

- 构建任何涉及患者记录的功能
- 为临床系统实现访问控制或身份验证
- 设计医疗数据的数据库 schema
- 构建返回患者或临床医生数据的 API
- 实现审计追踪或日志记录
- 审查代码中的数据暴露漏洞
- 为多租户医疗系统设置行级安全（RLS）

## 工作原理

医疗数据保护在三个层面运作：**分类**（什么是敏感数据）、**访问控制**（谁可以查看）和**审计**（谁已查看）。

### 数据分类

**PHI（受保护健康信息）** — 任何可识别患者身份且与其健康相关的数据：患者姓名、出生日期、地址、电话、电子邮件、国家身份证号（SSN、Aadhaar、NHS 编号）、病历号、诊断、用药、化验结果、影像、保险单号和理赔详情、预约和住院记录，或以上任意组合。

**医疗系统中的 PII（非患者敏感数据）**：临床医生/员工个人信息、医生诊费结构和支付金额、员工薪资和银行信息、供应商付款信息。

### 访问控制：行级安全

```sql
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;

-- 按机构限定访问范围
CREATE POLICY "staff_read_own_facility"
  ON patients FOR SELECT TO authenticated
  USING (facility_id IN (
    SELECT facility_id FROM staff_assignments
    WHERE user_id = auth.uid() AND role IN ('doctor','nurse','lab_tech','admin')
  ));

-- 审计日志：仅允许插入（防篡改）
CREATE POLICY "audit_insert_only" ON audit_log FOR INSERT
  TO authenticated WITH CHECK (user_id = auth.uid());
CREATE POLICY "audit_no_modify" ON audit_log FOR UPDATE USING (false);
CREATE POLICY "audit_no_delete" ON audit_log FOR DELETE USING (false);
```

### 审计追踪

每次 PHI 访问或修改都必须记录日志：

```typescript
interface AuditEntry {
  timestamp: string;
  user_id: string;
  patient_id: string;
  action: 'create' | 'read' | 'update' | 'delete' | 'print' | 'export';
  resource_type: string;
  resource_id: string;
  changes?: { before: object; after: object };
  ip_address: string;
  session_id: string;
}
```

### 常见泄露途径

**错误消息：** 永远不要在向客户端抛出的错误消息中包含患者识别数据。详细信息仅记录在服务端日志中。

**控制台输出：** 永远不要记录完整的患者对象。使用不透明的内部记录 ID（UUID）——而非病历号、国家身份证号或姓名。

**URL 参数：** 永远不要将患者识别数据放入可能出现在日志或浏览器历史记录中的查询字符串或路径段中。仅使用不透明的 UUID。

**浏览器存储：** 永远不要将 PHI 存储在 localStorage 或 sessionStorage 中。PHI 仅保留在内存中，按需获取。

**服务角色密钥：** 永远不要在客户端代码中使用 service_role 密钥。始终使用 anon/publishable 密钥，让 RLS 强制执行访问控制。

**日志与监控：** 永远不要记录完整的患者记录。仅使用不透明的记录 ID（非病历号）。在将堆栈跟踪发送到错误跟踪服务之前进行脱敏处理。

### 数据库 Schema 标注

在 schema 层面标记 PHI/PII 字段：

```sql
COMMENT ON COLUMN patients.name IS 'PHI: patient_name';
COMMENT ON COLUMN patients.dob IS 'PHI: date_of_birth';
COMMENT ON COLUMN patients.aadhaar IS 'PHI: national_id';
COMMENT ON COLUMN doctor_payouts.amount IS 'PII: financial';
```

### 部署检查清单

每次部署前：
- 错误消息和堆栈跟踪中无 PHI
- console.log/console.error 中无 PHI
- URL 参数中无 PHI
- 浏览器存储中无 PHI
- 客户端代码中无 service_role 密钥
- 所有 PHI/PII 表均启用 RLS
- 所有数据修改均有审计追踪
- 已配置会话超时
- 所有 PHI 端点均需 API 认证
- 已验证跨机构数据隔离

## 示例

### 示例 1：安全与不安全的错误处理

```typescript
// 不好——在错误中泄露 PHI
throw new Error(`Patient ${patient.name} not found in ${patient.facility}`);

// 好——通用错误，详细信息仅在服务端记录（仅使用不透明 ID）
logger.error('Patient lookup failed', { recordId: patient.id, facilityId });
throw new Error('Record not found');
```

### 示例 2：多机构隔离的 RLS 策略

```sql
-- A 机构的医生无法查看 B 机构的患者
CREATE POLICY "facility_isolation"
  ON patients FOR SELECT TO authenticated
  USING (facility_id IN (
    SELECT facility_id FROM staff_assignments WHERE user_id = auth.uid()
  ));

-- 测试：以 doctor-facility-a 身份登录，查询 facility-b 的患者
-- 预期：返回 0 行
```

### 示例 3：安全日志记录

```typescript
// 不好——记录可识别的患者数据
console.log('Processing patient:', patient);

// 好——仅记录不透明的内部记录 ID
console.log('Processing record:', patient.id);
// 注意：即使是 patient.id 也应该是不透明的 UUID，而非病历号
```
