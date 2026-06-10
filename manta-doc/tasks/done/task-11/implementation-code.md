# Task-11 구현 코드 가이드

이 문서는 `manta-repo/`에 직접 적용할 코드를 설명한다. 실제 코드 파일은 이 문서 작성 단계에서 수정하지 않는다.

## 핵심 구조

- `packages/cli/src/errors/cli-error-policy.ts`: CLI text error contract의 단일 owner.
- `packages/cli/src/cli.ts`: Commander program 생성과 top-level async parse 경계.
- `packages/cli/src/index.ts`: 실행 진입점. CLI 실행만 위임한다.
- `help`/`init` command: 직접 stderr를 만들지 않고 error policy를 호출한다.

## 파일: `packages/cli/src/errors/cli-error-policy.ts` (신규)

```diff
+import { CommanderError } from 'commander';
+
+// CLI exit code는 shell script와 AI agent가 가장 먼저 보는 실패 신호다.
+// 0: 성공/no-op, 1: 실행 중 실패, 2: 사용자가 명령을 잘못 호출한 경우.
+export const CLI_EXIT_CODES = {
+  success: 0,
+  runtimeFailure: 1,
+  usageError: 2,
+} as const;
+
+// JSON error는 이번 task 범위 밖이지만, text stderr에도 안정 식별자를 둔다.
+// 문구가 바뀌어도 AI와 로그 검색은 이 code로 분기할 수 있다.
+export type CliErrorCode = 'UNKNOWN_COMMAND' | 'USAGE_ERROR' | 'RUNTIME_FAILURE';
+
+export interface CliTextError {
+  code: CliErrorCode;
+  exitCode: number;
+  message: string;
+}
+
+const ANSI_ESCAPE_PATTERN = /\x1B\[[0-?]*[ -/]*[@-~]/g;
+const CONTROL_CHARACTER_PATTERN = /[\x00-\x1F\x7F]/g;
+const MAX_RENDERED_VALUE_LENGTH = 120;
+
+// 사용자 입력이 stderr에 그대로 들어가면 newline/ANSI escape로 로그를 속일 수 있다.
+// 오류 메시지에 echo되는 값은 한 줄짜리 printable string으로 정규화한다.
+export function sanitizeCliErrorValue(value: string): string {
+  const singleLineValue = value
+    .replace(ANSI_ESCAPE_PATTERN, '')
+    .replace(CONTROL_CHARACTER_PATTERN, ' ')
+    .replace(/\s+/g, ' ')
+    .trim();
+
+  if (singleLineValue.length <= MAX_RENDERED_VALUE_LENGTH) {
+    return singleLineValue;
+  }
+
+  return `${singleLineValue.slice(0, MAX_RENDERED_VALUE_LENGTH)}...`;
+}
+
+export function createUnknownCommandError(commandName: string): CliTextError {
+  return {
+    code: 'UNKNOWN_COMMAND',
+    exitCode: CLI_EXIT_CODES.usageError,
+    message: `Unknown command: ${sanitizeCliErrorValue(commandName)}. Run \`manta help\` to see available commands.`,
+  };
+}
+
+export function createUsageError(message: string): CliTextError {
+  return {
+    code: 'USAGE_ERROR',
+    exitCode: CLI_EXIT_CODES.usageError,
+    message: `Usage error: ${sanitizeCliErrorValue(stripCommanderErrorPrefix(message))}`,
+  };
+}
+
+export function createRuntimeFailureError(message: string): CliTextError {
+  return {
+    code: 'RUNTIME_FAILURE',
+    exitCode: CLI_EXIT_CODES.runtimeFailure,
+    message: `Runtime failure: ${sanitizeCliErrorValue(message)}`,
+  };
+}
+
+export function formatCliError(cliTextError: CliTextError): string {
+  return `[${cliTextError.code}] ${cliTextError.message}`;
+}
+
+// 모든 CLI 오류 출력은 이 함수로 모은다.
+// 새 command가 추가돼도 stderr format과 exit code가 흩어지지 않게 하기 위함이다.
+export function writeCliError(cliTextError: CliTextError): void {
+  console.error(formatCliError(cliTextError));
+  process.exitCode = cliTextError.exitCode;
+}
+
+// Commander parse 단계에서 발생하는 오류를 Manta의 세 분류로 수렴시킨다.
+export function createCliErrorFromCommanderError(commanderError: CommanderError): CliTextError {
+  if (commanderError.code === 'commander.unknownCommand') {
+    return createUnknownCommandError(extractUnknownCommandName(commanderError.message));
+  }
+
+  return createUsageError(commanderError.message);
+}
+
+// Commander 밖에서 새어 나온 예외는 runtime failure로 본다.
+// command action 내부에서 가능한 한 core Result를 명시적으로 변환하되, top-level guard도 둔다.
+export function createCliErrorFromUnknownError(error: unknown): CliTextError {
+  if (error instanceof Error) {
+    return createRuntimeFailureError(error.message);
+  }
+
+  return createRuntimeFailureError(String(error));
+}
+
+function stripCommanderErrorPrefix(message: string): string {
+  return message.replace(/^error:\s*/, '');
+}
+
+function extractUnknownCommandName(message: string): string {
+  const match = stripCommanderErrorPrefix(message).match(/^unknown command '(.+)'$/);
+  return match ? match[1] : stripCommanderErrorPrefix(message);
+}
```

## 파일: `packages/cli/src/cli.ts` (신규)

```diff
+import { Command, CommanderError } from 'commander';
+import { VERSION } from '@manta/core';
+import { commandFactories } from './commands/command-factories';
+import { commandHelpEntries } from './help/command-registry';
+import { formatForCommanderHook, renderOverview } from './help/render-help';
+import {
+  createCliErrorFromCommanderError,
+  createCliErrorFromUnknownError,
+  writeCliError,
+} from './errors/cli-error-policy';
+
+export function createMantaProgram(): Command {
+  const program = new Command()
+    .name('manta')
+    .description('File-based task management for humans and AI')
+    .version(VERSION)
+    // Commander가 process.exit()를 직접 호출하지 못하게 막는다.
+    // 그래야 top-level에서 CommanderError를 Manta error policy로 변환할 수 있다.
+    .exitOverride()
+    .configureHelp({
+      formatHelp: () => formatForCommanderHook(renderOverview(commandHelpEntries)),
+    })
+    .configureOutput({
+      // Commander 기본 stderr는 Manta 형식이 아니므로 숨긴다.
+      // 실제 출력은 catch block에서 writeCliError()로 한 번만 한다.
+      writeErr: () => {},
+      outputError: () => {},
+    });
+
+  for (const entry of commandHelpEntries) {
+    program.addCommand(commandFactories[entry.name]());
+  }
+
+  return program;
+}
+
+export async function runMantaCli(argv: readonly string[] = process.argv): Promise<void> {
+  const program = createMantaProgram();
+
+  try {
+    // init 같은 command action은 async 파일 I/O를 한다.
+    // parseAsync()를 써야 비동기 실패도 top-level policy가 기다리고 분류할 수 있다.
+    await program.parseAsync([...argv]);
+  } catch (error) {
+    if (error instanceof CommanderError) {
+      writeCliError(createCliErrorFromCommanderError(error));
+      return;
+    }
+
+    writeCliError(createCliErrorFromUnknownError(error));
+  }
+}
```

## 파일: `packages/cli/src/index.ts`

```diff
 #!/usr/bin/env node
 
