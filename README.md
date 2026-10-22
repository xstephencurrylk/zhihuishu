# 智慧树学习辅助脚本（Tampermonkey）

一个面向智慧树 MOOC 学习页的 Tampermonkey 用户脚本，提供视频自动续播、随堂练习检测、DeepSeek 答案提示，以及默认关闭的测试模式自动作答功能。

> 本项目不是智慧树官方项目。请仅在课程允许、教师授权或个人学习场景中使用；自动提交模式仅限授权的非计分测试账号和模拟题库。

## 功能概览

- **视频自动播放**：检测课程页面中的主视频，暂停后尝试继续播放。
- **自动切换下一节**：视频结束后，通过播放器内置的 `PlayerStarter` 回调切换下一节，不模拟点击课程目录。
- **随堂练习检测**：检测 `.video-config-dialog` 中的“AI随堂练习”入口并自动打开。
- **题目解析**：读取题目、题型和选项，支持当前页面中的单选、多选和判断题结构。
- **DeepSeek 答案建议**：将题目发送到 DeepSeek API，显示建议选项、解析和置信度。
- **测试模式自动提交**：显式启用后，自动选择建议选项并点击“提交作答”。
- **提交前校验**：逐题校验实际选中项是否与建议答案一致，不一致时停止提交。
- **本地配置**：API Key、接口地址和模型保存在 Tampermonkey 本地存储中。
- **异常点击规避**：v1.1.1 起不再对课程目录调用 `element.click()`，避免触发智慧树的 `isTrusted` 异常检测。

## 适用页面

当前脚本匹配：

```text
https://wisdom-mooc.zhihuishu.com/study/*
```

主要适配当前 `wisdom-mooc.zhihuishu.com` 学习页的以下结构：

- 播放器容器：`container`
- 播放器回调：`PlayerStarter.get('container').callback.playerNext`
- 课程目录：`.course-catalog .chapter-item`、`.chapter-content-second`
- 练习入口：`.video-config-dialog .dialog-item`
- 题目弹窗：`.ai-class-exercise-dialog`
- 题目和选项：`.item`、`.option`、`.class-question-select`、`.answer`
- 提交按钮：`.el-dialog__footer button`

## 兼容性说明

脚本**不能保证适配智慧树的所有课程**。

可能无法直接使用的情况包括：

- 旧版智慧树页面或非 `wisdom-mooc.zhihuishu.com` 页面
- 课程视频位于跨域 `iframe` 中
- 播放器不是当前 `ableVideoPlayerAiv2` 版本
- 播放器实例 ID 不是 `container`
- 填空题、问答题、连线题、拖拽题等非选择题型
- 图片题或需要读取图片内容的题目
- 不同课程使用的其他弹窗组件和 CSS 类名
- 智慧树前端更新或平台新增限制、验证码、人脸识别或异常检测

脚本不会绕过登录、权限、验证码、人脸识别、学习时间限制或平台风控。

## 安装

### 1. 安装 Tampermonkey

在浏览器中安装 Tampermonkey 扩展，并确认扩展已启用。

### 2. 添加脚本

下载或打开：

```text
zhihuishu-study-helper.user.js
```

然后选择一种方式：

1. 打开 Tampermonkey 管理面板 → “添加新脚本” → 粘贴脚本内容 → 按 `Ctrl+S` 保存。
2. 将 `.user.js` 文件拖入浏览器窗口，按 Tampermonkey 提示安装。
3. 上传到 GitHub 后，也可以使用仓库中脚本的 Raw 地址安装。

Raw 地址格式：

```text
https://raw.githubusercontent.com/<USER>/<REPO>/main/zhihuishu-study-helper.user.js
```

### 3. 刷新课程页面

打开智慧树学习页并刷新。页面右下角应出现“智慧树学习辅助”面板。

## DeepSeek API 配置

1. 展开面板中的“DeepSeek API 设置”。
2. 填写 API Key。
3. 接口地址默认：

```text
https://api.deepseek.com/chat/completions
```

4. 模型默认：

```text
deepseek-chat
```

5. 点击“保存设置”。

API Key 会通过 `GM_setValue` 保存在 Tampermonkey 本地存储中，不会写入脚本源码。但启用 AI 提示后，课程标题、题目和选项会发送到所配置的 API 地址。

请勿将 API Key 提交到 GitHub，也不要在截图、Issue 或日志中公开 Key。

