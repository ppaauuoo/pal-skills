---
name: beautiful-tui
description: >-
  Scaffold a beautiful terminal UI for any CLI tool or pipeline using ink
  (React for terminals) and termcn (shadcn-style terminal components). Use
  when the user asks to make a CLI pretty, add a TUI, build a terminal
  dashboard, add spinners/progress bars/tables to a script runner, or says
  "make it beautiful" for any command-line tool. Covers: project scaffold,
  termcn components (Spinner, ProgressBar, Table, ThemeProvider), pipeline
  orchestration with live progress, and theme customization.
---

# Beautiful TUI — ink + termcn

Build gorgeous terminal interfaces using React (ink) and termcn components.

---

## When to Use

- User says "make it beautiful", "add a TUI", "pretty CLI", "terminal dashboard"
- Wrapping a multi-step pipeline (Python, shell, etc.) with live progress
- Building interactive terminal tools with spinners, tables, progress bars
- Any CLI that currently prints ugly text and deserves better

---

## Step 0 — Scaffold the Project

Create a `cli/` directory (or whatever name fits) next to the scripts being wrapped.

### package.json

```json
{
  "name": "<project>-cli",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "tsx src/cli.tsx"
  },
  "dependencies": {
    "ink": "^7.1.0",
    "react": "^19.2.0",
    "cli-spinners": "^3.2.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/node": "^22.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.7.0"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "lib": ["ESNext"],
    "types": ["node"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist",
    "rootDir": "src",
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

### components.json (termcn registry)

```json
{
  "registries": {
    "@termcn": "https://termcn.dev/r/{name}.json"
  }
}
```

Then `npm install`.

---

## Step 1 — termcn Components

Copy these into `src/components/ui/`. They follow the termcn API exactly — same props, same behavior. This is the copy-paste model (like shadcn).

### Spinner (`src/components/ui/spinner.tsx`)

```tsx
import React, { useState, useEffect } from "react";
import { Text } from "ink";
import cliSpinners from "cli-spinners";

type SpinnerName = keyof typeof cliSpinners;

interface SpinnerProps {
  type?: SpinnerName;
  label?: string;
  color?: string;
}

export const spinnerNames = Object.keys(cliSpinners) as SpinnerName[];

export function Spinner({ type = "dots", label, color }: SpinnerProps) {
  const spinner = cliSpinners[type];
  const [frame, setFrame] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setFrame((f) => (f + 1) % spinner.frames.length);
    }, spinner.interval);
    return () => clearInterval(id);
  }, [spinner]);

  return (
    <Text>
      <Text color={color ?? "cyan"}>{spinner.frames[frame]}</Text>
      {label && <Text> {label}</Text>}
    </Text>
  );
}
```

### ProgressBar (`src/components/ui/progress-bar.tsx`)

```tsx
import React from "react";
import { Text } from "ink";

interface ProgressBarProps {
  value: number;
  total?: number;
  width?: number;
  showPercent?: boolean;
  fillChar?: string;
  emptyChar?: string;
  color?: string;
  label?: string;
}

export function ProgressBar({
  value,
  total = 100,
  width = 30,
  showPercent = true,
  fillChar = "█",
  emptyChar = "░",
  color,
  label,
}: ProgressBarProps) {
  const pct = Math.min(100, Math.round((value / total) * 100));
  const filled = Math.round((pct / 100) * width);
  const empty = width - filled;

  return (
    <Text>
      {label && <Text>{label} </Text>}
      <Text color={color ?? "green"}>{fillChar.repeat(filled)}</Text>
      <Text dimColor>{emptyChar.repeat(empty)}</Text>
      {showPercent && <Text> {pct}%</Text>}
      <Text dimColor> ({value}/{total})</Text>
    </Text>
  );
}
```

### ThemeProvider (`src/components/ui/theme-provider.tsx`)

```tsx
import React, { createContext, useContext, type ReactNode } from "react";

interface ThemeColors {
  primary: string;
  accent: string;
  success: string;
  warning: string;
  error: string;
  muted: string;
}

interface Theme {
  name: string;
  colors: ThemeColors;
}

