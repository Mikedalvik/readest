# Complete Explanation: GitHub Actions iOS Build Workflow

This document provides a line-by-line explanation of the GitHub Actions workflow that builds the Readest iOS app.

---

## Overview: What This Workflow Does

**Purpose:** Automatically build an iOS simulator app using GitHub's cloud macOS runners

**Input:** Your code pushed to GitHub  
**Output:** `Readest.app` file (iOS simulator build)  
**Time:** ~15-20 minutes  
**Cost:** Free (for public repositories)

---

## Full Workflow Structure

```
┌─────────────────────────────────────┐
│  Trigger (manual or on push)       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  GitHub Spins Up macOS VM           │
│  (Apple Silicon Mac in cloud)       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Install Tools (Node, Rust, pnpm)   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Download Dependencies              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Build Next.js Frontend             │
│  (JavaScript → HTML/CSS/JS bundle)  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Build Rust Backend                 │
│  (Rust → Native iOS library)        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Xcode Packages Everything          │
│  (Library + Assets → .app bundle)   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Upload Readest.app as Artifact     │
└─────────────────────────────────────┘
```

---

## Line-by-Line Breakdown

### Section 1: Workflow Metadata

```yaml
name: Build iOS
```

**What it does:** Names your workflow "Build iOS"  
**Where you see it:** In the GitHub Actions tab sidebar  
**Why it matters:** Helps you identify this workflow among others

---

```yaml
on: 
  workflow_dispatch:
```

**What it does:** Defines when the workflow runs  
**`workflow_dispatch`:** Manual trigger only (you click "Run workflow" button)  

**Alternative options:**
```yaml
on:
  push:
    branches: [ main ]  # Auto-run on every push to main
  pull_request:         # Run on PRs
  schedule:
    - cron: '0 0 * * *' # Run daily at midnight
```

**Why manual trigger:** Prevents wasting free build minutes on every small change

---

### Section 2: Job Definition

```yaml
jobs:
  build:
    runs-on: macos-latest
```

**`jobs:`** - A workflow can have multiple jobs (e.g., build, test, deploy)  
**`build:`** - Name of this specific job  
**`runs-on: macos-latest`** - Which type of machine to use

**Available runners:**
- `ubuntu-latest` - Linux (free, unlimited)
- `windows-latest` - Windows (free, limited)
- `macos-latest` - macOS Apple Silicon M1/M2 (free for public repos)
- `macos-13` - macOS Intel (older)

**Why macOS:** iOS apps can ONLY be built on macOS (Xcode requirement)

---

### Section 3: Setup Steps

#### Step 1: Checkout Code

```yaml
- name: Checkout code
  uses: actions/checkout@v4
  with:
    submodules: recursive
```

**What it does:** Downloads your repository code to the runner

**`uses: actions/checkout@v4`** - Pre-built action from GitHub  
**`with: submodules: recursive`** - Also downloads git submodules

**Why submodules matter:** Readest uses external libraries (like foliate-js) that are included as git submodules. Without this, those dependencies would be missing.

**What happens:**
```bash
# Equivalent to:
git clone https://github.com/YOUR-USERNAME/readest
cd readest
git submodule update --init --recursive
```

---

#### Step 2: Setup Node.js

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: 22
```

**What it does:** Installs Node.js version 22

**Why Node.js:** Next.js (the frontend framework) requires Node.js to build

**What gets installed:**
- `node` command (JavaScript runtime)
- `npm` command (package manager)

**Verification:**
```bash
node --version  # v22.x.x
npm --version   # 10.x.x
```

---

#### Step 3: Install pnpm

```yaml
- name: Install pnpm
  run: npm install -g pnpm
```

**What it does:** Installs pnpm package manager globally

**`run:`** - Executes a shell command  
**`-g`** - Global installation (available system-wide)

**Why pnpm instead of npm:**
- Faster (uses hard links instead of copying files)
- More efficient disk usage
- Readest project is configured to use pnpm

**What happens:**
```bash
npm install -g pnpm
# Now you can use: pnpm install, pnpm build, etc.
```

---

#### Step 4: Setup Rust

```yaml
- name: Setup Rust
  uses: dtolnay/rust-toolchain@stable
