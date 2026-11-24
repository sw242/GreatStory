# Install step-cli und ca
step-cli = step command line interface

## Step client
### Download of package
`curl -LO https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.31.0/step_linux_0.31.0_amd64.tar.gz`

### Extract
`tar -xzf step_linux_0.31.0_amd64.tar.gz`

### Install system-wide
__Copy binary to /usr/local/bin__<br>
`cp step_0.31.0/bin/step /usr/local/bin/`

__Make executable__<br>
`chmod +x /usr/local/bin/step`

### Test 
__Show version info__<br>
`step version`

## Step CA
### Download of package
TODO

### Auto start per systemd 
__Strategy__: store CA password in password file.

__Create folder for password file__<br>
`mkdir -p /etc/step-ca` 

__Create password file and enter password__<br>
`nano /etc/step-ca/password.txt` 

__Limit access to owner 'root'__<br>
`chmod 600 /etc/step-ca/password.txt`

#### Test start from command line
`step-ca ~/.step/config/ca.json --password-file /etc/step-ca/password.txt`
(CA should start without asking for password) 

#### Create systemd configuration
`nano /etc/systemd/system/step-ca.service`

```
[Unit] 
Description=Smallstep Certificate Authority 
After=network.target 

[Service] 
Type=simple 
ExecStart=/usr/local/bin/step-ca /root/.step/config/ca.json \ 
  --password-file /etc/step-ca/password.txt 
Restart=always 
RestartSec=5 

[Install] 
WantedBy=multi-user.target 
```

#### Load & activate 
```
systemctl daemon-reload 
systemctl enable step-ca 
systemctl start step-ca 
```

#### Check status 
`systemctl status step-ca `

## Install root certificate of the CA on the clientinstallieren 

### Transfer root certificat
`pct pull 101 /root/.step/certs/root_ca.crt /root/root_ca.crt`


### Root-CA im Debian-Zertifikatsspeicher installieren 
```
cp /root/root_ca.crt /usr/local/share/ca-certificates/step-home-ca.crt
update-ca-certificates
```

## Setup ACME on step CA
### Preconditions
- ACME activated on CA<br>
Test -> liefert JSON antwort (sonst nur bei curl -k …)<br>
`curl https://ca.fritz.box/acme/acme/directory`

- curl and socat installed

### Install acme.sh on node
- Download and install<br>
`curl https://get.acme.sh | sh -s email=stefan.wehrmann72@googlemail.com`

- Check version<br>
`~/.acme.sh/acme.sh --version`

### Register account on Step-CA
```
~/.acme.sh/acme.sh \
  --register-account \
  --server https://ca.fritz.box/acme/acme/directory \
  -m stefan.wehrmann72@googlemail.com
```
Output:<br>
```
[Tue Nov 24 11:19:38 PM CET 2026] Account key creation OK.
[Tue Nov 24 11:19:38 PM CET 2026] Registering account: https://ca.fritz.box/acme/acme/directory
[Tue Nov 24 11:19:38 PM CET 2026] Registered
[Tue Nov 24 11:19:39 PM CET 2026] ACCOUNT_THUMBPRINT='wjOFS0uillJEZCzrTFcicnFL38aGTgC-X-3Rdvnjr6s'
```

### Request first ACME certificate
```
~/.acme.sh/acme.sh \
  --issue \
  --standalone \
  -d proxmox10.fritz.box \
  --server https://ca.fritz.box/acme/acme/directory
```

### Install cert
```
~/.acme.sh/acme.sh \
  --install-cert \
  -d proxmox10.fritz.box \
  --fullchain-file /etc/pve/local/pve-ssl.pem \
  --key-file /etc/pve/local/pve-ssl.key \
  --reloadcmd "systemctl restart pveproxy"
```