const ThemeContext = createContext<Theme>({
  name: "default",
  colors: {
    primary: "#60a5fa",
    accent: "#a78bfa",
    success: "#34d399",
    warning: "#fbbf24",
    error: "#f87171",
    muted: "#6b7280",
  },
});

export function useTheme() {
  return useContext(ThemeContext);
}

export function createTheme(t: { name: string; colors: Partial<ThemeColors> }): Theme {
  return {
    name: t.name,
    colors: {
      primary: "#60a5fa",
      accent: "#a78bfa",
      success: "#34d399",
      warning: "#fbbf24",
      error: "#f87171",
      muted: "#6b7280",
      ...t.colors,
    },
  };
}

export function ThemeProvider({ theme, children }: { theme: Theme; children: ReactNode }) {
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}
```

### Table (`src/components/ui/table.tsx`)

```tsx
import React from "react";
import { Box, Text } from "ink";

interface Column<T> {
  key: keyof T & string;
  header: string;
  width?: number;
  align?: "left" | "right" | "center";
}

interface TableProps<T extends Record<string, unknown>> {
  data: T[];
  columns: Column<T>[];
  borderColor?: string;
}

export function Table<T extends Record<string, unknown>>({
  data,
  columns,
  borderColor,
}: TableProps<T>) {
  const widths = columns.map((col) => {
    const maxData = Math.max(...data.map((r) => String(r[col.key] ?? "").length));
    return col.width ?? Math.max(col.header.length, maxData) + 2;
  });

  const pad = (s: string, w: number, align: string = "left") => {
    if (align === "right") return s.padStart(w);
    if (align === "center") {
      const left = Math.floor((w - s.length) / 2);
      return " ".repeat(left) + s + " ".repeat(w - s.length - left);
    }
    return s.padEnd(w);
  };

  const border = (l: string, m: string, r: string, fill: string) =>
    l + widths.map((w) => fill.repeat(w)).join(m) + r;

  return (
    <Box flexDirection="column">
      <Text color={borderColor}>{border("╭", "┬", "╮", "─")}</Text>
      <Text color={borderColor}>
        {"│"}
        {columns.map((col, i) => (
          <Text key={col.key}>
            <Text bold>{pad(col.header, widths[i], col.align)}</Text>
            <Text color={borderColor}>│</Text>
          </Text>
        ))}
      </Text>
      <Text color={borderColor}>{border("├", "┼", "┤", "─")}</Text>
      {data.map((row, ri) => (
        <Text key={ri} color={borderColor}>
          {"│"}
          {columns.map((col, i) => (
            <Text key={col.key}>
              <Text>{pad(String(row[col.key] ?? ""), widths[i], col.align)}</Text>
              <Text color={borderColor}>│</Text>
            </Text>
          ))}
        </Text>
      ))}
      <Text color={borderColor}>{border("╰", "┴", "╯", "─")}</Text>
    </Box>
  );
}
```

---

## Step 2 — Build the CLI

The main `src/cli.tsx` pattern for wrapping a multi-step pipeline:

```tsx
import React, { useState, useEffect } from "react";
import { render, Box, Text } from "ink";
import { Spinner } from "./components/ui/spinner.js";
import { ProgressBar } from "./components/ui/progress-bar.js";
import { ThemeProvider, createTheme, useTheme } from "./components/ui/theme-provider.js";
import { spawn } from "child_process";
import path from "path";
import { fileURLToPath } from "url";

const theme = createTheme({
  name: "my-pipeline",
  colors: {
    primary: "#60a5fa",
    accent: "#c084fc",
    success: "#34d399",
  },
});

// Spawn a child process, stream lines to a callback
function runStep(cmd: string, args: string[], cwd: string, onLine: (line: string) => void): Promise<number> {
  return new Promise((resolve, reject) => {
    const proc = spawn(cmd, args, { cwd, shell: true, env: { ...process.env } });
    let buf = "";
    const handle = (data: Buffer) => {
      buf += data.toString();
      const lines = buf.split("\n");
      buf = lines.pop() ?? "";
      for (const l of lines) if (l.trim()) onLine(l.trim());
    };
    proc.stdout?.on("data", handle);
    proc.stderr?.on("data", handle);
    proc.on("close", (code) => resolve(code ?? 0));
    proc.on("error", reject);
  });
}