```

**What it does:** Installs the Rust programming language compiler

**`dtolnay/rust-toolchain`** - Popular Rust installation action  
**`@stable`** - Latest stable Rust version

**Why Rust:** Tauri's native backend is written in Rust. This compiles to native iOS code.

**What gets installed:**
- `rustc` (Rust compiler)
- `cargo` (Rust package manager)
- `rustup` (Rust version manager)

---

#### Step 5: Add iOS Targets

```yaml
- name: Add iOS targets
  run: |
    rustup target add aarch64-apple-ios-sim
```

**What it does:** Adds iOS simulator compilation target to Rust

**`rustup target add`** - Install cross-compilation support  
**`aarch64-apple-ios-sim`** - ARM64 iOS Simulator (for Apple Silicon Macs)

**Why this is needed:** By default, Rust only compiles for your current platform. To compile for iOS, you need to install the iOS target.

**Alternative targets:**
```bash
aarch64-apple-ios        # Real iOS devices (iPhone/iPad)
x86_64-apple-ios         # Intel Mac iOS simulator (older)
aarch64-apple-ios-sim    # Apple Silicon iOS simulator (M1/M2)
```

**What happens:**
```bash
# Downloads iOS standard library and compilation tools
rustup target add aarch64-apple-ios-sim
# Now: cargo build --target aarch64-apple-ios-sim
```

---

### Section 4: Install Dependencies

#### Step 6: Install Node Dependencies

```yaml
- name: Install dependencies
  run: pnpm install
```

**What it does:** Installs all JavaScript/TypeScript dependencies

**What happens:**
1. Reads `package.json` and `pnpm-lock.yaml`
2. Downloads ~500+ npm packages
3. Installs them in `node_modules/`
4. Takes 3-5 minutes (first time)

**Key dependencies installed:**
- Next.js (React framework)
- React (UI library)
- Tauri CLI (build tools)
- daisyUI (styling)
- PDF.js (PDF rendering)
- Many others...

**Equivalent to:**
```bash
cd readest
pnpm install  # Reads package.json and installs everything
```

---

#### Step 7: Setup PDF.js

```yaml
- name: Setup PDF.js
  run: pnpm --filter @readest/readest-app setup-pdfjs
```

**What it does:** Copies PDF.js library files to Next.js public directory

**`--filter @readest/readest-app`** - Run command only in specific workspace  
**`setup-pdfjs`** - Custom npm script defined in package.json

**Why needed:** PDF.js needs to be in the public folder so Next.js can serve it

**What happens:**
```bash
# Defined in package.json:
"scripts": {
  "setup-pdfjs": "cp -r node_modules/pdfjs-dist/build/pdf.worker.js public/"
}
```

---

#### Step 8: Initialize iOS Project

```yaml
- name: Initialize iOS
  run: pnpm tauri ios init
```

**What it does:** Generates the Xcode project structure

**What gets created:**
```
apps/readest-app/src-tauri/gen/apple/
├── Readest.xcodeproj/        # Xcode project
├── Readest_iOS/               # iOS app bundle config
├── Assets.xcassets/           # App icons, images
├── LaunchScreen.storyboard    # Splash screen
└── Info.plist                 # App metadata
```

**Why needed:** Tauri creates this structure dynamically. It doesn't exist in the git repo.

**Equivalent to:**
```bash
cd apps/readest-app
pnpm tauri ios init
# Tauri: "Initializing iOS project..."
# Creates Xcode project from templates
```

---

### Section 5: Configure Build Settings

#### Step 9: Set iOS Deployment Target

```yaml
- name: Set iOS deployment target to 14.0
  run: |
    cd apps/readest-app/src-tauri/gen/apple
    sed -i '' 's/IPHONEOS_DEPLOYMENT_TARGET = [0-9.]*;/IPHONEOS_DEPLOYMENT_TARGET = 14.0;/g' Readest.xcodeproj/project.pbxproj
