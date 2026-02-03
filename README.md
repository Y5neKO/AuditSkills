# 自动化代码审计Skills

## 流程

```
完整流程覆盖验证
  ┌───────────┬───────────────────────────┬───────────┬───────────────────────┬─────────┐
  │   阶段    │          子代理             │ Task 调用 │       产物文件        │  状态   │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 0   │ 准备工作                   │ -         │ -                     │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 1   │ web-entry-discovery       │ ✅ L51    │ web_entries.json      │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ sink-point-scanner        │ ✅ L57    │ sink_points.json      │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ security-asset-scanner    │ ✅ L63    │ security_assets.json  │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ data-model-analyzer       │ ✅ L69    │ data_models.json      │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 1.5 │ entry-function-analyzer   │ ✅ L86    │ entry_functions.json  │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 2   │ dataflow-tracer           │ ✅ L104   │ traces.json           │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ vulnerability-validator   │ ✅ L112   │ validated_vulns.json  │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ vulnerability-correlator  │ ✅ L120   │ correlated_vulns.json │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ poc-generator             │ ✅ L128   │ pocs/                 │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 2.5 │ forward-flow-tracer       │ ✅ L142   │ forward_traces.json   │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 3   │ access-control-auditor    │ ✅ L159   │ access_control.json   │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │           │ business-logic-auditor    │ ✅ L167   │ business_logic.json   │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 4   │ attack-chain-orchestrator │ ✅ L182   │ attack_chains.json    │ ✅      │
  ├───────────┼───────────────────────────┼───────────┼───────────────────────┼─────────┤
  │ Phase 5   │ report-generator          │ ✅ L197   │ report.md/json        │ ✅      │
  └───────────┴───────────────────────────┴───────────┴───────────────────────┴─────────┘
```



## 使用

```sh
cp -r AuditSkills/agent .claude
cp -r AuditSkills/skills .claude
```



## 案例

### SpringBlade 4.8.0.RELEASE

使用Skills版本：https://github.com/Y5neKO/AuditSkills/tree/6d3ffd152c8a5bfcf16c3b680e95469f5fe1a7ad

使用模型：GLM-4.7

版本：https://github.com/chillzhuang/SpringBlade/tree/fa431033fa19b3d50cec120c74313a38e51397c8

报告：[扫描报告](./audit_result/SpringBlade-fa431033fa19b3d50cec120c74313a38e51397c8/report.md)

### blade-tool 4.8.0.RELEASE

使用Skills版本：https://github.com/Y5neKO/AuditSkills/tree/a3ee8defae6d2c36b214e4c2a7e3d1d84f797ae8

使用模型：GLM-4.7

版本：https://github.com/chillzhuang/blade-tool/tree/80a2844c4a3cb9dd4372eb9a0441307a71a8ca9a

报告：[扫描报告](./audit_result/blade-tool-80a2844c4a3cb9dd4372eb9a0441307a71a8ca9a/report.md)

