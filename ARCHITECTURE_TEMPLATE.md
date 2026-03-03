# Architecture & Infrastructure Template

Use this document to replicate this project's architecture and infrastructure in a new, empty project. Replace the domain and content with your own; the structure, tooling, and conventions stay the same.

---

## Quick reference

| Area | Create / support |
|------|------------------|
| **Dirs** | `src/` (components, services, data, utils), `tests/` (e2e, regression, integration, fixtures, e2e/helpers), `public/`, `conf/`, optional `scripts/`, `docs/` |
| **Root config** | `package.json`, `vite.config.js`, `vitest.config.js`, `eslint.config.js`, `playwright.config.js`, `vercel.json`, `.env.example`, `index.html` |
| **SPA deploy** | `public/_redirects` (Netlify), `public/.htaccess` (Apache), `vercel.json` rewrites → `index.html` |
| **npm scripts** | `dev`, `build`, `preview`, `lint`; `test`, `test:unit`, `test:regression`, `test:integration`, `test:e2e`, `test:e2e:fast`, `test:all`; optional per-area `test:projection`, `test:tax`, etc. |
| **Testing layers** | Unit (Vitest, next to code in `__tests__/`), regression, integration, e2e (Playwright); `tests/setup.js`, `tests/e2e/global-setup.js`; fixtures and helpers under `tests/` |
| **Selectors** | Prefer `data-testid`; avoid text or CSS class selectors in e2e and component tests |
| **Cursor rules** | Run allowlisted commands (test, build, lint) without asking; test-as-part-of-development (reproduce with test, run suites after changes, add tests for new behavior) |

---

## 1. Directory layout

```
your-project/
├── src/
│   ├── components/       # React UI components; group by feature or route (e.g. Discussion/, CountryOptions/)
│   ├── services/         # Business logic, API clients, engines; colocate __tests__/ next to source
│   ├── data/             # Static JSON/config consumed by the app (not user data)
│   ├── utils/            # Pure helpers (formatting, validation)
│   ├── config/           # App-level config modules (e.g. feature flags, env-based settings)
│   ├── theme.js          # UI theme (e.g. Chakra extendTheme)
│   ├── i18n/             # i18n config and resources
│   ├── App.jsx           # Root component and route definitions
│   ├── main.jsx          # Entry: createRoot, providers, router
│   └── index.css         # Global styles
├── tests/
│   ├── setup.js          # Vitest global setup (e.g. localStorage mock, NODE_ENV)
│   ├── fixtures/         # Shared test data and helpers for unit/regression/integration
│   ├── regression/       # Regression / golden tests
│   ├── integration/      # Cross-service or cross-layer tests
│   └── e2e/
│       ├── global-setup.js  # Playwright: ensure browser available, env checks
│       ├── helpers/          # E2E helpers (selectors, flows, assertions)
│       └── *.spec.js        # Playwright specs
├── public/               # Static assets; copied as-is; SPA redirect/rewrite files
├── conf/                  # Non-code config (prompts, rates, INI); optional
├── scripts/               # Optional: pipeline/orchestration (e.g. Python); README describes stages
├── docs/                  # Project docs and this template
├── package.json
├── vite.config.js
├── vitest.config.js
├── eslint.config.js
├── playwright.config.js
├── vercel.json            # Or equivalent for your host
├── .env.example
└── index.html
```

- **src/components**: One place for all UI. Use subfolders for major areas (e.g. a hub, list/detail pages, shared widgets).
- **src/services**: Keep logic out of components. Services can depend on `src/data` and `src/config`. Put unit tests in `src/services/__tests__/` next to the file under test.
- **src/data**: Static JSON or JS that the app loads (e.g. question flows, reference tables). No secrets; use env or `conf/` for environment-specific values.
- **public**: Favicon, `_redirects`, `.htaccess`, and any static JSON or HTML that the app fetches at runtime (e.g. list indexes, per-item payloads). Optional: self-contained content folders (e.g. one folder per “optimization” or article with `content.html` and images).
- **conf**: Optional. Holds prompts, rate tables, or INI-style config that scripts or the app may load. Keep `.env` for secrets and env-specific overrides; document vars in `.env.example`.

---

## 2. Testing approach

### Layers

