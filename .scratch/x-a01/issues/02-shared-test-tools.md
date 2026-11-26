# 02: 共享测试工具

**What to build:** 一个供 03-10 复用的测试工具层：状态判定（failed > blocked > passed，管理员/助教判定树）、证据 recorder（raw 与 deliverable 分离）、脱敏（按值/secret pattern 和 PII，不误伤字段名）、官方结果 schema 校验、manifest 生成（时间/URL/status/SHA256）。所有纯函数用 fixture 测试，不依赖被测站。

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] decideStatus() 实现 failed > blocked > passed 和管理员/助教判定树；fixture 覆盖 blocked vs na、failed vs blocked。
- [ ] evidence recorder 把原始证据写 raw 目录、脱敏副本写 deliverable 目录；交付路径不出现原始 secret。
- [ ] redact() 扫描值/secret pattern 和 PII，不把字段名（authorization、cookie）当泄露；fixture 覆盖字段名保留。
- [ ] validateResultSchema() 按官方 X-A01 schema 校验；不通过 fail closed。
- [ ] manifest 生成记录时间、URL、状态码、SHA256。
- [ ] 所有工具用 fixture 测试，不依赖被测站。
