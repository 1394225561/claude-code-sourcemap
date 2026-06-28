# Claude Code 源码学习指南

欢迎来到 Claude Code 源码学习工作区！这里是你学习 Claude Code 内部架构的起点。

## 快速开始

### 第一步：查看学习计划
打开 [学习计划](lessons/0001-claude-code-学习计划.html) 了解完整的学习路径。

### 第二步：快速入门
查看 [快速入门指南](reference/quick-start.html) 用 10 分钟了解核心架构。

### 第三步：项目结构
查看 [项目结构速查表](reference/project-structure.html) 掌握文件布局。

## 源码说明

> **重要提示**: 本学习指南基于从 `@anthropic-ai/claude-code@2.1.88` 包中恢复的源码。
>
> - **源码恢复不完整**: 部分文件（如 `types/message.ts`）在代码中被引用但不存在
> - **构建时生成的代码**: 某些类型和常量可能在构建时生成
> - **Feature Flags**: 代码中大量使用 `feature()` 函数进行功能开关
> - **内部 API**: 部分代码标记为 `ant-only`（Anthropic 内部使用）

## 学习资源

### 核心文档
- [学习计划](lessons/0001-claude-code-学习计划.html) - 完整的学习路径和阶段规划
- [快速入门](reference/quick-start.html) - 10 分钟了解核心架构
- [项目结构速查表](reference/project-structure.html) - 文件和目录速查

### 参考资料
- [术语表](reference/glossary.html) - 核心概念和术语定义
- [项目概览](MISION.md) - 学习目标和范围
- [资源列表](RESOURCES.md) - 学习资源汇总

### 学习记录
- [学习记录](learning-records/) - 学习进度和心得

## 学习路径

### 阶段 1：入门基础（1-2 周）
- [ ] 阅读学习计划
- [ ] 理解项目整体架构
- [ ] 研究入口文件 `entrypoints/cli.tsx`
- [ ] 理解工具系统接口 `Tool.ts`

### 阶段 2：核心引擎（2-3 周）
- [ ] 研究查询引擎 `query.ts`
- [ ] 理解消息类型和状态管理
- [ ] 学习上下文管理

### 阶段 3：工具系统（2-3 周）
- [ ] 学习 3-5 个典型工具实现
- [ ] 理解工具权限系统
- [ ] 掌握工具扩展机制

### 阶段 4：服务层（2-3 周）
- [ ] 研究 API 客户端
- [ ] 理解 MCP 协议
- [ ] 学习认证流程

### 阶段 5：UI 和交互（1-2 周）
- [ ] 理解 Ink 框架使用
- [ ] 学习组件系统
- [ ] 研究用户交互流程

### 阶段 6：高级主题（持续）
- [ ] 插件开发
- [ ] 技能系统
- [ ] 多代理协作

## 核心文件清单

### 入口文件
| 文件 | 大小 | 职责 | 关键函数 |
|------|------|------|----------|
| `entrypoints/cli.tsx` | ~39KB | CLI 入口 | `main()` |
| `main.tsx` | ~800KB | 主入口 | `main()`, `run()` |
| `replLauncher.tsx` | ~4KB | REPL 启动器 | `launchRepl()` |

### 核心引擎
| 文件 | 大小 | 职责 | 关键函数/类 |
|------|------|------|-------------|
| `query.ts` | ~69KB | 查询引擎核心 | `query()` (generator) |
| `QueryEngine.ts` | ~47KB | 查询引擎类 | `QueryEngine` class |
| `Tool.ts` | ~30KB | 工具基类 | `Tool`, `ToolUseContext` |
| `tools.ts` | ~17KB | 工具注册表 | `getTools()` |
| `context.ts` | ~6KB | 上下文管理 | `getSystemContext()` |

### 工具系统
| 工具 | 复杂度 | 学习价值 |
|------|--------|----------|
| `BashTool` | 中 | Shell 执行、沙箱、权限 |
| `FileReadTool` | 低 | 文件操作基础 |
| `FileEditTool` | 中 | 差异匹配、验证 |
| `AgentTool` | 高 | 子代理、并发、隔离 |
| `MCPTool` | 高 | MCP 协议集成 |

## 学习建议

### DO - 推荐做法
- 从入口文件开始，跟着执行流程走
- 使用调试器跟踪代码执行
- 添加日志观察行为
- 画图总结理解
- 做笔记记录问题

### DON'T - 避免做法
- 不要试图记住每一行代码
- 不要跳过类型定义
- 不要忽视测试
- 不要孤立地看代码

## 工具推荐

### IDE 设置
- VS Code + TypeScript 支持
- 启用代码导航和重构
- 安装 GitLens 查看历史

### 调试工具
- Node.js 调试器
- Chrome DevTools (远程调试)
- 日志分析工具

## 获取帮助

### 遇到问题时
1. 查看术语表理解概念
2. 搜索相关代码
3. 阅读测试用例
4. 问我（你的 AI 助手）

### 有用的命令
```bash
# 搜索代码
grep -r "functionName" src/

# 查看文件结构
find src/ -type f -name "*.ts" | head -20

# 运行测试
npm test
```

## 下一步

现在你已经了解了学习框架，建议你：

1. 打开 [学习计划](lessons/0001-claude-code-学习计划.html) 查看详细内容
2. 从 `entrypoints/cli.tsx` 开始阅读代码
3. 遇到问题随时问我

祝你学习愉快！🚀
