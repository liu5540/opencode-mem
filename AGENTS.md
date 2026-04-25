# AGENTS.md — OpenCode Mem

## Commands

- **Build**: `bun run build`
- **Type check**: `bun run typecheck` (also runs in pre-commit)
- **Format**: `bun run format`
- **Format check**: `bun run format:check`
- **Run all tests**: `bun test`
- **Run single test file**: `bun test tests/config.test.ts`
- **Run single test by name**: `bun test --test-name-pattern="should default to"`
- **Install deps**: `bun install`

## Runtime & Package Manager

- Use **Bun** (`bun`) as both runtime and package manager.
- Do not use `npm` or `node` directly for scripts.

## TypeScript

- Target: `ESNext`, modules: `ESNext` with `moduleResolution: bundler`.
- `strict: true`, `noUncheckedIndexedAccess: true`, `noFallthroughCasesInSwitch: true`, `noImplicitOverride: true`.
- `verbatimModuleSyntax: true` — use `import type { Foo }` for type-only imports.
- `allowImportingTsExtensions: false` — import paths must end in `.js` (e.g., `./config.js`), even for `.ts` files.
- `noUnusedLocals: false` and `noUnusedParameters: false` — unused locals/parameters do not fail build.
- Prefer explicit return types on exported/public functions and methods.
- Use `as const` for literal return objects when appropriate.

## Code Style

- **Prettier** is enforced. Config:
  - `semi: true`
  - `singleQuote: false` (double quotes)
  - `tabWidth: 2`
  - `useTabs: false`
  - `printWidth: 100`
  - `trailingComma: "es5"`
  - `bracketSpacing: true`
  - `arrowParens: "always"`
  - `endOfLine: "lf"`
- Run `bun run format` before committing. Husky pre-commit runs `bun run typecheck && bunx lint-staged`.

## Imports

- Group imports:
  1. External packages (e.g., `@opencode-ai/plugin`, `zod`)
  2. Node built-ins with `node:` prefix (e.g., `node:fs`, `node:path`)
  3. Internal modules with `.js` extension (e.g., `./config.js`)
- Use `import type` for type-only imports.
- Avoid default exports unless required by a framework/plugin API.

## Naming Conventions

- `PascalCase` for classes, interfaces, type aliases, and enums.
- `camelCase` for variables, functions, methods, and file names.
- `UPPER_SNAKE_CASE` for module-level constants and `Symbol.for` keys.
- Boolean variables should read as predicates when possible (e.g., `isConfigured`, `hasNonEmptyChoices`).
- Private class members prefixed with `_` or use `private`/`protected` access modifiers.

## Error Handling

- Prefer returning result objects over throwing: `{ success: true as const, ... }` or `{ success: false as const, error: string }`.
- When catching errors, normalize to string: `error instanceof Error ? error.message : String(error)`.
- Use the project `log(message, data?)` utility for logging instead of `console.log`.
- Swallow unimportant errors silently only when explicitly intended (e.g., `.catch(() => {})` for toasts).

## Types & Interfaces

- Define reusable types in `src/types/index.ts` or adjacent `types.ts` files.
- Use `interface` for object shapes; use `type` for unions, mapped types, and aliases.
- Avoid `any` when possible; use `unknown` and narrow with type guards (e.g., `isErrorResponseBody`).
- Function signatures should be explicit; avoid implicit `any`.

## Testing

- Test framework: **Bun** built-in test runner (`bun:test`).
- Use `describe` / `it` / `expect` from `bun:test`.
- Tests live in `tests/` directory, named `*.test.ts`.
- Use `afterAll` / `beforeAll` for setup/teardown when needed.
- Prefer `await import()` inside tests when isolating module side effects.

## Project Structure

- Source code: `src/`
- Tests: `tests/`
- Built output: `dist/` (do not edit manually)
- Plugin entry: `src/plugin.ts`
- Config logic: `src/config.ts`

## Commit & Hooks

- Pre-commit hook runs typecheck and `lint-staged` (Prettier on staged files).
- Do not bypass hooks unless explicitly instructed.
