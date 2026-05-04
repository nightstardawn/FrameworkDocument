# Git 基础操作指南

> 版本：v0.3

## 学习资源

| 资源 | 链接 |
| --- | --- |
| Git 与 GitHub 简易教程 | https://www.bilibili.com/video/BV1s3411g7PS/ |
| Git 学习练习（交互式） | https://learngitbranching.js.org/?locale=zh_CN |

---

## 一、初始设置

```bash
# 设置用户名和邮箱
git config --global user.name "nightstardawn"
git config --global user.email "1321665118@qq.com"

# 初始化仓库（具体看项目是否有初始化）
git init
```

---

## 二、添加与提交文件

### 添加文件到暂存区

```bash
git add re01.txt   # 添加指定文件
git add .          # 添加当前目录下的所有文件
```

### 提交文件

```bash
# 方式一：通过 vim 编辑器输入提交信息
git commit
# 按 a 进入编辑模式 → 输入提交信息 → 按 Esc 退出编辑 → 输入 :wq 保存退出

# 方式二：直接附带提交信息（推荐）
git commit -m "提交信息"
```

### 查看提交记录

```bash
git log
```

### 提交信息规范

| 前缀 | 含义 |
| --- | --- |
| `feat` | 新功能 |
| `fix` | 修补 Bug |
| `docs` | 文档变更 |
| `style` | 格式调整（不影响代码运行） |
| `refactor` | 重构（非新增功能，非修复 Bug） |
| `test` | 增加测试 |
| `chore` | 构建过程或辅助工具的变动 |

示例：`fix(text): change content`

---

## 三、版本回退

```bash
git reset --hard HEAD^      # 回退到上一个版本
git reset --hard HEAD~1     # 回退到上上一个版本（~N 表示回退 N 个版本）
git reset --hard <版本编号>  # 回退到指定版本
```

---

## 四、分支管理

### 创建与查看

```bash
git branch <分支名>         # 创建分支（创建后修改需要 commit）
git branch                  # 查看所有本地分支
git branch -m <旧名> <新名> # 重命名分支
```

### 切换分支

```bash
git checkout <分支名>       # 切换到指定分支
git checkout -b <分支名>    # 创建并切换到新分支
```

### 合并与删除

```bash
git merge <分支名>          # 将指定分支合并到当前分支
git branch -d <分支名>      # 删除本地分支
```

### 推送分支到远程

```bash
git push <库名> <分支名>    # 推送指定分支
git push <库名> --all       # 推送所有分支
```

### 删除远程分支

```bash
git push <库名> -d <分支名>
```

> **提示**：在 GitHub 中设置默认分支：Settings → Branches → Default branch

---

## 五、远程仓库操作

### 克隆仓库

```bash
git clone <仓库url> .    # 克隆到当前文件夹
```

### 关联远程仓库

```bash
git remote add <远程库名> <url>   # 关联远程仓库
git remote -v                     # 查看当前关联的远程库
git remote remove <远程库名>      # 删除远程库关联
```

### 推送代码

```bash
git push -u <远程库名> <分支名>              # 推送并关联当前分支
git push <远程库> <本地分支>:<远程分支>      # 推送本地分支到指定远程分支
```

### 拉取更新

```bash
git fetch <库名>   # 下载远程仓库的所有更新，但不会自动合并，需手动 merge
git pull           # 拉取远程代码并自动合并到当前分支
```

> **注意**：如果本地版本低于远程库，`push` 会被拒绝。需要先通过 `fetch` + `merge` 或 `pull` 同步版本后再推送。

### 关闭 Pull Request

```bash
git commit --allow-empty -m "Close#4"
git push
```
