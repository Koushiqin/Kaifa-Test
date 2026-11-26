# 08: 证据 manifest 与脱敏/PII 审计

**What to build:** 对 raw 证据做脱敏和 PII 审计，把脱敏副本放入 deliverable 目录并生成 manifest：每个文件记录时间、URL、状态码、SHA256；扫描 secret 值和 PII 值，不误伤字段名；截图需要打码或人工目视；审计失败仍产出失败报告。

**Blocked by:** 04, 05, 06

**Status:** ready-for-agent

- [ ] raw 证据只存在本地忽略目录；deliverable 目录只放脱敏副本，原始 secret 永不进入交付路径。
- [ ] manifest 覆盖 deliverable 全部文件，含时间、URL、状态码和 SHA256。
- [ ] 审计按值/secret pattern 扫描密码、cookie、Authorization、token、session 和学生 PII；字段名（authorization、cookie 等）不判失败。
- [ ] 截图做打码或人工目视；纯文本扫描不足以放行。
- [ ] 审计失败时产出失败报告并阻止交付；09 可以报告该失败，10 被阻断。
- [ ] 原始 HAR/trace/video 不在交付物中；只保留脱敏摘要。
