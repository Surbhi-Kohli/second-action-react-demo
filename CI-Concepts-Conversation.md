# CI Concepts - Full Conversation Log

---

## Q1: Teach me CI basic concepts

### Answer:

**Reference Workflow:** `.github/workflows/4.artifacts-outputs.yml`

```yaml
name: Deploy website
on:
  push:
    branches:
      - main
jobs:
  test: # job1
    runs-on: ubuntu-latest
    steps:
      - name: Get code
        uses: actions/checkout@v3
      - name: Install dependencies
        run: npm ci
      # - name: Lint code
      #   run: npm run lint
      - name: Test code
        run: npm run test
  build: # job2
    needs: test
    runs-on: ubuntu-latest
    outputs: 
      script-file: ${{steps.publish.outputs.my-script-file}}
    steps:
      - name: Get code
        uses: actions/checkout@v3
      - name: Install dependencies
        run: npm ci
      - name: Build website
        run: npm run build
      - name: Upload Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist-files
          path: |
                dist
                package.json 
      - name: Publish JS files
        id: publish
        run: find dist/assets/*.js -type f -execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'         
  deploy: # job3
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Get build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-files
      - name: Output contents
        run: ls    
      - name: Output filename
        run: echo "${{needs.build.outputs.script-file }}"
      - name: Deploy
        run: echo "Deploying..."
```

### 1. What is CI?
Continuous Integration is the practice of automatically building, testing, and validating code changes whenever developers push code to a repository. It catches bugs early and ensures code quality.

### 2. Triggers (Events)
```yaml
on:
  push:
    branches:
      - main
```
- **What**: Defines when your workflow runs
- **Your example**: Runs on every push to the `main` branch
- **Common triggers**: `push`, `pull_request`, `schedule`, `workflow_dispatch`

### 3. Jobs
Your workflow has 3 jobs: `test`, `build`, `deploy`
- **What**: Independent units of work that run in parallel by default
- **Runners**: Each job runs on a fresh virtual machine (`runs-on: ubuntu-latest`)
- **Dependencies**: Use `needs` to create sequential workflows

```yaml
build:
  needs: test  # Waits for 'test' job to complete
```

### 4. Steps
The individual commands within a job:
- **Actions**: Reusable components (`uses: actions/checkout@v3`)
- **Commands**: Shell commands (`run: npm ci`)
- **Sequential**: Steps run in order, top to bottom

### 5. Key CI Patterns in the Workflow

#### Checkout Code
```yaml
- uses: actions/checkout@v3
```
Downloads your repository code to the runner.

#### Install Dependencies
```yaml
- run: npm ci
```
`npm ci` (clean install) ensures reproducible builds from `package-lock.json`.

#### Test
```yaml
- run: npm run test
```
Automated testing catches bugs before deployment.

#### Build
```yaml
- run: npm run build
```
Compiles/bundles your application.

### 6. Artifacts
Persist files between jobs:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist
```
- **Purpose**: Share build outputs between jobs
- **Your use case**: `build` job creates `dist` folder, `deploy` job downloads it

### 7. Outputs
Pass data between jobs:
```yaml
outputs: 
  script-file: ${{steps.publish.outputs.my-script-file}}
```
- **Job-level outputs**: Make step outputs available to other jobs
- **Access**: `${{needs.build.outputs.script-file}}`

### 8. CI Benefits Used in the Workflow
- Automated testing: Code is tested on every push
- Build validation: Ensures code compiles successfully
- Consistent environment: Same Ubuntu environment every time
- Fast feedback: Developers know immediately if something breaks
- Deployment pipeline: Automated path from code -> test -> build -> deploy

### 9. Best Practices in the Workflow
1. **Job dependencies**: Test before build, build before deploy
2. **Fail fast**: If tests fail, build/deploy never run
3. **Clean installs**: `npm ci` ensures reproducible builds
4. **Artifact management**: Build once, deploy the same artifact

---

## Q2: I am trying to learn basic CI patterns and concepts as a beginner that I must know

### Answer:

### Core CI Concepts

#### 1. The CI Pipeline
A series of automated stages that code goes through:
```
Code Push -> Test -> Build -> Deploy
```
**Why it matters**: Catches issues early, automates repetitive tasks, ensures quality.

#### 2. Version Control Integration
CI systems watch your repository and trigger on events:
- **Push**: Code merged to a branch
- **Pull Request**: Before merging changes
- **Schedule**: Run tests nightly
- **Manual**: Trigger deployments on-demand

#### 3. Build Automation
Every code change should build automatically:
```yaml
- run: npm ci          # Install dependencies
- run: npm run build   # Compile/bundle code
```
**Key principle**: "If it doesn't build automatically, it's broken"

#### 4. Automated Testing
Tests run on every change without human intervention:
```yaml
- run: npm run test    # Unit tests
- run: npm run lint    # Code quality checks
```

**Test pyramid** (from most to least):
- **Unit tests**: Test individual functions
- **Integration tests**: Test components together
- **E2E tests**: Test full user workflows

#### 5. Fail Fast
Stop the pipeline immediately when something breaks:
```yaml
test:
  runs-on: ubuntu-latest