- **Unit**: Vitest; tests live in `src/**/__tests__/*.test.js` or next to the module. Fast, isolated; mock external deps. Use a single `tests/setup.js` for global mocks (e.g. `localStorage`) and `process.env.NODE_ENV = 'test'`.
- **Regression**: Vitest; tests in `tests/regression/*.test.js`. Guard against regressions (e.g. golden outputs, key metrics). Use `tests/fixtures/` for shared inputs and expected data.
- **Integration**: Vitest; tests in `tests/integration/*.test.js`. Exercise interactions between services or layers (e.g. data flow from one engine to another).
- **E2E**: Playwright; specs in `tests/e2e/*.spec.js`. Run against the real app (dev server). Use `tests/e2e/helpers/` for reusable flows, selectors, and assertions.

### Conventions

- **Stable selectors**: Prefer `data-testid` (or another stable attribute) for any element that tests interact with or assert on. Add the attribute when introducing UI that will be covered by e2e or component tests. Avoid selecting by visible text or CSS class so tests don’t break when copy or styles change.
- **Vitest config**: Use `vitest.config.js` (or equivalent in Vite) with `environment: 'jsdom'`, `setupFiles: ['./tests/setup.js']`, and `include` covering `src/**/*.test.js`, `tests/regression/**/*.test.js`, `tests/integration/**/*.test.js`. Exclude e2e from Vitest.
- **Playwright config**: Set `testDir` to `tests/e2e`, `baseURL` to your dev server (e.g. `http://localhost:5173`), and `webServer` to start the dev server (e.g. `npm run dev`) with `reuseExistingServer: true`. Use a single browser project (e.g. Chrome) or allow override via env (e.g. `PLAYWRIGHT_CHROME_EXECUTABLE_PATH`). Use `globalSetup` to fail fast if the browser is missing or env is wrong.

### npm scripts

- `test` / `test:unit`: Run all Vitest tests (unit + regression + integration).
- `test:unit:watch`: Vitest in watch mode.
- `test:coverage`: Vitest with coverage (e.g. v8, output text/json/html).
- `test:regression`: Run only `tests/regression`.
- `test:integration`: Run only `tests/integration`.
- Optional per-area scripts: e.g. `test:projection`, `test:tax` that run a single `src/services/__tests__/*.test.js` file for fast feedback.
- `test:e2e`: Full Playwright run.
- `test:e2e:fast`: Playwright excluding slow or optional specs (e.g. with `--grep-invert`).
- `test:e2e:ui` / `test:e2e:headed` / `test:e2e:debug`: Interactive and headed modes.
- `test:all`: Run unit + regression + integration + e2e (e.g. before release or in CI).

### Workflow

- After every code change: run the relevant suite (unit, regression, integration, or e2e) before considering work done.
- For bugfixes: add a failing test that reproduces the bug, fix, then re-run the suite.
- For features: add unit or integration tests for the behavior; add or extend e2e tests when the flow is user-facing (e.g. a new UI option that must persist or affect results).
- For UI or config changes that affect e2e: update or add helpers in `tests/e2e/helpers/` and at least one e2e spec.

---

## 3. Styling approach

- **UI library**: Use a component library (e.g. Chakra UI) for layout, forms, and typography so styling is consistent and accessible.
- **Theme**: Define a single theme module (e.g. `src/theme.js`) that extends the library’s default theme with your brand colors, fonts, and global styles (e.g. body background and color). Export it and wrap the app in the library’s theme provider in the root entry (e.g. `main.jsx`).
- **Global CSS**: Use one global CSS file (e.g. `src/index.css`) for reset, box-sizing, body margin/font, and `#root` min-height. Keep component-specific overrides in the library’s system or in scoped styles rather than global classes.
- **Where it’s applied**: The theme provider and the global CSS import are applied at the root so all routes and components inherit the same look and tokens.

---

## 4. Build and tooling

- **Bundler**: Vite with the React plugin. Root HTML file has a single script entry (e.g. `/src/main.jsx`). Build output goes to `dist/` (or your configured folder).
- **ESLint**: Use the flat config. Apply to `**/*.{js,jsx}`; ignore `dist`. Use `@eslint/js`, `globals` (browser), and React-related plugins (e.g. `eslint-plugin-react-hooks`, `eslint-plugin-react-refresh`). Enable recommended and React-refresh rules so only components are exported from JSX files where appropriate.
- **Playwright**: One config file; `testDir` points to `tests/e2e`. Configure `webServer` to run the dev server so e2e runs against a real build. Use a single browser project for simplicity; allow CI to override (e.g. headed vs headless, executable path). Set a short default timeout (e.g. 3s) and override in individual tests when needed.
- **Scripts**: Provide `dev`, `build`, `preview`, `lint`, and the test scripts above. Optionally add scripts that generate fixtures or golden data (e.g. `generate:golden`) and corresponding test scripts (e.g. `test:component-golden`).

