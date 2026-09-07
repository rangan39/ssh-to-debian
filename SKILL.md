---
name: ssh-to-debian
description: Walks through connecting a Mac to a Debian (or other Linux) machine over SSH — installing and enabling OpenSSH server on the Debian side, finding its IP, connecting and disconnecting from the Mac's built-in Terminal, setting up passwordless login with SSH keys, adding a short host alias in ~/.ssh/config, and troubleshooting "connection refused" or timeout errors. Use this whenever the user wants to SSH into a home server, NAS, Raspberry Pi, homelab box, or remote Linux machine from their Mac, mentions setting up remote/headless access, asks about ssh-keygen or ssh-copy-id, wants a shorter `ssh` command instead of typing an IP every time, or is stuck on a failed SSH connection — even if they don't say "Debian" or "SSH" by name.
---

# Connect a Mac to a Debian Machine over SSH

This skill covers first-time setup of SSH between a Mac and a Debian (or Debian-derived, e.g. Ubuntu/Raspberry Pi OS) machine, and the common follow-on workflows people reach for right after: skipping the password, shortening the command, and unsticking a failed connection.

Commands below use `<username>` and `<debian-ip>` as placeholders — substitute the actual account name and address for this user's machine before handing over or running any command. Never leave the angle brackets in a command you actually give the user to run.

**Default to explaining/handing over commands rather than running them yourself.** Most of Step 1 requires `sudo` and runs *on the Debian machine*, which you typically don't have a terminal session on — and even when you do, installing packages and enabling system services are exactly the kind of system-level changes that should go through the user, interactively, so they see what's happening and can enter their own password. Give the user the commands to paste into their own terminal (Mac Terminal for Step 2, a Debian console/keyboard-and-monitor session or an existing session for Step 1) rather than trying to execute Step 1 for them. If you're already in a shell that's genuinely on the Debian box (e.g. the user has SSH'd in and handed you the terminal), it's fine to run the read-only checks (`ip a`, `systemctl status ssh`) yourself to help diagnose — just don't silently run `sudo apt install` or flip service state without the user asking you to.

## Step 1: Prepare the Debian machine

Run these on the Debian machine itself (not the Mac):

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

`enable --now` both starts the SSH daemon immediately and makes it start on every boot — that second part matters if this is a headless box the user won't have a monitor plugged into next time.

Then find the IP address to connect to:

```bash
ip a
```

Look for the `inet` line under the active network interface (commonly `eth0`, `wlan0`, or an `enp*`/`wlp*` name on newer kernels) — it'll usually start with `192.168.` or `10.`. That's the address the Mac will connect to. Note it's worth checking whether the router hands out a *static* or *reserved* IP for this machine, since a DHCP lease can change later and quietly break the connection — mention this if the user is setting up something they'll come back to repeatedly (like a home server).

## Step 2: Connect from the Mac

The Mac's Terminal app ships with an SSH client already — no install needed.

```bash
ssh <username>@<debian-ip>
```

For example, if the account is `alice` and the machine's address is `192.168.1.50`: `ssh alice@192.168.1.50`

First-time connections show a host key fingerprint prompt; typing `yes` accepts it and pins that fingerprint for future connections (this is what protects against a silent man-in-the-middle later — if the fingerprint ever changes unexpectedly on a *subsequent* connection, that's worth flagging to the user rather than dismissing). Then it prompts for the Debian account's password — the terminal shows no cursor movement or asterisks while typing, which is normal, not a hang.

## Step 3: Disconnect

```bash
exit
```

## Optional: passwordless login with SSH keys

Worth suggesting once the user has the basic connection working and plans to use it more than once or twice — typing a password every time gets old fast, and keys are also what's needed for anything scripted (rsync jobs, VS Code Remote-SSH, etc.) since those can't sit and wait for an interactive password.

On the Mac:

```bash
ssh-keygen -t ed25519
ssh-copy-id <username>@<debian-ip>
```

`ssh-keygen` only needs to be run once ever (skip it if `~/.ssh/id_ed25519` already exists — reusing the same key across machines is fine). `ssh-copy-id` will ask for the Debian password one last time, then copies the public key into `~/.ssh/authorized_keys` on the Debian side. After that, `ssh <username>@<debian-ip>` logs straight in.

## Optional: a short alias instead of the full command

Once the user is tired of typing the username/IP every time, add a `Host` block to `~/.ssh/config` on the Mac:

```
Host debian
    HostName <debian-ip>
    User <username>
```

From then on, `ssh debian` does the same thing as the full command. This is purely a Mac-side convenience file — nothing needs to change on the Debian machine. If the user has several such machines, this is also where each one's identity file (`IdentityFile ~/.ssh/id_ed25519`) or a non-default port can live, so it's the natural place to point them if they ask "how do I avoid retyping this."

## Troubleshooting a failed connection

Work through these roughly in order — each rules out one layer:

1. **"Connection refused"** almost always means the SSH server isn't running on the Debian box. On the Debian machine: `sudo systemctl status ssh` — if it's not `active (running)`, `sudo systemctl enable --now ssh` (from Step 1) fixes it.
2. **Timeout / hangs with no response** usually means a firewall is blocking port 22, or the Mac and Debian machine aren't actually on the same reachable network. Check `sudo ufw status` on Debian (if ufw is in use) — `sudo ufw allow ssh` opens it if it's active and blocking. Also confirm both machines are on the same LAN/subnet, since a timeout with the service otherwise healthy is often a network topology issue, not an SSH one.
3. **Wrong IP address** — DHCP leases change. Rerun `ip a` on the Debian machine to confirm the address hasn't moved; this is the single most common cause after everything worked once before.
4. **Wrong username or "permission denied"** — confirm the account name being used actually exists on the Debian machine (`ssh` errors here look similar to connection problems but the socket-level connection actually succeeded).