build:
  needs: test  # Won't run if test fails
deploy:
  needs: build # Won't run if build fails
```

### Essential CI Patterns

#### Pattern 1: Clean Environment
Every build starts fresh:
- New virtual machine
- No leftover files from previous runs
- Reproducible results
```yaml
runs-on: ubuntu-latest  # Fresh Ubuntu VM
```

#### Pattern 2: Dependency Management
Lock dependencies for consistency:
```yaml
- run: npm ci  # Uses package-lock.json (exact versions)
```
**Not**: `npm install` (can install different versions)

#### Pattern 3: Artifact Sharing
Build once, use everywhere:
```yaml
# Job 1: Build
- uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist

# Job 2: Deploy
- uses: actions/download-artifact@v4
  with:
    name: dist-files
```
**Why**: Ensures you deploy exactly what you tested.

#### Pattern 4: Job Dependencies
Control execution order:
```yaml
jobs:
  test:
    # runs first
  build:
    needs: test  # waits for test
  deploy:
    needs: build # waits for build
```

#### Pattern 5: Environment Isolation
Separate environments for different stages:
- **Development**: Frequent deployments, latest code
- **Staging**: Pre-production testing
- **Production**: Stable, tested releases

#### Pattern 6: Idempotency
Running the same build twice produces the same result:
- Fixed dependency versions
- No random data in tests
- Deterministic builds

### Key Terminology
- **Runner**: The machine that executes your jobs
- **Workflow**: The entire CI/CD configuration file
- **Job**: A group of steps that run on the same runner
- **Step**: A single task (command or action)
- **Action**: Reusable code package (like `checkout@v3`)
- **Artifact**: Files produced by a job (build outputs, test reports)

### CI Best Practices

**DO:**
- Keep builds fast (<10 minutes ideal)
- Test before deploying
- Use caching for dependencies
- Make builds reproducible
- Log everything for debugging
- Run CI on every commit

**DON'T:**
- Skip tests to save time
- Hardcode secrets in workflows
- Ignore failing tests
- Deploy without testing
- Use `npm install` instead of `npm ci`
- Make builds dependent on external state

### Common CI Workflow Structure
```yaml
name: CI Pipeline

on: [push, pull_request]

jobs:
  # Stage 1: Quality Checks
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run lint
  
  # Stage 2: Testing
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run test
  
  # Stage 3: Build
  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist
  
  # Stage 4: Deploy
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-output
      - run: echo "Deploy to production"
```

### Next Steps to Learn
1. **Caching**: Speed up builds by caching `node_modules`
2. **Matrix builds**: Test across multiple Node versions
3. **Environment variables**: Configure different environments
4. **Secrets management**: Secure API keys and credentials
5. **Status checks**: Require CI to pass before merging PRs

---

## Q3: What is the blob storage being mentioned here?

### Context (from GitHub Actions log output):
```
Multiple search paths detected. Calculating the least common ancestor of all paths
The least common ancestor is /home/runner/work/second-action-react-demo/second-action-react-demo. This will be the root directory of the artifact
With the provided path, there will be 6 files uploaded
Artifact name is valid!
Root directory input is valid!
Beginning upload of artifact content to blob storage
Uploaded bytes 57502
Finished uploading artifact content to blob storage!
SHA256 digest of uploaded artifact zip is e0e79c5668d3003b844061a2f63d9e50f4b27b2bead110ba6ce4c8b5f6707458
Finalizing artifact upload
Artifact dist-files.zip successfully finalized. Artifact ID 5329470604
Artifact dist-files has been successfully uploaded! Final size is 57502 bytes. Artifact ID is 5329470604
```

### Answer:
"Blob storage" here refers to GitHub Actions' internal object storage service where artifacts are uploaded and stored as blobs (binary large objects). The `actions/upload-artifact` action zips your files and uploads them to GitHub's backend storage (not your repo). It's essentially the temporary storage GitHub uses to keep artifacts so later jobs can download them.

---

## Q4: Where artifacts live, retention limits, and how to download them

### Answer:

### 1) Where artifacts live
Artifacts are stored in **GitHub's internal blob storage** (object storage), not in your repo.
They're scoped to the **workflow run**, and associated with your repository.

You can access them from:
- **Actions tab** -> select workflow run -> **Artifacts** section
- Or via the **GitHub API** (if you need programmatic access)

### 2) Retention limits
By default:
- **90 days** retention for GitHub Actions artifacts (can be changed)
- Configurable at repo/org level or per workflow using `retention-days`

Example:
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: dist
    retention-days: 7
```