---

## 5. SPA and deploy

- **Client-side routing**: The app is a SPA. All routes are handled by the router; direct links and refresh must serve the same root document so the router can load.
- **Fallback to index.html**: Configure the host so that for any path that does not match a static file, the server returns `index.html`. Common patterns:
  - **Netlify**: `public/_redirects` with a line like `/*    /index.html   200`.
  - **Vercel**: `vercel.json` with `rewrites: [{ "source": "/(.*)", "destination": "/index.html" }]`.
  - **Apache**: `public/.htaccess` with mod_rewrite: serve existing files and directories as-is; otherwise rewrite to `index.html`.
- **Public folder**: Put the redirect/rewrite files in `public/` so they are copied into the build output. Also place here favicon, and any static JSON or HTML the app fetches at runtime (e.g. list index, per-item payloads). Self-contained content (e.g. one folder per “article” with `content.html` and images) can live under `public/` and be loaded by route or ID; the app resolves asset paths relative to that folder.

---

## 6. Config and env

- **Where config lives**: Use a `conf/` directory for non-code config (prompts, rate tables, INI) if your app or scripts need it. Use `src/config/` for in-app config modules that read env or load from `conf/`.
- **Env**: Document all env vars (including those prefixed with your bundler’s public prefix, e.g. `VITE_*`) in `.env.example`. No secrets or real values; just variable names and short comments. The app and scripts read env at runtime; the bundler inlines only the vars it is configured to expose (e.g. `import.meta.env.VITE_*`).
- **Secrets**: Keep secrets in `.env` (gitignored). Do not commit them or put them in the template; the template describes the pattern (e.g. “document in `.env.example`; load in app or scripts”).

---

## 7. Development workflow (Cursor rules)

Adopt two high-level rules so agents and humans behave consistently:

**1. Run allowed commands directly**  
When the task requires running allowlisted commands (e.g. `npm run test`, `npm run test:projection`, `npm run build`, `npm run lint`), run them directly and report the result. Do not ask the user to confirm or to run the command themselves.

**2. Testing as part of development**  
- **Bugfixes**: Reproduce with a failing test first, then fix, then run the relevant suite to verify. Do not change production code to “fix” a bug until a test reproduces it.
- **After every change**: Run the relevant test suite (unit, regression, integration, or e2e) before considering work done. Add or extend tests so the new behavior or fix is covered (regression test for bugfixes; unit/integration for features; e2e when the flow is user-facing).
- **Where to add tests**: Unit tests next to the code (e.g. `src/services/__tests__/`); regression and integration under `tests/regression/` and `tests/integration/`; e2e under `tests/e2e/` using helpers in `tests/e2e/helpers/`. When adding new UI or options that e2e will use, add or extend the helpers and at least one e2e spec.
- **Stable selectors**: Use `data-testid` (or similar) for elements that tests interact with or assert on; avoid selecting by visible text or CSS class.
- **Checklist before done**: Relevant unit/regression/integration tests run and pass; new behavior or bug has a test that would catch a regression; for UI/config changes, e2e helper and at least one e2e test updated or added if the flow is user-critical.

---

## 8. Optional: scripts and pipeline pattern

If the project has a multi-step pipeline (e.g. generate data, transform, then publish to `public/`):

- Put pipeline scripts in a `scripts/` directory. Use a `scripts/README.md` to describe the stages, inputs, outputs, and order of execution. Do not encode domain-specific logic in the template; only the pattern (e.g. “Stage 1 produces X, Stage 2 reads X and writes Y, Stage 3 writes to `public/`”).
- Use a single config file or env for the pipeline (e.g. INI or `.env`) and document it. Paths in the README should be generic (e.g. “input dir”, “output dir”) so another project can substitute its own.
- The app should depend only on the final output (e.g. files under `public/`), not on the presence of the scripts. The template document does not need to describe the pipeline’s internals, only that optional orchestration can live in `scripts/` with a README.
