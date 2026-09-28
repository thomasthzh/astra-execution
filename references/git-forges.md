# GitHub 与自部署 GitLab 适配

仅在读取/更新远端任务、创建审查请求或跟踪 CI 时读取。提供执行协议，不要求更换平台、安装新服务或把内部资料搬到 GitHub。

## 先确定实例与项目

从用户指定项目、当前仓库配置及已有连接解析事实。若列出 remote，先避免输出 URL 中可能嵌入的凭据。记录：

```text
provider: github | gitlab | local
web_base_url: 实际实例 URL（保留端口和部署子路径）
api_base_url: 经过核实的 API 根地址
project_path: group/subgroup/project 或 owner/repo
project_id: GitLab 返回的数值项目 ID
source_project / target_project: fork 场景分别记录
target_branch / base_sha / credential_profile: 真实值；不写密钥
capabilities: 版本（可取得时）、合并策略、必需检查与可用审批/队列
```

`git@host:group/repo.git`、`ssh://git@host:port/group/repo.git` 和 HTTPS remote 都可能存在。SSH host/端口不一定是 Web/API host/端口；复用现有实例映射或核实后再访问。自部署实例可能有反向代理子路径，不机械拼接 host 根目录。移除 clone URL 的 `.git` 后仍须核对项目，别仅凭 host 名含 gitlab 就确认平台。

没有 GitLab 连接不等于没有访问路径：优先已有专用工具，其次已认证 `glab`，再用允许的本地 HTTP 客户端与 REST API。宿主只暴露 GitHub 工具时不能拿它操作 GitLab。没有远端访问就先做本地调查/实现/验证，具体缺口只在远端步骤必要时提出。

## 身份与访问

复用现有 credential helper、glab host 登录或连接器；`glab auth status --hostname <已确认主机>` 可检查所选主机状态，不使用显示 token 的选项。API 身份与 Git SSH/HTTPS 推送身份分别核对，能读 API 不等于能 push/merge。

不把 token 写进任务包、文件、URL、命令参数或日志；也不让用户在聊天贴密钥。确需登录时使用宿主安全认证流程。环境 token 可能覆盖保存的登录，跨实例时核对凭据来源与 host，不能把 A 实例凭据发给 B。CI job token 的端点权限有限，不把它当作通用访问令牌。按操作选择已有授权的最小权限，不默认要求管理员 token。

内网/VPN、代理或 CA 失败属于访问问题：核对当前环境和已配置的企业信任链；不能自动用 `-k`、跳过 TLS 校验或改全局 Git/CLI 配置来掩盖失败。读取本地信任配置与诊断可以继续，修改信任/认证配置依实际请求。公网搜索只查通用官方文档，不带内部源码、Issue 内容或敏感实例信息。

## GitLab 定向与 ID

REST 通常为 `<实例 base>/api/v4`，以实际部署为准。首次按 URL 编码后的完整项目路径取项目，再用数值 ID，减少转义歧义。`group/subgroup/repo` 作为一个路径参数编码为 `group%2Fsubgroup%2Frepo`；不要双重编码。Issue/MR 用项目内 `iid`，Pipeline 用 `id`，不可混用。

以下是只读命令形状，主机/项目/ID 均为示例，执行前替换为已验证事实并检查本机 `glab ... --help`：

```text
glab api --hostname gitlab.example.internal --method GET "projects/group%2Fsubgroup%2Frepo"
glab api --hostname gitlab.example.internal --method GET "projects/123/issues?state=opened&per_page=100" --paginate
glab mr list -R "https://gitlab.example.internal/group/subgroup/repo"
glab api --hostname gitlab.example.internal --method GET "projects/123/merge_requests/7"
glab api --hostname gitlab.example.internal --method GET "projects/123/merge_requests/7/pipelines" --paginate
glab api --hostname gitlab.example.internal --method GET "projects/123/pipelines/456"
```

`glab api` 用 `--hostname`；`mr`/`issue` 等项目命令使用它们支持的 `-R/--repo`。不要假设所有子命令都有相同定向参数。`api` 添加 field 参数可能改变默认 HTTP 方法，因此读操作显式 GET；多行写入优先结构化请求体或文件，避免 shell 插值。

标准 `--hostname` 路径不能表达当前特殊部署时，使用已核实 API base 的客户端，不静默退回 gitlab.com。CLI 字段/参数随版本不同，工具未支持时用该实例支持的 REST 契约，不反复猜命令。

## 任务到交付的映射

| 工作 | GitHub | GitLab（含自部署） |
| --- | --- | --- |
| 队列 | Issues / 已有项目板 | Project Issues / 已有看板；只读取被授权范围 |
| 身份 | host + repository + number | instance + project_id + issue/MR iid |
| 代码审查 | Pull Request | Merge Request，保存 `web_url`、源/目标项目与分支 |
| 验证 | Checks / Actions | Pipeline、Jobs、必要时 child/downstream Pipeline |
| 合并条件 | 保护规则、审批、merge queue | 保护分支、讨论/审批、pipeline、项目合并条件，支持时 merge train |
| 发版 | 项目发布流程 | 项目 Release / CI Environment / Deployment 流程 |

