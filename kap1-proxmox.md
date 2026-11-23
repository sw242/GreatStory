Connect to proxmox on node: 

https://proxmox10.fritz.box:8006/ 

https://192.168.10.10 

 

User name: root 

Passwort: bekannt 

 

Shell commands: 

 

pct enter 101 

-> open the terminal on node 101 

 

 

Linux: 

cat /etc/os-release 

-> print os version on terminal 

 

which step 

-> print path to executable step (based on PATH) 

 

echo $PATH 

-> print content of environment variable PATH 

 

python3 -m http.server 8080 

-> start http server in a folder 

-> in web browser: http://192.168.10.11:8080/root_ca.crt to download certificate 

 

 

tar -czvf /root/dotstep-backup.tar.gz /root/.step 

-c neues Archiv erstellen 

-z gzip-Komprimierung 

-v Dateinamen anzeigen 

-f Dateiname folgt 

 

 

 