### 3) Downloading artifacts

**In UI:**
Actions -> Workflow run -> Artifacts -> Download

**In another job:**
```yaml
- uses: actions/download-artifact@v4
  with:
    name: dist-files
```

**Locally via CLI:**
```bash
gh run download <run-id> -n dist-files
```

---

## Q5: For the step `Upload Artifacts`, does this download or upload artifact, and how do I download it?

### Context:
```yaml
- name: Upload Artifacts
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: |
          dist
          package.json
```

### Answer:
That step **uploads** an artifact.
It uses `actions/upload-artifact@v4`, which zips and uploads the files in `dist` and `package.json` to GitHub's artifact storage.

**How to download it:**

**Inside a later job** (like the `deploy` job):
```yaml
- name: Get build Artifacts
  uses: actions/download-artifact@v4
  with:
    name: dist-files
```

**From the GitHub UI:**
Actions -> open the workflow run -> Artifacts section -> download.

**Via CLI:**
```bash
gh run download <run-id> -n dist-files
```

---

## Q6: What is the `path` here in the Upload Artifacts step?

### Context:
```yaml
- name: Upload Artifacts
  uses: actions/upload-artifact@v4
  with:
    name: dist-files
    path: |
          dist
          package.json
```

### Answer:
`path` is the file/folder location **on the runner**, relative to the repo root (`$GITHUB_WORKSPACE`).

In this example:
- `dist` — the dist directory at the repo root
- `package.json` — the package.json file at the repo root

These are the files/folders that will be **uploaded as the artifact**.
You can also use absolute paths or glob patterns (e.g., `dist/**/*.js`).

---

## Q7: How does `ls` give the path of artifacts?

### Context:
```yaml
deploy: # job3
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Get build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: dist-files
      - name: Output contents
        run: ls
```

### Answer:
`ls` doesn't "find" the artifact path by itself. It just lists **the current working directory**.

By default, `actions/download-artifact@v4` extracts artifacts into the **workspace root** (`$GITHUB_WORKSPACE`). So when you run:
```yaml
- run: ls
```
it shows whatever was extracted into that directory (e.g., `dist/`, `package.json`).

**Want a specific folder?**
You can set a target path:
```yaml
- uses: actions/download-artifact@v4
  with:
    name: dist-files
    path: downloaded-artifacts
```
Then `ls downloaded-artifacts` will show the artifact contents.

---

## Q8: Does this action download to the job's runner machine or on GitHub's storage?

### Answer:
It downloads **to the job's runner machine** (the VM running that job).
GitHub's storage is only the **source** where the artifact is stored.
`actions/download-artifact@v4` pulls it **from GitHub storage onto the runner** so your subsequent steps can use the files.

---

## Q9: So the `path` here is the path within the runner?

### Answer:
Yes — in `actions/upload-artifact@v4`, the `path` is **a path on the runner's filesystem** (relative to the repo workspace).

Example:
```yaml
path: |
  dist
  package.json
```
Those are expected to exist **inside the runner** at `$GITHUB_WORKSPACE/dist` and `$GITHUB_WORKSPACE/package.json`.

---

## Q10: When I specify path as `download-artifacts`, will it be in `$GITHUB_WORKSPACE/download-artifacts`?

### Answer:
Yes — if you set `path: download-artifacts` in `actions/download-artifact@v4`, it will download to:
```
$GITHUB_WORKSPACE/download-artifacts
```
because relative paths are resolved from the workspace.
(If you provide an absolute path, it uses that directly.)

---

## Q11: What is `$GITHUB_WORKSPACE`?

### Answer:
`$GITHUB_WORKSPACE` is an environment variable set by GitHub Actions that points to the **working directory on the runner** where your repository is checked out.

So:
- Your repo files live under `$GITHUB_WORKSPACE`
- Relative paths in steps (like `dist/`) are resolved from there

Example on a hosted runner:
```
/home/runner/work/<repo>/<repo>
```

---

## Q12: I am unable to understand the part where we get output from the build job to access it in the deploy job

### Context:
The `outputs` tag, the `find dist/...` run command, and how it's accessible to the deploy job.

### Answer:

