# 实践课 4：Pi Agent 的模型调用与运行流程

- 姓名：阳凌杰
- 小组仓库：[superheror/pi-hunnu-2026](https://github.com/superheror/pi-hunnu-2026)
- 本地实验目录：`D:\study\开源\pi-hunnu-2026`
- 实验所用基础提交：`c3cc91632d0a1950fb122605b190233f2aa0006e`；实验时在本地修改了源码。

## 一、实验目的

按[教师作业要求](https://github.com/hunnuOpenSourceSoftware2026/assignments/blob/main/practice/04-Pi-Agent-run.md)，在实践课 3 的 Pi 项目中增加两条观察日志，重新构建，并比较直接问答与读取文件时的 Agent 循环。实验源码位于小组项目 `pi-hunnu-2026`，Pi 交互界面版本为 v1.5.2。本文只提交报告、截图和测试文本，不包含完整源码或构建依赖。

## 二、源码修改

文件：`packages/agent/src/agent-loop.ts`。

1. 在 `streamAssistantResponse` 中，紧靠 `await streamFunction(...)` 之前插入日志（截图中的第 412 行）：

   ```ts
   console.error("[P4] Agent 即将请求模型");
   const response = await streamFunction(config.model, llmContext, {
   ```

   这表示 Agent 即将把当前上下文交给模型调用函数。

   ![S1：模型请求前的日志及 streamFunction 调用](S1.png)

2. 在 `runLoop` 的 `for (const result of toolResults)` 循环内，紧靠 `currentContext.messages.push(result)` 之后插入日志（截图中的第 285 行）：

   ```ts
   for (const result of toolResults) {
       currentContext.messages.push(result);
       console.error("[P4] 工具结果已加入上下文");
       newMessages.push(result);
   }
   ```

   每条工具结果写入上下文时打印一次。

   ![S2：工具结果写入上下文后的日志](S2.png)

## 三、构建与启动

在小组项目根目录运行 `npm run build`。S3a 记录命令及构建过程；S3b 显示生成 `packages\coding-agent\dist\bundle`（84 个文件，9.4 MiB），命令返回提示符，随后执行 `echo %ERRORLEVEL%` 得到 `0`。截图使用的是 Windows CMD，故以 `%ERRORLEVEL%` 检查退出码。构建期间有 npm 配置警告，但最终构建成功。

![S3a：运行 npm run build](S3a.png)

![S3b：coding-agent 构建完成，退出码为 0](S3b.png)

随后启动编译后的 Pi，v1.5.2 成功进入交互界面。

![S4：Pi v1.5.2 交互界面](S4.png)

此前一次在线构建因连接 `models.dev` 超时失败；重新执行 `npm run build` 后成功。这里以最终成功构建的 S3a、S3b 为准。

## 四、两次会话的实际观察

| 项目 | A：直接问答 | B：读取文件 |
| --- | --- | --- |
| 输入 | “请用三句话解释 Git 分支是什么，以及它在多人协作中的作用。不要读取文件，也不要调用任何工具。” | “请用 read 工具读取当前目录的 practice4.txt，并告诉我文件中的实验代号。” |
| 工具 | 未观察到工具调用 | 调用 `read` 读取小组项目目录中的 `practice4.txt` |
| 截图可见的 `[P4]` 日志 | `Agent 即将请求模型` 一条 | `工具结果已加入上下文` → `Agent 即将请求模型` |
| 最终回复 | 给出 Git 分支及多人协作作用的三句话解释 | 返回 `P4-READ-2026` |
| 可确认的模型调用轮次 | 至少 1 轮；画面中只见 1 条请求日志 | 至少 1 轮后续请求；结合工具调用机制，读取前还需要一次模型请求，但该日志未进入截图画面 |

会话 A 中，模型直接回答，没有显示 `read` 或其他工具记录，也没有显示“工具结果已加入上下文”。

![S5：直接问答、最终回复与模型请求日志](S5.png)

会话 B 中，画面显示 `read` 记录、正确代号，以及“工具结果已加入上下文”后紧接“Agent 即将请求模型”。首次模型请求日志未出现在这张终端截图里；结合代码插入位置和工具调用记录，可还原“请求模型 → 本地读取文件 → 工具结果加入上下文 → 再次请求模型”的流程。交互终端可能交错刷新标准错误输出与界面内容，截图中的显示位置不等于严格执行时序。

![S6：read 工具读取文件、返回代号及后续两条日志](S6.png)

`practice4.txt` 的实验代号只存在于文件中，用户指令没有提前透露；所提交的[测试文本](practice4.txt)与截图中的回复一致。

## 五、运行流程分析

```text
用户输入
   │
   ▼
coding-agent（交互界面与 read 工具）
   │ 调用
   ▼
agent（循环、上下文、工具结果写入）
   │ 调用 streamFunction
   ▼
ai（模型请求接口） ───────► 远程模型服务
   ▲                            │
   └──────── 模型回复 / 工具调用 ─┘

B 中：模型提出 read 调用 → coding-agent 在本地读取文件
      → agent 将工具结果加入上下文 → ai 再请求远程模型生成最终答案
```

实际读文件的是本地 `read` 工具，由 Agent 调度；远程模型服务并不直接访问本机文件。工具完成后，`agent` 将工具结果消息写入 `currentContext.messages`。下一轮模型请求因此携带新增的文件读取结果，模型才能从中提取 `P4-READ-2026` 并回答。A 没有观察到工具结果，故无需该次工具结果回填与后续请求。