```

**What it does:** Sets minimum iOS version to 14.0

**Why 14.0:** The Swift code uses `Logger` API which requires iOS 14+

**`sed -i ''`** - Stream editor (find and replace in file)  
**Regular expression:** Finds `IPHONEOS_DEPLOYMENT_TARGET = 13.0;` and changes to `14.0`

**What this prevents:**
```
Error: 'Logger' is only available in iOS 14.0 or newer
```

**How it works:**
```bash
# Before:
IPHONEOS_DEPLOYMENT_TARGET = 13.0;

# After:
IPHONEOS_DEPLOYMENT_TARGET = 14.0;
```

---

### Section 6: Build Frontend

#### Step 10: Build Next.js

```yaml
- name: Build Next.js frontend
  working-directory: apps/readest-app
  env:
    NODE_OPTIONS: "--max-old-space-size=6144"
  run: pnpm build
```

**What it does:** Compiles Next.js React app to static files

**`working-directory:`** - Run command in this directory (like `cd`)  
**`env:`** - Set environment variables  
**`NODE_OPTIONS`** - Node.js runtime options

**Why `--max-old-space-size=6144`:**
- Node.js default memory limit: ~2GB
- Readest build needs more memory
- 6144 MB = 6 GB (prevents out-of-memory crash)

**What happens:**
```bash
cd apps/readest-app
NODE_OPTIONS="--max-old-space-size=6144" pnpm build

# Next.js compiles:
# 1. TypeScript → JavaScript
# 2. React components → Optimized bundles
# 3. CSS → Minified stylesheets
# 4. Images → Optimized formats
# 5. Creates static HTML pages

# Output: out/ directory with production build
```

**Build output:**
```
out/
├── index.html
├── _next/static/chunks/    # JavaScript bundles
├── _next/static/css/       # Stylesheets
└── assets/                 # Images, fonts
```

**Time:** ~2-3 minutes

---

### Section 7: Build Rust Backend

#### Step 11: Build Rust Library

```yaml
- name: Build Rust library for iOS simulator
  working-directory: apps/readest-app/src-tauri
  env:
    IPHONEOS_DEPLOYMENT_TARGET: "14.0"
  run: |
    cargo build --release --target aarch64-apple-ios-sim --lib
    
    echo "=== Finding built libraries ==="
    find ../../../target/aarch64-apple-ios-sim/release -name "*.a" -type f
    
    mkdir -p gen/apple/Externals/arm64/debug
    
    LIBRARY=$(find ../../../target/aarch64-apple-ios-sim/release -name "lib*.a" -type f | head -1)
    echo "Found library: $LIBRARY"
    cp "$LIBRARY" gen/apple/Externals/arm64/debug/libapp.a
```

**What it does:** Compiles Rust code to iOS native library

**Breaking it down:**

**`cargo build`** - Rust build command
- `--release` - Optimized production build (smaller, faster)
- `--target aarch64-apple-ios-sim` - Compile for iOS simulator
- `--lib` - Build library only (not executable)

**`IPHONEOS_DEPLOYMENT_TARGET: "14.0"`** - Tells Rust to target iOS 14.0+

**What gets compiled:**
```rust
// Rust code in src-tauri/src/
mod commands;
mod menu;
mod plugins;

// Compiles to:
libreadest.a  // Static library file
```

**File finding logic:**
```bash
# 1. Build creates library at:
target/aarch64-apple-ios-sim/release/libreadest.a

# 2. Find it:
LIBRARY=$(find ... -name "lib*.a" | head -1)
# Returns: libreadest.a

