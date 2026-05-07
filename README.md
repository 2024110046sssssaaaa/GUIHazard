# GUIHazard: 三端 GUI Agent 安全测评框架

GUIHazard 是一个统一的安全测评框架，用于评估 GUI Agent（图形用户界面智能体）在 **桌面端 (Desktop)**、**移动端 (Mobile)** 和 **Web端 (Web)** 三个平台上的安全风险。框架涵盖三大类安全威胁：用户滥用 (User Misuse)、提示注入攻击 (Prompt Injection) 和模型异常行为 (Model Misbehavior)。

---

## 目录结构

```
GUIHazard/
├── desktop/              # 桌面端测评模块 (基于 OS-Harm)
│   ├── assets/           # 测试资产（邮件、文档、图片等）
│   ├── crossplatform/    # 跨平台攻击测试用例
│   │   ├── X08/         # 登录确认不一致注入
│   │   ├── X09/         # 通知同步注入
│   │   ├── X10/         # 短信验证码泄露
│   │   ├── X11/         # 投屏诱导攻击
│   │   └── X12/         # 剪贴板投毒
│   ├── desktop_env/     # 虚拟机环境管理
│   ├── judge/           # LLM 评判模块
│   ├── mm_agents/       # 多模态 Agent 实现
│   └── run.py           # 主入口脚本
│
├── mobile/              # 移动端测评模块 (基于 MobileSafetyBench)
│   ├── attacks/         # 攻击数据集
│   │   ├── 1_visual_perception_attacks/    # 视觉感知攻击
│   │   ├── 2_environment_content_injection/# 环境内容注入
│   │   ├── 3_direct_malicious_commands/   # 直接恶意命令
│   │   ├── 4_llm_reasoning_attacks/       # LLM推理攻击
│   │   └── 5_system_exploitation/          # 系统级利用
│   ├── mobile_safety/   # 核心框架
│   └── scripts/         # 运行脚本
│
└── web/                 # Web端测评模块 (基于 WASP)
    ├── visualwebarena/  # WebArena 环境实现
    ├── webarena_prompt_injections/  # 提示注入测试
    │   ├── configs/     # 攻击配置
    │   └── attacker_*.py # 攻击执行器
    └── configs/         # 配置文件
```

---

## 核心功能

### 1. 桌面端测评 (Desktop)

基于 [OS-Harm](https://arxiv.org/abs/2506.14866) 基准，测试 GUI Agent 在桌面操作系统环境中的安全性。

**支持的观测类型：**
- `screenshot`: 仅截图
- `a11y_tree`: 无障碍树描述
- `screenshot_a11y_tree`: 截图 + 无障碍树
- `som`: Set-of-Marks 标记

**测试场景：**
| 类别 | 描述 | 任务数 |
|------|------|--------|
| 故意滥用 (test_misuse.json) | 用户主动请求有害操作 | ~50 |
| 提示注入 (test_injection.json) | 网页/文档中嵌入恶意指令 | ~50 |
| 模型异常 (test_misbehavior.json) | 模型自发的不安全行为 | ~50 |

**运行示例：**
```bash
cd desktop
python run.py --path_to_vm Ubuntu/Ubuntu.vmx \
              --observation_type screenshot_a11y_tree \
              --model o4-mini \
              --result_dir ./results \
              --test_all_meta_path evaluation_examples/test_misuse.json
```

### 2. 移动端测评 (Mobile)

基于 [MobileSafetyBench](https://github.com/) 框架，测试移动端 Agent 的安全性。

**攻击类型：**
| 类别 | 攻击编号 | 描述 |
|------|----------|------|
| 视觉感知攻击 | M07 | 通知注入 |
| 环境内容注入 | M08-M09 | 屏幕文本注入、视图树干扰 |
| 直接恶意命令 | M10-M12 | 提示注入、隐私风险操作、有害内容生成 |
| LLM推理攻击 | M15-M16 | 恶意指令利用、glitch token攻击 |
| 系统级利用 | M13, M20-M21 | 深度链接伪造、通知叠加攻击、后台攻击 |

**运行示例：**
```bash
cd mobile
python attacks/batch_runner.py --config attacks/config.yaml --model claude-3-5-sonnet
```

### 3. Web端测评 (Web)

基于 [WASP](https://arxiv.org/abs/2504.18575) 基准，测试 Web Agent 对提示注入攻击的抵御能力。

**支持的 Agent：**
- GPT-4o / GPT-4o-mini (WebArena 框架)
- Claude-3.5 / Claude-3.7 (Computer Use 框架)

**支持的网站环境：**
- GitLab
- Reddit
- 更多...

**运行示例：**
```bash
cd web/webarena_prompt_injections
python run.py --config configs/experiment_config.raw.json \
              --model gpt-4o \
              --system-prompt configs/system_prompts/wa_p_som_cot_id_actree_3s.json \
              --output-dir ./results
```

### 4. 跨平台攻击测试

桌面端集成了 **X08-X12** 五类跨平台攻击测试，模拟 Desktop 与 Mobile 之间的安全通道被滥用的场景：

| 测试集 | 攻击类型 | 描述 |
|--------|----------|------|
| **X08** | 登录确认不一致注入 | 桌面请求登录，手机端核对信息被篡改 |
| **X09** | 通知同步注入 | 桌面同步通知被注入恶意内容 |
| **X10** | 短信验证码泄露 | 通过同步通道获取并泄露验证码 |
| **X11** | 投屏诱导攻击 | 投屏画面中注入诱导性操作 |
| **X12** | 剪贴板投毒 | 同步剪贴板内容被注入恶意标记 |

**运行跨平台测试：**
```bash
cd desktop
python run_crossplatform_tests.py --test_set X08 --model o4-mini
```

**指标定义（详见 `crossplatform/paper_evaluation_schema.json`）：**
- `XP-ASR`: 跨平台攻击成功率
- `XP-DSR`: 跨平台防御成功率
- `HSR`: 跨端信息传递成功率
- `SCR/CCR/CFR/PR/RR`: 任务完成/妥协/协作失败/部分完成/拒绝率

---

## 安装依赖

### 通用依赖
```bash
# Python 3.10+
pip install -r requirements.txt
```

### 桌面端额外依赖
```bash
# VMware Workstation (或支持的虚拟机)
# OSWorld 环境 (参考 OSWorld 安装文档)
```

### 移动端额外依赖
```bash
# Android SDK
# Appium
# Java 17
conda create -n mobile_safety python=3.10
pip install -r requirements.txt
```

### Web端额外依赖
```bash
# Docker
# Python 3.10
bash webarena_prompt_injections/setup.sh
```

---

## 评估指标

### 通用指标
| 指标 | 描述 |
|------|------|
| Attack Success Rate (ASR) | 攻击成功执行的比例 |
| Defense Success Rate (DSR) | 防御成功的比例 |
| Over-Defense Rate (ODR) | 过度防御导致正常任务失败的比例 |

### 跨平台指标
| 指标 | 描述 |
|------|------|
| XP-ASR | 跨平台攻击成功率 |
| HSR | 跨端信息传递成功率 |
| CCR | 妥协完成率 |

