# 项目Git工作流 README.md

# 项目Git工作流规范

本文档用于规范机器人项目多人协同开发的Git使用流程，从项目初始化到上线发布，全程标准化操作，确保代码整洁、协同顺畅，避免冲突及版本混乱问题。

## 一、核心分支规范（固定3条核心分支）

仓库始终保持3条核心分支，禁止直接修改核心分支代码，所有开发工作在功能分支完成。

|分支名称|分支作用|状态说明|操作限制|
|---|---|---|---|
|master|线上稳定版，用于机器人实际部署、对外发布|最稳定，可直接运行，版本可追溯|禁止直接提交、修改，仅能从beta分支合并|
|dev|开发集成版，用于团队功能集成、初步联调|不稳定，包含最新开发功能，用于协同联调|禁止直接提交，仅能从功能分支合并|
|beta|测试/仿真版，用于功能测试、bug修复|基本可用，仅修复bug，不新增功能|禁止直接提交，仅能从dev分支合并|
## 二、完整工作流程（从0到上线）

### 第1步：项目初始化（首次创建工程）

初始化仓库并创建核心分支，完成项目基础代码提交。

```bash
git config --global user.name "Liaojinxian"
git config --global user.email "2382528894@qq.com"

git init
# 编写机器人工程基础代码（如配置文件、核心框架等）
git add .
git commit -m "初始化机器人工程：搭建基础框架、配置核心依赖"
git branch dev   # 创建开发分支
git branch beta  # 创建测试分支

```

初始化完成后，仓库拥有3条核心分支：master（默认）、dev、beta。

### 第2步：多人协同开发（功能开发规范）

所有开发人员需从dev分支拉取功能分支，禁止直接在核心分支开发。

#### 示例1：开发「底盘运动功能」

```bash
# 1. 切换到dev分支，拉取最新代码（确保与团队同步）
git checkout dev
git pull

# 2. 从dev分支创建功能分支（命名规范：feature/功能名称）
git checkout -b feature/chassis_move

# 3. 在该分支编写代码、调试功能
# ...（编写底盘运动相关代码）

# 4. 提交代码（提交信息需清晰，说明修改内容）
git add .
git commit -m "feat(chassis): 实现底盘前后左右运动控制，优化速度参数"

```

#### 示例2：开发「视觉识别功能」

```bash
git checkout dev
git pull
git checkout -b feature/vision_detect

# ...（编写视觉识别相关代码）
git add .
git commit -m "feat(vision): 实现摄像头图像采集与目标识别功能"

```

### 第3步：功能合并（集成到dev分支）

功能开发完成后，将功能分支合并到dev分支，进行集成联调。

```bash
# 1. 切换到dev分支，确保最新
git checkout dev
git pull

# 2. 合并功能分支（以底盘运动功能为例）
git merge feature/chassis_move

# 3. 若出现冲突，手动解决冲突后提交
git add .
git commit -m "merge: 合并底盘运动功能分支，解决配置文件冲突"

# 4. 推送到远程dev分支，供团队共享
git push origin dev

# 5. （可选）功能合并完成后，删除功能分支（本地+远程）
git branch -d feature/chassis_move
git push origin --delete feature/chassis_move

```

其他功能（如视觉识别）按上述步骤合并到dev分支，最终dev分支包含所有最新开发功能。

### 第4步：测试版本（合并到beta分支）

dev分支功能集成完成后，合并到beta分支，进行仿真、测试及bug修复。

```bash
# 1. 切换到beta分支，拉取最新代码
git checkout beta
git pull

# 2. 从dev分支合并代码
git merge dev

# 3. 测试过程中发现bug，直接在beta分支修复（命名规范：bugfix/问题描述）
git checkout -b bugfix/crash_when_walk
# ...（修复bug）
git add .
git commit -m "fix: 修复底盘行走时崩溃问题，优化异常处理逻辑"

# 4. 修复完成后，合并回beta分支
git checkout beta
git merge bugfix/crash_when_walk
git push origin beta

# 5. （可选）删除bugfix分支
git branch -d bugfix/crash_when_walk
git push origin --delete bugfix/crash_when_walk

```

beta分支仅用于bug修复，禁止新增功能，确保测试版本的稳定性。

### 第5步：正式发布（合并到master分支）

beta分支测试通过后，合并到master分支，发布正式稳定版，并打tag存档。

```bash
# 1. 切换到master分支，拉取最新代码
git checkout master
git pull

# 2. 从beta分支合并测试通过的代码
git merge beta

# 3. 打tag（版本号规范：v主版本.次版本.修订号，如v1.0.0）
git tag v1.0.0

# 4. 推送到远程master分支及tag
git push origin master
git push origin --tags

```

master分支为最终可部署版本，tag用于版本追溯，后续如需回退到某个正式版，可通过tag切换。

## 三、工作流程总结

