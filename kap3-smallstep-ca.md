# Step-cli und ca installieren 

## Step client
### Download of package
curl -LO https://dl.smallstep.com/gh-release/cli/gh-release-header/v0.31.0/step_linux_0.31.0_amd64.tar.gz 

### Extract
tar -xzf step_linux_0.31.0_amd64.tar.gz 

### Install system-wide
__Copy binary to /usr/local/bin__<br>
cp step_0.31.0/bin/step /usr/local/bin/ 

__Make executable__<br>
chmod +x /usr/local/bin/step 

### Test 
__Show version info__<br>
step version 

## Step CA
### Download of package
TODO

# Auto start per systemd 
## Strategy: store CA password in password file.

__Create folder for password file__<br>
mkdir -p /etc/step-ca 

__Create password file and enter password__<br>
nano /etc/step-ca/password.txt 

__Limit access to owner 'root'__<br>
chmod 600 /etc/step-ca/password.txt 

## Test 

step-ca ~/.step/config/ca.json --password-file /etc/step-ca/password.txt 

(starts CA without asking for password) 

 
## Create systemd configuration
nano /etc/systemd/system/step-ca.service 

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

 

//Load & activate 

systemctl daemon-reload 

systemctl enable step-ca 

systemctl start step-ca 

 

//Check status 

systemctl status step-ca 

 

// Root -Zertifikat der CA auf Client installieren 

pct pull 101 /root/.step/certs/root_ca.crt /root/root_ca.crt 

 

#Root-CA im Debian-Zertifikatsspeicher installieren 

cp /root/root_ca.crt /usr/local/share/ca-certificates/step-home-ca.crt 

update-ca-certificates 

 

# Test -> liefert JSON antwort (sonst nur bei curl -k …) 

curl https://ca.fritz.box/acme/acme/directory 

