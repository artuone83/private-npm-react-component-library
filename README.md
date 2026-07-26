# Building a Private React Primitives Library

> [!NOTE]
> Web guides
>
> - https://dev.to/receter/how-to-create-a-react-component-library-using-vites-library-mode-4lma
> - https://dev.to/epilot/building-a-scalable-react-component-library-lessons-from-concorde-elements-kdi
>   [Shadcn - Tailwind Manual Instalation](https://ui.shadcn.com/docs/installation/manual)

Stack:

- React
- TypeScript
- shadcn/ui
- Vite (library mode)
- Storybook (deployed to GitHub Pages)
- GitHub + GitHub Actions
- npm (private scope)
- tailwindcss

---

## 0. Decide on the registry first

There are two ways to make a package "private but installable as a dependency":

| Option                                                                                                                | Cost                                                   | Install experience                                                                                                                                            |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A. npm scoped private package** (`@yourscope/components` on registry.npmjs.org, `publishConfig.access: restricted`) | Requires a paid npm **Pro** user or **Teams/Org** plan | `npm install @yourscope/components` — consumers just need an `.npmrc` with an npm read token                                                                  |
| **B. GitHub Packages npm registry**                                                                                   | Free with a private GitHub repo                        | `npm install @yourscope/components` — consumers need an `.npmrc` pointing `@yourscope:registry` at `npm.pkg.github.com` + a GitHub token with `read:packages` |

> [!NOTE]
>
> This guide is written for **Option A** and highlights the small changes required for **Option B** where relevant (mainly `.npmrc`, `publishConfig.registry`, and GitHub Actions authentication).
>
> Everything else is identical. If you choose **Option B**, follow **Steps 1–8** and **Step 10** as written, then continue with the **Option B: Step-by-Step** section for all registry-specific configuration in one place.

- [Introduction to GitHub Packages](https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages)

---

## 1. Repo scaffold

> [!NOTE]
> **Why not [tsdx](https://tsdx.io/docs)?** tsdx was a popular zero-config scaffold for TS component libraries, but it's been unmaintained since ~2021 (old Rollup/TS versions, no real path to Tailwind CSS extraction or shadcn-style setup). It also scaffolds its own Rollup build pipeline, which would conflict with the Vite library-mode config this guide builds in step 5. Scaffold manually and use Vite alone — it's actively maintained and everything below (Tailwind, Storybook, CSS extraction) is built around it directly.
>
> **Why not `npm create vite@latest -- --template react-ts`?** That template scaffolds a deployable _app_ (`index.html`, `public/`, `src/main.tsx` DOM bootstrap, `src/App.tsx`), not a library — none of which a component package ships. `vite.config.ts` would still need to be rewritten from scratch for library mode (`build.lib`, `rollupOptions.external`, the `dts` plugin, see step 5), so the template mostly just adds app-specific files you'd delete. Starting from `npm init -y` produces the exact folder structure below with nothing to strip out.

```bash
mkdir components && cd components
git init
npm init -y
npm pkg set name="@yourscope/components" version="0.1.0" type="module"
```

Folder structure you're building toward:

```bash
components/
├── .changeset/
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── .storybook/
├── src/
│   ├── components/
│   │   └── Button/
│   │       ├── Button.tsx
│   │       ├── Button.stories.tsx
│   │       ├── Button.test.tsx
│   │       └── index.ts
│   ├── lib/
│   │   └── utils.ts
│   ├── styles/
│   │   └── globals.css
│   └── index.ts
├── .npmrc
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── eslint.config.js
└── vitest.config.ts
```

---

## 2. Install dependencies

```bash
# Core
npm install react react-dom
npm install -D typescript @types/react @types/react-dom

# Vite + library build
npm install -D vite @vitejs/plugin-react vite-plugin-dts vite-plugin-lib-inject-css

# Styling / shadcn foundations
npm install class-variance-authority clsx tailwind-merge
npm install -D tailwindcss postcss autoprefixer tailwindcss-animate

# Radix components (shadcn wraps these — add per-component as needed)
npm install @radix-ui/react-slot

# Storybook
npx storybook@latest init --type react_vite

# Testing
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom

# Linting
npm install -D eslint typescript-eslint eslint-plugin-react-hooks eslint-plugin-storybook

# Release automation
npm install -D @changesets/cli
npx changeset init
```

`react` and `react-dom` should end up in `peerDependencies`, not `dependencies` — fixed in package.json below.

---

## 3. `package.json`

```json
{
  "name": "@yourscope/components",
  "version": "0.1.0",
  "description": "UI components for [Team] projects — React, TypeScript, shadcn/ui.",
  "license": "UNLICENSED",
  "type": "module",
  "sideEffects": ["**/*.css"],
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    },
    "./styles.css": "./dist/style.css",
    "./package.json": "./package.json"
  },
  "files": ["dist"],
  "publishConfig": {
    "access": "restricted",
    "registry": "https://registry.npmjs.org/"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/yourorg/components.git"
  },
  "peerDependencies": {
    "react": ">=18.0.0",
    "react-dom": ">=18.0.0"
  },
  "scripts": {
    "dev": "storybook dev -p 6006",
    "build": "npm run typecheck && vite build",
    "build-storybook": "storybook build -o storybook-static",
    "typecheck": "tsc -p tsconfig.build.json --noEmit",
    "lint": "eslint .",
    "test": "vitest run",
    "test:watch": "vitest",
    "changeset": "changeset",
    "version-packages": "changeset version",
    "release": "npm run build && changeset publish"
  }
}
```

Key points:

- `sideEffects` scoped to CSS so bundlers can still tree-shake your JS.
- `exports["./styles.css"]` is how consumers pull in the compiled Tailwind output: `import "@yourscope/components/styles.css"`.
- `access: restricted` is what makes a **scoped** package private on npm (default for scopes is actually already restricted, but be explicit).
- **Option B (GitHub Packages) diff:** change `publishConfig.registry` to `"https://npm.pkg.github.com"`.

---

## 4. TypeScript config

**`tsconfig.json`** (editor/dev config, includes Storybook/tests):

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "declaration": false,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src", ".storybook"]
}
```

**`tsconfig.build.json`** (what actually ships — excludes stories/tests):

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "declaration": true,
    "emitDeclarationOnly": true,
    "outDir": "dist"
  },
  "include": ["src"],
  "exclude": ["src/**/*.stories.tsx", "src/**/*.test.tsx"]
}
```

---

## 5. Vite [library-mode](https://vite.dev/guide/build#library-mode) config

**`vite.config.ts`**

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import dts from "vite-plugin-dts";
import { libInjectCss } from "vite-plugin-lib-inject-css";
import { resolve } from "path";

export default defineConfig({
  plugins: [
    react(),
    libInjectCss(),
    dts({
      include: ["src"],
      exclude: ["src/**/*.stories.tsx", "src/**/*.test.tsx"],
      rollupTypes: true,
    }),
  ],
  build: {
    lib: {
      entry: resolve(__dirname, "src/index.ts"),
      formats: ["es", "cjs"],
      fileName: (format) => `index.${format === "es" ? "mjs" : "cjs"}`,
    },
    rollupOptions: {
      external: ["react", "react-dom", "react/jsx-runtime"],
      output: {
        globals: { react: "React", "react-dom": "ReactDOM" },
      },
    },
    sourcemap: true,
    emptyOutDir: true,
    cssCodeSplit: false,
  },
});
```

This gives you `dist/index.mjs`, `dist/index.cjs`, `dist/index.d.ts`, and `dist/style.css` — everything referenced in `package.json` `exports`.

---

## 6. Tailwind + shadcn/ui foundation

**`tailwind.config.ts`**

```ts
import type { Config } from "tailwindcss";

export default {
  darkMode: ["class"],
  content: ["./src/**/*.{ts,tsx}", "./.storybook/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
} satisfies Config;
```

**`postcss.config.js`**

```js
export default {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};
```

**`src/styles/globals.css`** — run `npx shadcn@latest init` and let it write the CSS variable block here (or paste the standard shadcn `:root { --background: ...; }` theme tokens). Import this once in `src/index.ts` so `libInjectCss` bundles it into `dist/style.css`.

**`src/lib/utils.ts`** (the standard shadcn `cn` helper):

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

Add shadcn components via its CLI — it scaffolds source files into `src/components/ui`, which you then re-export as your own components:

```bash
npx shadcn@latest add button
```

**`src/components/Button/Button.tsx`** (shadcn's generated component, kept as-is or trimmed to your API):

```tsx
import * as React from "react";
import { Slot } from "@radix-ui/react-slot";
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/lib/utils";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:opacity-90",
        outline: "border border-border bg-transparent hover:bg-accent",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 px-3",
      },
    },
    defaultVariants: { variant: "default", size: "default" },
  },
);

