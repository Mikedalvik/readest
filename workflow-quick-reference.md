# GitHub Actions iOS Build - Quick Reference Card

## Workflow at a Glance

```yaml
name: Build iOS
on: workflow_dispatch  # Manual trigger
runs-on: macos-latest  # Apple Silicon Mac
```

**Total time:** ~15-20 minutes  
**Cost:** $0 (public repos)  
**Output:** Readest.app (iOS simulator)

---

## Build Steps Summary

| # | Step | Time | Purpose |
|---|------|------|---------|
| 1 | Checkout code | 10s | Download repository |
| 2 | Setup Node.js | 15s | Install Node v22 |
| 3 | Install pnpm | 10s | Package manager |
| 4 | Setup Rust | 1-2m | Rust compiler |
| 5 | Add iOS targets | 1m | iOS cross-compilation |
| 6 | Install dependencies | 3-5m | npm packages |
| 7 | Setup PDF.js | 30s | Copy PDF library |
| 8 | Initialize iOS | 30s | Generate Xcode project |
| 9 | Set deployment target | 5s | iOS 14.0 minimum |
| 10 | Build Next.js | 2-3m | Compile frontend |
| 11 | Build Rust | 6-8m | Compile backend |
| 12 | Create xcconfig | 5s | Disable signing |
| 13 | Disable Rust script | 5s | Skip rebuild |
| 14 | Build with Xcode | 1-2m | Assemble app |
| 15 | Find outputs | 5s | Locate .app file |
| 16 | Upload artifact | 30s | Save to GitHub |

**Total:** ~15-20 minutes

---

## Key Commands Explained

### Node.js Memory Fix
```yaml
env:
  NODE_OPTIONS: "--max-old-space-size=6144"  # 6GB RAM
```
**Why:** Prevents out-of-memory during Next.js build

### Rust Cross-Compilation
```bash
rustup target add aarch64-apple-ios-sim  # Install iOS target
cargo build --target aarch64-apple-ios-sim  # Build for iOS
```
**Why:** Compiles Rust code for iOS simulator

### Code Signing Bypass
```bash
CODE_SIGN_IDENTITY=""
CODE_SIGNING_REQUIRED=NO
CODE_SIGNING_ALLOWED=NO
```
**Why:** Simulator builds don't need Apple Developer account

### Xcode Build
```bash
xcodebuild \
  -project Readest.xcodeproj \
  -scheme Readest_iOS \
  -configuration Debug \
  -sdk iphonesimulator \
  -arch arm64
```
**Why:** Assembles app from all built components

---

## File Locations

### Input Files
```
readest/
├── .github/workflows/build-ios.yml  # This workflow
├── apps/readest-app/
│   ├── package.json                 # Dependencies
│   ├── src-tauri/
│   │   ├── Cargo.toml              # Rust config
│   │   └── src/                    # Rust source
│   └── src/                        # Next.js source
└── pnpm-lock.yaml                  # Locked versions
```

### Generated Files
```
apps/readest-app/
├── out/                            # Next.js build output
├── src-tauri/
│   ├── gen/apple/                  # Xcode project (created)
│   │   ├── Readest.xcodeproj/
│   │   ├── DisableCodeSigning.xcconfig  # Our config
│   │   └── Externals/arm64/debug/
│   │       └── libapp.a            # Rust library
│   └── target/
│       └── aarch64-apple-ios-sim/
│           └── release/
│               └── libreadest.a    # Rust build
```

### Output
```
DerivedData/.../Build/Products/Debug-iphonesimulator/
└── Readest.app/                    # Final app (uploaded)
```

---

## Important Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `NODE_OPTIONS` | `--max-old-space-size=6144` | Node.js 6GB memory |
| `IPHONEOS_DEPLOYMENT_TARGET` | `14.0` | Minimum iOS version |
| `CARGO_INCREMENTAL` | `0` | Faster Rust builds |

---

## Troubleshooting Quick Fixes

### Problem: Out of memory
```yaml
env:
  NODE_OPTIONS: "--max-old-space-size=8192"  # Increase to 8GB
```

### Problem: Wrong architecture
```bash
# Add to xcodebuild:
-arch arm64 \
ARCHS=arm64 \
VALID_ARCHS=arm64
```

### Problem: Code signing errors
```bash
# Add to xcodebuild:
CODE_SIGN_IDENTITY="" \
CODE_SIGNING_REQUIRED=NO \
CODE_SIGNING_ALLOWED=NO \
DEVELOPMENT_TEAM=""
```

### Problem: iOS version too old
```bash
# In project.pbxproj:
sed -i '' 's/IPHONEOS_DEPLOYMENT_TARGET = .*/IPHONEOS_DEPLOYMENT_TARGET = 14.0;/g'
```