# 3. Copy to where Xcode expects:
cp libreadest.a gen/apple/Externals/arm64/debug/libapp.a
```

**Why copy:** Xcode looks for `libapp.a` specifically, but Rust creates `libreadest.a` (based on project name)

**Time:** ~6-8 minutes (longest step)

---

### Section 8: Disable Code Signing

#### Step 12: Create xcconfig File

```yaml
- name: Create xcconfig to disable code signing
  run: |
    cat > apps/readest-app/src-tauri/gen/apple/DisableCodeSigning.xcconfig << 'XCCONFIG'
    CODE_SIGN_IDENTITY =
    CODE_SIGNING_REQUIRED = NO
    CODE_SIGNING_ALLOWED = NO
    CODE_SIGN_STYLE = Manual
    DEVELOPMENT_TEAM =
    PROVISIONING_PROFILE_SPECIFIER =
    ARCHS = arm64
    ONLY_ACTIVE_ARCH = YES
    VALID_ARCHS = arm64
    IPHONEOS_DEPLOYMENT_TARGET = 14.0
    XCCONFIG
```

**What it does:** Creates configuration file to bypass code signing

**What is code signing:**
- Apple requires apps to be "signed" with certificates
- Proves who created the app
- Requires Apple Developer account ($99/year)

**How we bypass it:**
- Simulator builds don't need signing
- We force Xcode to skip the signing step

**xcconfig file format:**
```
KEY = VALUE
KEY2 = VALUE2
```

**Key settings explained:**

| Setting | Value | Meaning |
|---------|-------|---------|
| `CODE_SIGN_IDENTITY` | (empty) | Don't use any certificate |
| `CODE_SIGNING_REQUIRED` | NO | Signing is not required |
| `CODE_SIGNING_ALLOWED` | NO | Don't try to sign |
| `CODE_SIGN_STYLE` | Manual | Don't auto-manage signing |
| `DEVELOPMENT_TEAM` | (empty) | No team ID |
| `ARCHS` | arm64 | Build for ARM64 only |
| `IPHONEOS_DEPLOYMENT_TARGET` | 14.0 | Minimum iOS 14.0 |

**Why this works:**
- Simulator builds run on macOS, not real iOS
- macOS doesn't enforce app signing like iOS does
- We're building for testing, not App Store distribution

---

#### Step 13: Disable Rust Build Script

```yaml
- name: Disable Build Rust Code script in Xcode
  run: |
    cd apps/readest-app/src-tauri/gen/apple
    sed -i '' 's|shellScript = "pnpm tauri ios xcode-script|shellScript = "echo Skipping Rust build - already built|g' Readest.xcodeproj/project.pbxproj
```

**What it does:** Prevents Xcode from trying to rebuild Rust code

**The problem:**
- Xcode project has a "Build Rust Code" script phase
- It runs `pnpm tauri ios xcode-script`
- This expects a dev server (doesn't exist in CI)
- Results in: `failed to read missing addr file` error

**The solution:**
- Replace the script with harmless echo command
- We already built Rust manually (step 11)
- Xcode just packages the pre-built library

**How it works:**
```bash
# Find this in project.pbxproj:
shellScript = "pnpm tauri ios xcode-script ...";

# Replace with:
shellScript = "echo Skipping Rust build - already built";
```

**Why this is safe:**
- We built `libapp.a` already
- Xcode just needs to link it
- No need to rebuild

---

### Section 9: Build iOS App

#### Step 14: Run Xcode Build

```yaml
- name: Build iOS app with Xcode
  working-directory: apps/readest-app
  run: |
    xcodebuild \
      -project src-tauri/gen/apple/Readest.xcodeproj \
      -scheme Readest_iOS \
      -configuration Debug \
      -sdk iphonesimulator \
      -arch arm64 \
      -xcconfig src-tauri/gen/apple/DisableCodeSigning.xcconfig \
      CODE_SIGN_IDENTITY="" \
      CODE_SIGNING_REQUIRED=NO \
      CODE_SIGNING_ALLOWED=NO \
      DEVELOPMENT_TEAM="" \
      ONLY_ACTIVE_ARCH=YES \
      VALID_ARCHS=arm64 \
      IPHONEOS_DEPLOYMENT_TARGET=14.0 \
      build