export interface ButtonProps
  extends
    React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean;
}

export const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button";
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    );
  },
);
Button.displayName = "Button";
```

**`src/components/Button/index.ts`**

```ts
export * from "./Button";
```

**`src/index.ts`** (root barrel — this is your public API surface):

```ts
import "./styles/globals.css";

export * from "./components/Button";
// export * from "./components/..." as you add components
```

---

## 7. Storybook

`storybook init` already scaffolded `.storybook/main.ts` and `.storybook/preview.ts`. Two edits:

**`.storybook/main.ts`** — make sure it points at your stories and uses the Vite/React framework:

```ts
import type { StorybookConfig } from "@storybook/react-vite";

const config: StorybookConfig = {
  stories: ["../src/**/*.stories.@(ts|tsx)"],
  addons: ["@storybook/addon-essentials", "@storybook/addon-interactions"],
  framework: { name: "@storybook/react-vite", options: {} },
};
export default config;
```

**`.storybook/preview.ts`** — import your compiled styles so components render correctly in the docs:

```ts
import type { Preview } from "@storybook/react";
import "../src/styles/globals.css";

const preview: Preview = {
  parameters: {
    controls: { matchers: { color: /(background|color)$/i, date: /Date$/i } },
  },
};
export default preview;
```

**`src/components/Button/Button.stories.tsx`**

```tsx
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "Primitives/Button",
  component: Button,
  tags: ["autodocs"],
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Default: Story = { args: { children: "Click me" } };
export const Outline: Story = {
  args: { children: "Outline", variant: "outline" },
};
```

Run locally with `npm run dev`. This is also what CI will build as a static site — see below for publishing it so nobody has to clone the repo just to browse components.

### Publish Storybook to GitHub Pages

`npm run build-storybook` already produces a static site (`storybook-static/`). Deploying that to GitHub Pages means anyone on the team can browse components, props, and code examples straight from a URL — no local clone, no `npm install`, no `npm run dev`.

One-time repo setup: **Settings → Pages → Source: GitHub Actions**.

**`.github/workflows/storybook.yml`**

```yaml
name: Deploy Storybook to Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build-storybook
      - uses: actions/upload-pages-artifact@v3
        with:
          path: storybook-static
      - uses: actions/deploy-pages@v4
        id: deployment
