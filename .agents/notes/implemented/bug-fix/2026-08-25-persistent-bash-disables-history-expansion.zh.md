# Agent Note: 持久 Bash 在执行命令前关闭 history expansion

Status: implemented

[English](2026-08-25-persistent-bash-disables-history-expansion.md) | 中文

## 问题

持久 Bash 工具会把每条模型命令放入单行 marker wrapper，再执行 `eval`。交互式 Bash 仍开启 history expansion 时，命令文本中的裸 `!` 可能在 wrapper 的 `eval` 解析阶段失败，导致开始或结束 marker 都没有输出。工具随后会一直等待不可能到达的 marker，直到墙钟超时并关闭 shell。

## 决策

持久 Bash 初始化现在执行 `stty -echo; set +H`。回显仍保持关闭，后端拥有的提示符仍可用于就绪检测；同时在任何模型命令执行前关闭 Bash history expansion。命令 wrapper 和 marker 协议保持不变。

## 考虑过的替代方案

**在每条命令传给 `eval` 前转义感叹号。** 不采纳：工具需要在不改变引号、注释、heredoc 和其他 Bash 语法的情况下转换 shell 源码，这会重复 Bash 解析规则，并可能改变模型命令。

**移除 `eval`，改为启动独立脚本进程执行命令。** 不采纳：持久 shell 负责跨调用保留 cwd、环境变量、函数、后台任务和 shell 状态；子进程无法保留这些语义。

**只在每条命令执行期间关闭 history expansion。** 不采纳：wrapper 在命令体运行前就已经被解析，设置必须在初始化阶段生效，早于第一条 wrapper 命令。

## 后果

包含感叹号的命令会按字面执行，不再因 history expansion 丢失完成 marker。持久 shell 不再提供交互式 history expansion；这是有意的，因为该 shell 是命令执行面而不是人工交互终端。既有 marker wrapper 和基于提示符的就绪行为保持兼容。

## 测试

真实 loader 组合测试在初始化后执行包含 `# !ok` 的命令并断言结果，同时保留普通多行命令和 heredoc 用例。手动 PTY 复现证明修复前 wrapper 会得到 `bash: !ok: event not found` 且丢失 marker；执行 `set +H` 后同一命令以 marker 状态 `0` 完成。