```

**What it does:** Assembles all pieces into final iOS app

**`xcodebuild`** - Command-line interface to Xcode

**Parameters explained:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-project` | Path to .xcodeproj | Which Xcode project to build |
| `-scheme` | Readest_iOS | Which build scheme (debug/release config) |
| `-configuration` | Debug | Debug build (faster, includes symbols) |
| `-sdk` | iphonesimulator | Build for simulator, not device |
| `-arch` | arm64 | Apple Silicon architecture |
| `-xcconfig` | Path to xcconfig | Use our signing-disabled config |
| `CODE_SIGN_IDENTITY=""` | (empty) | Override: no certificate |
| `CODE_SIGNING_REQUIRED=NO` | NO | Override: don't require signing |
| `DEVELOPMENT_TEAM=""` | (empty) | Override: no team |
| `IPHONEOS_DEPLOYMENT_TARGET` | 14.0 | Override: iOS 14.0 minimum |
| `build` | (command) | Execute the build |

**What Xcode does:**

```
1. Compiles Swift/Objective-C code (if any)
   ↓
2. Links libapp.a (our Rust library)
   ↓
3. Processes assets (icons, images)
   ↓
4. Copies Next.js build (out/ folder)
   ↓
5. Creates Info.plist
   ↓
6. Bundles everything into Readest.app
   ↓
7. Places in: DerivedData/.../Debug-iphonesimulator/
```

**Output structure:**
```
Readest.app/
├── Readest              # Executable binary
├── Info.plist           # App metadata
├── Assets.car           # Compiled assets
├── out/                 # Next.js web app
│   ├── index.html
│   └── _next/...
└── Frameworks/          # Linked libraries
```

**Time:** ~1-2 minutes

---

### Section 10: Find and Upload Results

#### Step 15: Find Build Output

```yaml
- name: Find and list build outputs
  run: |
    echo "=== Searching for .app files recursively ==="
    find apps/readest-app -name "*.app" -type d 2>/dev/null || echo "No .app found yet"
    
    echo "=== Checking Xcode DerivedData ==="
    find ~/Library/Developer/Xcode/DerivedData -name "Readest.app" -type d 2>/dev/null || echo "Not in DerivedData"
    
    echo "=== Listing src-tauri/gen/apple contents ==="
    ls -la apps/readest-app/src-tauri/gen/apple/ || echo "Directory doesn't exist"
```

**What it does:** Searches for the built .app file

**Why search multiple locations:**
- Xcode can put output in different places
- Depends on configuration and Xcode version
- We search everywhere to find it

**Commands explained:**

**`find apps/readest-app -name "*.app" -type d`**
- Search recursively for directories ending in .app
- `2>/dev/null` - Hide error messages
- `|| echo "..."` - If find fails, print message

**Common locations:**
```bash
# Location 1: Project build directory
apps/readest-app/src-tauri/gen/apple/build/Debug-iphonesimulator/Readest.app

# Location 2: Xcode DerivedData
~/Library/Developer/Xcode/DerivedData/Readest-xyz/Build/Products/Debug-iphonesimulator/Readest.app
```

**This step is diagnostic only** - helps debug if upload fails

---

#### Step 16: Upload Artifact

```yaml
- name: Upload iOS artifacts
  uses: actions/upload-artifact@v4
  with:
    name: readest-ios-simulator-app
    retention-days: 90
    path: |
      apps/readest-app/src-tauri/gen/apple/build/Debug-iphonesimulator/*.app
      ~/Library/Developer/Xcode/DerivedData/**/Build/Products/Debug-iphonesimulator/*.app
    if-no-files-found: warn
```

**What it does:** Uploads the built app to GitHub for download

**`uses: actions/upload-artifact@v4`** - GitHub's artifact uploader

**Parameters:**

**`name:`** - Name shown in artifacts list (what you click to download)

**`retention-days:`** - How long GitHub keeps the file (default: 90 days)

**`path:`** - Where to find files to upload
- Can use wildcards (`*.app`)
- Can specify multiple paths (each on new line)
- Searches both possible locations

