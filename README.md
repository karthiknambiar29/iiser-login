# iiser-login

Automatic login for the IISER Bhopal campus network's captive portal (FortiGate), on Linux.

Normally, when your network session expires, Firefox shows **"Open network login page"** and you have to type your username and password again. `iiser-login` does that for you. It can run once on demand, or stay in the background: it keeps your session alive and logs you back in whenever you get disconnected.

## Features

- Detects the login page the same way Firefox does (`detectportal.firefox.com`).
- Finds the portal address automatically, so it keeps working if the gateway's host, IP or port changes.
- Reads the login form's hidden fields (`magic`, `4Tredir`) directly from the page.
- Handles special characters in passwords correctly.
- Watch mode checks every 60 seconds, visits the portal's keepalive link to keep the session from expiring, and logs in again if you are dropped anyway.
- Writes a log to `~/.cache/iiser-login.log` for troubleshooting.

## Requirements

- Linux with `bash`, `curl` and `grep` (with `-P` support, i.e. GNU grep)
- `systemd`, only for running in the background

On Debian/Ubuntu, `curl` is the only thing you may need to install:

```bash
sudo apt install curl
```

## Installation

**1. Get the script**

```bash
git clone https://github.com/karthiknambiar29/iiser-login.git
cd iiser-login
```

**2. Install it**

```bash
mkdir -p ~/.local/bin
install -m 755 iiser-login ~/.local/bin/iiser-login
```

Make sure `~/.local/bin` is on your `PATH`:

```bash
which iiser-login
```

If that prints nothing, add this line to `~/.bashrc` and open a new terminal:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

**3. Save your credentials**

The background service can't ask you for a password, so it reads your credentials from a file:

```bash
mkdir -p ~/.config/iiser-login
cat > ~/.config/iiser-login/credentials <<'EOF'
IISER_USER='your_username'
IISER_PASS='your_password'
EOF
chmod 600 ~/.config/iiser-login/credentials
```

Keep the single quotes around the values. `chmod 600` makes the file readable only by you.

> **Never commit this file to Git.** It lives in `~/.config`, outside the repository, for exactly this reason.

**4. Test it**

Log out of the portal (or wait until your session expires), then run:

```bash
iiser-login
```

You should see `Login successful.` If you're already connected, you'll see `Already online.` instead.

## Running it all the time (systemd)

Set up a background service so the login happens automatically at boot and after every disconnect.

**1. Create the service**

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/iiser-login.service <<'EOF'
[Unit]
Description=IISER captive portal auto-login
After=network-online.target

[Service]
ExecStart=%h/.local/bin/iiser-login --watch
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
EOF
```

**2. Enable and start it**

```bash
systemctl --user daemon-reload
systemctl --user enable --now iiser-login.service
```

**3. Keep it running when you're not logged in**

This is useful for machines you reach over SSH or that sit on the login screen:

```bash
sudo loginctl enable-linger $USER
```

**Managing the service**

| Task | Command |
|---|---|
| Check status | `systemctl --user status iiser-login` |
| Follow the log | `tail -f ~/.cache/iiser-login.log` |
| Restart (after updating the script) | `systemctl --user restart iiser-login` |
| Stop and disable | `systemctl --user disable --now iiser-login` |

## Checking that it works

**Is the service set up correctly?**

```bash
systemctl --user is-active iiser-login     # active
systemctl --user is-enabled iiser-login    # enabled
loginctl show-user $USER -p Linger         # Linger=yes
```

**Does the login work without a terminal?** This runs the script the way the service does, with no keyboard input:

```bash
env -i HOME=$HOME PATH=/usr/bin:/bin ~/.local/bin/iiser-login < /dev/null
```

It should print `Login successful.` or `Already online.` and never ask for a password.

**Does it recover from a logout?** Follow the log in one terminal:

```bash
tail -f ~/.cache/iiser-login.log
```

In another terminal, log out of the portal. Take the keepalive link from the log and change `keepalive` to `logout`:

```bash
grep 'Keepalive URL' ~/.cache/iiser-login.log | tail -1
curl -sk "https://172.31.1.254:1003/logout?XXXXXXXX"
```

Within about a minute the log should show:

```
Offline - logging in again.
Portal: https://172.31.1.254:1003/fgtauth?...
Login successful.
Keepalive URL: https://172.31.1.254:1003/keepalive?...
```

## Configuration

These optional settings go in the credentials file or in the environment:

| Variable | Default | Meaning |
|---|---|---|
| `IISER_USER` | *(asked)* | Portal username |
| `IISER_PASS` | *(asked)* | Portal password |
| `IISER_INSECURE` | `1` | `1` skips the portal's certificate check (see below); `0` enforces it |
| `IISER_CRED_FILE` | `~/.config/iiser-login/credentials` | Location of the credentials file |
| `IISER_LOG` | `~/.cache/iiser-login.log` | Location of the log file |

### About the certificate check

The portal is reached by its IP address (`172.31.1.254`), but its certificate is issued for `*.iiserb.ac.in`. The names don't match, so `curl` would reject the connection, just as Firefox shows a warning. For this reason the script skips the check by default (`IISER_INSECURE=1`). This is the same as clicking through Firefox's warning, and it is reasonable on the campus network.

If you want to keep the check, edit the script:

1. Add `--resolve gateway.iiserb.ac.in:1003:172.31.1.254` to the `CURL=(...)` line.
2. Add this line right after the `auth_url=$(...)` block:
   ```bash
   auth_url="${auth_url/172.31.1.254/gateway.iiserb.ac.in}"
   ```
3. Set `IISER_INSECURE=0` in your credentials file.

## Troubleshooting

Start with the log:

```bash
tail -30 ~/.cache/iiser-login.log
```

| Log message | Likely cause | Fix |
|---|---|---|
| `No portal redirect found` | The portal's redirect format changed, or the network isn't intercepting traffic | Check the saved raw probe response in the log. Run `curl -4 -sv http://detectportal.firefox.com/success.txt` while logged out |
| `SSL certificate problem` / `no alternative certificate subject name matches` | Certificate check enabled | Set `IISER_INSECURE=1` (see above) |
| `Login failed` followed by a portal page mentioning invalid credentials | Wrong username or password | Fix `~/.config/iiser-login/credentials` |
| Script asks for a username when run by the service | Credentials file missing or unreadable | Recreate it (installation step 3) |
| Old behaviour after updating | Old copy still installed or shadowing the new one | Run `type -a iiser-login`, reinstall, then `systemctl --user restart iiser-login` |

## Uninstall

```bash
systemctl --user disable --now iiser-login
rm ~/.config/systemd/user/iiser-login.service
rm ~/.local/bin/iiser-login
rm -r ~/.config/iiser-login
rm ~/.cache/iiser-login.log
```

## Security notes

- Your password is stored in plain text in `~/.config/iiser-login/credentials`. It is protected only by file permissions (`chmod 600`), so don't use this on a shared account.
- The password is never written to the log.
- Use this only with your own account, and in line with the institute's network policy.