**Goal:** Grab a filename found in the `build` job and make it available in the `deploy` job.

### Step 1: Step creates an output
```yaml
- name: Publish JS files
  id: publish
  run: find dist/assets/*.js -type f -execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'
```
- This step **finds a JS file**
- It **writes a key-value output** to `$GITHUB_OUTPUT`
- The key is `my-script-file`

### Step 2: Job exposes that output
```yaml
outputs:
  script-file: ${{ steps.publish.outputs.my-script-file }}
```
- Job output named `script-file`
- It maps to the step's output `my-script-file`

### Step 3: Deploy job reads it
```yaml
run: echo "${{ needs.build.outputs.script-file }}"
```
- `deploy` depends on `build`
- So it can access `build` outputs via `needs.build.outputs`

### Easy way to remember (3-step rule)

**Mnemonic: _Step -> Job -> Needs_**

1. **Step creates output**
   (`echo "name=value" >> $GITHUB_OUTPUT`)
2. **Job exports output**
   (`outputs: jobKey: ${{ steps.stepId.outputs.name }}`)
3. **Next job reads output**
   (`${{ needs.jobId.outputs.jobKey }}`)

### Translation table:

| Purpose | YAML piece | Example |
|--------|------------|---------|
| Step output | `$GITHUB_OUTPUT` | `my-script-file=...` |
| Job output | `outputs:` | `script-file: ${{ steps.publish.outputs.my-script-file }}` |
| Downstream use | `needs.<job>.outputs` | `${{ needs.build.outputs.script-file }}` |

### In plain words (very simple):
- In `build`, you **find a file** and **save its name**
- You **expose that saved name** as a job output
- In `deploy`, you **read the saved name**

---

## Q13: Explain `find dist/assets/*.js -type f -execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'` in an easy to remember way

### Answer:

**Plain English:** "Find JS files in `dist/assets` and **save the filename as an output** named `my-script-file`."

### Breaking it down (tiny pieces):

- `find dist/assets/*.js` -> look for all `.js` files in that folder
- `-type f` -> only files (not folders)
- `-execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'` -> for each file found, write `my-script-file=<that file name>` into `$GITHUB_OUTPUT`

### Easy way to remember (3 words):
**Find -> Format -> Output**

1. **Find** the file
2. **Format** as `key=value`
3. **Output** to `$GITHUB_OUTPUT`

---

## Q14: Why this iteration behavior of "for each file, create a key value pair"? Seems like the key would be same for all. Why this?

### Answer:
The key **is the same for every file**, and that means **only the last one wins**.

When you write multiple lines like:
```
my-script-file=file1.js
my-script-file=file2.js
```
GitHub Actions outputs will keep **the last value** (`file2.js`). So yes — this command iterates, but **it doesn't store all values**.

### Why do this then?
Sometimes people:
- expect only **one** file to match, or
- just want **the last file** in the list.

### If you want all files:
You need to combine them into one output value, for example:
```bash
echo "my-script-files=$(ls dist/assets/*.js | tr '\n' ' ')" >> $GITHUB_OUTPUT
```

So the behavior is:
- iterate: yes
- accumulate: no (unless you build the list yourself)

---

## Q15: Does `-execdir` do the iterations?

### Answer:
No — **`find` does the iteration**, not `-execdir`.

- `find ...` walks through matching files **one by one**
- `-execdir ...` just says **"for each file found, run this command in that file's directory"**

So the loop comes from `find`, and `-execdir` is the action run per match.

---

## Q16: What if the last file is not the one we want? How and why this "last file" logic was taken?

### Answer:
It's **not a deliberate "last file" logic** — it's just a side-effect of how outputs work.
Each `echo my-script-file=... >> $GITHUB_OUTPUT` **overwrites the previous value**, so the last one written becomes the final output.

### If the last file isn't the one you want, you have to choose explicitly:

**Option 1: pick the first match**
```bash
echo "my-script-file=$(find dist/assets -name '*.js' -type f | head -n 1)" >> $GITHUB_OUTPUT
```

**Option 2: pick a specific file pattern**
```bash
echo "my-script-file=$(find dist/assets -name 'main*.js' -type f | head -n 1)" >> $GITHUB_OUTPUT
```

**Option 3: sort and pick newest**
```bash
echo "my-script-file=$(ls -t dist/assets/*.js | head -n 1)" >> $GITHUB_OUTPUT
```

So: **the "last file" isn't chosen on purpose** — it's just the last value written. If you need a specific one, you must filter or sort.

---

## Q17: That means `my-script-file=file1.js` and `my-script-file=file2.js` does not happen at a time — it stores only one value?

