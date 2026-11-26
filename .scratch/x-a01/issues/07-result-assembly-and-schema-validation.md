# 07: 结果组装与官方 schema 校验

**What to build:** 把三个用例结果片段组装成工作台官方 expected.json：填 runId、expectedRevision、environment、build、时间戳、measurementScope、metrics、caseResults、notes；environment/build 从 03 预检结果和用户 commit 读取；本地通过官方 X-A01 schema 校验；不导入工作台。

**Blocked by:** 04, 05, 06

**Status:** ready-for-agent

- [ ] caseResults 恰好覆盖 X-A01-01/02/03 各一次；状态为官方枚举；failed/blocked/na 必须有 note。
- [ ] 元数据：唯一 runId、expectedRevision 1（组装前重读）、environment 来自 03 预检证据（无证据才写“测试环境，隔离性待核实”）、build 优先用户 commit（没有才写“unknown”）、时间戳 +08:00、measurementScope functional、metrics {}、thresholds []、artifacts []。
- [ ] notes 只引用确定性的证据 manifest 位置，不依赖 manifest 内容，以便与 08 并行。
- [ ] 候选版本未知和隔离性待核实写入 notes，作为任务级限制。
- [ ] 本地 schema 校验通过；不通过则 fail closed。
- [ ] 不执行任何工作台导入。