```

This is deliberately a separate workflow from `ci.yml` and `release.yml` — it only needs to build and ship a static site, and keeping it decoupled means a Storybook deploy never blocks (or gets blocked by) the changesets/publish flow.

Notes:

- This is a **project page** (`https://<owner>.github.io/<repo>/`), so assets are served from a subpath. Storybook's Vite builder emits relative asset paths by default, so this works out of the box — no `base` config needed in `.storybook/main.ts`.
- If the repo is private, GitHub Pages for private repos requires **GitHub Pro/Team/Enterprise** (same tier gate as Option A's private npm scope) — public repos get Pages for free regardless of plan. If that's a blocker, Chromatic's free tier is a private-repo-friendly alternative for hosted Storybook docs.
- Runs on every push to `main`, so the deployed docs always reflect what's merged — not just tagged releases.

---

## 8. Testing (optional but recommended)

**`vitest.config.ts`**

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: { environment: "jsdom", setupFiles: "./vitest.setup.ts" },
});
```

**`vitest.setup.ts`**

```ts
import "@testing-library/jest-dom/vitest";
```

---

## 9. `.npmrc`

**Option A (npm private scope):**

```
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

**Option B (GitHub Packages):**

```
@yourscope:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

Never commit an actual token — this file just reads from an env var, injected by GitHub Actions (or your own shell locally, `export NPM_TOKEN=...`).

Generate the npm token: npmjs.com → Access Tokens → **Automation** token (bypasses 2FA prompts, safe for CI). Add it to the repo as a GitHub secret named `NPM_TOKEN` (Settings → Secrets and variables → Actions).

---

## 10. Changesets (versioning)

`.changeset/config.yml` (written by `changeset init`, tweak the default branch):

```yaml
{
  "$schema": "https://unpkg.com/@changesets/config/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "access": "restricted",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
}
```

Workflow going forward, per change:

```bash
npx changeset          # interactively pick bump type + write a summary
git add . && git commit -m "feat: add Button primitive"
git push
```

The Actions release workflow below turns accumulated changesets into a version bump + CHANGELOG + npm publish automatically.

---

## 11. GitHub Actions — CI

**`.github/workflows/ci.yml`**

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test
      - run: npm run build
      - run: npm run build-storybook
```