### Answer:
Correct — **only one value is stored** for a given output key.
Even if you write:
```
my-script-file=file1.js
my-script-file=file2.js
```
the output **ends up as just** `file2.js` (last write wins).
So **multiple values aren't preserved** unless you combine them into one string yourself.

---

## Q18: What is `needs` in `needs.build.outputs` and `steps` in `steps.publish.outputs`?

### Answer:

### `needs.build.outputs`
Inside `deploy`, `needs` lets you access outputs from **other jobs** you depend on.

```yaml
needs: build
```

So you can do:
```yaml
${{ needs.build.outputs.script-file }}
```
Meaning: "get the `script-file` output from the `build` job."

### `steps.publish.outputs`
Inside `build`, `steps` lets you access outputs from **steps in the same job**.

```yaml
- id: publish
  run: ...
```

Then:
```yaml
${{ steps.publish.outputs.my-script-file }}
```
Meaning: "get the output `my-script-file` from the step with id `publish`."

### Easy way to remember:
- **`steps` = same job**
- **`needs` = other job (dependency)**

---

## Q19: Are these context objects?

### Answer:
Yes — in GitHub Actions terms, `needs` and `steps` are **context objects** used inside expressions like `${{ ... }}`.

- `steps` = context for steps in **the same job**
- `needs` = context for **dependent jobs**

Docs call them **"contexts"** (e.g., `github`, `env`, `steps`, `needs`, etc.).

---

## Q20: Is `find` a Linux command or specific to GitHub Actions?

### Answer:
`find` is a **standard Linux/Unix command**, not specific to GitHub Actions.
It works the same way on any Linux machine; GitHub Actions just runs it on the Ubuntu runner.

---

## Q21: Does the syntax `find dist/assets/*.js -type f -execdir echo '::set-output name=script-file::{}'';'` work?

### Answer:
No — that's **outdated syntax** that was deprecated.

`::set-output` was the old way (before 2022), but GitHub Actions now requires:
```bash
echo "script-file=value" >> $GITHUB_OUTPUT
```

The `::set-output` syntax **will not work** in current runners and should be avoided.

---

## Q22: But if it's a Linux-specific thing then why is it outdated for GitHub Actions?

### Answer:
Two different things were being mixed up:

1. **`find`** = Linux command (still works fine)
2. **`::set-output`** = GitHub Actions' **own syntax** for setting outputs (not a Linux thing)

GitHub Actions deprecated **their own `::set-output` syntax** and replaced it with `$GITHUB_OUTPUT` for security reasons. The Linux `find` command itself didn't change — just how GitHub Actions handles outputs.

So:
- `find` command works the same
- `::set-output` was GitHub's old way of capturing output
- `>> $GITHUB_OUTPUT` is GitHub's new way

---

## Q23: Explain the Cache dependencies step

### Context:
```yaml
- name: Cache dependencies
  uses: actions/cache@v5
  with:
    path: ~/.npm
    key: deps-node-modules-${{hashFiles('**/package-lock.json')}}
```

### Answer:
This step caches npm dependencies to **speed up builds**.

**`path: ~/.npm`**
Caches npm's cache directory (where npm stores downloaded packages).

**`key: deps-node-modules-${{hashFiles('**/package-lock.json')}}`**
Creates a unique cache key based on:
- Static prefix: `deps-node-modules-`
- Dynamic suffix: hash of `package-lock.json` content

### How it works:

1. **First run**: No cache exists -> downloads all dependencies -> saves cache
2. **Subsequent runs**: Same `package-lock.json` -> cache hit -> skips downloads
3. **Dependencies change**: Different `package-lock.json` -> new hash -> cache miss -> downloads fresh

### Why it speeds things up:
Instead of downloading packages every time:
- **Without cache**: `npm ci` downloads ~50MB+ of packages
- **With cache**: `npm ci` reuses local cache -> much faster

---

## Q24: Is the path `~/.npm` a standard?

### Answer:
Yes, `~/.npm` is **npm's default cache directory** on Linux/macOS.

When you run `npm install` or `npm ci`, npm stores downloaded packages in `~/.npm` to avoid re-downloading them later. This is the standard location npm uses by default.

You can verify this by running:
```bash
npm config get cache
```

On Windows, it would be different (like `%AppData%/npm-cache`), but since GitHub Actions runners use Ubuntu, `~/.npm` is the correct standard path.

---

## Q25: What is `~/`? Is it in the home directory?

### Answer:
Yes, `~` is shorthand for the **home directory**.

- `~/` = `/home/username/` (on Linux)
- `~/.npm` = `/home/username/.npm`