function App() {
  const { colors } = useTheme();
  // ... state, useEffect to run steps, render Spinner/ProgressBar per step
  return (
    <Box flexDirection="column" padding={1}>
      {/* Header box with unicode borders */}
      {/* StepIndicator components per pipeline step */}
      {/* Summary on completion */}
    </Box>
  );
}

render(<ThemeProvider theme={theme}><App /></ThemeProvider>);
```

---

## Step 3 — Patterns & Recipes

### Header with Box Drawing

```tsx
function Header({ title }: { title: string }) {
  const { colors } = useTheme();
  const w = title.length + 4;
  return (
    <Box flexDirection="column" marginBottom={1}>
      <Text color={colors.accent} bold>{"╭" + "─".repeat(w) + "╮"}</Text>
      <Text color={colors.accent} bold>{"│ "}<Text color={colors.primary}>{title}</Text>{" │"}</Text>
      <Text color={colors.accent} bold>{"╰" + "─".repeat(w) + "╯"}</Text>
    </Box>
  );
}
```

### Step Indicator (status icon + spinner/progress)

```tsx
type StepStatus = "pending" | "running" | "done" | "error";

function StepIndicator({ label, status, progress, total, lastLog, elapsed }: {
  label: string; status: StepStatus; progress: number; total: number; lastLog: string; elapsed: number;
}) {
  const { colors } = useTheme();
  const icon = { pending: "○", running: "›", done: "✓", error: "✗" }[status];
  const iconColor = { pending: colors.muted, running: colors.accent, done: colors.success, error: colors.error }[status];

  return (
    <Box flexDirection="column" marginBottom={1}>
      <Box gap={1}>
        <Text color={iconColor} bold>{icon}</Text>
        <Text bold={status === "running"}>{label}</Text>
        {status === "done" && <Text dimColor>({(elapsed / 1000).toFixed(1)}s)</Text>}
      </Box>
      {status === "running" && (
        <Box marginLeft={3} flexDirection="column">
          {total > 0
            ? <ProgressBar value={progress} total={total} color={colors.accent} width={35} />
            : <Spinner type="dots" label="working..." color={colors.accent} />}
          {lastLog && <Text dimColor>  {lastLog}</Text>}
        </Box>
      )}
    </Box>
  );
}
```

### Parse Progress from stdout

Match `[N/M]` patterns, "Found X items", or any structured output:

```tsx
const match = line.match(/\[(\d+)\/(\d+)\]/);
if (match) {
  setProgress(parseInt(match[1], 10));
  setTotal(parseInt(match[2], 10));
}
```

---

## Step 4 — Run

```bash
# One-shot
cd cli && npx tsx src/cli.tsx

# Or a .bat / .sh wrapper
@echo off
cd /d "%~dp0cli"
npx tsx src/cli.tsx
```

Environment variables for config (no arg-parsing lib needed):

```tsx
const CONFIG = {
  input: process.env.INPUT ?? "default_input",
  output: process.env.OUTPUT ?? "default_output",
  limit: parseInt(process.env.LIMIT ?? "20", 10),
};
```

---

## Available Themes

Pass to `createTheme()`:

| Name | Primary | Accent | Vibe |
|------|---------|--------|------|
| default | `#60a5fa` | `#a78bfa` | Cool blue/purple |
| dracula | `#bd93f9` | `#ff79c6` | Dark purple/pink |
| nord | `#88c0d0` | `#81a1c1` | Arctic blue |
| catppuccin | `#cba6f7` | `#f5c2e7` | Pastel lavender |
| tokyo-night | `#7aa2f7` | `#bb9af7` | Neon night |

Or install official termcn themes: `npx shadcn@latest add @termcn/theme-dracula`

---

## Notes

- **Requires Node.js 22+** (ink 7 requirement)
- `tsx` handles TypeScript + JSX without a build step
- The `runStep()` helper works for any child process (Python, shell, Go, etc.)
- Progress parsing is regex-based — adapt patterns to match your script output
- For interactive features (selectable tables, text input), see termcn docs: https://www.termcn.dev/docs/registry
