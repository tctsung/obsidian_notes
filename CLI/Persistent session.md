---
created: 2025-09-28T20:03
updated: 2026-11-17T14:18
---

## Intro
- usage: allow persistent code run in **<span style="color:rgb(255, 0, 0)">terminal</span>
- can create/detach/manage multiple shell session (each is a **screen**)


## tmux
- create a new session
```bash
tmux new -s <name>   # new session with custom name
```
- detach session (code will keep running)
```
Ctrl+B, then D   # press Ctrl+B, release, then press D
```
- end session (running code will stop)
```
exit                          # inside the session
tmux kill-ses -t <name>       # end a specific session (kill-ses = kill-session)
```
- resume session
```
tmux ls             # list all sessions
tmux a -t <name>    # resume specific session
```
- scroll up output (mouse wheel doesn't work by default)
```
Ctrl+B, then [   # enter scroll mode, use ↑↓ / PgUp PgDn; Q to exit
```

## screen
- create a new session
	- can create multiple sessions if needed
```bash
screen             # default
screen -S <name>   # new session with custom name
```
- detach session (code will keep running)
```
screen -d   # detach current session
Ctrl+A, D   # detach current session
```
- end session (running code will stop)
```
screen -X quit   # end current attached session
screen -X -S <session id> quit # end a specific session
pkill screen  # end all session
```
- resume session
```
screen -ls  # list out all existing session ID & names
screen -r <name or session id>  # resume specific session
```

