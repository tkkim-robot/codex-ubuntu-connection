
# Ubuntu Codex Desktop Connection Features

Short, repeatable notes for installing the Ubuntu/Linux Codex Desktop build with the newer connection UI.

The Linux wrapper can expose the connection UI, but current Codex Desktop builds may still assume some macOS-only behavior in the remote-control flow. The extra hotfix steps below cover the Linux-specific gaps found during setup.

## Links

- Reference: https://zenn.dev/robustonian/articles/codex_mobile_connect_to_ubuntu?locale=en
- Linux wrapper repo: https://github.com/robustonian/codex-desktop-linux
- Branch used here: https://github.com/robustonian/codex-desktop-linux/tree/feat/install-latest-installer
- Upstream Codex page: https://openai.com/codex/

## 1. Clone the Linux wrapper

```bash
git clone -b feat/install-latest-installer --single-branch \
  https://github.com/robustonian/codex-desktop-linux.git \
  "$HOME/codex-desktop-linux"

cd "$HOME/codex-desktop-linux"
```

## 2. Install prerequisites

Normal Ubuntu path:

```bash
sudo apt update
sudo apt install -y git curl unzip tar python3 build-essential
```

If your Ubuntu `7z` is too old or missing, the repo can bootstrap `7zz`:

```bash
bash scripts/install-deps.sh
```

No-sudo `7zz` fallback:

```bash
mkdir -p "$HOME/.local/bin" "$HOME/.local/share/7zip"
curl -fL -o /tmp/7z.tar.xz https://www.7-zip.org/a/7z2600-linux-x64.tar.xz
tar -C "$HOME/.local/share/7zip" -xf /tmp/7z.tar.xz 7zz
ln -sf "$HOME/.local/share/7zip/7zz" "$HOME/.local/bin/7zz"
ln -sf "$HOME/.local/bin/7zz" "$HOME/.local/bin/7z"
```

## 3. Install the app

Preferred path if you can use `sudo`:

```bash
bash scripts/install-latest.sh
```

User-local path, useful when you do not want to replace system packages:

```bash
mkdir -p "$HOME/.local/opt/codex-desktop-linux"

PATH="$HOME/.local/bin:$PATH" \
CODEX_INSTALL_ROOT="$HOME/.local/opt/codex-desktop-linux" \
CODEX_INSTALL_DIR="$HOME/.local/opt/codex-desktop-linux/codex-app" \
./install.sh --fresh

cp Codex.dmg "$HOME/.local/opt/codex-desktop-linux/Codex.dmg"
./contrib/user-local-install/install-user-local.sh
```

After this, these commands should exist:

```bash
codex-desktop
codex-desktop-version
codex-desktop-update
```

## 4. Enable the feature flags

Edit `~/.codex/config.toml` and make sure the `[features]` table contains:

```toml
[features]
remote_connections = true
remote_control = true
```

If `[features]` already exists, add only the two key/value lines under the existing table.

## 5. Make sure the Codex CLI works from non-interactive shells

Remote connection setup may call the `codex` CLI from a shell that does not load your normal interactive `PATH`.

Check:

```bash
codex --version
```

If this fails with an error like:

```text
/usr/bin/env: 'node': No such file or directory
```

make sure the Node.js binary installed by the wrapper is on `PATH`, or replace your user-local `codex` launcher with a small wrapper that prepends the wrapper-installed Node.js directory before executing the real CLI.

After the fix, this should work from a plain SSH shell:

```bash
$HOME/.local/bin/codex --version
```

## 6. Apply the Linux remote-control hotfixes

The robustonian branch exposes the Linux remote-control UI, but the authorization flow may still try to use a macOS-only native device-key module. Without a Linux fallback, **Control other devices** can fail with:

```text
Remote control device keys are only available on macOS
```

Apply the Linux remote-control key hotfix:

```bash
cd "$HOME/codex-desktop-linux"
bash scripts/apply-linux-remote-control-key-hotfix.sh
```

Current builds may also need two additional Linux fixes:

```bash
bash scripts/apply-linux-remote-control-availability-hotfix.sh
bash scripts/apply-linux-remote-control-settings-hotfix.sh
```

These fixes address cases where the backend remote-control manager is working, but the Settings UI still only shows SSH connections or hides the **Control this Mac / Control other devices** tabs.

The tab may still say **Control this Mac** on Linux. That is upstream UI text; on Linux it means “control this machine.”

The hotfixes repack or patch installed Codex Desktop assets, back up previous files under `~/codex-app-backups/`, and should be re-applied after updating Codex Desktop.

Security note: on Linux, the fallback device key is stored locally in:

```text
~/.local/state/codex-desktop/remote-control-device-keys.json
```

The file should be written with mode `0600`, but it is not hardware-backed like the macOS implementation.

## 7. Run Codex Desktop

```bash
codex-desktop --x11
```

Then open:

```text
Settings -> Connections
```

You should see tabs for controlling this machine, controlling other devices, and SSH connections.

You can also open the Connections settings page directly from the desktop session:

```bash
xdg-open 'codex://settings/connections'
```

## Verification and logs

Check installed metadata:

```bash
codex-desktop-version
```

Watch the launcher log:

```bash
tail -f "$HOME/.cache/codex-desktop/launcher.log"
```

Useful success/failure log patterns:

```bash
rg -i "refresh_remote_control|remote_control_client_enrollment|remote_control_authorize|device_key|finish_response|authorize_failed|remote-connections-settings" \
  "$HOME/.cache/codex-desktop/launcher.log"
```

Useful success patterns include:

```text
refresh_remote_control_started
refresh_remote_control_completed
```

## Troubleshooting

- `Remote control device keys are only available on macOS`: run `bash scripts/apply-linux-remote-control-key-hotfix.sh`.
- Connections page only shows SSH connections: apply the availability and settings hotfixes, restart Codex Desktop, then reopen `codex://settings/connections`.
- `codex --version` fails over SSH because `node` is missing: fix the user-local Codex CLI wrapper or ensure the wrapper-installed Node.js directory is available on `PATH`.
- `ssh: connect to host ... port 22: Connection refused`: unrelated saved SSH remote; fix or remove that SSH connection in Settings.
- `unsupported feature enablement auth_elicitation`: noisy but not usually the remote-control blocker.
- If the app does not stay open, clear stale PID files and relaunch:

```bash
rm -f "$HOME/.local/state/codex-desktop/app.pid" \
      "$HOME/.local/state/codex-desktop/webview.pid"
codex-desktop --x11
```

## Background behavior

Codex Desktop remote control is not currently a pure headless service on Linux. It still expects a real graphical desktop session through X11 or Wayland.

You can keep it running in the background after launch, but it should still be started inside the logged-in desktop session:

```bash
DISPLAY=:1 XDG_RUNTIME_DIR=/run/user/$(id -u) codex-desktop --x11
```