### Problem: Library not found
```bash
# Find actual library:
find target -name "lib*.a" -type f
# Copy with correct name:
cp path/to/libname.a expected/location/libapp.a
```

---

## Platform-Specific Notes

### macOS Runner (GitHub Actions)
- **CPU:** Apple M1/M2 (arm64)
- **RAM:** 14 GB
- **Xcode:** Pre-installed (16.x)
- **Fresh every run:** No persistent state

### iOS Simulator Target
- **Architecture:** arm64 (Apple Silicon)
- **SDK:** iphonesimulator 18.5
- **Minimum iOS:** 14.0
- **No signing required:** Debug builds

### Build Modes
- **Debug:** Faster, larger, includes symbols, no signing
- **Release:** Slower, smaller, optimized, requires signing

---

## Cost Breakdown

### GitHub Actions Free Tier

**Public repositories:**
- macOS: Unlimited minutes ✅
- Ubuntu: Unlimited minutes
- Windows: Unlimited minutes

**Private repositories:**
- All platforms: 2,000 minutes/month
- macOS multiplier: 10x (1 min = 10 billing min)
- Your workflow: 20 min real = 200 billing min
- Limit: ~10 builds/month free

**Paid (if exceeded):**
- macOS: $0.08/minute
- 20-min build = $1.60

---

## Customization Options

### Auto-trigger on push
```yaml
on:
  push:
    branches: [ main ]
  workflow_dispatch:
```

### Build multiple configurations
```yaml
strategy:
  matrix:
    configuration: [Debug, Release]
```

### Add caching
```yaml
- uses: actions/cache@v3
  with:
    path: ~/.cargo
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
```

### Notifications
```yaml
- name: Notify on failure
  if: failure()
  run: # send notification
```

---

## Comparison Table

| Aspect | GitHub Actions | Local Mac | Other CI |
|--------|---------------|-----------|----------|
| **Cost** | Free | $599+ | Varies |
| **Setup** | 5 min | Hours | Medium |
| **Speed** | ~20 min | ~10 min | ~20 min |
| **Access** | Anywhere | Physical | Cloud |
| **Maintenance** | None | Yes | Some |

---

## Common File Types

| Extension | Description | Created By |
|-----------|-------------|------------|
| `.yml` | Workflow definition | You (manually) |
| `.xcodeproj` | Xcode project | `tauri ios init` |
| `.xcconfig` | Build settings | Workflow |
| `.a` | Static library | `cargo build` |
| `.app` | iOS app bundle | `xcodebuild` |
| `.ipa` | iOS package | `xcodebuild archive` |

---

## Useful Commands

### Locally test parts of the workflow

```bash
# Install dependencies
pnpm install

# Build frontend
pnpm build

# Build Rust for iOS sim
cargo build --target aarch64-apple-ios-sim

# Initialize iOS
pnpm tauri ios init

# Build iOS (requires Mac)
pnpm tauri ios build --debug
```

### Check workflow syntax
```bash
# Install act (local GitHub Actions)
brew install act

# Test workflow locally
act -j build
```

---

## Success Indicators

### ✅ Build succeeded if you see:

```
✓ Build iOS app with Xcode
✓ Find and list build outputs
✓ Upload iOS artifacts
```

### 📦 Download location:

```
GitHub repo → Actions → Workflow run → 
Bottom of page → Artifacts → 
Click "readest-ios-simulator-app"
```

---

## Performance Tips

1. **Enable caching** - Saves 5-10 minutes
2. **Use matrix builds** - Build multiple configs in parallel
3. **Skip unnecessary steps** - Comment out when debugging
4. **Incremental builds** - Reuse previous build artifacts

---

## Security Best Practices

1. **Never commit secrets** - Use GitHub Secrets
2. **Review dependencies** - Check Dependabot alerts
3. **Pin action versions** - `actions/checkout@v4` not `@main`
4. **Limit permissions** - Use minimal required permissions
5. **Scan artifacts** - Check uploaded files

---

## Next Steps

- [ ] Add automated testing
- [ ] Set up automatic releases
- [ ] Build for real devices (needs Apple account)
- [ ] Add other platforms (Android, Windows, Linux)
- [ ] Optimize with caching
- [ ] Add deployment step

---

## Resources

- **Full explanation:** See `github-actions-workflow-explained.md`
- **GitHub Actions docs:** https://docs.github.com/en/actions
- **Tauri docs:** https://v2.tauri.app/
- **Xcode build settings:** https://xcodebuildsettings.com/

---

**Remember:** This workflow runs on **GitHub's servers**, not your computer!
