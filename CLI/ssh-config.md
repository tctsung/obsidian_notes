# SSH Config

- Config: `~/.ssh/config` (per-user) or `/etc/ssh/ssh_config` (system-wide)
- Keys: `~/.ssh/id_rsa`, `~/.ssh/id_ed25519` (private), `.pub` (public)

---

## Host Alias — `ssh <alias>` instead of full hostname

```
Host <alias>
  HostName <actual-hostname>
  User <username>
  ProxyCommand <proxy-binary> %h
  ServerAliveInterval <seconds>
  ServerAliveCountMax <count>
```

Example:
```
Host cloud
  HostName <dev-desktop-hostname>
  User <username>
  ProxyCommand=/usr/local/bin/wssh proxy %h
  ServerAliveInterval 15
  ServerAliveCountMax 44
  LocalForward 7777 localhost:7777
```
→ `ssh cloud` connects to the dev desktop through wssh proxy, and auto-forwards port 7777.

---

## EC2 GPU Instance — SSH over SSM (no public IP, no VPN)

The GPU instance has no public IP. SSH tunnels through SSM instead.

### Prerequisites
1. `ada credentials update --account=<account-id> --provider=conduit --role=<role-name> --once`
2. Public key must exist in `/home/ec2-user/.ssh/authorized_keys` on the instance

### Config
```
Host p4
  HostName <instance-id>
  User ec2-user
  IdentityFile ~/.ssh/id_ecdsa
  ProxyCommand aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p --region ap-northeast-1
  StrictHostKeyChecking no
```

### Usage
```bash
ada credentials update --account=<account-id> --provider=conduit --role=<role-name> --once
ssh p4                              # interactive shell
scp file.py p4:~/<username>/           # copy file
rsync -avz ./project/ p4:~/<username>/project/   # sync directory
```

### How it works
- `ProxyCommand` uses `aws ssm start-session` as the transport layer (replaces TCP)
- SSM authenticates via your local AWS credentials (from `ada`), not SSH keys
- SSH keys authenticate you to `sshd` on the instance (standard SSH auth)
- No security group changes, no public IP, no VPN needed

### Push a new user's key (via SSM send-command)
```bash
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["echo '\''<pubkey>'\'' >> /home/ec2-user/.ssh/authorized_keys"]' \
  --region ap-northeast-1
```

---

## Directives

| Directive | Meaning |
|---|---|
| `Host <pattern>` | Block header — alias (`cloud`), exact host, or glob (`*.amazon.com`) |
| `HostName` | Real address (required when `Host` is an alias) |
| `User` | Login username |
| `ProxyCommand` | Route through a proxy instead of direct TCP. `%h`=host, `%p`=port |
| `ServerAliveInterval` | Keepalive ping every N sec |
| `ServerAliveCountMax` | Drop after N missed pings. Max idle = `Interval × Count` |
| `ForwardAgent yes` | Forward local SSH agent to remote (for git, nested ssh) |
| `AddKeysToAgent yes` | Auto-add key to agent on first use |
| `IdentityFile` | Private key path for this host |
| `IdentitiesOnly yes` | Only use specified key, skip agent keys |
| `LocalForward <local-port> <remote-host>:<remote-port>` | Auto port-forward on connect. Access remote service at `localhost:<local-port>` |
| `StrictHostKeyChecking` | `accept-new` = trust first connect, reject if changed |
| `Match all` | Catch-all block, goes at bottom |

## Eval Order
- Top-to-bottom, **first match wins** per directive
- Specific hosts above wildcards, `Match all` last

## Debug
```bash
ssh -G <alias>   # dry-run: show resolved config
ssh -v <alias>   # verbose connection log
```

---

## Background Tunnels — `ssh -f -N`

| Flag | Meaning |
|---|---|
| `-f` | Fork to background after authentication |
| `-N` | No remote command — just hold the tunnel open |
| `-L` | Local port forward (see `LocalForward` above) |

```bash
# Background tunnel: forward local 8080 → remote localhost:8080
ssh -f -N -L 8080:localhost:8080 cloud

# With SOCKS proxy (dynamic forward)
ssh -f -N -D 1080 cloud

# Kill it later
pkill -f "ssh -f -N.*cloud"
# or find PID:  ps aux | grep "ssh -f -N"
```

→ `-f -N` is the standard way to run a port-forward tunnel in the background without opening a shell.

---

## AutoSSH — auto-reconnecting tunnels

- `autossh` wraps `ssh` and monitors the connection; restarts it automatically if it drops
- Install: `brew install autossh`
- Use `-M 0` to skip the monitor port and rely on `ServerAliveInterval` keepalives instead
- Supports all normal `ssh` flags (`-L`, `-D`, `-N`, `-f`, etc.)

```bash
# Auto-reconnecting local forward (backgrounds itself with -f)
autossh -M 0 -f -N -L 8080:localhost:8080 cloud

# Auto-reconnecting SOCKS proxy
autossh -M 0 -f -N -D 1080 cloud

# Kill: pkill -3 autossh
```

- Prefer `autossh` over plain `ssh -f -N` for long-lived tunnels (e.g., cloud desktop, port forwards that need to survive Wi-Fi drops)
