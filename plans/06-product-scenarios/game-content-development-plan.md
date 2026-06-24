---
id: "plans-game-content-development-001"
title: "游戏内容开发业务智能体计划"
type: "plan"
category: "plans/product-scenarios"
tags: ["plan", "game", "content-development", "agent-team"]
version: "0.1.0"
created: "2026-05-10"
updated: "2026-05-10"
status: "draft"
---

# 游戏内容开发业务智能体计划

## 目标

用 Legion 组织一个游戏研发团队，覆盖策划、美术、程序、QA、运营五类工种，展示多 Agent 协作、知识沉淀、流程审批和成本调度能力。

## 组织模板

```text
制作人 Agent
├── 策划主管 Agent
│   ├── 系统策划 Agent
│   ├── 数值策划 Agent
│   └── 文案策划 Agent
├── 美术主管 Agent
│   ├── 概念美术 Agent
│   └── UI/动效 Agent
├── 技术主管 Agent
│   ├── 客户端 Agent
│   ├── 服务端 Agent
│   └── 工具链 Agent
├── QA 主管 Agent
│   ├── 功能测试 Agent
│   └── 回归测试 Agent
└── 运营 Agent
```

## 核心工作流

| 阶段 | 输入 | Agent | 输出 | 治理点 |
|------|------|-------|------|--------|
| 玩法立项 | 市场/目标用户/竞品 | 制作人/策划主管 | feature brief | 策略审批 |
| GDD 编写 | feature brief | 系统策划/文案策划 | GDD 页面 | 写入 Know 待审核 |
| 数值设计 | GDD | 数值策划 | 表格/公式/边界 | Aegis + 人工抽查 |
| 原型实现 | GDD/数值表 | 客户端/服务端 | playable prototype | 预算门控 |
| 美术接入 | GDD/UI flow | 美术/UI Agent | 资源清单/规范 | 版权与风格审核 |
| QA 回归 | build | QA Agent | bug list/test report | 质量门控 |
| 版本沉淀 | 交付物/报告 | 运营/Know | 发布记录/复盘 | K50 审核 |

## 知识库设计

| 知识域 | 示例 |
|--------|------|
| GDD | 玩法、系统、关卡、角色、经济系统 |
| Art Bible | 风格、色彩、UI 规范、资源命名 |
| Tech Wiki | 架构、接口、构建、部署 |
| QA Wiki | 测试用例、Bug 模式、回归清单 |
| LiveOps Wiki | 活动模板、用户反馈、数据复盘 |

## 验收

- 一个功能从 brief 到测试报告可通过工作流串起来。
- GDD 被 Know 摄取并可被程序/美术/QA Agent 查询。
- 数值和实现变更有审计和版本记录。
- 高成本模型只在策划推理、架构和质量评审中使用。

