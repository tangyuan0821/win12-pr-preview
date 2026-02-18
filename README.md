# win12-pr-preview

面向上游仓库 https://github.com/tjy-gitnub/win12 的全自动 PR 预览发布方案，全部逻辑运行在本仓库，零改动上游仓库配置。

## 1）前置准备（账户、权限、Secrets、Pages）
1. **大号PAT（只读拉代码）**
   - 路径：GitHub Settings → Developer settings → Personal access tokens → Fine-grained token → Generate new token
   - 选择资源库：限定为 `tjy-gitnub/win12`
   - 权限最小化：`Contents: Read-only`、`Metadata: Read-only`、`Pull requests: Read-only`
2. **小号PAT（仅PR评论）**
   - 路径同上，Fine-grained token
   - 资源库：`tjy-gitnub/win12`
   - 权限最小化：`Pull requests: Read & write`（若界面需要 Issues 才能发评论，可勾选 `Issues: Read & write`）
3. **在本仓库新增 Secrets**
   - `UPSTREAM_READ_PAT`：大号PAT
   - `PREVIEW_COMMENT_PAT`：小号PAT
   - `PREVIEW_COMMENT_USERNAME`：小号 GitHub 用户名，用于定位并更新评论
4. **开启 GitHub Pages（本仓库 Settings → Pages）**
   - Source 选择 **Deploy from a branch**
   - Branch 选择 `gh-pages`（Workflow 会自动创建/更新）
5. **可选加速触发（无需改动上游）**
   - 可在任意机器/Actions 监听 GitHub Events 接口，一旦捕获上游 PR `opened/synchronize/reopened` 事件，通过 `repository_dispatch` 触发本 Workflow。**此调用需要对本仓库拥有 `contents: write` 的令牌**（可单独创建 `DISPATCH_PAT`，或在本仓库的 Actions 中直接使用 `${{ secrets.GITHUB_TOKEN }}`）：  
   ```
   gh api repos/{owner}/win12-pr-preview/dispatches \
     -f event_type=upstream-pr \
     -f client_payload='{"number":123}' \
     -H "Authorization: token $DISPATCH_PAT"
   ```
   - 若不接入外部监听，Workflow 也会每 15 分钟轮询自动处理最新的打开状态非草稿 PR。
6. **时间配置**
   - 评论时间默认使用 `Asia/Shanghai`（UTC+8，以满足需求），如需覆盖可在 Workflow `env.PREVIEW_TIMEZONE` 中调整。

## 2）可直接复制的 GitHub Actions Workflow（中文注释）
文件路径：`.github/workflows/upstream-pr-preview.yml`