**`if-no-files-found:`** - What to do if nothing found
- `warn` - Show warning but don't fail
- `error` - Fail the workflow
- `ignore` - Do nothing

**What happens:**
```
1. GitHub Actions finds Readest.app
2. Compresses it into .zip
3. Uploads to GitHub's artifact storage
4. Shows download link at bottom of workflow page
```

**Where to download:**
```
GitHub repo → Actions tab → Your workflow run → Scroll to bottom → 
"Artifacts" section → Click "readest-ios-simulator-app"
```

**Download size:** ~20-30 MB (compressed)

---

## Complete Build Flow Diagram

```
┌────────────────────────────────────────────────────┐
│ 1. TRIGGER WORKFLOW                                │
│    • Manual click "Run workflow"                   │
│    • Or auto-trigger on git push                   │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 2. PROVISION RUNNER                                │
│    • GitHub allocates macOS VM                     │
│    • Apple Silicon M1/M2 Mac                       │
│    • Fresh, clean environment                      │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 3. SETUP ENVIRONMENT                               │
│    • Checkout code (git clone + submodules)        │
│    • Install Node.js 22                            │
│    • Install pnpm                                  │
│    • Install Rust + iOS target                     │
│    Time: ~1 minute                                 │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 4. INSTALL DEPENDENCIES                            │
│    • pnpm install (500+ npm packages)              │
│    • Setup PDF.js                                  │
│    • Initialize iOS project structure              │
│    Time: ~3-5 minutes                              │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 5. CONFIGURE BUILD                                 │
│    • Set iOS deployment target to 14.0             │
│    • Create xcconfig to disable signing            │
│    • Modify Xcode project settings                 │
│    Time: ~5 seconds                                │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 6. BUILD FRONTEND                                  │
│    • Next.js compiles React → static files         │
│    • TypeScript → JavaScript                       │
│    • CSS optimization                              │
│    • Output: out/ directory                        │
│    Time: ~2-3 minutes                              │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 7. BUILD BACKEND                                   │
│    • Cargo compiles Rust → libreadest.a            │
│    • Cross-compile for iOS simulator               │
│    • Copy to Xcode expected location               │
│    Time: ~6-8 minutes (longest!)                   │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 8. ASSEMBLE APP                                    │
│    • Xcode links library + frontend                │
│    • Processes assets (icons, images)              │
│    • Creates app bundle: Readest.app               │
│    Time: ~1-2 minutes                              │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 9. UPLOAD ARTIFACT                                 │
│    • Compress Readest.app → .zip                   │
│    • Upload to GitHub artifact storage             │
│    • Available for download                        │
│    Time: ~30 seconds                               │
└────────────────┬───────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────┐
│ 10. CLEANUP                                        │
│    • GitHub deallocates VM                         │
│    • All temporary files deleted                   │
│    • Only artifact remains                         │
└────────────────────────────────────────────────────┘
```

**Total time:** ~15-20 minutes  
**Cost:** $0 (free for public repos)

---

## Key Concepts Explained

### What is a GitHub Actions "Runner"?

A **runner** is a virtual machine (VM) that GitHub provides to run your workflows.

**Types:**
- `ubuntu-latest` - Linux VM
- `windows-latest` - Windows VM
- `macos-latest` - macOS VM (what we use)

**Specifications (macos-latest):**
- **CPU:** Apple M1/M2 (3-4 cores)
- **RAM:** 14 GB
- **Storage:** 14 GB available
- **OS:** macOS 14 (Sonoma) or newer
- **Xcode:** Pre-installed (multiple versions)

**Lifecycle:**
```
1. Workflow triggered
2. GitHub allocates fresh VM
3. Workflow runs
4. VM is destroyed
5. Artifacts saved
```

**Important:** Each run gets a completely fresh machine - nothing persists between runs!

---

### What is an "Artifact"?

An **artifact** is a file (or files) that you save from a workflow run.

**Purpose:**
- Save build outputs (your .app file)
- Save logs for debugging
- Share results between jobs
- Download for later use

