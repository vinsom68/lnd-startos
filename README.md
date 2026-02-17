# Wrapper for LND

This project wraps [LND](https://github.com/lightningnetwork/lnd) for StartOS. The Lightning Network Daemon (lnd) - is a complete implementation of a Lightning Network node.

## Build environment
Before building the LND package, your build environment must be setup for building StartOS services. Instructions for setting up the proper build environment can be found in the [Developer Docs](https://docs.start9.com/latest/developer-docs/packaging).

## Dependencies

- [deno](https://deno.land/)
- [docker](https://docs.docker.com/get-docker)
- [docker-buildx](https://docs.docker.com/buildx/working-with-buildx/)
- [make](https://www.gnu.org/software/make/)
- [start-sdk](https://github.com/Start9Labs/start-os/blob/v0.3.5.1/core/install-sdk.sh)
- [yq (version 4)](https://mikefarah.gitbook.io/yq)

## Cloning

Clone the project locally.

```
git clone git@github.com:Start9Labs/lnd-startos.git
cd lnd-startos
```

## Building

To build the project run the command: `make`

Alternatively the package can be built for individual architectures by specifying the architecture as follows:

```
make x86
```

or

```
make arm
```

## Installing (on StartOS)

```
start-cli auth login
#Enter your StartOS password
start-cli --host https://server-name.local package install lnd.s9pk
```

If you already have your `start-cli` config file setup with a default `host`, you can install simply by running:

```
make install
```

**Tip:** You can also install the lnd.s9pk using **Sideload Service** under the **StartOS > SETTINGS** section.

## Verify Install

Go to your StartOS Services page, select **LND**, configure and start the service.

**Done!**

## Expose LND External IP via External VPS (Safer Reverse Tunnel)

This guide uses a cheap VPS (for example RackNerd) as a public endpoint and a reverse SSH tunnel from StartOS to expose `lnd` externally.

### 1. Harden SSH on VPS

Reference: https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo nano /etc/ssh/sshd_config
```

Set:

```conf
GatewayPorts clientspecified
```

Restart SSH:

```bash
sudo systemctl restart ssh || sudo systemctl restart sshd
```

### 2. Configure UFW on VPS

```bash
sudo apt install ufw
sudo ufw allow 22/tcp
sudo ufw allow 9735/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

### 3. Configure Fail2Ban on VPS

```bash
sudo apt install fail2ban -y
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Add/update:

```conf
[sshd]
enabled  = true
port     = ssh
logpath  = /var/log/auth.log
backend  = systemd
maxretry = 5
bantime  = 10m
findtime = 10m
```

Check status:

```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

### 4. Configure Reverse AutoSSH from Start9

On Start9:

```bash
sudo -i
```

Run the following to switch into the persistent chroot environment:

```bash
sudo /usr/lib/startos/scripts/chroot-and-upgrade
```

Install autossh:

```bash
sudo apt install autossh -y
```

Create SSH keypair (example key path):

```bash
ssh-keygen -f /root/.ssh/start9racknerd
```

Copy key to VPS:

```bash
cat /root/.ssh/start9racknerd.pub | ssh root@your.vps.ip.addr "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Optional key auth test:

```bash
ssh -i /root/.ssh/start9racknerd -p 22 root@your.vps.ip.addr
```

Test reverse tunnel from Start9:

```bash
autossh -M 0 -N -R 0.0.0.0:9735:lnd.embassy:9735 -i /root/.ssh/start9racknerd -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -o "ExitOnForwardFailure yes" -p 22 root@your.vps.ip.addr
```

Create a systemd service:

```bash
cat > /etc/systemd/system/reverse.autossh.service <<'EOL'
[Unit]
Description=Reverse Autossh service
Wants=podman.service
After=podman.service

[Service]
Type=simple
Restart=always
RestartSec=3
Environment=AUTOSSH_POLL=60
ExecStart=/usr/bin/autossh -M 0 -N -R 0.0.0.0:9735:lnd.embassy:9735 -i /root/.ssh/start9racknerd -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -o "ExitOnForwardFailure yes" -p 22 root@your.vps.ip.addr

[Install]
WantedBy=multi-user.target
EOL
```

Enable and check:

```bash
systemctl daemon-reload
systemctl enable --now reverse.autossh
systemctl status reverse.autossh
```

Exit chroot (this reboots StartOS). Type `exit` and allow reboot to complete:

```bash
exit
```

After reboot, set the LND `External IP` setting to your VPS public IP and port `9735`.

### 5. Final SSH hardening

Confirm VPS SSH password login remains disabled after key setup:

```conf
PasswordAuthentication no
```

Then apply and restart SSH on VPS:

```bash
sudo systemctl restart ssh || sudo systemctl restart sshd
```
