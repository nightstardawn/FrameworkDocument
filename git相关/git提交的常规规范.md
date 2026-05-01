# Git 提交规范

## 格式

```
<type>(<scope>): <subject>
```

- `type` — 必填，提交类型
- `scope` — 可选，影响的模块范围
- `subject` — 必填，简短描述（50字以内）

## Type 类型

| Type | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 Bug |
| `refactor` | 重构（不改变功能） |
| `art` | 美术资源增删改 |
| `docs` | 文档变更 |
| `chore` | 构建/配置/工具变更 |
| `perf` | 性能优化 |
| `test` | 测试相关 |

## Scope 范围

按项目模块划分，常用范围：

- `UI` — UI框架及窗口
- `Event` — 事件系统
- `Timer` — 计时器
- `Log` — 日志系统
- `Base` — 单例/基础管理器
- `Editor` — 编辑器工具

## 示例

```
feat(UI): 添加背包窗口及物品展示逻辑
fix(Timer): 修复暂停后恢复计时偏移问题
art: 添加主界面背景图和按钮切图
refactor(Event): 将事件ID改为自增分配方式
chore: 更新 .gitignore 忽略 Bundles 目录
docs: 补充 UIFramework 生命周期说明
```

## 原则

- 一个 commit 只做一件事，不混合不相关的改动
- subject 描述"做了什么"，而非"改了哪个文件"
- 有关联的 Issue 或任务可在 body 中注明