---

## 12. GitHub Actions — Release / Publish

**`.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    branches: [main]

concurrency: ${{ github.workflow }}-${{ github.ref }}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://registry.npmjs.org"
          cache: npm

      - run: npm ci

      - name: Create release PR or publish
        uses: changesets/action@v1
        with:
          publish: npm run release
          version: npm run version-packages
          title: "chore: version packages"
          commit: "chore: version packages"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

How this behaves:

1. You merge PRs to `main` that each include a changeset file.
2. This workflow runs, sees pending changesets, and opens/updates a **"Version Packages" PR** with the bumped `package.json`, updated `CHANGELOG.md`, and the changesets consumed.
3. When _that_ PR is merged, the workflow runs again, this time with no pending changesets — so `changesets/action` runs `npm run release` instead, which builds and `npm publish`es to the registry.

**Option B (GitHub Packages) diff:** set `registry-url: "https://npm.pkg.github.com"` and drop `NODE_AUTH_TOKEN` in favor of the default `GITHUB_TOKEN` (which already has `packages:write` if you enable it in workflow `permissions`).

---

## 13. Consuming the package from another project

**Option A consumers** need an `.npmrc` (repo-level, not committed with a real token) or a CI secret:

```
//registry.npmjs.org/:_authToken=${NPM_TOKEN}
```

Then:

```bash
npm install @yourscope/components
```

```tsx
import { Button } from "@yourscope/components";
import "@yourscope/components/styles.css";
```

**Option B consumers** need:

```
@yourscope:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

(token needs `read:packages` scope).

---

## 14. First release checklist

```bash
git remote add origin https://github.com/yourorg/components.git
git add .
git commit -m "chore: scaffold components library"
git push -u origin main

# create your first npm scope/org if you haven't:
npm org create yourscope   # or via npmjs.com UI, requires paid plan for private scoped packages

npx changeset             # describe the initial Button as a minor/patch
git add . && git commit -m "chore: initial changeset" && git push
```

Push to `main` → CI runs → Release workflow opens the "Version Packages" PR → merge it → package publishes to npm automatically.

---

## Option B: step-by-step (GitHub Packages)