So when GitHub Actions uses `~/.npm`, it's referring to the home directory of the user running on the Ubuntu runner (typically `/home/runner/.npm`).

---

## Q26: How does `.npm` contents look like?

### Answer:
The `.npm` directory contains **npm's package cache** and looks something like this:

```
~/.npm/
├── _cacache/           # Main cache storage (binary data)
├── _logs/              # npm operation logs
├── _locks/             # Lock files for concurrent operations
├── anonymous-cli-metrics.json
└── _npx/               # npx cache (if used)
```

**Key parts:**
- **`_cacache/`** - The actual package data (tarballs, metadata) stored as content-addressed blobs
- **`_logs/`** - Debug logs from npm operations
- **`_locks/`** - Prevents concurrent npm processes from interfering

**Example cache structure:**
```
_cacache/
├── content-v2/         # Actual package files (binary)
├── index-v5/           # Index of what's cached
└── tmp/               # Temporary files during operations
```

It's all managed automatically by npm — you never need to manually edit these files. The caching action just preserves this entire structure between workflow runs to avoid re-downloading packages.

---

## Q27: I don't understand the contents of the `.npm` folder. If it is to contain dependencies, it should be containing them.

### Answer:
The confusion was between two different things:

**`.npm` (cache)**
- Raw package **downloads** (compressed tarballs)
- Used to avoid **re-downloading** from npm registry
- Not directly usable code

**`node_modules` (installed dependencies)**
- **Extracted, runnable** JavaScript code
- What your app actually `require()`s or `import`s
- Created by `npm install` or `npm ci`

### How they work together:
1. **Without cache**: `npm ci` -> downloads from internet -> extracts to `node_modules`
2. **With cache**: `npm ci` -> uses `.npm` cache -> extracts to `node_modules`

So caching `.npm` speeds up **step 1** (downloading), but you still need **step 2** (extraction to `node_modules`) every time.

**Think of it as**: `.npm` = "zip files storage", `node_modules` = "unzipped, ready-to-use files"

---

## Q28: Why should the caching of dependencies be done before the installation of dependencies? It sounds counter-intuitive, if let's say we are installing dependencies for the first time.

### Answer:
The **caching step runs first to CHECK** if there's an existing cache, not to create one initially.

### First time (no cache exists):
1. **Cache step**: Looks for cache with key `deps-node-modules-[hash]` -> **finds nothing** -> does nothing
2. **Install step**: `npm ci` downloads fresh from internet -> installs to `node_modules`
3. **After job completes**: Cache action **automatically saves** the `.npm` folder for next time

### Second time (cache exists):
1. **Cache step**: Looks for cache with same key -> **finds it** -> restores `.npm` folder
2. **Install step**: `npm ci` uses restored cache -> much faster install
3. **After job completes**: Cache already exists, so no new save needed

### Key insight:
The cache action is **smart** — it both:
- **Restores** cache at the beginning (if found)
- **Saves** cache at the end (if cache miss occurred)

So the order makes sense:
1. Try to restore cache first
2. Install (using cache if available)
3. Auto-save cache if it was a miss

**Think of it as**: "Check for shortcuts before doing the work"

---

## Q29: Will the dependencies be cached if we run the workflow again? Will the test step use the cache?

### Answer:
Yes, **if** the `package-lock.json` hasn't changed, the cache will be reused.

However, looking at the current workflow:

### What will happen:
- **Test job**: Will use cache (caching step was added)
- **Build job**: Won't use cache (no caching step was added at that point)

### To make both jobs use cache:
Add the same cache step to the `build` job:
```yaml
build:
  steps:
    - name: Get code
      uses: actions/checkout@v3
    - name: Cache dependencies
      uses: actions/cache@v5
      with:
        path: ~/.npm
        key: deps-node-modules-${{hashFiles('**/package-lock.json')}}
    - name: Install dependencies
      run: npm ci
```

**Result**: Both jobs will share the same cache and be faster on subsequent runs.

---

## Q30: `Cache step: Looks for cache with same key -> finds it -> restores .npm folder` — what does it mean by "restore .npm folder"?

### Answer:
"Restore .npm folder" means **downloading the cached folder from GitHub's storage back to the runner**.

### What happens step-by-step:

**Without cache (first run):**
- Runner starts with empty filesystem
- No `.npm` folder exists
- `npm ci` downloads everything from internet -> creates `.npm` cache
- GitHub Actions saves `.npm` folder to their cache storage

**With cache (subsequent runs):**
- Runner starts with empty filesystem again
- Cache action **downloads** the saved `.npm` folder from GitHub's cache storage
- **Recreates** the `.npm` folder on the runner at `~/.npm`
- Now `npm ci` can use the local cache instead of downloading