## 使用说明

面板包含以下开关：

| 开关 | 默认值 | 功能 |
| --- | --- | --- |
| 自动播放 | 开启 | 尝试让当前课程视频持续播放 |
| 自动下一节 | 开启 | 视频结束后通过播放器回调切换下一节 |
| 自动打开题目 | 开启 | 检测“AI随堂练习”入口并打开题目 |
| DeepSeek 提示 | 开启 | 请求 API 并显示建议答案与解析 |
| 测试模式：自动选择并提交 | 关闭 | 自动勾选建议答案并点击提交 |

### 下一节切换机制

v1.1.1 不再调用课程目录节点的 `click()`。当前实现为：

```js
pageWindow.PlayerStarter
  .get('container')
  .callback
  .playerNext();
```

这样可以复用播放器自身的切课逻辑，不会生成不可信的 `PointerEvent`，从而避免智慧树因 `isTrusted === false` 判定异常脚本并跳转到主页。

### 测试模式自动提交

启用“测试模式：自动选择并提交”后，会弹出二次确认。确认后脚本会：

1. 等待 DeepSeek 返回建议答案。
2. 按选项字母自动选择答案。
3. 检查每题实际选中项是否与建议完全一致。
4. 找到并点击“提交作答”。

以下情况会自动停止提交：

- 未配置 API Key
- API 请求失败或返回格式异常
- 某题没有建议答案
- 建议答案包含页面中不存在的选项
- 页面实际选中项与建议答案不一致
- 未找到“提交作答”按钮

> 自动提交功能默认关闭。请仅在你拥有授权的非计分测试账号和模拟题库中使用，并遵守课程及平台规则。

## 页面更新后失效怎么办

当前脚本依赖智慧树页面的 CSS 类和播放器对象，平台改版后可能需要更新选择器。

排查顺序：

1. 确认使用的版本为 `v1.1.1` 或更高。
2. 删除或停用同一页面的旧版本脚本，避免重复执行。
3. 刷新页面后观察右下角状态信息。
4. 若提示“未找到播放器内置的下一节接口”，检查页面是否仍存在 `window.PlayerStarter`。
5. 若题目未识别，检查页面是否仍使用 `.ai-class-exercise-dialog` 和 `.class-question-select`。
6. 若浏览器阻止自动播放，先手动点击一次播放按钮。

## 已知限制

- DeepSeek 的建议答案可能错误，不等于课程标准答案。
- 脚本目前主要识别选择题结构，不处理主观题和图片题。
- 自动播放、自动切换和自动提交都依赖页面当前版本的前端实现。
- 平台更新后可能需要重新适配。
- 本项目不包含验证码识别、风控绕过或账号安全机制。

## 测试情况

开发过程中进行了以下本地模拟测试：

- JavaScript 语法检查通过。
- 模拟 AI 返回多选题 `A,C` 和单选题 `B`，脚本成功选中 `A,C,B` 并触发提交。
- 模拟播放器 `PlayerStarter` 回调，脚本成功调用下一节接口，且未触发课程目录模拟点击。

## 文件结构

```text
.
├── README.md
├── GITHUB_DESCRIPTION.md
├── zhihuishu-study-helper.user.js
└── zhihuishu-study-helper-review-only.user.js  # 可选：仅提示、不自动提交的版本
```

## 更新记录

### v1.1.1

- 修复视频结束后切换到下一节时跳转智慧树主页的问题。
- 改用播放器内置 `playerNext` 回调，不再模拟点击课程目录。
- 增加 `unsafeWindow` 权限以访问页面播放器对象。

### v1.1.0

- 增加默认关闭的测试模式自动选择与提交。
- 增加二次确认和提交前答案校验。

### v1.0.0

- 实现视频自动播放与下一节切换。
- 实现随堂练习检测、题目读取和 DeepSeek 答案提示。
- 实现本地 API Key 配置面板。

## 免责声明

本项目仅用于个人学习、前端兼容性研究和经授权的非计分测试。使用者应自行确认其使用方式符合课程要求、学校规定、教师授权以及智慧树平台规则。作者不对因使用本脚本产生的成绩、账号、数据或平台风险负责。

## License

当前仓库尚未声明开源许可证。公开发布或允许他人分发前，请根据实际需求添加合适的 `LICENSE` 文件。