-import { Command } from 'commander';
-import { VERSION } from '@manta/core';
-import { commandFactories } from './commands/command-factories';
-import { commandHelpEntries } from './help/command-registry';
-import { formatForCommanderHook, renderOverview } from './help/render-help';
+import { runMantaCli } from './cli';
 
-const program = new Command()
-  .name('manta')
-  .description('File-based task management for humans and AI')
-  .version(VERSION)
-  .configureHelp({
-    formatHelp: () => formatForCommanderHook(renderOverview(commandHelpEntries)),
-  });
-
-for (const entry of commandHelpEntries) {
-  program.addCommand(commandFactories[entry.name]());
-}
-
-program.parse();
+void runMantaCli();
```

## 파일: `packages/cli/src/commands/help.ts`

```diff
 import { commandHelpEntries, findHelpEntry } from '../help/command-registry';
 import { applyHelpEntryToCommand } from '../help/apply-help-entry';
 import { renderCommand, renderOverview, toCommandJson, toOverviewJson } from '../help/render-help';
+import { createUnknownCommandError, writeCliError } from '../errors/cli-error-policy';
@@
     const targetEntry = findHelpEntry(commandName);
     if (!targetEntry) {
-      console.error(
-        `Unknown command: ${commandName}. Run \`manta help\` to see available commands.`,
-      );
-      process.exitCode = 1;
+      // help 내부 unknown도 root unknown과 같은 error policy를 탄다.
+      writeCliError(createUnknownCommandError(commandName));
       return;
     }
