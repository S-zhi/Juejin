# Issue 状态与 Agent 调度

本项目已实现 **Issue 状态流转、TypeSafe Jev 评审和 action 标签分配**。要让整个流程运转，还需要配置 API Key，并接入一个真正执行任务的 Agent。Jev 只判断是否需要更强模型，不负责开发、测试或创建 PR。

## 启用前需要补齐什么

1. **部署 Workflow 并创建标签**：将整个 `.github/` 放进目标仓库的默认分支，启用 GitHub Actions。先按下面的“手动创建与更新仓库标签”步骤，将 `.github/label.yaml` 同步到 GitHub。
2. **配置 Jev 密钥**：在仓库 **Settings → Secrets and variables → Actions** 添加 `TYPESAFE_API_KEY`。默认模型为 `jev-latest`，配置见 [action-policy.yaml](.github/action-policy.yaml)。
3. **接入执行 Agent**：提供任务监听、阶段执行、结果反馈和失败处理能力，具体要求见下一节。在 `action-policy.yaml` 的 `agents` 中填写实际模型及能力；这里仅提供评审上下文，填写配置不会自动启动 Agent。

## 手动创建与更新仓库标签

**直接在 GitHub Actions 页面运行，无需本地安装依赖。** `.github/label.yaml` 是配置，`sync-labels.yml` 负责读取它并创建或更新仓库标签。只上传 YAML 不会自动同步，需要手动运行一次：

1. 确保 `.github/label.yaml`、`.github/scripts/sync_labels.py` 和 `.github/workflows/sync-labels.yml` 已提交到默认分支，并启用 GitHub Actions。
2. 打开仓库 **Actions → Sync labels → Run workflow**，选择 `main` 分支，再点击 **Run workflow**。
3. 等待运行成功，日志中会出现各标签的 `已同步标签: ...`。
4. 打开 [仓库 Labels 页面](https://github.com/S-zhi/Juejin/labels)，确认六个 `state:` 标签和两个 `action:` 标签已创建。

Workflow 自动准备 Python 和 PyYAML，使用运行环境预装的 GitHub CLI，以及自动提供的 `GITHUB_TOKEN`（已声明 `issues: write`）。不需要 Jev API Key，也无需额外配置个人 Token；手动运行者需要仓库写权限。

以后修改 `.github/label.yaml` 后，再手动运行 **Sync labels** 即可。同名标签更新颜色和描述，不重复创建；未声明的手工标签保留，也不会修改已有 Issue 的标签分配。从配置中移除标签不会删除 GitHub 上的旧标签。失败后可修复问题并重新运行。

<details>
<summary>可选：在本地运行同步脚本</summary>

**1. 准备环境并进入仓库根目录。** 先安装 Python（建议 3.12）、Git 和 GitHub CLI（`gh`）。尚未下载项目时执行：

```bash
git clone https://github.com/S-zhi/Juejin.git
cd Juejin
```

如果已经有本地项目，直接进入包含 `.github/` 和 `README.md` 的目录，不必再次克隆。以下命令都在该目录执行：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install PyYAML==6.0.2
```

以上激活命令适用于 macOS/Linux；Windows PowerShell 使用 `.venv\Scripts\Activate.ps1`。`.venv/` 是本地 Python 环境，不需要提交。

**2. 登录 GitHub。** 使用对目标仓库有标签写权限的账号；如果用细粒度 PAT，需要授予目标仓库 `Issues: write` 权限。

```bash
gh auth login --hostname github.com
gh auth status
```

**3. 先预览，再实际同步。** 以下以本仓库 `S-zhi/Juejin` 为目标；用于其他仓库时替换 `--repo` 后面的值。

```bash
# 只校验配置并打印计划执行的命令，不修改 GitHub
python .github/scripts/sync_labels.py --file .github/label.yaml --repo S-zhi/Juejin --dry-run

# 实际创建或更新 GitHub 仓库标签
python .github/scripts/sync_labels.py --file .github/label.yaml --repo S-zhi/Juejin
```

</details>

## 执行 Agent 必须具备什么能力

- **发现并领取任务**：通过轮询或外部 Webhook 发现带有 `action:agent` 的未关闭 Issue，读取描述、评论和当前 `state:`；避免重复领取，并在反馈前确认任务仍属于当前阶段。不要仅依赖另一个监听 `labeled` 的 GitHub Actions 来启动执行器，内置 `GITHUB_TOKEN` 添加的标签不会触发它。
- **执行阶段任务**：需求评审、方案设计、编码与测试、代码审查，以及跟进关联 PR 的实际合并。执行器自行配置模型凭据、运行环境和必要的仓库权限；Jev Key 不能代替执行模型的凭据。
- **成功反馈**：先在 Issue 中记录结果、验证证据和相关 PR，再移除 `action:agent`，保留当前 state，由状态控制推进下一阶段。
- **失败反馈**：先添加 `state:agent-failed`，再删除原 state，最后移除 `action:agent`。失败或取消后暂停，不自动重试或继续推进。

执行器修改标签时应使用具备 Issue 写权限的 **GitHub App Token 或 PAT**，才能触发现有的后续状态流转。若执行器使用内置 `GITHUB_TOKEN`，必须另外显式调用后续流程，不能只移除标签后等待新 Workflow。

完整接口、幂等要求和操作顺序见 [Agent 接入协议](.github/protocols/agent-action-protocol.md)。本项目尚未提供远端执行器、任务领取服务或执行结果回调服务。

## 接入后如何运转

```text
新建 Issue → state:needs-triage → Jev 评审
  ├─ 常规模型足够 → action:agent → 执行 Agent 处理
  └─ 需要更强模型 → action:human → 升级处理方处理
完成并移除最后一个 action → 下一阶段 → 再次评审
失败 → state:agent-failed → 暂停
```

正常阶段顺序：

```text
needs-triage → assess-plan → coding → code-review → wait-auto-merge
```

升级处理方完成后同样记录结果并移除自己的 `action:human`。末阶段只有在关联 PR **实际合并**后才算完成；释放 action 后不再推进，也不自动关闭 Issue。

`state-control.yaml` 处理新建 Issue 和 action 标签移除，并在状态确实变化后直接调用评审。`action-control.yaml` 只监听 `labeled`，只对新增 `state:` 标签执行评审。

## 评审依据与首次验证

发送给 Jev 的内容包括 Issue 标题、完整描述、全部可读取的 Issue 评论、当前阶段和 Agent 能力配置。通用及各阶段的提示词放在 [state-review.yaml](.github/prompts/state-review.yaml)，判断的是是否需要更强模型；信息暂缺、等待 CI 或未配置能力本身不作为升级理由。评论读取失败、模型调用失败或评审期间上下文变化时，不添加 action 标签。

完成配置后，创建一个测试 Issue，确认初始 state 和 action 都已添加；再确认执行器能记录结果、移除 action，并触发下一阶段。最后验证一次失败任务能够进入 `state:agent-failed` 并停止。现有测试使用模拟接口，不能代替接入真实 Jev 和执行 Agent 后的验证。
