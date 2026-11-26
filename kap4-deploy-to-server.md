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