### "Restore" = "Put back"
Think of it as:
1. **Save**: "Take `.npm` folder and store it in GitHub's cache"
2. **Restore**: "Take stored `.npm` folder and put it back on the runner"

So the runner gets the **same `.npm` folder contents** as the previous run, making `npm ci` much faster.

---

## Q31: How long are the dependencies stored? Is it like for a repo, all different workflows will have their own cache of dependencies?

### Answer:

### Cache retention: 7 days by default
GitHub Actions automatically deletes caches after 7 days of no access, or when the total cache size exceeds limits (10GB for free accounts).

### Cache scope: Repository-wide, but key-specific
**All workflows in the same repo share the same cache storage**, but they access it by **cache key**.

Since the key is:
```yaml
key: deps-node-modules-${{hashFiles('**/package-lock.json')}}
```

**Same key = shared cache:**
- Workflow A runs -> creates cache with key `deps-node-modules-abc123`
- Workflow B runs -> uses same key -> **reuses Workflow A's cache**
- Both workflows benefit from the same cached dependencies

**Different key = separate cache:**
- Different `package-lock.json` -> different hash -> different cache key
- Each branch with different dependencies gets its own cache

### Cache sharing examples:
- **Shared**: Multiple workflows with same dependencies
- **Shared**: Same workflow running on different commits (if `package-lock.json` unchanged)
- **Not shared**: Different repositories (even if same owner)
- **Not shared**: Different dependency versions (different `package-lock.json`)

So yes, the `test` and `build` jobs (and any other workflows) will share the same dependency cache as long as they use the same key.

---

## Q32: Is it the `actions/cache@v5` that stores the dependencies in GitHub storage?

### Answer:
Yes, `actions/cache@v5` is **exactly** what handles storing and retrieving cache in GitHub's storage.

### How it works:

**On cache miss (first time):**
1. Cache action looks for key -> doesn't find it
2. Your job runs (`npm ci` creates `.npm` folder)
3. **At job end**: `actions/cache@v5` automatically **uploads** `.npm` folder to GitHub's cache storage

**On cache hit (subsequent runs):**
1. Cache action looks for key -> **finds it**
2. `actions/cache@v5` **downloads** cached `.npm` folder from GitHub storage
3. Restores it to the runner at `~/.npm`
4. Your `npm ci` uses the restored cache

### So yes:
`actions/cache@v5` is the **complete caching solution** — it handles:
- **Storing** cache in GitHub's backend
- **Retrieving** cache from GitHub's backend
- **Managing** cache keys and expiration
- **Restoring** files to the correct paths

Without this action, there's no caching — it's the bridge between your workflow and GitHub's cache storage.

---

## Q33: What is `deps-node-modules-${{hashFiles('**/package-lock.json')}}`?

### Answer:
That's the **cache key template** that creates a unique identifier for your cache.

### Breaking it down:
- **`deps-node-modules-`** = Static prefix (always the same)
- **`${{hashFiles('**/package-lock.json')}}`** = Dynamic hash based on `package-lock.json` content

### Example results:
If your `package-lock.json` contains specific dependencies, the final key might look like:
```
deps-node-modules-a1b2c3d4e5f6...
```

If you change dependencies (modify `package-lock.json`), you get a different hash:
```
deps-node-modules-z9y8x7w6v5u4...
```

### Why this works:
- **Same dependencies** = Same hash = **Reuse existing cache**
- **Different dependencies** = Different hash = **Create new cache**

This ensures you don't accidentally use outdated cached dependencies when your `package-lock.json` changes.

---

## Q34: What is `runs-on: group: duo-ubuntu-latest`?

### Answer:
`runs-on: group: duo-ubuntu-latest` specifies a **self-hosted runner group** instead of a GitHub-hosted runner.

### Difference:

**GitHub-hosted runner:**
```yaml
runs-on: ubuntu-latest  # Uses GitHub's shared VMs
```

**Self-hosted runner group:**
```yaml
runs-on:
  group: duo-ubuntu-latest  # Uses organization's own runners
```

### What is a runner group?
- Organizations can set up **their own machines** as GitHub Actions runners
- These are organized into **groups** (e.g., `duo-ubuntu-latest`)
- Useful for: custom hardware, security, private network access, cost control

### Why use it?
- **Performance**: Dedicated machines, no queue wait times
- **Security**: Code stays on your own infrastructure
- **Access**: Can reach internal services/databases
- **Cost**: Avoid GitHub's per-minute billing for Actions

