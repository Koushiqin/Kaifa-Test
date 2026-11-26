# 01: 执行器骨架与账号库

**What to build:** 一个可运行的执行器骨架：按账号键加载共享账号库，生成运行配置（task ID、runId、证据目录、headed/headless），启动真实 Chrome，并为每个角色创建独立 browser context。它建立 raw 与 deliverable 两层证据目录，但不执行 X-A01 用例，也不产出正式结果文件。

**Blocked by:** None (can start immediately)

**Status:** ready-for-agent

- [ ] 按账号键读取共享账号库；缺少键时清晰失败。
- [ ] 密码不出现在 stdout、日志或配置输出中。
- [ ] 启动真实 Chrome，至少创建两个独立 context；cookie/storage 不串。
- [ ] 生成唯一 runId，创建 raw 证据目录和 deliverable 证据目录；raw 目录在本地忽略路径下，deliverable 只允许脱敏副本。
- [ ] 默认禁用 Playwright trace/video；如开启，只写 raw 目录，不进入 deliverable。
- [ ] 一条命令可端到端运行骨架，且不访问被测站。
- [ ] 不产出 X-A01 用例结果或正式结果文件。
