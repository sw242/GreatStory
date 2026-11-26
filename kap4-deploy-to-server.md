# Deploy to server


Since you already have:

✅ Ubuntu LXC
✅ nginx
✅ git

you mainly need:

Python virtual environment
Clone the repo
Install dependencies
Configure Gunicorn
Configure Nginx reverse proxy
Create a systemd service

## Install sqlite
`apt update`<br>
`apt install sqlite3`

## Install Python requirements

- First verify Python:<br>
`python3 --version`

- Install required packages:<br>
```
apt update
apt install -y python3 python3-venv python3-pip build-essential
```

## Clone the private repository

For a private repository, SSH access is usually easiest.

- Generate a key:
`ssh-keygen -t ed25519`<br>
(ed25519 specifies the type of key)
- Display the key:
`cat ~/.ssh/id_ed25519.pub`

- Add the public key to GitHub: GitHub
→ Settings
→ SSH and GPG Keys
→ New SSH Key

- Test:<br>
`ssh -T git@github.com`<br>
(when connecting for the first time, there will be a warning and GitHub 
asks if the SSH host key is trusted. This needs to be confirmed by typing 'yes')

- Create folder and clone code from git:
```
mkdir -p /opt/retinento
cd /opt/retinento

git clone git@github.com:sw242/Retinento.git .
```

- Copy the sqlite database from dev workstation
  - Name of database: vocable_trainer.db
  - Location on dev workstation: /home/stefan/Retinento/instance/vocable_trainer.db
```
scp /home/stefan/Retinento/instance/vocable_trainer.db \
    root@retinento-test.fritz.box:/opt/retinento/instance/
```
- Check access rights. The db needs read and write access for the user root.<br>
`ls -l /opt/retinento/instance`<br>
Expected output:
```
root@retinento-test:/opt/retinento/instance# ls -lisa
total 2144
5459    4 drwxr-xr-x 2 root root    4096 Nov 26 12:18 .
  23    4 drwxr-xr-x 9 root root    4096 Nov 25 22:11 ..
2229 2136 -rw-r--r-- 1 root root 2187264 Nov 26 12:18 vocable_trainer.db
root@retinento-test:/opt/retinento/instance# 
```

- Test access to db:<br>
`sqlite3 /opt/retinento/instance/vocable_trainer.db ".tables"`<br>
In the output, the tables of the db will be shown:
```
root@retinento-test:/opt/retinento/instance# sqlite3 /opt/retinento/instance/vocable_trainer.db ".tables"
card_progress           cards                   library_sets          
card_sets               course_sets             study_runs            
card_study_preferences  courses                 users                 
root@retinento-test:/opt/retinento/instance# 
```


## Create a Python virtual environment
- Install venv
```
cd /opt/retinento

python3 -m venv venv
source venv/bin/activate
```

- Upgrade pip:<br>
`pip install --upgrade pip`

- Install project dependencies:<br>
`pip install -r requirements.txt`

## Test locally first

- Run flask inside the venv:<br>
`flask run --host=0.0.0.0`

- Open in web browser:<br>
http://192.168.10.12:5000

Verify the application works before adding nginx.

## Install Gunicorn
- In venv run:<br>
`pip install gunicorn`

- Start web app in gunicorn:<br>
`gunicorn --bind 0.0.0.0:8000 "app:create_app()"`<br>
(or whatever the correct module path is)

- Test:<br>
http://192.168.10.12:8000<br>
http://retinento-test.fritz.box:8000

## Create a systemd service for gunicorn

- Create systemd file<br>
`nano /etc/systemd/system/retinento-test.service`<br>
with content:
```
[Unit]
Description=Retinento Test Flask Application
After=network.target

[Service]
Type=simple
User=root
Group=root

WorkingDirectory=/opt/retinento

Environment="PATH=/opt/retinento/venv/bin"

ExecStart=/opt/retinento/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:8000 \
    "app:create_app()"

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

- Load, enable and start system.d service
```
systemctl daemon-reload
systemctl enable retinento-test
systemctl start retinento-test
```

- Check:
`systemctl status retinento-test`

- Get logs:
`journalctl -u retinento -f`

## Configure nginx for https

- Preconditions
  - Certificates created: retinento-test.crt, retinento-test.key
  - Certificated copied to: /etc/nginx/certs
```
root@retinento-test:/etc/nginx/certs# ls -lisa
total 16
1273 4 drwx------ 2 root root 4096 Nov 25 21:40 .
 584 4 drwxr-xr-x 9 root root 4096 Nov 25 21:39 ..