**Storage:**
- Kept on GitHub servers
- Default retention: 90 days
- Can download any time during retention period
- Counts against storage quota (but generous)

**How to download:**
```
Repo → Actions → Workflow run → Scroll to bottom → 
Artifacts section → Click to download
```

---

### What is Code Signing?

**Code signing** is Apple's way of verifying who created an app.

**How it works:**
```
1. Developer creates app
2. Signs it with private certificate
3. Apple verifies with public certificate
4. iPhone checks signature before running
```

**Requirements:**
- Apple Developer account ($99/year)
- Certificate from Apple
- Provisioning profile

**Why we skip it:**
- Simulator doesn't require signing
- We're building for testing, not distribution
- Saves the $99/year cost

**Limitations:**
- Can't install on real iPhone/iPad
- Can't submit to App Store
- Simulator-only builds

---

### What is Cross-Compilation?

**Cross-compilation** is building code for a different platform than where you're building.

**Example:**
```
Building ON:  macOS (x86_64)
Building FOR: iOS Simulator (arm64)
```

**Why it's complex:**
- Different CPU architecture
- Different operating system
- Different system libraries
- Need special tools (like Rust targets)

**How Rust handles it:**
```bash
# Install target
rustup target add aarch64-apple-ios-sim

# Compile for that target
cargo build --target aarch64-apple-ios-sim
```

---

## Troubleshooting Guide

### Common Errors and Solutions

#### Error: "Out of memory"
```
FATAL ERROR: JavaScript heap out of memory
```

**Solution:** Increase Node.js memory
```yaml
env:
  NODE_OPTIONS: "--max-old-space-size=6144"  # 6GB
```

---

#### Error: "No such file or directory: libapp.a"
```
cp: libapp.a: No such file or directory
```

**Solution:** Find the actual library name
```bash
LIBRARY=$(find target -name "lib*.a" | head -1)
cp "$LIBRARY" expected/location/libapp.a
```

---

#### Error: "Code signing required"
```
error: No profiles for 'com.bilingify.readest' were found
```

**Solution:** Add signing override flags
```bash
xcodebuild \
  CODE_SIGN_IDENTITY="" \
  CODE_SIGNING_REQUIRED=NO \
  CODE_SIGNING_ALLOWED=NO
```

---

#### Error: "Architecture mismatch"
```
error: None of the architectures in ARCHS (x86_64) are valid
```

**Solution:** Build for arm64 only
```bash
xcodebuild \
  -arch arm64 \
  -sdk iphonesimulator
```

---

#### Error: "Logger API not available"
```
error: 'Logger' is only available in iOS 14.0 or newer
```

**Solution:** Set deployment target
```bash
sed -i '' 's/IPHONEOS_DEPLOYMENT_TARGET = .*/IPHONEOS_DEPLOYMENT_TARGET = 14.0;/g' project.pbxproj
```

---

## Performance Optimization

### Caching Dependencies

You can speed up builds by caching node_modules and Cargo dependencies:

```yaml
- name: Cache Node modules
  uses: actions/cache@v3
  with:
    path: |
      ~/.pnpm-store
      node_modules
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}

- name: Cache Cargo
  uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
```

**Benefit:** Reduces build time from ~20min to ~10min on subsequent runs

---

### Parallel Jobs

You could build multiple platforms in parallel:

```yaml
jobs:
  build-ios:
    runs-on: macos-latest
    # ... iOS build steps
    
  build-android:
    runs-on: ubuntu-latest
    # ... Android build steps
    
  build-web:
    runs-on: ubuntu-latest
    # ... Web build steps
```

**Benefit:** All platforms build simultaneously

---

## Security Considerations

### Secrets Management

If you need to add secrets (API keys, certificates):

```yaml
- name: Use secret
  env:
    API_KEY: ${{ secrets.MY_API_KEY }}
  run: echo "Using API key"
```

**Add secrets:**
```
Repo → Settings → Secrets and variables → Actions → 
New repository secret
```

**Never commit secrets to git!**

---

### Dependency Security

GitHub automatically scans for vulnerable dependencies:

