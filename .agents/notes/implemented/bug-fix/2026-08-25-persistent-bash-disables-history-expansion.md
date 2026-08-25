# Agent Note: Persistent Bash disables history expansion before command evaluation

Status: implemented

English | [中文](2026-08-25-persistent-bash-disables-history-expansion.zh.md)

## Problem

The persistent Bash tool evaluates each model command inside a one-line marker wrapper. Interactive Bash history expansion remained enabled, so command text containing a bare `!` could fail during the wrapper's `eval` parse before the start or end marker ran. The tool then waited for a marker that could never arrive until its wall-clock timeout closed the shell.

## Decision

Persistent Bash initialization runs `stty -echo; set +H`. Echo remains disabled while the backend-owned prompt stays available for readiness detection, and Bash history expansion is disabled before any model command is evaluated. The command wrapper and its marker protocol remain unchanged.

## Alternatives considered

**Escape exclamation marks in every command before `eval`.** Rejected because the tool would need to transform shell source without changing quoted strings, comments, heredocs, or other Bash syntax. That parser would duplicate Bash rules and could alter model commands.

**Remove `eval` and send the command as a separate script process.** Rejected because the persistent shell owns cwd, environment, functions, background jobs, and shell state across calls; a child process would not preserve those semantics.

**Disable history expansion only around each command.** Rejected because the wrapper itself is parsed before the command body runs. The setting must be active during initialization, before the first wrapped command arrives.

## Consequences

Commands containing exclamation marks are evaluated literally and no longer lose their completion marker to history expansion. Interactive history expansion is unavailable in the persistent shell, which is intentional because this shell is a command execution surface rather than a human interactive terminal. The existing marker wrapper and prompt-based readiness behavior remain compatible.

## Testing

The real loader composition test executes a command containing `# !ok` after initialization and asserts the command result. It also retains the ordinary multiline and heredoc cases. A manual PTY reproduction showed the pre-fix wrapper losing its marker with `bash: !ok: event not found`; the same command completed with marker status `0` after `set +H`.