2347 4 -rw-r--r-- 1 root root 1620 Nov 25 21:40 retinento-test.crt
2358 4 -rw-r--r-- 1 root root  227 Nov 25 21:40 retinento-test.key
root@retinento-test:/etc/nginx/certs# 
```

- Create nginx configuration file for retinento-test<br>
`nano /etc/nginx/sites-available/retinento-test`

- Content:<br>
```
server {
    listen 80;
    server_name retinento-test.fritz.box;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name retinento-test.fritz.box;

    ssl_certificate     /etc/nginx/certs/retinento-test.crt;
    ssl_certificate_key /etc/nginx/certs/retinento-test.key;

    location / {
        proxy_pass http://127.0.0.1:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

- Enable site:<br>
`ln -s /etc/nginx/sites-available/retinento-test /etc/nginx/sites-enabled/`

- Test correctness of configuration:<br>
`nginx -t`

- Reload nginx:<br>
`systemctl reload nginx`

- Test in web browser:<br>
https://retinento-test.fritz.box

## Setup ACME certificate update

root@retinento-test:~# systemctl stop nginx
root@retinento-test:~# acme.sh \
  --server https://ca.fritz.box/acme/acme/directory \
  --issue \
  -d retinento-test.fritz.box \
  --standalone
[Thu Nov 26 17:32:19 UTC 2026] Using CA: https://ca.fritz.box/acme/acme/directory
[Thu Nov 26 17:32:19 UTC 2026] Standalone mode.
[Thu Nov 26 17:32:19 UTC 2026] Account key creation OK.
[Thu Nov 26 17:32:19 UTC 2026] Registering account: https://ca.fritz.box/acme/acme/directory
[Thu Nov 26 17:32:19 UTC 2026] Registered
[Thu Nov 26 17:32:19 UTC 2026] ACCOUNT_THUMBPRINT='WPA4Quco1fOuUXO7YRl3I7ABiY07xlIl-a1Z65uIqnM'
[Thu Nov 26 17:32:19 UTC 2026] Creating domain key
[Thu Nov 26 17:32:19 UTC 2026] The domain key is here: /root/.acme.sh/retinento-test.fritz.box_ecc/retinento-test.fritz.box.key
[Thu Nov 26 17:32:19 UTC 2026] Single domain='retinento-test.fritz.box'
[Thu Nov 26 17:32:20 UTC 2026] Getting webroot for domain='retinento-test.fritz.box'
[Thu Nov 26 17:32:20 UTC 2026] Verifying: retinento-test.fritz.box
[Thu Nov 26 17:32:20 UTC 2026] Standalone mode server
[Thu Nov 26 17:32:21 UTC 2026] Success
[Thu Nov 26 17:32:21 UTC 2026] Verification finished, beginning signing.
[Thu Nov 26 17:32:21 UTC 2026] Let's finalize the order.
[Thu Nov 26 17:32:21 UTC 2026] Le_OrderFinalize='https://ca.fritz.box/acme/acme/order/0GwZXwcTiByBvAx9Do8pjOE6FOIfQoKy/finalize'
[Thu Nov 26 17:32:21 UTC 2026] Downloading cert.
[Thu Nov 26 17:32:21 UTC 2026] Le_LinkCert='https://ca.fritz.box/acme/acme/certificate/QDPOBhIzQSpOVlghZ2toHvFiXjXWUmvZ'
[Thu Nov 26 17:32:21 UTC 2026] Cert success.
-----BEGIN CERTIFICATE-----
MIICITCCAcegAwIBAgIRALm7caUW1BD7tRBy6oljspkwCgYIKoZIzj0EAwIwSDEa
MBgGA1UEChMRU21hbGxzdGVwIEhvbWUgQ0ExKjAoBgNVBAMTIVNtYWxsc3RlcCBI
b21lIENBIEludGVybWVkaWF0ZSBDQTAeFw0yNjExMjYxNzMxMjBaFw0yNzAyMjQx
NzMyMjBaMCMxITAfBgNVBAMTGHJldGluZW50by10ZXN0LmZyaXR6LmJveDBZMBMG
ByqGSM49AgEGCCqGSM49AwEHA0IABMRDb+VFE/nGAnKUaIVeirKbvVqvB0eeBg1g
Vt7mWm9W82wD57Vw7arXymok29kGOj+fnwQeoKGnodYxkGu+CO+jgbYwgbMwDgYD
VR0PAQH/BAQDAgeAMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNV
HQ4EFgQUfmdXiLFWw0l0DzS/vYObqV8E2qIwHwYDVR0jBBgwFoAUdAWmnnpw4UQ2
D+L/ocw90Pve4cMwIwYDVR0RBBwwGoIYcmV0aW5lbnRvLXRlc3QuZnJpdHouYm94
MB0GDCsGAQQBgqRkxihAAQQNMAsCAQYEBGFjbWUEADAKBggqhkjOPQQDAgNIADBF
AiEAvTTBEAZwhn5ECoJ8FwHGdtrwbpLHSDsWqO5+sv/gPkQCIHJ/81J9ycW0LPOb
lJhzwCjiQv2LvSDmOqHSU+IbuW+q
-----END CERTIFICATE-----
[Thu Nov 26 17:32:21 UTC 2026] Your cert is in: /root/.acme.sh/retinento-test.fritz.box_ecc/retinento-test.fritz.box.cer
[Thu Nov 26 17:32:21 UTC 2026] Your cert key is in: /root/.acme.sh/retinento-test.fritz.box_ecc/retinento-test.fritz.box.key
[Thu Nov 26 17:32:21 UTC 2026] The intermediate CA cert is in: /root/.acme.sh/retinento-test.fritz.box_ecc/ca.cer
[Thu Nov 26 17:32:21 UTC 2026] And the full-chain cert is in: /root/.acme.sh/retinento-test.fritz.box_ecc/fullchain.cer
root@retinento-test:~# 