```
Repo → Security tab → Dependabot alerts
```

Enable auto-updates:
```
Repo → Settings → Code security and analysis → 
Enable Dependabot security updates
```

---

## Cost Analysis

### GitHub Actions Pricing

**Free tier (public repos):**
- Ubuntu: Unlimited minutes
- Windows: 2,000 minutes/month
- macOS: 2,000 minutes/month

**Free tier (private repos):**
- All platforms: 2,000 minutes/month
- macOS uses 10x multiplier (1 min = 10 min)

**Your workflow:**
- ~20 minutes per run
- Public repo = FREE unlimited
- Private repo = 200 billing minutes per run
- Private: ~10 builds per month free

**Paid pricing (if you exceed free tier):**
- macOS: $0.08 per minute
- 20-minute build = $1.60
- 100 builds/month = $160

**Recommendation:** Keep repo public for free unlimited builds!

---

## Comparison: GitHub Actions vs Alternatives

### vs Local Mac Build

| Aspect | GitHub Actions | Local Mac |
|--------|---------------|-----------|
| **Cost** | $0 (public) | $599+ (Mac Mini) |
| **Setup** | 5 minutes | Hours (install Xcode, etc.) |
| **Maintenance** | None | OS updates, Xcode updates |
| **Consistency** | Always same environment | May drift over time |
| **Accessibility** | From any device | Need physical Mac |
| **Speed** | ~20 min | ~10-15 min (faster CPU) |

---

### vs Other CI Services

| Service | Free macOS? | Complexity | Integration |
|---------|------------|------------|-------------|
| **GitHub Actions** | ✅ Yes (public) | Easy | Native GitHub |
| **CircleCI** | ❌ Paid only | Medium | Good |
| **Travis CI** | ❌ Limited | Easy | Good |
| **Jenkins** | ❌ Self-hosted | Hard | Flexible |
| **GitLab CI** | ❌ Paid only | Medium | Native GitLab |

**Winner for most users:** GitHub Actions (free + easy + integrated)

---

## Advanced Customization

### Matrix Builds

Build for multiple iOS versions:

```yaml
strategy:
  matrix:
    ios-version: [14.0, 15.0, 16.0]

steps:
  - run: |
      xcodebuild \
        IPHONEOS_DEPLOYMENT_TARGET=${{ matrix.ios-version }}
```

---

### Conditional Steps

Run steps only in certain conditions:

```yaml
- name: Deploy to TestFlight
  if: github.ref == 'refs/heads/main'
  run: # deployment commands
```

---

### Notifications

Send notifications on build completion:

```yaml
- name: Notify on Slack
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Summary

Your GitHub Actions workflow is a sophisticated automated build pipeline that:

✅ **Eliminates** the need for a physical Mac  
✅ **Provides** consistent, reproducible builds  
✅ **Costs** $0 for public repositories  
✅ **Takes** ~20 minutes per build  
✅ **Produces** a ready-to-test iOS app  

**Key technologies used:**
- GitHub Actions (CI/CD platform)
- macOS runner (Apple Silicon VM)
- Node.js + pnpm (JavaScript ecosystem)
- Rust + Cargo (Native compilation)
- Tauri (App framework)
- Xcode (iOS build tools)

**What you've built:**
A professional-grade build automation system that many companies pay thousands of dollars for!

---

## Further Learning

### Official Documentation

- **GitHub Actions:** https://docs.github.com/en/actions
- **Tauri:** https://v2.tauri.app/
- **Rust:** https://doc.rust-lang.org/book/
- **Xcode Build Settings:** https://xcodebuildsettings.com/

### Next Steps

1. **Add automated testing** - Run tests before building
2. **Set up automatic releases** - Auto-create GitHub releases
3. **Build for real devices** - Add code signing for real iPhones
4. **Multi-platform builds** - Add Android, Windows, Linux
5. **Performance optimization** - Add caching to speed up builds

---

**Congratulations on understanding this complex workflow!** You now have deep knowledge of modern iOS CI/CD. 🎉