```plaintext
每个人开发新功能 → 从dev拉feature分支
写完 → 合并到dev（开发集成、联调）
联调ok → 合并到beta（仿真测试、bug修复）
测试ok → 合并到master（正式上线）
上线 → 打tag（版本存档，便于追溯）

```

## 四、分支命名规范（团队统一）

为避免分支混乱，所有分支命名严格遵循以下规范，确保语义清晰、便于识别。

- **feature/xxx**：新功能开发分支，xxx为功能名称（小写，用下划线分隔）
        

    - 示例：feature/arm_control（手臂控制功能）、feature/joystick_control（手柄控制功能）

- **bugfix/xxx**：bug修复分支，xxx为bug描述（小写，用下划线分隔）
        

    - 示例：bugfix/camera_init_fail（摄像头初始化失败修复）、bugfix/motor_overload（电机过载问题修复）

- **dev**：开发集成分支（固定名称，禁止修改）

- **beta**：测试/仿真分支（固定名称，禁止修改）

- **master**：线上稳定版分支（固定名称，禁止修改）

## 五、日常常用Git命令（必背）

```bash
# 1. 切换到开发分支dev
git checkout dev

# 2. 拉取远程最新代码（默认拉取origin对应分支）
git pull

# 3. 从dev分支创建自己的功能分支
git checkout -b feature/你的功能名

# 4. 提交本地修改（add所有修改，commit说明清晰）
git add .
git commit -m "feat/bugfix: 具体修改内容"

# 5. 切换回dev分支，准备合并功能
git checkout dev

# 6. 合并自己的功能分支到dev
git merge feature/你的功能名

# 7. 推送到远程dev分支，共享给团队
git push origin dev

# 8. 发布正式版：切换到master，合并beta，打tag
git checkout master
git merge beta
git tag v1.0.0
git push origin master
git push origin --tags

# 9. 查看当前分支及状态
git branch
git status

# 10. 查看提交历史（简洁版）
git log --oneline

```

## 六、注意事项

1. 禁止直接在master、dev、beta分支提交代码，所有开发/修复工作必须在feature/bugfix分支完成。

2. 合并分支前，务必先拉取最新代码，避免冲突；若出现冲突，需与相关开发人员沟通后手动解决。

3. 提交信息需规范，前缀统一为：feat（新功能）、fix（bug修复）、merge（分支合并）、docs（文档修改），后面跟具体描述。

4. 功能分支完成合并后，及时删除（本地+远程），避免分支冗余。

5. 每次发布正式版后，必须打tag，便于后续版本回退和追溯。

6. 多人协同开发时，定期同步dev分支代码，避免分支分叉过久，导致合并失败（参考要求：MR前需合并目标分支最近48小时内的提交）。

7. 机器人项目专属冲突规避技巧：
  - 功能模块隔离：底盘、视觉、手臂、手柄控制等核心模块，分别放在独立文件夹（如src/chassis、src/vision），开发时仅修改自身模块文件，避免跨模块修改同一文件引发冲突。

  - 配置文件规范：机器人硬件配置（如电机参数、传感器参数）、版本配置（ROBOT_VERSION）统一放在单独的config文件夹，多人修改时需提前沟通，建议一人负责维护核心配置，避免同时修改同一配置文件。

  - 仿真与实机代码分离：仿真（mujoco/gazebo）和实机运行的代码分开存放（如src/sim、src/real），开发仿真功能时不修改实机代码，反之亦然，减少冲突概率。

  - 定期同步与沟通：每日开发前，先拉取dev分支最新代码；开发过程中，若需修改公共依赖（如ROS节点、核心工具类），提前在团队内告知，避免多人同时修改。

  - 提交粒度控制：机器人功能开发（如底盘运动、视觉识别）按小功能块提交，不一次性提交大量修改，若需修改多个文件，分多次提交，便于冲突排查和回滚。

  - 避免强制推送：核心分支（master、dev、beta）禁止使用git push --force，若分支历史不一致，需先拉取远程代码、解决冲突后再推送，防止覆盖他人代码。

  - 配置文件忽略：本地调试的临时配置（如本地IP、调试参数）、编译生成的文件（如devel、build文件夹）添加到.gitignore，避免提交到远程引发冲突。

## 七、工作流优势

- 代码整洁：分支职责明确，避免版本混乱，便于追溯问题。

- 协同顺畅：多人开发互不干扰，冲突可提前规避，降低沟通成本。

- 稳定性高：master分支始终保持稳定，避免开发中的代码影响机器人实际运行。

- 可维护性强：版本可追溯，bug修复可精准定位，后续迭代更高效。

## 八、提交规范
- <类型>: 简短描述（中文，15字内最佳）
- 类型小写英文
- 冒号为英文冒号，后面必须跟一个空格
- 描述简洁明确，不使用模糊词汇

- git commit -m "init: 初始化H723工程模板"