```yaml
name: upstream-pr-preview

on:
  # 手动触发，便于指定单个PR验证
  workflow_dispatch:
    inputs:
      pr_number:
        description: '仅处理指定的上游PR编号（可选）'
        required: false
  # 供外部调用 repository_dispatch（例如自建监听服务转发上游PR事件）
  repository_dispatch:
    types:
      - upstream-pr
  # 定时轮询，保证无需改动上游仓库也能自动发现最新PR
  schedule:
    - cron: "*/15 * * * *"

permissions:
  contents: write

concurrency:
  group: upstream-pr-preview
  cancel-in-progress: true

env:
  UPSTREAM_REPO: tjy-gitnub/win12
  PREVIEW_ROOT: site
  PREVIEW_TIMEZONE: Asia/Shanghai

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      prs: ${{ steps.collect.outputs.prs }}
    steps:
      - name: 收集需要处理的PR列表（大号，只读）
        id: collect
        env:
          GH_TOKEN: ${{ secrets.UPSTREAM_READ_PAT }}
          INPUT_PR: ${{ github.event.inputs.pr_number || '' }}
          PAYLOAD_PR: ${{ github.event.client_payload.pull_request.number || github.event.client_payload.number || '' }}
        run: |
          set -euo pipefail
          if [ -n "$PAYLOAD_PR" ]; then
            PRS="[$PAYLOAD_PR]"
          elif [ -n "$INPUT_PR" ]; then
            PRS="[$INPUT_PR]"
          else
            PRS=$(gh api "repos/${UPSTREAM_REPO}/pulls" --paginate --jq '[.[] | select(.state=="open" and .draft==false) | .number]')
          fi
          if [ -z "$PRS" ]; then
            PRS="[]"
          fi
          echo "prs=$PRS" >> "$GITHUB_OUTPUT"
          echo "Target PRs: $PRS"

  deploy:
    runs-on: ubuntu-latest
    needs: prepare
    permissions:
      contents: write
    env:
      UPSTREAM_REPO: tjy-gitnub/win12
      PR_LIST: ${{ needs.prepare.outputs.prs }}
    steps:
      - name: 检出当前仓库（默认token，仅用于部署）
        uses: actions/checkout@v4

      - name: 拉取并构建预览（大号，只读）
        id: build
        env:
          GH_TOKEN: ${{ secrets.UPSTREAM_READ_PAT }}
          PREVIEW_ROOT: site
        run: |
          set -euo pipefail
          mkdir -p "$PREVIEW_ROOT"
          echo "$PR_LIST" | jq -e . > /tmp/prs.json
          : > /tmp/comment_info.jsonl
          : > /tmp/validate.list

          PR_COUNT=$(jq 'length' /tmp/prs.json)
          if [ "$PR_COUNT" -eq 0 ]; then
            echo "No open PRs found; will clean gh-pages."
            echo "暂无进行中的PR / No active PR previews" > "$PREVIEW_ROOT/README.md"
          fi

          for pr in $(jq -r '.[]' /tmp/prs.json); do
            if ! [[ "$pr" =~ ^[0-9]+$ ]]; then
              echo "Invalid PR number ${pr}, skipping (PR编号格式异常，已跳过)"
              continue
            fi
            data=$(gh api "repos/${UPSTREAM_REPO}/pulls/${pr}")
            state=$(echo "$data" | jq -r '.state')
            draft=$(echo "$data" | jq -r '.draft')
            if [ "$state" != "open" ] || [ "$draft" = "true" ]; then
              echo "Skip #$pr (state=$state draft=$draft)"
              continue
            fi

            sha=$(echo "$data" | jq -r '.head.sha')
            head_repo=$(echo "$data" | jq -r '.head.repo.full_name')

            if [ -z "$head_repo" ] || [ "$head_repo" = "null" ]; then
              echo "Head repo missing for PR #$pr, skipping (源仓库缺失，已跳过)"
              continue
            fi

            if ! printf '%s' "$head_repo" | grep -Eq '^[A-Za-z0-9._-]+/[A-Za-z0-9._-]+$'; then
              echo "Unexpected head repo format for PR #$pr, skipping (源仓库格式异常，已跳过)"
              continue
            fi

            archive="pr-${pr}.zip"
            curl -L --fail --max-filesize 104857600 \
              -H "Authorization: Bearer $GH_TOKEN" \
              -o "$archive" \
              "https://api.github.com/repos/${head_repo}/zipball/${sha}"
            unzip -q "$archive" -d "/tmp/src-$pr"
            SRC_DIRS=$(find "/tmp/src-$pr" -maxdepth 1 -type d -not -path "/tmp/src-$pr" || true)
            SRC_DIR=$(printf "%s\n" "$SRC_DIRS" | head -n 1)
            if [ -z "$SRC_DIR" ]; then
              echo "Unable to locate extracted source directory for PR #$pr, skipping (解压目录缺失，已跳过)"
              continue
            fi
            SRC_COUNT=$(printf "%s\n" "$SRC_DIRS" | grep -c . || true)
            if [ "${SRC_COUNT:-0}" -ne 1 ]; then
              echo "Unexpected archive layout for PR #$pr, skipping (解压目录数量异常，已跳过)"
              continue
            fi
            rsync -a --delete "${SRC_DIR}/" "$PREVIEW_ROOT/pr-${pr}/"
            echo "$pr $sha" >> /tmp/validate.list
          done

          if [ -s /tmp/validate.list ]; then
            pushd "$PREVIEW_ROOT" >/dev/null
            python3 -m http.server 8000 >"/tmp/http-server.log" 2>&1 &
            SERVER_PID=$!
            trap "kill $SERVER_PID 2>/dev/null" EXIT
            sleep 2
            while read -r pr sha; do
              if curl -fsS "http://localhost:8000/pr-${pr}/desktop.html" >/dev/null; then
                printf '{"number":%s,"sha":"%s"}\n' "$pr" "$sha" >> /tmp/comment_info.jsonl
              else
                echo "desktop.html validation failed for PR #$pr, skipping comment and deployment (desktop.html 校验失败，已跳过)" >&2
                tail -n 50 "/tmp/http-server.log" || true
                rm -rf "${PREVIEW_ROOT}/pr-${pr}"
              fi
            done < /tmp/validate.list
            trap - EXIT
            kill "$SERVER_PID" 2>/dev/null || true
            popd >/dev/null
          fi

      - name: 部署到 GitHub Pages（默认token）
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: site
          publish_branch: gh-pages
          commit_message: "chore: update PR previews"

      - name: 发布/更新预览评论（小号，仅PR写权限）
        if: success()
        env:
          GH_TOKEN: ${{ secrets.PREVIEW_COMMENT_PAT }}
          BOT_USERNAME: ${{ secrets.PREVIEW_COMMENT_USERNAME }}
          UPSTREAM_REPO: tjy-gitnub/win12
          PREVIEW_ROOT_URL: https://${{ github.repository_owner }}.github.io/${{ github.event.repository.name }}
        run: |
          set -euo pipefail
          MARKER="🤖 PR预览自动化机器人"
          if [ ! -s /tmp/comment_info.jsonl ]; then
            echo "No successful builds; skip commenting."
            exit 0
          fi

          while IFS= read -r line; do
            pr=$(echo "$line" | jq -r '.number')
            sha=$(echo "$line" | jq -r '.sha')
            url="${PREVIEW_ROOT_URL}/pr-${pr}/desktop.html"
            deployed_at=$(env TZ="${PREVIEW_TIMEZONE}" date "+%Y-%m-%d %H:%M:%S %Z")

            body=$(cat <<EOF
${MARKER}

- 预览链接：${url}
- 提交哈希：${sha}
- 部署时间（${PREVIEW_TIMEZONE}）：${deployed_at}
- 原项目：https://github.com/${UPSTREAM_REPO} （EPL-2.0）

> 若PR继续提交，新预览将自动覆盖并更新本评论。
EOF
)

            existing=$(gh api "repos/${UPSTREAM_REPO}/issues/${pr}/comments" --jq 'map(select(.user.login==$bot and (.body | contains($marker))))[0].id' --arg bot "$BOT_USERNAME" --arg marker "$MARKER" || true)
            if [ -n "$existing" ] && [ "$existing" != "null" ]; then
              gh api "repos/${UPSTREAM_REPO}/issues/comments/${existing}" -X PATCH -f body="$body"
            else
              gh api "repos/${UPSTREAM_REPO}/issues/${pr}/comments" -X POST -f body="$body"
            fi
          done < /tmp/comment_info.jsonl
```

