# Ubuntu Codex Desktop Connection Features

Short, repeatable notes for installing the Ubuntu/Linux Codex Desktop build with the newer connection UI, including the extra Linux hotfix needed for **Control other devices**.

Tested on Ubuntu 24.04.3 LTS on 2026-05-21. But this might only gonna work for less than a month. 

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

## 5. Apply the Linux remote-control key hotfix

The robustonian branch exposes the Linux remote-control UI, but the authorization flow still tries to use a macOS-only native device-key module. Without this hotfix, **Control other devices** can fail with:

```text
Remote control device keys are only available on macOS
```

This checkout includes a local helper script for that gap:

```bash
cd "$HOME/codex-desktop-linux"
bash scripts/apply-linux-remote-control-key-hotfix.sh
```

The hotfix repacks the installed `app.asar`, backs up the previous archive under `~/codex-app-backups/`, and restarts Codex Desktop with `--x11`.

Security note: on Linux this fallback stores the ECDSA private key in:

```text
~/.local/state/codex-desktop/remote-control-device-keys.json
```

The file is written with mode `0600`, but it is not hardware-backed like the macOS implementation.

## 6. Run Codex Desktop

```bash
codex-desktop --x11
```

Then open:

```text
Settings -> Connections -> Control other devices
```

Click **Set up**. If authorization succeeds, the device appears as a remote-control environment.

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
rg -i "remote_control_client_enrollment|remote_control_authorize|device_key|finish_response|authorize_failed" \
  "$HOME/.cache/codex-desktop/launcher.log"
```

## Troubleshooting

- `Remote control device keys are only available on macOS`: run `bash scripts/apply-linux-remote-control-key-hotfix.sh`.
- `ssh: connect to host ... port 22: Connection refused`: unrelated saved SSH remote; fix or remove that SSH connection in Settings.
- `unsupported feature enablement auth_elicitation`: noisy but not the remote-control blocker in this setup.
- If the app does not stay open, clear stale PID files and relaunch:

```bash
rm -f "$HOME/.local/state/codex-desktop/app.pid" \
      "$HOME/.local/state/codex-desktop/webview.pid"
codex-desktop --x11
```
