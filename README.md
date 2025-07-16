
🔁 Reusing node_modules Across GitHub Actions Jobs
This workflow demonstrates how to cache and reuse node_modules between jobs in GitHub Actions without needing to reinstall dependencies every time via npm ci.

✅ Why this approach?
Avoids redundant npm ci execution in every job.

Ensures consistency of dependencies across jobs (e.g., lint, test, build).

Works around limitations of actions/upload-artifact not preserving file permissions well for binary files.

📁 Workflow Overview

name: workflow-1
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -  
 
🧱 Job 1: Install Dependencies
```yaml
  Install-Dependencies:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - run: npm ci

      - run: tar -czf node_modules.tar.gz node_modules

      - uses: actions/upload-artifact@v4
        with:
          name: npm_ci-dependencies
          path: node_modules.tar.gz
```

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

📏 Job 2: Linting

```yaml
  Linting-Stage:
    runs-on: ubuntu-latest
    needs: Install-Dependencies
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - uses: actions/download-artifact@v4
        with:
          name: npm_ci-dependencies

      - run: |
          tar -xzf node_modules.tar.gz
          rm -rf node_modules.tar.gz

      - run: echo "$(pwd)/node_modules/.bin" >> $GITHUB_PATH

      - run: npm run lint
```     
     
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

🧪 Job 3: Testing

```yaml
  Testing-Stage:
    runs-on: ubuntu-latest
    needs: Linting-Stage
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - uses: actions/download-artifact@v4
        with:
          name: npm_ci-dependencies

      - run: |
          tar -xzf node_modules.tar.gz
          rm -rf node_modules.tar.gz

      - run: echo "$(pwd)/node_modules/.bin" >> $GITHUB_PATH
      - run: npm run test
```

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

🏗️ Job 4: Build

```yaml
  Build-Stage:
    runs-on: ubuntu-latest
    needs: Testing-Stage
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - uses: actions/download-artifact@v4
        with:
          name: npm_ci-dependencies

      - run: |
          tar -xzf node_modules.tar.gz
          rm -rf node_modules.tar.gz

      - run: echo "$(pwd)/node_modules/.bin" >> $GITHUB_PATH
      - run: npm run build
```

🧠 Notes
-> We tar the node_modules directory after npm ci to preserve file structure and permissions.

-> actions/upload-artifact and download-artifact are used to pass the .tar.gz file between jobs.

-> Each consuming job must extract node_modules.tar.gz and ensure node_modules/.bin is in the PATH.



# 🧪 GitHub Actions: Reuse `node_modules` Between Jobs

This note documents the working strategy (and the failed ones) to **reuse `node_modules` between multiple jobs** in a GitHub Actions workflow using `npm ci`, so you don’t have to reinstall dependencies in each job.

---

## ✅ What Worked (Success)

### Strategy: Use `tar` to archive `node_modules`

1. In the `Install-Dependencies` job:

   * Run `npm ci` to install dependencies.
   * Archive the `node_modules` folder as `node_modules.tar.gz`.
   * Upload the `.tar.gz` file as an artifact.

2. In downstream jobs (`Linting`, `Testing`, `Build`):

   * Download the artifact.
   * Extract the `node_modules.tar.gz`.
   * Add `node_modules/.bin` to the `PATH` to run tools like `eslint`, `vitest`, etc.

### Key Reasons This Worked

* Preserved `node_modules` structure + permissions.
* Ensured all scripts like ESLint are found under `.bin`.
* One-time installation of dependencies.

---

## ❌ What Failed (Pitfalls)

### ❌ Uploading `node_modules/` directly with `actions/upload-artifact`

* **Issue**: Uploading the directory directly caused missing `.bin` executables (e.g. `eslint not found`).
* **Why**: `upload-artifact` does not preserve file permissions or symlinks reliably.

### ❌ Downloading without extracting (no `.tar.gz`)

* **Issue**: Could not locate expected structure or paths.
* **Fix**: Wrapping `node_modules` in a `.tar.gz` solves this.

### ❌ Setting `path: ./node_modules/` during upload

* Same issue: artifact corrupted / incomplete on download.

---

## ✅ Working Workflow Snippet (Copy-Paste Ready)

### `Install-Dependencies` Job

```yaml
tar -czf node_modules.tar.gz node_modules
```

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: npm_ci-dependencies
    path: node_modules.tar.gz
```

### In Dependent Jobs (e.g. `Linting`, `Testing`, `Build`):

```yaml
- uses: actions/download-artifact@v4
  with:
    name: npm_ci-dependencies

- run: |
    tar -xzf node_modules.tar.gz
    rm -rf node_modules.tar.gz

- run: echo "$(pwd)/node_modules/.bin" >> $GITHUB_PATH
```

---

## 📁 Structure

```
node_modules.tar.gz  <--- Uploaded & downloaded across jobs
node_modules/
├── .bin/
│   └── eslint (and other executables)
├── react/
├── vite/
└── ...
```

---

## 🔄 Benefits

* Speeds up CI: `npm ci` only once.
* Prevents dependency drift between jobs.
* Keeps workflows DRY and efficient.

---

## 🧠 Pro Tips

* Always use `.tar.gz` to preserve executable file integrity.
* Add `node_modules/.bin` to `PATH` in every job that runs npm scripts.
* Validate with `ls -l node_modules/.bin/` if in doubt.

---

✅ Use this approach any time you have multiple CI stages that all depend on the same `node_modules`.

---