This is typically seen in **enterprise/organization** workflows, not personal projects. The current workflow uses `runs-on: ubuntu-latest` (GitHub-hosted), which is fine for learning.

## Q35 : explain :  - name: Publish JS files
        id: publish
        run: find dist/assets/*.js -type f -execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'    
### Answer:
## What that step is doing

```yaml
- name: Publish JS files
  id: publish
  run: find dist/assets/*.js -type f -execdir echo 'my-script-file={}' >> $GITHUB_OUTPUT ';'
```

### 1) It searches for `.js` files
- `find dist/assets/*.js -type f ...`

Because of the `*.js`, your shell expands the glob *before* `find` runs. So `find` will receive something like:

- `find dist/assets/app.js dist/assets/vendor.js -type f ...`

This is slightly unusual usage of `find` (more on that in “Gotchas”).

### 2) For each found file, it writes a GitHub Actions output
- `-execdir ... ';'` tells `find`:
  - **run a command once per match**
  - **run it from the directory where the matched file lives** (`-execdir`)

The command being executed is:

```sh
echo 'my-script-file={}' >> $GITHUB_OUTPUT
```

Key parts:
- **`{}`** is replaced by the current matched file (with `-execdir`, it’s typically the path relative to that directory, often just the filename).
- **`$GITHUB_OUTPUT`** is a special file path that GitHub Actions provides. Appending `name=value` lines into it sets **step outputs**.
- Because the step has `id: publish`, later steps can read this output as:
  - `${{ steps.publish.outputs.my-script-file }}`

### 3) Why the weird `';'`
- In `find`, the `-exec...` action must be terminated by a semicolon `;`.
- The semicolon is quoted as `';'` so the *shell* doesn’t treat it as a command separator. It gets passed to `find` literally, which is what `find` needs.

---

## Important gotchas / behavior to be aware of

### A) Multiple `.js` files → multiple writes to the same output key
If there are multiple JS files, this will append multiple lines like:

```
my-script-file=app.js
my-script-file=vendor.js
```

GitHub Actions outputs aren’t really meant to be “multi-valued” this way. In practice, **the last one usually wins** when you read `${{ steps.publish.outputs.my-script-file }}`.

If you intended to output *all* JS files, you should join them into one value (space/newline/JSON), or use a different approach.

### B) The `find dist/assets/*.js` pattern is fragile
If no files match `dist/assets/*.js`, many shells will pass the literal string `dist/assets/*.js` to `find`, which can produce errors like “No such file or directory”.

A more idiomatic `find` is:

```sh
find dist/assets -type f -name '*.js' ...
```

---

## Safer/cleaner alternatives (depending on what you want)

### If you want the **first** JS file as output
```sh
file="$(find dist/assets -type f -name '*.js' | head -n 1)"
echo "my-script-file=$file" >> "$GITHUB_OUTPUT"
```

### If you want **all** JS files as one output (space-separated)
```sh
files="$(find dist/assets -type f -name '*.js' -print | tr '\n' ' ' | sed 's/ $//')"
echo "my-script-files=$files" >> "$GITHUB_OUTPUT"
```

---

## Status
- **Explained** what `find ... -execdir ... >> $GITHUB_OUTPUT ';'` does, how it creates step outputs, and the main pitfalls (glob expansion + multiple matches).        
 ## Q 36: what is $GITHUB_OUTPUT

 ### Answer:
 ## What is `$GITHUB_OUTPUT`?

`$GITHUB_OUTPUT` is an **environment variable provided by GitHub Actions** that contains the **path to a temporary file** for the *current step*.

If you **append** lines of the form:

```bash
name=value
```

to that file, GitHub Actions will treat them as **step outputs**.

So in your step:

```bash
echo "my-script-file=dist/assets/app.js" >> "$GITHUB_OUTPUT"
```

you are setting an output called `my-script-file` for the step with `id: publish`.

Then you can read it later as:

```yaml
${{ steps.publish.outputs.my-script-file }}
```

---

## Key points

- **Scope**: step-level outputs (available within the same job via `steps.<id>.outputs.<name>`).
- **Why it exists**: it’s the modern, secure replacement for the old `::set-output` command (which was deprecated).
- **File format**: typically `key=value` per line. (There’s also a multiline “heredoc” format if needed.)

---

## Related: job outputs vs step outputs

In your workflow you also have:

```yaml
outputs:
  script-file: ${{ steps.publish.outputs.my-script-file }}
```

That promotes the *step output* to a **job output**, so downstream jobs can access it using:

```yaml
${{ needs.build.outputs.script-file }}
```

If you want, paste your `outputs:` section for the `build` job and I’ll point out exactly how to consume it in the `deploy` job.