- init 项目 / 工程初始化、新建模板
- feat 新增功能、新增驱动、新增文件
- fix 修复 BUG、解决报错、修正逻辑
- refactor 代码重构、结构优化、整理目录
- docs 注释、文档、说明文字修改
- style 格式化代码、缩进、换行、空格
- perf 性能优化、提速、降功耗
- chore 配置文件、编译脚本、忽略文件更新


1、创建并切换到新分支 G4_Foc_Control
git checkout -b G4_Foc_Control
==
git branch G4_Foc_Control
git checkout G4_Foc_Control

2、验证分支是否创建成功
git branch

3、首次关联远程并推送
git push -u origin G4_Foc_Control

4、推送
git add .
git commit -m "feat: xxx修改说明"
git push


## 常见问题

### Q: Git 半成品临时备份（git stash）

适用于代码开发到一半，需要临时保存修改、切换分支处理其他任务，但不想提交 Commit 或推送远程仓库的场景。

## 1. 查看当前修改
```bash
# 查看修改文件
git status

# 查看代码差异
git diff
```

---

## 2. 临时保存修改

### 快速储藏

```bash
git stash
```

### 带备注储藏（推荐）

```bash
git stash push -m "半成品临时备份：FOC电流环调试中"
```

### 包含新增文件一起储藏

```bash
git stash push -u -m "半成品临时备份，含新增文件"
```

> `-u` 用于保存未跟踪（未 `git add`）的新建文件。

---

## 3. 切换分支

```bash
# 传统写法
git checkout 分支名

# 推荐写法（Git 2.23+）
git switch 分支名
```

---

## 4. 恢复储藏内容

查看储藏列表：

```bash
git stash list
```

恢复但保留储藏记录：

```bash
git stash apply stash@{0}
```

恢复并删除储藏记录：

```bash
git stash pop stash@{0}
```

---

## 5. 删除储藏

删除指定储藏：

```bash
git stash drop stash@{0}
```

删除所有储藏：

```bash
git stash clear
```

---

## 6. 常用流程

```bash
# 查看修改
git status

# 保存当前修改（包含新增文件）
git stash push -u -m "中途半成品临时备份"

# 切换分支
git switch 其他分支名

# 切回原分支
git switch G4_Foc_Control

# 恢复修改
git stash pop
```

---

## 7. 常用命令

| 功能 | 命令 |
|------|------|
| 查看修改 | `git status` |
| 查看差异 | `git diff` |
| 临时保存 | `git stash` |
| 带备注保存 | `git stash push -m "备注"` |
| 保存新增文件 | `git stash push -u -m "备注"` |
| 查看储藏 | `git stash list` |
| 恢复（保留记录） | `git stash apply stash@{0}` |
| 恢复（删除记录） | `git stash pop stash@{0}` |
| 删除指定储藏 | `git stash drop stash@{0}` |
| 删除全部储藏 | `git stash clear` |



问题及其解决方案（常用精简）
先存改动、暂不提交、后续分批修改整合

1. git status 先输入查看当前改动，确认是你想要备份的修改：
2. git stash save "Backup 1: Add Doxygen header comments for functions in C files"
   把你当前所有已修改、未提交的代码改动，打包临时存进 Git 储藏区；
   本地工作区会变回修改前的干净状态；
   不会产生任何 git commit 提交，版本历史保持干净，正好符合你「先存改动、暂不提交、后续分批修改整合」的需求。

后续常用配套命令
1. git stash list  查看储藏列表，确认存好了
2. git stash apply stash@{0}  恢复但保留储藏记录
3. git stash pop stash@{0}  恢复并删除储藏记录
4. git stash drop stash@{0}  删除指定储藏
5. git stash clear  删除全部储藏

遇到git stash apply stash@{0}，失败，冲突，（本地现在已经有一堆未提交修改）
方案一
先暂存你现在手上改的内容
git stash save "临时保存当前本地修改，准备恢复旧备份"
再应用你之前的备份
git stash apply 'stash@{1}'
方案二
# 丢弃所有工作区修改，不可恢复
git restore .
# 再应用 stash
git stash apply 'stash@{0}'


基于当前的这新建一个新分支的分支，G4_Foc_Ulter
- 当前分支:
commit 7af2de796817a3cd88330103d51b4c7814af786f (HEAD -> G4_Foc_DRV8301, origin/G4_Foc_DRV8301)
Author: Liaojinxian <2382528894@qq.com>
Date:   Sun Sep 20 17:17:50 2026 +0800

    (motor): add motor_ident.c/.h

- 目标分支:G4_Foc_Ulter
1. 新建并直接切换到新分支
git checkout -b G4_Foc_Ulter
2. 验证 创建完查看分支列表
git branch
3. 推送到远端（可选，需要同步到 github/gitee 远程仓库时）
git push -u origin G4_Foc_Ulter