```

## 파일: `packages/cli/src/commands/init.ts`

```diff
 import chalk from 'chalk';
 import { resolveTaskDirPath, initializeMantaProject, getMantaDataDir } from '@manta/core';
 import { findHelpEntry } from '../help/command-registry';
 import { applyHelpEntryToCommand } from '../help/apply-help-entry';
+import { createRuntimeFailureError, writeCliError } from '../errors/cli-error-policy';
@@
         console.log(chalk.yellow(result.message));
         return;
       }
-      console.error(chalk.red(`Error: ${result.message}`));
-      process.exitCode = 1;
+      // core Result 실패는 Commander usage error가 아니라 실행 중 실패다.
+      // 따라서 exit 1과 RUNTIME_FAILURE code를 사용한다.
+      writeCliError(createRuntimeFailureError(result.message));
       return;
     }
```

## 파일: `packages/cli/src/errors/cli-error-policy.test.ts` (신규)

```diff
+import { CommanderError } from 'commander';
+import {
+  CLI_EXIT_CODES,
+  createCliErrorFromCommanderError,
+  createRuntimeFailureError,
+  createUnknownCommandError,
+  createUsageError,
+  formatCliError,
+  sanitizeCliErrorValue,
+} from './cli-error-policy';
+
+describe('cli error policy', () => {
+  it('should classify unknown command as usage error with stable code', () => {
+    const cliTextError = createUnknownCommandError('missing');
+
+    expect(cliTextError).toEqual({
+      code: 'UNKNOWN_COMMAND',
+      exitCode: CLI_EXIT_CODES.usageError,
+      message: 'Unknown command: missing. Run `manta help` to see available commands.',
+    });
+  });
+
+  it('should classify runtime failures separately from usage errors', () => {
+    const cliTextError = createRuntimeFailureError('Permission denied: /project');
+
+    expect(cliTextError.code).toBe('RUNTIME_FAILURE');
+    expect(cliTextError.exitCode).toBe(CLI_EXIT_CODES.runtimeFailure);
+    expect(formatCliError(cliTextError)).toBe(
+      '[RUNTIME_FAILURE] Runtime failure: Permission denied: /project',
+    );
+  });
+
+  it('should normalize Commander usage messages', () => {
+    const cliTextError = createUsageError("error: unknown option '--bad'");
+
+    expect(cliTextError).toEqual({
+      code: 'USAGE_ERROR',
+      exitCode: CLI_EXIT_CODES.usageError,
+      message: "Usage error: unknown option '--bad'",
+    });
+  });
+
+  it('should map Commander unknown command to the Manta unknown command contract', () => {
+    const commanderError = new CommanderError(
+      1,
+      'commander.unknownCommand',
+      "error: unknown command 'xyz'",
+    );
+
+    expect(createCliErrorFromCommanderError(commanderError)).toEqual(
+      createUnknownCommandError('xyz'),
+    );
+  });
+
+  it('should map Commander parse failures to usage errors', () => {
+    const commanderError = new CommanderError(
+      1,
+      'commander.unknownOption',
+      "error: unknown option '--bad'",
+    );
+
+    expect(createCliErrorFromCommanderError(commanderError)).toEqual(
+      createUsageError("error: unknown option '--bad'"),
+    );
+  });
+
+  it('should sanitize user-controlled values before rendering stderr', () => {
+    expect(sanitizeCliErrorValue('bad\n\x1b[31mvalue\x1b[0m\tname')).toBe('bad value name');
+  });
+
+  it('should truncate very long user-controlled values', () => {
+    const sanitizedValue = sanitizeCliErrorValue('a'.repeat(200));
+
+    expect(sanitizedValue).toHaveLength(123);
+    expect(sanitizedValue.endsWith('...')).toBe(true);
+  });
+});
```

## 파일: `packages/cli/src/cli.test.ts` (신규)

```diff
+import { runMantaCli } from './cli';
+
+describe('runMantaCli error contract', () => {
+  let errorSpy: jest.SpyInstance;
+  let logSpy: jest.SpyInstance;
+
+  beforeEach(() => {
+    process.exitCode = 0;
+    errorSpy = jest.spyOn(console, 'error').mockImplementation(() => {});
+    logSpy = jest.spyOn(console, 'log').mockImplementation(() => {});
+  });
+
+  afterEach(() => {
+    errorSpy.mockRestore();
+    logSpy.mockRestore();
+    process.exitCode = 0;
+  });
+
+  it('should emit UNKNOWN_COMMAND with exit code 2 for root unknown command', async () => {
+    await runMantaCli(['node', 'manta', 'xyz']);
+
+    expect(logSpy).not.toHaveBeenCalled();
+    expect(errorSpy).toHaveBeenCalledWith(
+      '[UNKNOWN_COMMAND] Unknown command: xyz. Run `manta help` to see available commands.',
+    );
+    expect(process.exitCode).toBe(2);
+  });
+
+  it('should emit UNKNOWN_COMMAND with exit code 2 for help unknown command', async () => {
+    await runMantaCli(['node', 'manta', 'help', 'xyz']);
+
+    expect(logSpy).not.toHaveBeenCalled();
+    expect(errorSpy).toHaveBeenCalledWith(
+      '[UNKNOWN_COMMAND] Unknown command: xyz. Run `manta help` to see available commands.',
+    );
+    expect(process.exitCode).toBe(2);
+  });
+
+  it('should emit USAGE_ERROR with exit code 2 for bad options', async () => {
+    await runMantaCli(['node', 'manta', 'init', '--bad']);
+
+    expect(logSpy).not.toHaveBeenCalled();
+    expect(errorSpy).toHaveBeenCalledWith("[USAGE_ERROR] Usage error: unknown option '--bad'");
+    expect(process.exitCode).toBe(2);
+  });
+
+  it('should emit USAGE_ERROR with exit code 2 for excess arguments', async () => {
+    await runMantaCli(['node', 'manta', 'help', 'init', 'extra']);
+
+    expect(logSpy).not.toHaveBeenCalled();
+    expect(errorSpy.mock.calls[0][0]).toContain('[USAGE_ERROR] Usage error: too many arguments');
+    expect(process.exitCode).toBe(2);
+  });
+});
```

## 파일: `packages/cli/src/commands/help.test.ts`

```diff
-  it('should write to stderr and set exitCode=1 for unknown command name', () => {
+  it('should write to stderr and set exitCode=2 for unknown command name', () => {
     runHelp(['xyz']);
     expect(errorSpy).toHaveBeenCalledTimes(1);
-    expect(errorSpy.mock.calls[0][0] as string).toContain('Unknown command: xyz');
-    expect(process.exitCode).toBe(1);
+    expect(errorSpy.mock.calls[0][0]).toBe(
+      '[UNKNOWN_COMMAND] Unknown command: xyz. Run `manta help` to see available commands.',
+    );
+    expect(process.exitCode).toBe(2);
   });
 });
```

## 검증 순서

코드 적용 후 `manta-repo/`에서 실행한다.

```bash
npm run build
npm test
npm run lint
```

수동 확인:

```bash
node packages/cli/dist/index.js xyz
node packages/cli/dist/index.js help xyz
node packages/cli/dist/index.js init --bad
node packages/cli/dist/index.js help init extra
```

기대:

- `xyz`, `help xyz`: stderr `[UNKNOWN_COMMAND] ...`, stdout empty, exit `2`
- bad option, excess args: stderr `[USAGE_ERROR] ...`, stdout empty, exit `2`
- runtime failure: stderr `[RUNTIME_FAILURE] ...`, stdout empty, exit `1`
- help 정상 경로는 task-10 출력 계약 유지
