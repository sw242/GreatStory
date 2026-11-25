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

## Install root certificate of the CA on the client 

### Transfer root certificat
`pct pull 101 /root/.step/certs/root_ca.crt /root/root_ca.crt`


### Root-CA im Debian-Zertifikatsspeicher installieren 
```
cp /root/root_ca.crt /usr/local/share/ca-certificates/step-home-ca.crt
update-ca-certificates
```

## Create certificate for the client
- Create certificate on the CA node for the client proxmox10.fritz.box \
with a validity of 90 days. This will create the named crt and the key file<br>
```
step ca certificate \
  --not-after=2160h \
  proxmox10.fritz.box \
  proxmox10.crt \
  proxmox10.key
```
- Check the generated certificate, e. g. the validity<br>
`step certificate inspect /root/certs/proxmox10.crt`


- The pveserver will use the certificate in the communication with the web browser. Therefore, we need to install them on the proxmox server (proxmox10)

- On proxmox10, copy the certificates from the CA node (101) and store them \
under /root<br>
```
pct pull 101 /root/certs/proxmox10.crt /root/proxmox10.crt
pct pull 101 /root/certs/proxmox10.key /root/proxmox10.key
```
- Now install (make a backup of the existing certificates):<br>
```
cp /root/proxmox10.crt /etc/pve/local/pve-ssl.pem
cp /root/proxmox10.key /etc/pve/local/pve-ssl.key
```
- Restart the web proxy:<br>
`systemctl restart pveproxy`

- Check status:<br>
`systemctl status pveproxy`

- Call proxmox10 pveserver from web browser:<br>
`https://proxmox10.fritz.box:8006`
- If everything is correct, there will be no data protection error and the \
browser will show the "lock" icon

## Setup ACME on step CA
### Activate ACME on CA
- Check if ACME is already activated in ca.json:<br>
`grep -i acme ~/.step/config/ca.json`
- If no output, ACME provisioner needs to be added/created

- Create ACME provisioner<br>
`step ca provisioner add acme --type ACME`

- Verify:<br>
`step ca provisioner list`
- Output should show acme beside the JWK provisioner

- Then, restart CA:<br>
`systemctl restart step-ca`

- Test on command line:<br>
`curl -k https://ca.fritz.box/acme/acme/directory`
- Should return JSON data, no error 404

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