GitHub 复用已连接工具或已配置 CLI，保留当前 host 与仓库，按宿主要求关联创建的 PR。GitLab MR 不通过 GitHub 专用接口附件化；工具未声明支持时直接返回完整 MR 链接。

GitLab 常用 REST 资源：项目 `projects/:id`；Issue `projects/:id/issues/:issue_iid`；MR `projects/:id/merge_requests/:iid`；diffs、discussions、pipelines 为相应子资源；Pipeline `projects/:id/pipelines/:pipeline_id`。按实际需要读取，不每轮全量拉取。

分页使用客户端能力或响应提供的下一页/Link；不把第一页当完整 backlog，也不盲跟到未确认的其他主机。只取本次授权任务范围。列表为空、401、403、404 要区分：404 可能是权限/编码/目标错误，不能直接认定“没有项目”。429 按返回等待建议有界退避，其他错误先分类。

## MR 与 Pipeline 的正确验收

合并前重新读取 MR 的当前源 head、目标分支、draft、冲突、`detailed_merge_status`、未解决讨论、实际要求的审批和检查。异步字段为空或处于 checking 时等待有界更新，不能按通过处理。

- **源分支 / 普通 MR Pipeline：** 检查关联的当前源版本及必要 jobs；旧提交成功不覆盖新提交。
- **Merged-results Pipeline / Merge train：** SHA 是合成提交。核对与当前 MR、源提交、目标上下文或队列位置的关联，不能要求其等于源 head，也不能因“最新且绿色”就接受旧合成结果。关联证据不足时先查证。
- failed、canceled、pending、manual、skipped 各有意义。按项目实际必需 job 和阻塞 manual job 判断；不自动点部署 job，也不把 skipped 当成功。外部/child/downstream 检查在项目要求时一并核对。
- 版本、套餐、项目设置影响 required approvals、merged-results、merge trains、auto-merge 可用性。以实例实际能力为准；无 train 时使用单一集成者与正常保护分支检查，不绕过规则。Free/旧实例不假装有企业级功能。`/approval_state` 规则端点需相应套餐；无审批规则时 `approved=true` 不证明有人审核，403/404 也不证明无需审批。
- 已获合并授权且条件满足时直接执行。REST 合并支持时携带刚核对的源 `sha`，防止检查后新 push 的竞态。409/版本变化要刷新和重新验收，不能改成无 SHA 强行合并。
- `auto_merge` 与旧 `merge_when_pipeline_succeeds` 存在版本差异，按实例和客户端支持选用；预约合并只代表已排队，须读到实际 merged 才算完成。未获合并授权不设置 auto-merge。

## 重试与发布

开 MR 前按源/目标项目和分支查询已有 MR；请求超时先查返回结果再决定是否重试。外部 Issue/Release/部署也使用已有 ID 与真实查询去重；不把所有 REST 写入视作天然幂等。

用户仅要求“修复并开 MR”时，到真实 MR 与要求的检查结果为止；要求“合并并发布”时，继续核对合并、部署及必要冒烟证据。创建 MR、启动 Pipeline、排入 train、发布成功分别入账。

未提供实例或访问能力时，可以报告“协议支持、尚未联调”；不能报告已接入/已测试用户 GitLab。本 skill 不默认安装 glab、不更改服务端配置、不创建 CI runner。

## 官方依据（核查于 2026-09-29）

- [glab api](https://docs.gitlab.com/cli/api/) / [mr list](https://docs.gitlab.com/cli/mr/list/) / [CLI 配置](https://docs.gitlab.com/cli/configuration/)
- [REST API、编码与分页](https://docs.gitlab.com/api/rest/) / [Projects](https://docs.gitlab.com/api/projects/) / [Issues](https://docs.gitlab.com/api/issues/)
- [MR API](https://docs.gitlab.com/api/merge_requests/) / [Pipelines](https://docs.gitlab.com/api/pipelines/)
- [Merged-results](https://docs.gitlab.com/ci/pipelines/merged_results_pipelines/) / [Merge trains](https://docs.gitlab.com/ci/pipelines/merge_trains/) / [Auto-merge](https://docs.gitlab.com/user/project/merge_requests/auto_merge/)
- [MR 审批](https://docs.gitlab.com/user/project/merge_requests/approvals/) / [CLI 认证状态](https://docs.gitlab.com/cli/auth/status/)
- [Approvals API](https://docs.gitlab.com/api/merge_request_approvals/) / [CLI 认证优先级](https://docs.gitlab.com/cli/authentication/) / [相对 URL 部署](https://docs.gitlab.com/install/relative_url/)
