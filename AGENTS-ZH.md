# AGENTS.md

Microduck 的强化学习（RL）训练环境，基于 [mjlab](https://github.com/mujocolab/mjlab)（MuJoCo Warp）和 PPO（rsl_rl）构建。Microduck 是一款重约 800 g、高约 25 cm、配备 14 个 Dynamixel XL330 舵机的双足机器人。策略在此以 50 Hz 训练，导出为 ONNX，再由 `pollen-robotics/microduck` 仓库中的运行时部署到真实机器人。仿真到现实迁移（Sim2real）是整个项目的核心：下面的每一条约定，都源于违反它之后出现过“策略在查看器中表现正常，在硬件上却失败”的情况。

## 命令

```bash
uv run list-envs                                    # 当前实际注册的任务列表
uv run train <TASK_ID> --env.scene.num-envs 4096    # 训练（添加 --hf-jobs 可使用 Hugging Face Jobs）
uv run train <TASK_ID> --env.scene.num-envs 64 --agent.max_iterations 5   # 冒烟测试——始终先运行
uv run play <TASK_ID> --wandb-run-path <entity/project/run_id>
uv run scripts/export.py <TASK_ID> --wandb-run-path <...>   # → ONNX（将观测归一化器固化进去——必须使用此路径）
uv run publish --task <TASK_ID> --wandb-run-path <...> --checkpoint N --repo <user>/microduck-<name> --kind episodic --duration-s 4.0
                              # → HF Hub 仓库（policy.onnx + schema-2 manifest.json + README），守护进程通过 `robotctl policy add` 加载
                            #   添加 --share-training 可同时打包训练检查点（需主动选择，见下文）
uv run share-training --wandb-run-path <...> --checkpoint N [--onnx hf://<policy-repo>/<file>.onnx]
                                    # → <owner>/microduck-training 中的训练包（完整 .pt + 来源信息 + 教师模型）
uv run train <TASK_ID> --init-checkpoint hf://<owner>/microduck-training/bundles/<exp>/<run>_<iter>
                                    # 从训练包恢复训练（使用 MICRODUCK_WARM_START=1 可热启动另一个任务）
uv run scripts/infer_policy.py --walking out.onnx   # CPU MuJoCo 部署预演（与训练相同的 BAM M6 执行器；--no-bam = XML PD）
uv run --with pytest pytest tests/
```

用 64 个环境运行 5 次迭代的冒烟测试，只需几美分就能发现约 95% 的配置错误。未经冒烟测试，绝不要启动长时间训练。

## 仓库结构

- `src/mjlab_microduck/tasks/mdp.py` —— 所有自定义 MDP 函数（奖励、事件、观测、命令、课程）。新函数都添加在这里，并按任务分组。
- `src/mjlab_microduck/tasks/microduck_*_env_cfg.py` —— 每个任务家族一个 cfg 模块。`microduck_velocity_env_cfg.py` 既是主要的行走训练方案，也是其他环境构建或参照的共享基础（机器人、域随机化 DR、观测、命令）。
- `src/mjlab_microduck/tasks/__init__.py` —— 任务注册（基础任务 + `-Backlash-` 变体）。
- `src/mjlab_microduck/tasks/backlash.py` —— 将任意环境配置包装成对应的齿隙版本。
- `src/mjlab_microduck/robot/microduck_constants.py` —— 机器人配置、HOME 帧、BAM 执行器配置。
- `src/mjlab_microduck/robot/microduck/` —— 从 Onshape 导出的 MJCF（onshape-to-robot，每个模型对应一个 `config_mjcf_*.json`）+ 场景 + `add_backlash.py`。碰撞模型家族：`walk`（仅脚部）、`groundcontact`（精选的地面接触部件集合）、`allcollisions`（所有部件；XL330 外壳由 `name_servo_collision_geoms` 命名为 `*_servo_collision`，供 VelStand 的舵机冲击传感器使用）。每种模型都有一个通过 `add_backlash.py <xml> --backlash-deg 2.0` 生成的 `_backlash` 对应版本。
- `src/mjlab_microduck/actuator/friction_dr_bam.py` —— BAM 执行器 + 摩擦域随机化 + 齿隙编码器。
- `src/mjlab_microduck/export.py` —— ONNX 导出（归一化器已固化）；`scripts/export.py` 对其进行封装。
- `src/mjlab_microduck/publish/` —— `uv run publish`：schema-2 清单构建器 + ONNX 形状/冒烟检查关卡 + Hub 上传；`training.py` 对应 `uv run share-training`（训练包）。
- `src/mjlab_microduck/checkpoints.py` —— 检查点规格（`hf://...` 训练包或本地路径）、训练参数 `--init-checkpoint`，供训练、导出和蒸馏教师模型使用。契约见 `microduck` 仓库中的 `docs/policy-manifest.md`；仅支持发布恒定命令的回合式/持续式策略（phase/posture-flag 由策略集负责）。
- `scripts/` —— 导出封装、推理、仿真与现实对比、wandb 辅助工具。
- `tests/` —— 配置不变量与 MDP 函数回归测试（在 CPU 上运行，无需 GPU）。

## 不变量——不得破坏

- **观测布局为 61 维（actor），整个策略家族共享此布局**，以便运行时热切换策略：48 维基础本体感知 + 13 维命令块 `[twist(3), head_pose(4), body_pose(6)]`，顺序固定。不使用某个命令槽的环境要对其补零（保留观测项，使用极小的采样范围），绝不要删除槽位。
- **关节布局**（14 个舵机，在 walk/groundcontact 模型上，控制索引 = 关节索引）：0–4 为左腿（hip_yaw、hip_roll、hip_pitch、knee、ankle），5–8 为颈部/头部（neck_pitch、head_pitch、head_yaw、head_roll），9–13 为右腿。在滚轮/齿隙模型中，被动关节会交错插入，因此绝不要在 MDP 函数中硬编码关节索引；应使用 mdp.py 中的 `_servo_joint_ids` / `_servo_joint_pos` 辅助函数（在普通模型上是恒等映射，在其他模型上也能正确处理）。
- **所有无驱动关节都命名为 `passive_*`**（轮子、齿隙铰链）。所有执行器/观测/奖励选择器都使用 `^(?!passive_).*`。新增关节时保持这一前缀约定；新增的 `passive_` 正则表达式不能误匹配齿隙关节（使用 `^passive_.*wheel`，而非 `^passive_.*`）。
- **执行器使用 BAM**（电压控制的 XL330 模型，摩擦由执行器计算）。这意味着：任何独立环境配置都必须注册 `expand_bam_friction_fields` 启动事件；关节摩擦域随机化必须缩放执行器的 `friction_scale`。BAM 下的 `dof_frictionloss` 会被清零，因此随机化它不会产生任何效果，也不会报错。
- **观测归一化已开启** → 必须将归一化器固化到 ONNX 中。`scripts/export.py` 会完成这一步；仿真中的 play 会掩盖这个问题（因为它始终应用归一化器），所以绝不要手工转换检查点。
- **策略不使用滤波**（训练中不对动作做低通滤波）。没有配套的运行时开关和迁移测试，就不要添加 EMA 滤波；训练时使用、部署时不使用，或反过来，都会破坏迁移效果。
- **域随机化不得跨重置累积。** mjlab 1.3.0 中 `operation="add"/"scale"` 的 `dr.*` 操作原生不会累积（它们会重新读取编译时默认值）；自定义 DR 函数必须先恢复默认值，再应用随机化。曾有一个会累积的质心（CoM）随机化器，导致数月间的每次长时间训练都退化。
- 如果某个观测被重映射到传感器视角（齿隙编码器、偏置），针对同一物理量的跟踪奖励也必须使用同一视角进行测量，否则策略会因为纠正自己看到的误差而受罚。
- `-Backlash-` 任务变体必须与基础任务使用相同类型的机器人模型（walk / groundcontact / rollers），避免齿隙 A/B 对比受到混杂因素影响。

## 构建新环境——工作流程

1. **选择最接近的模板**并在其基础上构建，不要从零开始：运动控制 → velocity 方案；以某个姿态结束的回合式技巧 → standup；命令控制的双状态切换 → sitstand；动态动作 → roulade（阅读其 cfg 文档字符串，其中记录了 5 次训练的经验演进）。基于 `make_microduck_velocity*_env_cfg` 构建，可以自动保持 DR / 观测 / 噪声 / 延迟同步；如果从 mjlab 基础模板独立构建，必须自行移植完整的 DR + 观测噪声 + NaN 防护体系（搜索 velocity 接入的内容：带 `_safe` 的 critic 观测项、带 sensor_names 的 `nan_state` 终止条件、`expand_bam_friction_fields`、编码器偏置、IMU 安装偏差）。
2. **训练前先在仿真中验证物理假设**——这是最能节省时间的一步：
   - 目标/静止姿态必须是稳定平衡态：从带噪声的初始状态出发，保持其控制输入 3 秒，检查倾斜程度，而不只是高度（只记录 z 的稳定性测试，会把已经倒下的状态报告为“静止正常”）。
   - 在仿真中使用实际机器人模型测量目标高度（例如站立策略下的躯干 z），绝不要跨模型版本沿用。曾有一个偏差仅 5 mm 的 STAND_Z，让目标连续数天都无法实现。
3. **配置约定**：在 cfg 文件顶部放置 `ENABLE_*` 开关和调好的常量；工厂函数采用 `make_..._env_cfg(play: bool, rough: bool)`；在 `tasks/__init__.py` 中注册（适用时还要加入 `_BACKLASH_TASKS` 表）；使用独立的 `RslRl...RunnerCfg` 和不同的 `experiment_name`。可以使用对称镜像损失（`symmetry.py` 中有 61 维映射表），但默认关闭，非对称任务绝不能使用。
4. **编写 cfg 测试**（参见 `tests/test_*_cfg.py`）：确认关节索引能在实际模型上正确解析、奖励权重符号符合预期、门控在预期条件下开启/关闭。这些测试在 CPU 上运行，用于锁定不变量。
5. **冒烟测试**（64 个环境、5 次迭代）：能够构建并步进，无 NaN，观测为 61 维，每个奖励项都能计算，ONNX 能够导出。
6. 开始训练并观察日志（见下文），预计需要 2–5 轮迭代来逐个解决奖励漏洞利用问题。这是正常现象，下面的经验能帮助避开其中大部分问题。

## 奖励设计——每条规则都来自实际教训

- **符号约定（曾让四个环境踩坑）：** mdp.py 中有两种惩罚风格。mjlab 基础代价函数返回值 ≥ 0 → 使用负权重。microduck 自带负号的函数（返回值 ≤ 0 的 `*_penalty`、`*_l1`）→ 使用正权重。给自带负号的惩罚配置负权重，会双重取负，变成对违规行为的奖励，策略就会反复刷取它（用屁股弹跳、猛砸着坐下）。**可靠的检查方法：每次训练中，wandb 的每一个 `Episode_Reward/<penalty>` 都必须 ≤ 0。**
- **RL 会严格按奖励的字面定义优化。** 每个约束不完整的自由度都会被利用（用抛射式甩动代替翻滚、用肩部侧滚代替矢状面翻滚、用头部支成三脚架代替站立）。应使用基于状态的硬门控（支撑接触、朝向轴检查、锁存状态）来定义什么才算目标动作，而不是依赖小幅惩罚来引导。
- **不要设置“大奖”：** 任何“到达 X”的奖励都必须限制发放速率或让目标渐变。提前到达后持续逐步领奖的目标状态，就是足以让策略不惜采取任意剧烈动作的大奖。对于命令驱动的状态转换，应跟踪一个逐渐变化的内部目标（恒定速率混合）；超前于斜坡目标不会获得奖励，因此缓慢运动本身就是最优解。单靠速度上限惩罚，积分后的代价有界，最终会败给这种奖励。
- **绝不要以处于坏状态作为正奖励的触发条件**（倒地、低位），否则策略会停在代价最低且符合条件的姿态上刷奖励。应改用基于势函数的奖励塑形（奖励 Δ进度，例如 Δcos(tilt)：起身有奖励，保持不动为零，无法停留刷取）。对于静止任务，应针对每种稳定倒地姿态（仰卧 / 俯卧 / 侧卧）逐项审查正奖励：如果倒地仍能保留大部分奖励，策略就会倒地。
- **以目标姿态落定的回合式任务：** 从 t=0 开始使用单一固定目标（关节和高度上的高斯 + L1，标准差适当放宽）+ |a_z| 冲击惩罚 + 两层直立约束。不要使用关键帧/途经点轨迹（策略会停留在途经点）。路径本来就应该由 RL 自行发现。
- **正则项分两类。** 阻碍运动的项（body_ang_vel、angular_momentum、姿态标准差）会惩罚动态动作在物理上必需的运动，动态任务中应保持低权重。平滑项（action_rate、joint_torque_rate）能抑制抖动而不阻碍缓慢的大幅运动，可以放心赋予权重，但必须在技能被发现之后再引入（通过课程从接近 0 开始）：探索困难技能时，任何尝试成本都会让“什么都不做”成为赢家。缓慢精细的任务（如伸手触及）需要比行走更强的平滑约束。
- **在环境间复制正则项时，应比较奖励总量，而不只是权重。** PPO 看到的是相对优势：当正向任务奖励总量增大到 4 倍时，相同 action_rate 权重的相对作用就会减弱到四分之一。
- **跟踪高斯的标准差：** 应约等于你仍然在意的误差，而不是最大误差；过宽时，小误差处没有梯度。但在收紧前，要先判断误差是策略能够消除的，还是目标行为本身固有的（占体重 38% 的头部在行走时必然摆动；过紧的瞬时头部跟踪标准差曾对行走施加过重惩罚，导致策略原地不动）。只对能够消除的部分计价，例如对 1 秒 EMA 施加 L1 惩罚，可以惩罚直流偏置，同时让振荡相互抵消。
- **在目标状态附近，乘法组合优于加法求和：** 当加法奖励组合存在折中吸引盆地时（例如通过倾斜让每项都拿到 80%），高斯乘积会因任一因子不足而大幅下降。但标准差必须足够宽，让当前策略能获得明显的分数，否则梯度不可见，行为不会改变。
- **关节停在硬限位上：** 对问题关节添加基于 qpos 的接近限位惩罚；原生 `dof_pos_limits` 仅在活动范围最后约 7.5% 的区域内触发，控制命令侧的惩罚无效（较宽的 ctrlrange 是有意设计的，低 kp 舵机需要超出目标的控制量）。

## 命令、观测与失活权重

- **始终没有非零值的命令输入，其权重会一直失活。** 每个命令槽从第 0 步起都应保留较小的非零采样范围（即使奖励权重为 0），使对应输入神经元保持活跃，供后续课程使用。
- **必须显式训练零命令行为**（采用 `zero_command_prob` 一类的精确零值采样）：均匀采样几乎永远不会产生全零命令，而全零恰好是部署时的空闲状态。
- 稀少但重要的命令区域需要明确分桶，例如原地转向（`rel_turn_in_place_envs`）：独立均匀采样曾让旋转只占经验的约 2%，最终始终未能学会。

## 课程学习

- 步数指环境步数：`iteration × 24`（`NUM_STEPS_PER_ENV = 24`）。
- 使用经过验证的职责划分：`microduck_mdp.reward_weight` 用于权重调度，专门的参数课程用于命令/事件范围。`mdp.reward_weight` 是阶跃函数，并不进行插值，因此要将斜坡离散成多个阶段。
- 通过管理器修改项配置（`env.event_manager.get_term_cfg(...)`），绝不要修改 `env.cfg.events[...]`。管理器初始化时会深拷贝配置，因此写入 `env.cfg` 不会产生效果，也不会报错（强制设置生成状态的评估脚本也会踩这个坑）。
- **每个阶段都要与策略实际学会的能力对齐**：当前训练子集尚未巩固前，不要提高初始状态混合分布的难度；技能尚未形成前，不要引入额外成本。如果 wandb 指标恰好在课程阶段边界出现阶跃下降，就说明节奏不对，应延长阶段或推迟引入时间，绝不要提前。
- 逆向课程初始状态（让回合从动作中途开始，包括接近完成的位置）是解决“学会开头，却始终学不会最后一步”的可靠方法，否则能力前沿无法获得同策略数据。

## 训练操作与日志解读

- wandb 项目为 `mjlab_microduck`；日志位于 `logs/<experiment_name>/`；使用 `--agent.load-checkpoint model_XXXX.pt --agent.resume True` 恢复训练。
- **热启动 ≠ 恢复训练。** mjlab 的 runner 会在检查点中保存 `common_step_counter`，并在加载时将它与迭代次数一起恢复，因此加载另一个任务的检查点，会让所有基于步数的课程在第 1 次迭代就跳到最终阶段。添加前缀 `MICRODUCK_WARM_START=1`（mdp.py Patch 5，会转发到 HF Jobs），可在保留权重/归一化器/优化器的同时，将两种计数都重置为 0。还要在目标配置中将继承自源任务的课程收敛到最终阶段（参见 velstand 中的 `_collapse_curricula_to_final`），因为加载的策略是在这些条件下训练的。只允许在归一化器兼容的任务之间热启动（stand 专家策略的 twist 标准差为 0.005，绝不要用它初始化行走环境）。
- 已部署策略的来源追溯：自 2026-10-27 起导出的 ONNX，会在元数据中携带来源信息（`run_path`、`task_id`、`checkpoint`、`checkpoint_sha256`）；更早的 Hub 文件只有 `run_path=None`。`pollen-robotics/microduck-training` 中的训练包会记录这些信息（各 `bundle.json` 中的 `onnx[]`，通过 sha256 匹配）。对于尚未打包的 ONNX，`scripts/find_onnx_source.py` 会与 wandb 检查点进行精确权重匹配来定位训练运行（2026-09：walk = 441tzs6d@3750，stand = 69u48n8l@9750；2026-10 Hub v7 velstand = j4i6yoq2@1250；谱系见 `docs/velstand_policy.md`，sitstand 见 `docs/sitstand_policy.md`）。
- 每次迭代都要观察：平均奖励上升，同时回合长度符合任务要求；每个惩罚项 ≤ 0；主要任务奖励项确实在增长（总奖励可能仅因正则项改善而上升，技巧却始终没有发生）。`Episode_Reward/<term>` 记录的是加权值：某项权重为 0 时，无论行为如何都记录为 0，因此必须结合权重调度解读。
- 预算：简单回合式技巧在 4096 个环境下约需 1000 次迭代；步态和课程较重的恢复任务需要 4000–6000 次迭代。
- **先测量，再推断。** 当某次训练“失败”时，在修改奖励前，先对实际检查点进行无界面评估（按初始状态类型分别开展成组测试、最终状态聚类、角速度曲线分析）。过去一些“失败”最终发现其实是检查点太早、成功判据将同一行为簇拆成两半，或者奖励上限与测得的物理规律冲突。仿真指标可能通过，但视频在肉眼看来仍然不合格；既要看视频，也要检查具体是哪个几何体/哪条轴发生接触。
- 如实报告 rollout 展现的行为（“能翻滚，但三次中有一次脸着地”），不要只说“成功了！”。由用户决定何时算足够好。

## 共享训练检查点（训练包）

完整参考文档：`docs/training_checkpoints.md`。智能体必须遵守以下规则：

- **始终需要用户主动选择。** 除非用户在本次会话中明确要求，否则绝不要上传检查点，也不要将训练包仓库改为公开。`publish` 不带 `--share-training` 时不会额外共享训练内容；`share-training` 默认创建私有仓库，除非指定 `--public`。
- **绝不要把 `.pt` 文件放进策略（ONNX）仓库。** 训练包放在单独的 `<owner>/microduck-training` 仓库中，每个训练包对应 `bundles/<experiment>/<run>_<iter>/` 下的一个文件夹。
- **当 Hub 上的策略更新时**（通过 `publish`，或手动更新 `pollen-robotics/microduck-policies`），如果用户希望共享检查点：对准确对应的检查点运行 `share-training`，并传入 `--onnx <the deployed file>`。它会拒绝记录 actor 权重无法逐位精确匹配的 ONNX；如果被拒绝，说明来源对应关系有误。应使用 `scripts/find_onnx_source.py` 找到正确检查点，绝不要删除 `--onnx` 来绕过检查。标签与策略集保持一致（如 `--tag velstand-v9`）。
- **教师模型会自动随包包含**（从该次运行自身的 `bc_cfg` 读取），因此 velstand 训练包无需访问 wandb 也能训练。不要手动上传教师模型。
- **训练包不可变**（以运行 + 迭代次数为键；同一已有 ID 下不允许放入不同检查点）。只有 `onnx` 记录可以合并到已有训练包中。
- **加载：** `--init-checkpoint <spec>` 替代 `--agent.resume` + `--wandb-run-path`（同时使用会报错）；它执行的是恢复训练（课程继续）。跨任务则应热启动：添加前缀 `MICRODUCK_WARM_START=1`（见上文“热启动 ≠ 恢复训练”）。首先检出训练包中的 `code.commit`，并应用 `uncommitted.diff`。
- `share-training` 的 `--checkpoint-file` 模式（不使用 wandb）会将当前检出版本的 HEAD 记录为来源，并拒绝工作区有未提交修改的情况；共享前需先提交并推送。

## 仿真到现实迁移的易错点（曾耗费数周实机调试）

- 全新执行一次 `uv sync` 才是判断依据（HF Jobs 就会这样做）：任何仅靠本地手动安装的软件包才能工作的内容，到了远端都会失败。务必让 `pyproject.toml` 如实声明依赖。
- **Wheel 包区分架构。** 在 linux-`aarch64`（DGX Spark / GB10）上，PyPI 的 torch wheel 仅支持 CPU（`2.9.1+cpu`，`torch.version.cuda is None`），因此 `torch.cuda.device_count() == 0`，mjlab 的 `select_gpus()` 会索引空列表，在第 0 次迭代之前就触发 `IndexError`。`[tool.uv.sources]` 仅针对 `aarch64` 将 torch 路由到 cu129 索引（cu129 与 warp 打包的 CUDA 工具包匹配；x86_64/HF Jobs 继续使用 PyPI）。有两个会悄然失效的点，均由 `tests/test_aarch64_cuda_torch.py` 锁定：torch 必须保持为直接依赖（uv 仅对直接依赖应用 `[tool.uv.sources]`，删除看似冗余的 `torch==` 固定版本声明会让路由失效）；版本固定必须保持使用 `==`，因为 CUDA 索引中的构建版本比 PyPI 更新（曾有一个 `>=` 悄悄将 torch 从 2.9.1 升到 2.13.0）。
- 符合物理规律的限制：25 cm 高的机器人自然翻滚时，角速度会达到 3.5–5.5 rad/s。不要通过限速强加人类尺度的速度直觉；应通过冲击和乱甩指标（|a_z|、action_rate、支撑门控）来抑制剧烈行为，而不是限制旋转速度。
- IMU 域随机化以零为中心，训练的是对安装偏差幅度的容忍能力，无法补偿系统性的安装偏置（这需要运行时校准）。
- 实机部署会在共享观测契约下热切换 ONNX 策略（行走 / 站立 / 技巧）。在接触真实机器人之前，先通过 `scripts/infer_policy.py` 预演，并正确写入命令槽（姿态标志位放在 twist 的 vx 槽中；全零输入表示“站立”，看起来就会像“策略无视按钮”）。