Steps 1–8 (go to step [Go to Repo scaffold](#1-repo-scaffold)) and step 10 (repo scaffold, dependencies, TypeScript, Vite, Tailwind/shadcn, Storybook, testing, Changesets) are **identical** to Option A — do those first, then come back here for the registry-specific pieces: `package.json`, `.npmrc`, tokens, the release workflow, and consuming the package.

### B.1 Naming constraint (read this first)

GitHub Packages ties every npm package to a GitHub repository, and the package **scope must exactly match your GitHub username or organization name** (case-sensitive). You don't get to pick an arbitrary scope like you can on npmjs.com.

- Repo at `github.com/wozniaka83/components` → package must be published as `@wozniaka83/components`.
- Repo owned by an org `github.com/your-org/components` → package must be `@your-org/components`.

If that doesn't match, `npm publish` fails with a 404/403. Set the name accordingly in step 1:

```bash
npm pkg set name="@wozniaka83/components" version="0.1.0" type="module"
```

### B.2 `package.json`

Same file as Option A (section 3), except `publishConfig` and `repository` point at GitHub:

```json
{
  "publishConfig": {
    "access": "restricted",
    "registry": "https://npm.pkg.github.com"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/wozniaka83/components.git"
  }
}
```

Everything else in `package.json` (exports, peerDependencies, scripts) stays the same as section 3.

### B.3 `.npmrc`

For local development (publishing or installing from your own machine):

```
@wozniaka83:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

Never commit a real token — this reads from an env var. Locally: `export GITHUB_TOKEN=...` (bash) or `$env:GITHUB_TOKEN="..."` (PowerShell).

### B.4 Generate a token

Two different tokens are involved, for two different purposes:

1. **Local dev / manual publish** — a classic Personal Access Token:
   GitHub → Settings → Developer settings → Personal access tokens (classic) → scopes: `read:packages` (to install) and `write:packages` (to publish). Export it as `GITHUB_TOKEN` in your shell.
2. **CI publish (Actions)** — no PAT needed. The workflow's built-in `secrets.GITHUB_TOKEN` already has `packages:write` for the repo it runs in, as long as the workflow grants it (see B.5). This is the whole appeal of Option B: no secret to create or rotate.

### B.5 GitHub Actions — Release workflow

Same `.github/workflows/release.yml` structure as Option A (section 12), with the registry and auth swapped:

```yaml
name: Release

on:
  push:
    branches: [main]

concurrency: ${{ github.workflow }}-${{ github.ref }}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      packages: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: "https://npm.pkg.github.com"
          cache: npm

      - run: npm ci

      - name: Create release PR or publish
        uses: changesets/action@v1
        with:
          publish: npm run release
          version: npm run version-packages
          title: "chore: version packages"
          commit: "chore: version packages"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Differences from Option A: `registry-url` points at `npm.pkg.github.com`, `permissions.packages: write` replaces the npm token, and `NODE_AUTH_TOKEN` reuses the same `secrets.GITHUB_TOKEN` — no `NPM_TOKEN` secret to create.

One-time repo setting: **Settings → Actions → General → Workflow permissions → Read and write permissions**, otherwise the default `GITHUB_TOKEN` can't publish.

### B.6 Consuming the package from another project

The consumer needs an `.npmrc` (not committed with a real value):

```
@wozniaka83:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

```bash
npm install @wozniaka83/components
```

```tsx
import { Button } from "@wozniaka83/components";
import "@wozniaka83/components/styles.css";
```

Two things npm's Option A install doesn't require:

- The token needs `read:packages` scope (a classic PAT — fine-grained tokens don't yet support package reads reliably).
- Because GitHub Packages inherits your repo's visibility, whoever installs the package must also have at least read access to the **repo itself** (as a collaborator, or via org membership), not just the token.

### B.7 First release checklist (Option B)

```bash
git remote add origin https://github.com/wozniaka83/components.git
git add .
git commit -m "chore: scaffold components library"
git push -u origin main

# no npm org/scope to create — the GitHub repo/org is the scope

npx changeset             # describe the initial Button as a minor/patch
git add . && git commit -m "chore: initial changeset" && git push
```

Push to `main` → CI runs → Release workflow opens the "Version Packages" PR → merge it → package publishes to `npm.pkg.github.com` automatically, visible under the repo's **Packages** tab.

### Switching to Option A later

Nothing above locks you in. To move to the public npm registry later:

1. `package.json`: change `publishConfig.registry` back to `https://registry.npmjs.org/`.
2. Add an `NPM_TOKEN` repo secret (npmjs.com → Access Tokens → Automation) and point `release.yml`'s `registry-url` / `NODE_AUTH_TOKEN` at it (section 12).
3. Update consumers' `.npmrc` to the npm registry line (section 13).
4. `npm publish` once manually the first time if you want the version history to carry over, since GitHub Packages and npmjs.com are separate registries with separate published-version history.

---

## Summary of what each piece is for

| Piece                               | Role                                                                                      |
| ----------------------------------- | ----------------------------------------------------------------------------------------- |
| Vite (lib mode) + `vite-plugin-dts` | Bundles `src/` into ESM+CJS with generated `.d.ts` types                                  |
| `vite-plugin-lib-inject-css`        | Extracts all Tailwind/shadcn CSS into one shippable `dist/style.css`                      |
| shadcn CLI                          | Scaffolds accessible Radix-based components into your source, which you own and customize |
| Storybook                           | Local dev/preview + living documentation for your team                                    |
| Storybook on GitHub Pages           | Browsable component docs/examples with no local clone or install required                 |
| Changesets                          | Semver bumps, CHANGELOG generation, coordinated publish                                   |
| GitHub Actions (CI)                 | Lint/typecheck/test/build on every PR                                                     |
| GitHub Actions (Release)            | Automates the version-PR → merge → `npm publish` cycle                                    |
| GitHub Actions (Pages deploy)       | Rebuilds and redeploys the Storybook site on every push to `main`                         |
