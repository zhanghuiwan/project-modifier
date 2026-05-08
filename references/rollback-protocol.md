# 回退操作详情

本文档详细说明三级回退策略的具体操作步骤，以及修改过程中的 Git 安全机制。

---

## 一、Git 安全机制

### 1.1 修改前的安全准备

在开始任何修改之前，确保当前项目状态已保存在 Git 中：

```bash
# 检查当前状态
git status

# 如果有未提交变更，先提交
git add .
git commit -m "chore: save current state before modification"

# 记录当前 commit hash（作为回退锚点）
git rev-parse HEAD
```

**回退锚点**：修改前的最新 commit 是本次修改的回退锚点。所有 Level 2 回退都回到这个锚点。

### 1.2 中大型修改的检查点

对于中大型修改，在规划文档中的每个关键步骤完成后创建检查点：

```bash
# 完成一个步骤后
git add .
git commit -m "feat(<scope>): complete step N - <description>"

# 如果是重要节点，打 tag
git tag "mod-checkpoint-<N>"
```

### 1.3 修改完成后的提交

```bash
# 提交所有变更
git add .
git commit -m "<type>(<scope>): <description>"

# 类型参考：
# fix: Bug 修复
# feat: 新功能
# refactor: 重构
# docs: 文档更新
# test: 测试相关
# chore: 构建/配置变更
```

---

## 二、三级回退操作详解

### Level 1 — 修复代码

**适用场景**：测试失败，原因是代码 bug，修改方案本身没有问题。

**操作步骤**：

1. 分析测试失败输出，定位 bug
2. 修改代码修复 bug
3. 重新运行失败的测试
4. 运行全量已有测试
5. 全部通过后继续

**Git 操作**：
```bash
git add .
git commit -m "fix(<scope>): <bug description>"
```

**注意**：Level 1 不改变修改方向，只是修复实现中的 bug。

---

### Level 2 — 回退到本次修改前

**适用场景**：修改方向出现偏差，或引入了难以修复的问题，需要重新来过。

**操作步骤**：

1. 确认回退锚点（修改前的 commit hash）
   ```bash
   # 方法A：查看 git log 找到修改前的 commit
   git log --oneline -20

   # 方法B：如果有 tag
   git tag -l "mod-*"
   ```

2. 回退到修改前的状态
   ```bash
   # 方法A：使用 git revert（保留历史，推荐）
   # 找到本次修改涉及的所有 commit
   git log <anchor-commit>..HEAD --oneline
   # 逆序 revert
   git revert <commit-range>

   # 方法B：使用 git reset（删除历史）
   git reset --hard <anchor-commit>
   ```

3. 重新执行修改
   - 如果有规划文档 → 回到规划文档，重新执行
   - 如果没有规划文档 → 回到 M2 重新分析修改点

4. 运行全量测试

5. 通过后继续

**Git 操作**：
```bash
# 回退后重新实现
git add .
git commit -m "<type>(<scope>): re-implement after rollback - <description>"
```

**记录**：在 CHANGELOG 中记录回退和重新实现的过程。

---

### Level 3 — 回退到接手前（需用户确认）

**适用场景**：需要回退到本次修改开始之前的 Git 状态，可能影响非本次修改的已有内容。

**⚠️ 重要：必须用户确认**

回退到接手前的状态意味着：
- 可能丢失其他人在此期间的提交
- 可能丢失项目已有的未提交修改
- 项目状态将回到一个完全不同的时间点

**操作步骤**：

1. 向用户说明回退的影响
   - 将回退到哪个 commit
   - 会丢失哪些变更
   - 回退后项目的状态

2. 用户确认后执行回退
   ```bash
   # 查看目标 commit
   git log --oneline -30

   # 回退
   git reset --hard <target-commit>
   ```

3. 重新从 M1（代码理解）开始
   - 因为项目状态已经改变，之前的理解可能不再准确
   - 重新评估项目状态
   - 重新分析修改点
   - 重新制定规划文档（如果需要）

4. 重新执行修改

**Git 操作**：
```bash
git reset --hard <target-commit>
# 重新理解和修改...
git add .
git commit -m "chore: restart modification after Level 3 rollback"
```

---

## 三、回退决策辅助

### 决策流程

```
修改/测试失败
     │
     ▼
 是代码bug吗？
     │
  YES└──→ Level 1: 修复代码 → 重测 → 继续
     │
     NO
     ▼
 修改方向有偏差吗？
     │
  YES└──→ Level 2: 回退到本次修改前 → 重新执行
     │
     NO
     ▼
 需要回退到接手前吗？
     │
  YES└──→ 询问用户确认 → Level 3: 回退 → 从M1重新开始
     │
     NO
     ▼
 修改方案本身有问题
     │
     ▼
 更新规划文档 → 重新评估影响 → 用户确认 → 重新执行
```

### 回退记录

每次执行回退时，在 Git commit message 和 CHANGELOG 中记录：
- 回退级别（Level 1/2/3）
- 回退原因
- 回退目标（哪个 commit/tag）
- 重新执行的方案

---

## 四、无 Git 项目的情况

如果项目没有 Git 管理：

1. **M0 阶段已初始化 Git**（如果按流程执行，不应出现此情况）
2. **如果跳过了 M0**：立即初始化
   ```bash
   git init
   git add .
   git commit -m "chore: initial commit"
   ```
3. 初始化后，回退策略正常适用

---

## 五、多人协作项目的注意事项

如果项目是多人协作的（有远程仓库、多人提交）：

1. **修改前先 pull**：确保本地代码是最新的
2. **回退前先沟通**：Level 2/3 回退可能影响其他人的工作
3. **使用分支**：建议在独立分支上修改
   ```bash
   # 创建修改分支
   git checkout -b modification/<brief-description>

   # 修改完成后合并
   git checkout main
   git merge modification/<brief-description>
   ```
4. **避免 force push**：除非确认安全
