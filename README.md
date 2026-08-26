# maestro-runner on Sauce Labs

Examples of running [Maestro](https://maestro.mobile.dev) flows on **Sauce Labs**
Android/iOS **real devices**, **emulators**, and **simulators** using
[maestro-runner](https://github.com/devicelab-dev/maestro-runner). Background on how this
works: [`docs/cloud-providers/saucelabs.md`](https://github.com/devicelab-dev/maestro-runner/blob/main/docs/cloud-providers/saucelabs.md).

## Quickstart

Four steps to run your first flow (iOS, EU data center):

1. Install `maestro-runner` - via npm:
   ```bash
   npm install --save-dev maestro-runner   # or use `npx maestro-runner ...` with no install step
   ```
   or via shell (macOS/Linux; on Windows use WSL):
   ```bash
   curl -fsSL https://open.devicelab.dev/install/maestro-runner | bash
   ```
2. Export your Sauce Labs credentials:
   ```bash
   export SAUCE_USERNAME="your-sauce-username"
   export SAUCE_ACCESS_KEY="your-sauce-access-key"
   ```
3. Upload the iOS app build to [Sauce Storage](https://docs.saucelabs.com/mobile-apps/app-storage/):
   ```bash
   curl -u "$SAUCE_USERNAME:$SAUCE_ACCESS_KEY" \
     -X POST "https://api.eu-central-1.saucelabs.com/v1/storage/upload" \
     -F "payload=@apps/SauceLabs-Demo-App.ipa" \
     -F "name=SauceLabs-Demo-App.ipa"
   ```
4. Run the flow:
   ```bash
   maestro-runner \
     --driver appium \
     --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com/wd/hub" \
     --caps provider-caps/ios-real-device.json \
     test flows/ios/login_standard_user.yaml
   ```

That's it - the sections below cover Android, other data centers, running a full suite,
tag filtering, and everything else in more detail.

## Folder structure

```
apps/                          # local copies of the demo app builds (upload these to Sauce storage)
  SauceLabs-Demo-App.ipa
  SauceLabs-Demo-App.Simulator.zip

flows/
  android/
    login_standard_user.yaml        # tags: smoke, login
    login_locked_out_user.yaml      # tags: regression, login, negative
    add_to_cart.yaml                # tags: smoke, cart
    view_about_screen.yaml          # tags: regression, menu
  ios/
    login_standard_user.yaml        # tags: smoke, login
    login_validation_error.yaml     # tags: regression, login, negative
    add_to_cart.yaml                # tags: smoke, cart
    view_about_screen.yaml          # tags: regression, menu

provider-caps/
  android-real-device.json
  android-emulator.json
  android-arm-emulator.json
  ios-real-device.json
  ios-simulator.json
```

**Why one folder per platform:** `maestro-runner ... test <dir>` only discovers `*.yaml`/`*.yml`
files **one level deep** in the directory you pass. That also means each platform needs its own
run (with its own `--caps` file and its own `appId`), so keeping `flows/android/` and `flows/ios/`
separate matches how the tool is actually invoked - and how different the two apps are.

**Why one `provider-caps/` folder for everything:** caps files are reused across every flow for a
given platform/device-type combo, so they live independently of the flows rather than being
duplicated per test.

## Prerequisites

1. Install [`maestro-runner`](https://github.com/devicelab-dev/maestro-runner) - via npm
   (`npm install --save-dev maestro-runner`, or `npx maestro-runner ...` with no install step) or
   via shell (`curl -fsSL https://open.devicelab.dev/install/maestro-runner | bash`; on Windows use
   WSL).
2. Export your Sauce Labs credentials:
   ```bash
   export SAUCE_USERNAME="your-sauce-username"
   export SAUCE_ACCESS_KEY="your-sauce-access-key"
   ```
3. Upload each build in `apps/` to [Sauce Storage](https://docs.saucelabs.com/mobile-apps/app-storage/)
   under the exact filename its caps file expects (`appium:app: storage:filename=...`):
   - `apps/SauceLabs-Demo-App.apk` → used by `provider-caps/android-real-device.json`,
     `provider-caps/android-emulator.json`, and `provider-caps/android-arm-emulator.json` - from
     [my-demo-app-android releases](https://github.com/saucelabs/my-demo-app-android/releases)
   - `apps/SauceLabs-Demo-App.ipa` → used by `provider-caps/ios-real-device.json` - from
     [my-demo-app-ios releases](https://github.com/saucelabs/my-demo-app-ios/releases)
   - `apps/SauceLabs-Demo-App.Simulator.zip` → used by `provider-caps/ios-simulator.json` - same
     releases page

   If you swap in a different build, keep the uploaded filename and each caps file's
   `storage:filename` in sync.

   Example upload with `curl`:
   ```bash
   curl -u "$SAUCE_USERNAME:$SAUCE_ACCESS_KEY" \
     -X POST "https://api.eu-central-1.saucelabs.com/v1/storage/upload" \
     -F "payload=@apps/SauceLabs-Demo-App.ipa" \
     -F "name=SauceLabs-Demo-App.ipa"
   ```
4. Pick your region's Appium endpoint. Examples below use the **EU data center**
   (`eu-central-1`) - swap in `us-west-1`, `us-east-4`, etc. if you need a different region.

## Run commands

This is the core of the repo - everything else here just sets up these calls. Each example
builds on the last: a single flow, a parallel batch, a tag-filtered parallel batch, and a
full-suite run collapsed onto one reused device.

### 1. Run a single flow on an Android real device

```bash
maestro-runner \
  --driver appium \
  --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com/wd/hub" \
  --caps provider-caps/android-real-device.json \
  test flows/android/login_standard_user.yaml
```

### 2. Run the whole iOS suite in parallel across 4 real devices

```bash
maestro-runner \
  --driver appium \
  --parallel 4 \
  --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com/wd/hub" \
  --caps provider-caps/ios-real-device.json \
  test flows/ios/
```

`--parallel 4` opens 4 concurrent Sauce Labs sessions and spreads the flows in `flows/ios/`
across them.

### 3. Run only `login`-tagged flows in parallel across 10 sessions

```bash
maestro-runner \
  --driver appium \
  --parallel 10 \
  --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com/wd/hub" \
  --caps provider-caps/android-real-device.json \
  test --include-tags login flows/android/
```

Tag filtering (`--include-tags`/`--exclude-tags`) and `--parallel` combine freely - this narrows
the run down to `login`-tagged flows first, then fans whatever matches out across 10 sessions.
`--parallel 10` is a ceiling, not a fixed count: only as many sessions are created as there are
matching flows, so if fewer than 10 flows carry the `login` tag, fewer than 10 sessions get
created.

### 4. Run the full iOS simulator suite on a single reused simulator

```bash
maestro-runner \
  --driver appium \
  --parallel 1 \
  --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com/wd/hub" \
  --caps provider-caps/ios-simulator.json \
  test flows/ios/
```

`--parallel 1` is spelled out explicitly here, but it's also the default when `--parallel` is
omitted - every flow in `flows/ios/` runs sequentially **on the same simulator session** instead
of spinning up a new one per flow. This only works cleanly because every iOS
flow starts with `launchApp`: Maestro terminates and relaunches the app between flows, so each
test still starts from the same clean app state even though the underlying device/session is
shared. This is a good way to run a full suite cheaply when you don't need per-test device
isolation.

### 5. Android ARM emulators (beta)

`provider-caps/android-arm-emulator.json` targets Sauce Labs' newer **Android ARM emulators**
(`armRequired: true`), which run on ARM-based host hardware instead of the standard x86 emulator
farm. This device type is currently in **beta** - if you need access, contact
[support@saucelabs.com](mailto:support@saucelabs.com).

```bash
maestro-runner \
  --driver appium \
  --appium-url "https://$SAUCE_USERNAME:$SAUCE_ACCESS_KEY@ondemand.eu-central-1.saucelabs.com:443/wd/hub" \
  --caps provider-caps/android-arm-emulator.json \
  test flows/android/
```

## Tags used in these examples

| Tag          | Meaning                                       |
|--------------|------------------------------------------------|
| `smoke`      | Core happy-path checks (login, add to cart)     |
| `regression` | Broader coverage, run less frequently           |
| `login`      | Any flow exercising the login screen            |
| `negative`   | Invalid-input / error-path assertions           |
| `cart`       | Cart-related flows                              |
| `menu`       | Menu/navigation flows                           |


## Job naming on Sauce Labs

None of the `provider-caps/*.json` files set `sauce:options.name`, so Sauce Labs falls back to the
YAML flow's basename (without extension) as the job name - e.g. `login_standard_user`. Add a
`name` under `sauce:options` in a caps file if you want a fixed job name instead.
