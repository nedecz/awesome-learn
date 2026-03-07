# TypeScript Tooling

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [tsconfig.json Deep Dive](#tsconfigjson-deep-dive)
   - [Essential Compiler Options](#essential-compiler-options)
   - [Strict Mode Options](#strict-mode-options)
   - [Module and Resolution Options](#module-and-resolution-options)
   - [Project References](#project-references)
   - [Composite Projects](#composite-projects)
3. [ESLint with TypeScript](#eslint-with-typescript)
   - [Setting Up typescript-eslint](#setting-up-typescript-eslint)
   - [Recommended Configurations](#recommended-configurations)
   - [Custom Rules](#custom-rules)
   - [Performance Optimization](#performance-optimization)
4. [Prettier Configuration](#prettier-configuration)
   - [Prettier Setup for TypeScript](#prettier-setup-for-typescript)
   - [Integrating Prettier with ESLint](#integrating-prettier-with-eslint)
5. [Build Tools](#build-tools)
   - [tsc - The TypeScript Compiler](#tsc---the-typescript-compiler)
   - [esbuild](#esbuild)
   - [SWC](#swc)
   - [Rollup](#rollup)
   - [Webpack](#webpack)
   - [Vite](#vite)
6. [Package Managers](#package-managers)
   - [npm with TypeScript](#npm-with-typescript)
   - [Yarn with TypeScript](#yarn-with-typescript)
   - [pnpm with TypeScript](#pnpm-with-typescript)
7. [Monorepo Tools](#monorepo-tools)
   - [Nx](#nx)
   - [Turborepo](#turborepo)
   - [Lerna](#lerna)
8. [Editor Integration](#editor-integration)
   - [VS Code](#vs-code)
   - [IntelliJ and WebStorm](#intellij-and-webstorm)
9. [CI/CD Integration](#cicd-integration)
   - [Type Checking in Pipelines](#type-checking-in-pipelines)
   - [GitHub Actions Example](#github-actions-example)
10. [Debugging TypeScript](#debugging-typescript)
    - [Source Maps](#source-maps)
    - [VS Code Debugging](#vs-code-debugging)
    - [Node.js Debugging](#nodejs-debugging)
11. [Migration Tooling](#migration-tooling)
    - [ts-migrate](#ts-migrate)
    - [dts-gen](#dts-gen)
    - [Incremental Migration Strategies](#incremental-migration-strategies)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

TypeScript's power extends far beyond its type system — the surrounding ecosystem of tools, build systems, and developer experience enhancements is what makes TypeScript productive at scale. A well-configured toolchain catches errors early, enforces consistency, speeds up builds, and streamlines the development workflow from local editing to production deployment.

This guide covers the essential tools that every TypeScript project should consider, from compiler configuration and linting to build systems, monorepo orchestration, and debugging. Each section provides practical configuration examples and explains the trade-offs between different approaches.

### Target Audience

- **TypeScript developers** — looking to optimize their development workflow and understand best-practice tooling configurations
- **Tech leads and architects** — evaluating build tools, monorepo strategies, and CI/CD pipelines for TypeScript projects
- **Teams migrating to TypeScript** — needing guidance on migration tooling and incremental adoption strategies

### Scope

- Deep dive into tsconfig.json compiler options and project references
- ESLint and Prettier setup for TypeScript codebases
- Comparison of build tools including tsc, esbuild, SWC, Rollup, Webpack, and Vite
- Package manager and monorepo tool considerations
- Editor integration, CI/CD pipelines, and debugging workflows
- Migration tooling for adopting TypeScript in existing JavaScript projects

---

## tsconfig.json Deep Dive

The `tsconfig.json` file is the foundation of every TypeScript project. It controls how the compiler interprets your code, what JavaScript it emits, and how modules are resolved.

### Essential Compiler Options

A well-structured tsconfig starts with a clear understanding of the most impactful options:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

**Key options explained:**

- **`target`** — The ECMAScript version for emitted JavaScript. `ES2022` supports top-level await, class fields, and `Object.hasOwn`.
- **`module`** — Determines the module system for output. `NodeNext` respects `package.json` `"type"` fields.
- **`isolatedModules`** — Ensures each file can be transpiled independently, required for tools like esbuild and SWC.
- **`declaration`** — Generates `.d.ts` type declaration files, essential for library authors.
- **`declarationMap`** — Enables "Go to Definition" to navigate to the original `.ts` source instead of `.d.ts`.

### Strict Mode Options

The `strict` flag is an umbrella that enables multiple individual checks. Understanding each one helps when migrating gradually:

```json
{
  "compilerOptions": {
    "strict": true,

    "noUncheckedIndexedAccess": true,
    "noPropertyAccessFromIndexSignature": true,
    "exactOptionalPropertyTypes": true
  }
}
```

The `strict` flag enables:

```typescript
// strictNullChecks — null and undefined are distinct types
function getLength(str: string | null): number {
  // Error: 'str' is possibly 'null'
  // return str.length;
  return str?.length ?? 0;
}

// strictFunctionTypes — contravariant parameter checking
type Handler = (event: MouseEvent) => void;
// Error: Type '(event: Event) => void' is not assignable
// const handler: Handler = (event: Event) => {};

// noImplicitAny — parameters must have explicit types
function processItem(item: unknown): string {
  // Error without annotation: Parameter 'item' implicitly has an 'any' type
  return String(item);
}

// strictPropertyInitialization — class properties must be initialized
class User {
  name: string;
  email: string;

  constructor(name: string, email: string) {
    this.name = name;
    this.email = email;
  }
}
```

Additional strict-adjacent options beyond the `strict` umbrella:

```typescript
// noUncheckedIndexedAccess — indexed access returns T | undefined
const scores: Record<string, number> = { math: 95 };
const mathScore = scores["math"]; // type is number | undefined
if (mathScore !== undefined) {
  console.log(mathScore.toFixed(2)); // safe to use
}

// exactOptionalPropertyTypes — distinguishes missing from undefined
interface Config {
  debug?: boolean;
}
const config: Config = {};
// config.debug = undefined; // Error with exactOptionalPropertyTypes
delete config.debug; // OK — removes the property
```

### Module and Resolution Options

Module resolution has become more nuanced with the rise of ESM:

```json
{
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",

    "paths": {
      "@app/*": ["./src/*"],
      "@lib/*": ["./src/lib/*"],
      "@tests/*": ["./tests/*"]
    },
    "baseUrl": "."
  }
}
```

For bundler-based projects (Vite, webpack, etc.), use the `bundler` resolution:

```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true
  }
}
```

### Project References

Project references enable incremental builds across multiple related projects within a repository:

```json
{
  "compilerOptions": {
    "composite": true,
    "declaration": true,
    "declarationMap": true
  },
  "references": [
    { "path": "./packages/shared" },
    { "path": "./packages/server" },
    { "path": "./packages/client" }
  ],
  "files": []
}
```

Each referenced project has its own `tsconfig.json`:

```json
{
  "compilerOptions": {
    "composite": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true
  },
  "references": [
    { "path": "../shared" }
  ],
  "include": ["src/**/*"]
}
```

Build with references using:

```bash
# Build all referenced projects in dependency order
tsc --build

# Build with verbose output
tsc --build --verbose

# Clean all build outputs
tsc --build --clean

# Force a full rebuild
tsc --build --force
```

### Composite Projects

Composite projects combine project references with `composite: true` to enable efficient incremental compilation. The compiler stores build metadata in `.tsbuildinfo` files:

```json
{
  "compilerOptions": {
    "composite": true,
    "incremental": true,
    "tsBuildInfoFile": "./dist/.tsbuildinfo"
  }
}
```

A typical monorepo structure with composite projects:

```
monorepo/
├── tsconfig.json              # Root config with references
├── packages/
│   ├── shared/
│   │   ├── tsconfig.json      # composite: true
│   │   └── src/
│   ├── api/
│   │   ├── tsconfig.json      # references shared
│   │   └── src/
│   └── web/
│       ├── tsconfig.json      # references shared
│       └── src/
```

---

## ESLint with TypeScript

ESLint is the standard linter for TypeScript projects. The `typescript-eslint` project provides the parser and rules needed to lint TypeScript code effectively.

### Setting Up typescript-eslint

Install the required packages:

```bash
npm install --save-dev eslint @eslint/js typescript-eslint
```

Create an `eslint.config.mjs` using the flat config format:

```typescript
// eslint.config.mjs
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  },
  {
    ignores: ["dist/", "node_modules/", "*.js"],
  }
);
```

### Recommended Configurations

`typescript-eslint` provides several preset configurations with increasing strictness:

```typescript
// eslint.config.mjs
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,

  // Choose ONE of these presets:
  // Level 1: Basic recommended rules
  ...tseslint.configs.recommended,

  // Level 2: Recommended + type-checked rules (uses type information)
  // ...tseslint.configs.recommendedTypeChecked,

  // Level 3: Strict rules (opinionated, catches more issues)
  // ...tseslint.configs.strict,

  // Level 4: Strict + type-checked (maximum safety)
  // ...tseslint.configs.strictTypeChecked,

  // Optional: Stylistic rules for consistent code style
  // ...tseslint.configs.stylisticTypeChecked,
);
```

### Custom Rules

Configure rules to match your team's preferences:

```typescript
// eslint.config.mjs
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // Enforce explicit return types on exported functions
      "@typescript-eslint/explicit-function-return-type": [
        "error",
        { allowExpressions: true },
      ],

      // Prevent floating promises (missing await)
      "@typescript-eslint/no-floating-promises": "error",

      // Require handling of promise rejections
      "@typescript-eslint/no-misused-promises": "error",

      // Prefer nullish coalescing over logical OR
      "@typescript-eslint/prefer-nullish-coalescing": "warn",

      // Enforce consistent type imports
      "@typescript-eslint/consistent-type-imports": [
        "error",
        { prefer: "type-imports", fixStyle: "inline-type-imports" },
      ],

      // Naming conventions
      "@typescript-eslint/naming-convention": [
        "error",
        {
          selector: "interface",
          format: ["PascalCase"],
        },
        {
          selector: "typeAlias",
          format: ["PascalCase"],
        },
        {
          selector: "enum",
          format: ["PascalCase"],
        },
      ],
    },
  },
  {
    // Test-specific rule overrides
    files: ["**/*.test.ts", "**/*.spec.ts"],
    rules: {
      "@typescript-eslint/no-explicit-any": "off",
      "@typescript-eslint/no-non-null-assertion": "off",
    },
  }
);
```

### Performance Optimization

Type-checked rules require the TypeScript compiler, which adds overhead. Optimize ESLint performance with these strategies:

```typescript
// eslint.config.mjs — use projectService for better caching
export default tseslint.config(
  {
    languageOptions: {
      parserOptions: {
        // projectService is faster than specifying project directly
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
  }
);
```

```bash
# Run ESLint with caching to speed up repeated runs
eslint --cache --cache-location .eslintcache .

# Lint only changed files using lint-staged
npx lint-staged
```

---

## Prettier Configuration

Prettier enforces consistent code formatting, eliminating style debates and reducing diff noise in code reviews.

### Prettier Setup for TypeScript

Install Prettier:

```bash
npm install --save-dev prettier
```

Create a `.prettierrc` configuration:

```json
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

Add a `.prettierignore` file:

```
dist/
node_modules/
coverage/
*.tsbuildinfo
pnpm-lock.yaml
package-lock.json
```

Add format scripts to `package.json`:

```json
{
  "scripts": {
    "format": "prettier --write \"src/**/*.{ts,tsx,json,css}\"",
    "format:check": "prettier --check \"src/**/*.{ts,tsx,json,css}\""
  }
}
```

### Integrating Prettier with ESLint

To prevent ESLint and Prettier from conflicting, use `eslint-config-prettier` to disable formatting-related ESLint rules:

```bash
npm install --save-dev eslint-config-prettier
```

```typescript
// eslint.config.mjs
import eslint from "@eslint/js";
import tseslint from "typescript-eslint";
import prettierConfig from "eslint-config-prettier";

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.recommended,
  prettierConfig, // Must be last to override formatting rules
);
```

---

## Build Tools

Choosing the right build tool impacts developer experience, build speed, and output quality. Here is how the major options compare for TypeScript projects.

### tsc - The TypeScript Compiler

The official compiler is the baseline for correctness. It performs full type checking and emits JavaScript:

```bash
# Compile the project
tsc

# Watch mode for development
tsc --watch

# Type check without emitting files
tsc --noEmit

# Build with project references
tsc --build
```

A production-oriented tsconfig for a Node.js library:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "isolatedModules": true
  },
  "include": ["src/**/*"]
}
```

### esbuild

esbuild is an extremely fast bundler written in Go. It transpiles TypeScript but does **not** perform type checking:

```bash
npm install --save-dev esbuild
```

```typescript
// build.mjs
import * as esbuild from "esbuild";

await esbuild.build({
  entryPoints: ["src/index.ts"],
  bundle: true,
  outfile: "dist/index.js",
  platform: "node",
  target: "node20",
  format: "esm",
  sourcemap: true,
  minify: process.env.NODE_ENV === "production",
  external: ["express", "pg"],
});
```

```json
{
  "scripts": {
    "build": "tsc --noEmit && node build.mjs",
    "build:fast": "node build.mjs"
  }
}
```

The key pattern is to use `tsc --noEmit` for type checking alongside esbuild for fast bundling.

### SWC

SWC is a Rust-based compiler that serves as a drop-in replacement for tsc transpilation:

```bash
npm install --save-dev @swc/cli @swc/core
```

Create a `.swcrc` configuration:

```json
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": {
      "syntax": "typescript",
      "tsx": false,
      "decorators": true,
      "dynamicImport": true
    },
    "target": "es2022",
    "loose": false,
    "externalHelpers": false
  },
  "module": {
    "type": "es6"
  },
  "sourceMaps": true,
  "minify": false
}
```

```bash
# Transpile a directory
npx swc src -d dist --source-maps

# Watch mode
npx swc src -d dist --watch
```

### Rollup

Rollup excels at building libraries with tree-shaking and multiple output formats:

```bash
npm install --save-dev rollup @rollup/plugin-typescript @rollup/plugin-node-resolve
```

```typescript
// rollup.config.mjs
import typescript from "@rollup/plugin-typescript";
import resolve from "@rollup/plugin-node-resolve";

export default {
  input: "src/index.ts",
  output: [
    {
      file: "dist/index.cjs",
      format: "cjs",
      sourcemap: true,
    },
    {
      file: "dist/index.mjs",
      format: "es",
      sourcemap: true,
    },
  ],
  plugins: [
    resolve(),
    typescript({
      tsconfig: "./tsconfig.json",
      declaration: true,
      declarationDir: "./dist/types",
    }),
  ],
  external: [/node_modules/],
};
```

### Webpack

Webpack remains widely used for complex application builds. Use `ts-loader` or `babel-loader` with TypeScript:

```bash
npm install --save-dev webpack webpack-cli ts-loader
```

```typescript
// webpack.config.ts
import path from "path";
import type { Configuration } from "webpack";

const config: Configuration = {
  entry: "./src/index.ts",
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: "ts-loader",
        exclude: /node_modules/,
      },
    ],
  },
  resolve: {
    extensions: [".tsx", ".ts", ".js"],
  },
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "dist"),
  },
  devtool: "source-map",
};

export default config;
```

For faster builds, use `transpileOnly` mode with `ts-loader` alongside a separate type-checking step:

```typescript
// webpack.config.ts (partial)
import ForkTsCheckerWebpackPlugin from "fork-ts-checker-webpack-plugin";

const config: Configuration = {
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        use: {
          loader: "ts-loader",
          options: { transpileOnly: true },
        },
      },
    ],
  },
  plugins: [
    // Runs type checking in a separate process
    new ForkTsCheckerWebpackPlugin(),
  ],
};
```

### Vite

Vite provides a fast development experience using native ESM and esbuild for transpilation:

```bash
npm create vite@latest my-app -- --template vanilla-ts
```

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
  build: {
    target: "es2022",
    sourcemap: true,
    lib: {
      entry: "src/index.ts",
      formats: ["es", "cjs"],
      fileName: (format) => `index.${format === "es" ? "mjs" : "cjs"}`,
    },
    rollupOptions: {
      external: [/node_modules/],
    },
  },
});
```

For applications, Vite's dev server provides near-instant hot module replacement:

```bash
# Start development server
npx vite

# Build for production
npx vite build

# Preview production build
npx vite preview
```

**Build tool comparison summary:**

| Tool | Type Check | Speed | Tree Shaking | Use Case |
|------|-----------|-------|-------------|----------|
| tsc | Yes | Moderate | No | Type checking, declaration generation |
| esbuild | No | Very fast | Yes | Applications, fast bundling |
| SWC | No | Very fast | No | Transpilation, Jest transforms |
| Rollup | Via plugin | Moderate | Yes | Libraries, multiple output formats |
| Webpack | Via plugin | Moderate | Yes | Complex applications, legacy support |
| Vite | No | Very fast | Yes | Modern applications, dev experience |

---

## Package Managers

Each package manager has TypeScript-specific considerations for type declarations, workspace management, and performance.

### npm with TypeScript

npm is the default package manager that ships with Node.js:

```bash
# Initialize a TypeScript project
npm init -y
npm install --save-dev typescript @types/node

# Install type declarations for third-party packages
npm install express
npm install --save-dev @types/express

# Check for missing type declarations
npx typesync
```

Configure scripts in `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "format": "prettier --write ."
  }
}
```

### Yarn with TypeScript

Yarn provides PnP (Plug'n'Play) which requires extra configuration for TypeScript:

```bash
# Initialize with Yarn
yarn init -2
yarn add --dev typescript @types/node

# Enable TypeScript PnP support
yarn dlx @yarnpkg/sdks vscode
```

For Yarn PnP, add the following to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "moduleResolution": "bundler"
  }
}
```

### pnpm with TypeScript

pnpm's strict node_modules structure works well with TypeScript. Its workspace feature is excellent for monorepos:

```bash
# Initialize
pnpm init
pnpm add -D typescript @types/node

# Workspace setup
pnpm add -D typescript -w  # -w flag for workspace root
```

Create a `pnpm-workspace.yaml` for monorepo projects:

```yaml
packages:
  - "packages/*"
  - "apps/*"
```

---

## Monorepo Tools

Monorepos benefit from specialized tools that orchestrate builds, cache results, and manage dependencies across packages.

### Nx

Nx provides intelligent build orchestration with computation caching and affected-project detection:

```bash
# Create a new Nx workspace
npx create-nx-workspace@latest my-org --preset=ts
```

A typical Nx project structure:

```
my-org/
├── nx.json
├── tsconfig.base.json
├── packages/
│   ├── shared/
│   │   ├── project.json
│   │   ├── tsconfig.json
│   │   └── src/
│   ├── api/
│   │   ├── project.json
│   │   ├── tsconfig.json
│   │   └── src/
│   └── web/
│       ├── project.json
│       ├── tsconfig.json
│       └── src/
```

```json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true
    },
    "typecheck": {
      "dependsOn": ["^typecheck"],
      "cache": true
    },
    "lint": {
      "cache": true
    }
  }
}
```

```bash
# Build only affected projects
npx nx affected --target=build

# Run type checking across all projects
npx nx run-many --target=typecheck

# Visualize project dependency graph
npx nx graph
```

### Turborepo

Turborepo focuses on fast, cached task execution with minimal configuration:

```bash
# Create a new Turborepo
npx create-turbo@latest
```

Configure `turbo.json`:

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "lint": {},
    "test": {
      "dependsOn": ["build"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

```bash
# Build all packages respecting dependency order
turbo build

# Run only affected tasks
turbo build --filter=...[HEAD~1]

# Run tasks in a specific package and its dependencies
turbo build --filter=@myorg/api...
```

### Lerna

Lerna is a mature monorepo tool focused on package publishing and versioning:

```bash
npm install --save-dev lerna
npx lerna init
```

```json
{
  "$schema": "node_modules/lerna/schemas/lerna-schema.json",
  "version": "independent",
  "npmClient": "pnpm",
  "command": {
    "publish": {
      "conventionalCommits": true
    }
  }
}
```

```bash
# Run TypeScript build across all packages
npx lerna run build

# Publish changed packages
npx lerna publish

# List changed packages since last release
npx lerna changed
```

---

## Editor Integration

A well-configured editor is the most impactful productivity tool for TypeScript developers.

### VS Code

VS Code has first-class TypeScript support built in. Enhance it with a workspace configuration:

```json
// .vscode/settings.json
{
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "explicit"
  },
  "typescript.preferences.importModuleSpecifier": "non-relative",
  "typescript.suggest.completeFunctionCalls": true,
  "typescript.inlayHints.parameterNames.enabled": "all",
  "typescript.inlayHints.functionLikeReturnTypes.enabled": true
}
```

Recommended VS Code extensions for TypeScript:

```json
// .vscode/extensions.json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "streetsidesoftware.code-spell-checker"
  ]
}
```

### IntelliJ and WebStorm

JetBrains IDEs provide robust TypeScript support. Key configuration points:

- **Languages & Frameworks → TypeScript** — Set the TypeScript version to the project's `node_modules/typescript` for consistency.
- **Code Style → TypeScript** — Configure formatting rules. If using Prettier, enable the Prettier plugin and set it as the default formatter.
- **ESLint** — Enable automatic ESLint configuration detection under **Languages & Frameworks → JavaScript → Code Quality Tools → ESLint**.
- **File Watchers** — Add file watchers for automatic compilation if not using a build tool with watch mode.

---

## CI/CD Integration

Integrating TypeScript tooling into CI/CD pipelines catches type errors, lint violations, and formatting issues before code reaches production.

### Type Checking in Pipelines

A robust CI pipeline runs type checking, linting, and tests as separate steps:

```bash
#!/bin/bash
# ci-check.sh — Run all TypeScript quality checks

set -euo pipefail

echo "=== Type Checking ==="
npx tsc --noEmit

echo "=== Linting ==="
npx eslint . --max-warnings 0

echo "=== Format Check ==="
npx prettier --check .

echo "=== Tests ==="
npx vitest run --coverage

echo "=== All checks passed ==="
```

### GitHub Actions Example

A complete GitHub Actions workflow for a TypeScript project:

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npx eslint . --max-warnings 0

      - name: Format check
        run: npx prettier --check .

      - name: Test
        run: npx vitest run --coverage

      - name: Build
        run: npm run build
```

For monorepos, use affected-based pipelines:

```yaml
# Monorepo CI with Turborepo
- name: Build affected packages
  run: npx turbo build --filter=...[origin/main]

- name: Test affected packages
  run: npx turbo test --filter=...[origin/main]
```

---

## Debugging TypeScript

Effective debugging requires proper source map configuration and editor integration to step through original TypeScript code rather than compiled JavaScript.

### Source Maps

Source maps create a mapping between your TypeScript source and the emitted JavaScript, enabling debuggers to show TypeScript code:

```json
{
  "compilerOptions": {
    "sourceMap": true,
    "inlineSources": true,
    "declarationMap": true
  }
}
```

Source map options:

- **`sourceMap: true`** — Generates `.js.map` files alongside `.js` output
- **`inlineSourceMap: true`** — Embeds the source map inside the `.js` file (useful for simple setups)
- **`inlineSources: true`** — Includes the original TypeScript source in the source map (enables debugging without access to `.ts` files)
- **`declarationMap: true`** — Maps `.d.ts` files back to `.ts` sources for "Go to Definition"

### VS Code Debugging

Create a VS Code launch configuration for debugging TypeScript:

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug TypeScript",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npx",
      "runtimeArgs": ["tsx"],
      "args": ["${file}"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "node_modules/**"]
    },
    {
      "name": "Debug Tests (Vitest)",
      "type": "node",
      "request": "launch",
      "runtimeExecutable": "npx",
      "runtimeArgs": [
        "vitest",
        "run",
        "--reporter=verbose",
        "--no-coverage"
      ],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "skipFiles": ["<node_internals>/**", "node_modules/**"]
    },
    {
      "name": "Attach to Process",
      "type": "node",
      "request": "attach",
      "port": 9229,
      "skipFiles": ["<node_internals>/**", "node_modules/**"]
    }
  ]
}
```

### Node.js Debugging

Debug TypeScript directly in Node.js using `tsx` or `ts-node`:

```bash
# Debug with tsx (recommended — fast, supports ESM)
node --inspect-brk -r tsx/cjs src/index.ts

# Debug with ts-node
node --inspect-brk -r ts-node/register src/index.ts

# Debug with native Node.js loader (Node 20+)
node --inspect-brk --import tsx src/index.ts
```

For debugging specific issues, add conditional breakpoints programmatically:

```typescript
function processOrder(order: Order): Receipt {
  // The debugger statement pauses execution when DevTools is attached
  if (order.total > 10000) {
    debugger;
  }

  const tax = calculateTax(order);
  const receipt = generateReceipt(order, tax);
  return receipt;
}
```

---

## Migration Tooling

Migrating an existing JavaScript project to TypeScript can be done incrementally with the help of specialized tools.

### ts-migrate

`ts-migrate` from Airbnb automates the initial conversion of JavaScript files to TypeScript by adding type annotations and suppressing errors with `@ts-expect-error` comments:

```bash
# Install ts-migrate
npx ts-migrate-full <project-directory>
```

The tool performs the following steps:

1. Renames `.js` / `.jsx` files to `.ts` / `.tsx`
2. Adds a base `tsconfig.json` if one does not exist
3. Inserts `@ts-expect-error` comments for type errors
4. Adds basic type annotations where inferable

After running `ts-migrate`, systematically resolve the `@ts-expect-error` comments:

```typescript
// Before: ts-migrate output
// @ts-expect-error -- ts-migrate: could not determine type
function processData(data) {
  return data.map((item) => item.value);
}

// After: manually fix with proper types
interface DataItem {
  value: number;
  label: string;
}

function processData(data: DataItem[]): number[] {
  return data.map((item) => item.value);
}
```

### dts-gen

`dts-gen` generates TypeScript declaration files for existing JavaScript libraries that lack type definitions:

```bash
# Install dts-gen
npm install --save-dev dts-gen

# Generate declarations from a module
npx dts-gen -m <module-name>

# Generate declarations from a specific file
npx dts-gen -e "require('./my-module')"
```

The generated `.d.ts` file provides a starting point that you can refine:

```typescript
// Generated by dts-gen
export function calculate(a: any, b: any): any;
export function format(value: any): any;

// Refined manually
export function calculate(a: number, b: number): number;
export function format(value: Date | string): string;
```

### Incremental Migration Strategies

Enable JavaScript files in your TypeScript project for a gradual migration:

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": false,
    "outDir": "./dist",
    "strict": false
  },
  "include": ["src/**/*"]
}
```

A recommended phased migration plan:

```typescript
// Phase 1: Rename files to .ts, use allowJs for unchanged files
// tsconfig.json: allowJs: true, strict: false

// Phase 2: Enable checkJs to find issues in JavaScript files
// tsconfig.json: checkJs: true

// Phase 3: Add types to critical paths first
// Focus on shared interfaces and API boundaries

// Phase 4: Enable strict mode incrementally
// Start with strictNullChecks, then add others one at a time

// Phase 5: Full strict mode
// tsconfig.json: strict: true, allowJs: false
```

Track migration progress with a script:

```bash
#!/bin/bash
# migration-progress.sh — Track TypeScript migration status

total_ts=$(find src -name "*.ts" -o -name "*.tsx" | wc -l)
total_js=$(find src -name "*.js" -o -name "*.jsx" | wc -l)
total=$((total_ts + total_js))

if [ "$total" -gt 0 ]; then
  percentage=$((total_ts * 100 / total))
  echo "Migration progress: ${percentage}% (${total_ts}/${total} files)"
  echo "  TypeScript files: ${total_ts}"
  echo "  JavaScript files: ${total_js}"
else
  echo "No source files found."
fi

# Count remaining ts-expect-error comments
errors=$(grep -r "@ts-expect-error" src/ --include="*.ts" --include="*.tsx" | wc -l)
echo "  Remaining @ts-expect-error: ${errors}"
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and type inference |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [03-NODE-AND-BACKEND.md](03-NODE-AND-BACKEND.md) | Node.js and Backend | Server-side TypeScript with Express, NestJS, Fastify, and databases |
| [04-FRONTEND-FRAMEWORKS.md](04-FRONTEND-FRAMEWORKS.md) | Frontend Frameworks | React, Angular, and Vue with TypeScript |
| [05-TESTING.md](05-TESTING.md) | Testing | Unit testing, integration testing, and E2E testing with TypeScript |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Coding standards and recommended approaches |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Guided progression through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial TypeScript Tooling documentation |
