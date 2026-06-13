---
name: commit-by-feature
description: Commits code changes by feature with a standardized multi-line commit message. Runs when the user makes code changes and asks to commit, or when the user explicitly requests a commit. Amends the last unpushed commit message when the user switches to another feature and has unpushed commits.
---

# 按功能提交（Commit by Feature）

在用户完成代码修改并要求提交、或用户明确要求提交时，按「功能」进行 git commit，并采用下方固定格式的 commit message。若用户在一个功能已 commit 后又在做另一功能的修改，且存在未 push 的 commit，则对**最近一次未 push 的 commit** 使用 `git commit --amend` 修改其 message，使其描述新功能。

## 何时执行

- 用户说「帮我 commit」「提交一下」「把更改 commit」等，或
- 用户完成一批代码修改后要求提交，或
- 用户说「修改刚才的 commit 说明」「改一下上次的 commit message」等（则执行 amend 流程）。

## Commit Message 格式（必须严格遵循）

整段作为 **commit message 正文**（不要用 `-m` 只写一行，用 `-m "第一行" -m "正文..."` 或写进文件再 `git commit -F`）。格式如下：

```
修改描述： [feat] 或 [fix] 或 [refactor] + 一句话概括（如：精简视频裁剪页面与回灌页面：在视频裁剪页移除xxx；在回灌页将设备 IP 默认改为 0.0.0.0。）
影响分析： all
是否自测： 是
自测内容： ui（或 doc / 接口 / 改动对应的文件夹路径 /等，按实际改动的自测范围填写）
测试建议： all
代码审查： 刘江涛
NOTE： all
项目编号： all
```

- **修改描述**：首行用 `[feat]` / `[fix]` / `[refactor]` 等前缀，后接简洁的功能/修复说明，多块功能用分号或「；」分隔。
- **影响分析 / 测试建议 / NOTE / 项目编号**：无特殊要求时填 `all`。
- **是否自测**：一般为「是」。
- **自测内容**：根据改动填 `ui`、`doc`、`接口`、`导出` 等。
- **代码审查**：固定为「刘江涛」。

## 执行步骤

### 1. 仅修改上一次 commit 的 message（amend）

当用户明确说「改 commit 说明」「修改上次的 commit」「amend」且当前有未 push 的 commit 时：

1. 运行 `git log -1 --oneline` 确认最近一次 commit。
2. 根据**当前工作区+暂存区的最新改动**（或用户口述）归纳「修改描述」。
3. 按上面格式写出完整 commit message。
4. 执行：`git commit --amend -m "修改描述： ..." -m "影响分析： all" -m "是否自测： 是" -m "自测内容： ..." -m "测试建议： all" -m "代码审查： 刘江涛" -m "NOTE： all" -m "项目编号： all"`
   或使用临时文件：`git commit --amend -F commit_msg.txt`（内容为上述多行）。
5. 告知用户已 amend 完成。

### 2. 按功能提交当前改动（新 commit）

当用户要求提交当前更改时：

1. **查看改动**：`git status`、`git diff`（及 `git diff --staged`）。根据修改的文件路径和 diff 内容判断属于哪一个或哪几个功能（例如：视频裁剪、回灌、帧头导出、日志分析、通用功能、系统设置等）。
2. **决定 commit 数量**：
   - 通常**一个功能一个 commit**；若改动明显属于同一功能，只生成一个 commit。
   - 若改动横跨多个互不相关的功能，可拆成多个 commit（先 add 第一批，commit；再 add 下一批，commit），每个 commit 的 message 都按本格式填写。
3. **撰写「修改描述」**：
   - 用 `[feat]` / `[fix]` / `[refactor]` 等开头。
   - 一句话概括该 commit 涉及的功能或修复，多点时用分号分隔。
   - 仅描述本 commit 包含的改动，不要写未改动的功能。
4. **填写其余字段**：影响分析、是否自测、自测内容、测试建议、代码审查、NOTE、项目编号按格式填好。
5. **执行提交**：
   - 先 `git add` 本功能相关文件，再 `git commit`，message 使用上述多行格式（可用多个 `-m` 或 `-F` 文件）。
   - 若拆多个 commit，重复 add + commit，每个 commit 对应一个功能的描述。

## 示例

**单功能单 commit：**

```
修改描述： [feat] 帧头导出支持 MP4 视频并修复导出 0 帧问题；导出服务 JSON 改为使用 orjson。
影响分析： all
是否自测： 是
自测内容： 导出、MP4 视频
测试建议： all
代码审查： 刘江涛
NOTE： all
项目编号： all
```

**Amend 场景：**
用户已为「视频裁剪精简」commit 过一次，未 push，接着又做了「回灌页设备 IP 默认 0.0.0.0」的修改并希望把这两块合进同一个 commit 说明。则对最近一次 commit 执行 `git commit --amend`，将 message 改为同时包含「视频裁剪页精简」和「回灌页设备 IP 默认与选网卡自动填充」的修改描述。

## 注意事项

- 不要用单行 `-m "..."` 把多行格式挤成一行；用多行 message 或 `-F` 文件。
- 若仓库有 `.gitignore` 或未跟踪文件，只 add 用户明确要提交的或与当前功能明显相关的文件。
- 不执行 `git push`，除非用户明确要求推送。