## 3）0-1 上线操作步骤
1. Fork 或创建个人仓库（即本仓库）并拉取到本地/在线编辑。
2. 按“前置准备”生成大号、小号 PAT，并写入本仓库 Secrets：`UPSTREAM_READ_PAT`、`PREVIEW_COMMENT_PAT`、`PREVIEW_COMMENT_USERNAME`。
3. 打开 Settings → Pages，选择 `gh-pages` 分支启用 Pages。
4. 确认 `.github/workflows/upstream-pr-preview.yml` 已存在（本仓库已提供）。
5. 首次可通过 **Actions → upstream-pr-preview → Run workflow** 手动触发；如需单个PR，输入 PR 号。
6. 若要即时响应上游 PR 事件，在外部监听后调用 `repository_dispatch`（示例命令见上）。
7. Workflow 会：过滤草稿/关闭PR → 拉取PR源分支 → 构建 `pr-{PR号}` 目录 → 本地校验 `desktop.html` → 推送至 `gh-pages`。
8. 部署成功后，小号会在对应上游 PR 发布/更新单条评论，包含预览直链、SHA、时间（默认 `PREVIEW_TIMEZONE=Asia/Shanghai` 即 UTC+8，可按需覆盖）、项目地址与许可声明。
9. Pages 访问规则：`https://{owner}.github.io/win12-pr-preview/pr-<PR号>/desktop.html`。
10. PR 合并或关闭后，定时/手动运行会重建 `gh-pages`，未列出的目录被移除，实现自动清理。

## 4）常见问题排查
- **拉取代码失败**：确认 `UPSTREAM_READ_PAT` 权限为 Contents/Pull requests 只读；若 PR 来自私有 fork，则需开放可见性或临时提权。
- **部署失败或校验未通过**：查看 Actions 日志中的 `curl -fsS ...desktop.html` 输出；确保 PR 中仍包含 `desktop.html`，并能通过本地静态服务访问。
- **评论未发送**：确认 `PREVIEW_COMMENT_PAT` 具备 PR 写权限，`PREVIEW_COMMENT_USERNAME` 与小号用户名一致；部署失败时设计为不留言。
- **重复评论/刷屏**：Workflow 会查找小号且包含“PR预览自动化机器人”的评论并进行 PATCH 更新，不会新建多条。
- **Pages 未生效**：确保 Settings → Pages 已指向 `gh-pages`，并等待几分钟 CDN 生效；若仓库为私有需确认 Pages 允许公开访问。

## 5）开源合规注意事项（EPL-2.0）
- 保留并同步上游仓库的作者署名和 LICENSE 文件（Workflow 使用 rsync 全量保留）。
- 预览站点仅作展示，不修改上游授权条款；评论中附上原项目地址与 EPL-2.0 说明。
- 部署目录隔离（`pr-{PR号}`），不会覆盖或篡改上游代码；仅在个人仓库的 Pages 空间输出静态预览。
- 不向上游提交任何配置变更，遵循“只读拉取、独立部署、最小权限”原则。
