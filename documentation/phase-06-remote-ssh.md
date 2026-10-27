# Phase 6 — Remote SSH administration

[Previous: networking toolkit](phase-05-networking-toolkit.md) · [Next: multi-server networking](phase-07-multi-server-networking.md)

**Run on:** Ubuntu and **macOS Terminal**, as labeled. **Goal:** Connect to the correct VM, install a dedicated public key, and inspect authentication logs. Keep the UTM console available.

## Step 1 — Prepare the Ubuntu SSH server

In **Ubuntu**:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo sshd -t
systemctl status ssh.service ssh.socket --no-pager
sudo ss -lntp '( sport = :22 )'
ip -br -4 addr
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Some Ubuntu releases use socket activation: `ssh.socket` can listen before `ssh.service` is active. Check the actual listener. If neither is running, start `ssh.service` with `sudo systemctl start ssh.service`, then recheck TCP 22. Record the host-key fingerprint and the current management IPv4 address.

## Step 2 — Connect from the Mac

In **macOS Terminal**, substitute your username and current VM address:

```bash
LAB_USER=your_ubuntu_username
LAB_IP=192.168.64.3
ssh "$LAB_USER@$LAB_IP"
```

On the first connection, compare the displayed ED25519 fingerprint with Step 1 before accepting the host key. Enter your Ubuntu password interactively. If a known host key changes, verify the VM's console fingerprint and determine why before editing `known_hosts`.

Inside the remote session:

```bash
whoami
hostname
printf '%s\n' "$SSH_CONNECTION"
exit
```

**Checkpoint:** You reached `dc-network-01` from the Mac and can identify both ends of the SSH session.

## Step 3 — Create a dedicated client key

In **macOS Terminal**:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_dc_lab -C dc-lab
```

Choose a passphrase. If the file already exists, reuse it or choose a different filename; do not overwrite an existing key. The `.pub` file is the public key. The file without `.pub` is private and stays on the Mac.

## Step 4 — Install the public key

In the same **macOS Terminal** with `LAB_USER` and `LAB_IP` still set:

```bash
cat ~/.ssh/id_ed25519_dc_lab.pub | ssh "$LAB_USER@$LAB_IP" \
  'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys'
```

Run this once per VM. It appends your public key to the Ubuntu account's allowed keys. Keep the console and password login available while learning; this guide does not change SSH authentication policy.

## Step 5 — Confirm key authentication

In **macOS Terminal**:

```bash
ssh -i ~/.ssh/id_ed25519_dc_lab -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no "$LAB_USER@$LAB_IP"
```

The private key's local passphrase prompt is different from the Ubuntu account password. Successful login with password and keyboard-interactive authentication disabled confirms the public-key path.

In **Ubuntu**, inspect recent logs:

```bash
sudo journalctl -u ssh.service --since '10 minutes ago' --no-pager
sudo journalctl _COMM=sshd --since '10 minutes ago' --no-pager
```

Look for a successful public-key authentication event and the client address. If your release writes authentication events to `/var/log/auth.log`, inspect that file as well.

## Step 6 — Verify console recovery and diagnose failures

1. Open the VM in UTM and sign in locally, even while SSH is working.
2. Keep that console available before later firewall, route, or service changes.
3. For a timeout, check the current VM address, UTM network, route, and firewall.
4. For connection refused, inspect the TCP 22 listener and SSH socket/service.
5. For permission denied, confirm username, key selection, file ownership/permissions, and authentication logs. Use `ssh -v -i ~/.ssh/id_ed25519_dc_lab "$LAB_USER@$LAB_IP"` on the Mac when needed.

**Checkpoint:** Both console login and remote key-based login work. Repeat Steps 1–6 for the two VMs created in Phase 7. Copy only public keys to the VMs.

Reference: [Ubuntu OpenSSH server guide](https://ubuntu.com/server/docs/how-to/security/openssh-server/).
