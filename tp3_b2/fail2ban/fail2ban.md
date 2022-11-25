# Module 7 : Fail2Ban

Fail2Ban c'est un peu le cas d'école de l'admin Linux, je vous laisse Google pour le mettre en place.

C'est must-have sur n'importe quel serveur à peu de choses près. En plus d'enrayer les attaques par bruteforce, il limite aussi l'imact sur les performances de ces attaques, en bloquant complètement le trafic venant des IP considérées comme malveillantes

Faites en sorte que :

- si quelqu'un se plante 3 fois de password pour une co SSH en moins de 1 minute, il est ban
- vérifiez que ça fonctionne en vous faisant ban
- afficher la ligne dans le firewall qui met en place le ban
- lever le ban avec une commande liée à fail2ban

> Vous pouvez vous faire ban en effectuant une connexion SSH depuis `web.tp2.linux` vers `db.tp2.linux` par exemple, comme ça vous gardez intacte la connexion de votre PC vers `db.tp2.linux`, et vous pouvez continuer à bosser en SSH.

```
[archi@db ~]$ sudo dnf install epel-release -y
[...]
Complete!
```

```
[archi@db ~]$ sudo dnf install -y fail2ban-firewalld
[...]
Complete!
```

```
[archi@db ~]$ sudo systemctl start fail2ban
[archi@db ~]$ sudo systemctl enable fail2ban
Created symlink /etc/systemd/system/multi-user.target.wants/fail2ban.service → /usr/lib/systemd/system/fail2ban.service.
[archi@db ~]$ sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2022-11-25 09:18:54 CET; 11s ago
       Docs: man:fail2ban(1)
   Main PID: 1332 (fail2ban-server)
      Tasks: 3 (limit: 5905)
     Memory: 10.3M
        CPU: 61ms
     CGroup: /system.slice/fail2ban.service
             └─1332 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 25 09:18:54 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 25 09:18:54 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 25 09:18:54 db.tp2.linux fail2ban-server[1332]: 2022-11-25 09:18:54,900 fail2ban.configreader   [1332]: WARNING 'al>
Nov 25 09:18:54 db.tp2.linux fail2ban-server[1332]: Server ready
```

```
[archi@db ~]$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

[archi@db ~]$ sudo nano /etc/fail2ban/jail.local
[archi@db ~]$ sudo cat /etc/fail2ban/jail.local
[...]
[DEFAULT]
bantime = 1h
findtime = 1h
maxretry = 3


[archi@db ~]$ sudo mv /etc/fail2ban/jail.d/00-firewalld.conf /etc/fail2ban/jail.d/00-firewalld.local
```

```
[archi@db ~]$ sudo systemctl restart fail2ban
[archi@db ~]$ sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2022-11-25 09:25:29 CET; 10s ago
       Docs: man:fail2ban(1)
    Process: 1388 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 1389 (fail2ban-server)
      Tasks: 3 (limit: 5905)
     Memory: 10.6M
        CPU: 65ms
     CGroup: /system.slice/fail2ban.service
             └─1389 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 25 09:25:29 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 25 09:25:29 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 25 09:25:29 db.tp2.linux fail2ban-server[1389]: 2022-11-25 09:25:29,242 fail2ban.configreader   [1389]: WARNING 'al>
Nov 25 09:25:29 db.tp2.linux fail2ban-server[1389]: Server ready
```

```
[archi@db ~]$ sudo nano /etc/fail2ban/jail.d/sshd.local
[archi@db ~]$ sudo cat /etc/fail2ban/jail.d/sshd.local
[sshd]
enabled = true

bantime = 1d
maxretry = 3
```

```
[archi@db ~]$ sudo systemctl restart fail2ban
[archi@db ~]$ sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/usr/lib/systemd/system/fail2ban.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2022-11-25 09:27:53 CET; 1s ago
       Docs: man:fail2ban(1)
    Process: 1409 ExecStartPre=/bin/mkdir -p /run/fail2ban (code=exited, status=0/SUCCESS)
   Main PID: 1411 (fail2ban-server)
      Tasks: 5 (limit: 5905)
     Memory: 14.0M
        CPU: 77ms
     CGroup: /system.slice/fail2ban.service
             └─1411 /usr/bin/python3 -s /usr/bin/fail2ban-server -xf start

Nov 25 09:27:53 db.tp2.linux systemd[1]: fail2ban.service: Deactivated successfully.
Nov 25 09:27:53 db.tp2.linux systemd[1]: Stopped Fail2Ban Service.
Nov 25 09:27:53 db.tp2.linux systemd[1]: Starting Fail2Ban Service...
Nov 25 09:27:53 db.tp2.linux systemd[1]: Started Fail2Ban Service.
Nov 25 09:27:53 db.tp2.linux fail2ban-server[1411]: 2022-11-25 09:27:53,301 fail2ban.configreader   [1411]: WARNING 'al>
Nov 25 09:27:53 db.tp2.linux fail2ban-server[1411]: Server ready
```

```
[archi@db ~]$ sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd
```

```
[archi@db ~]$ sudo fail2ban-client get sshd maxretry
3
```

```
[archi@web ~]$ ssh archi@10.102.1.12
archi@10.102.1.12's password:
Permission denied, please try again.
archi@10.102.1.12's password:
Permission denied, please try again.
archi@10.102.1.12's password:
archi@10.102.1.12: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).
ssh archi@10.102.1.12
```

```
[archi@db ~]$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     3
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   10.102.1.11
```

```
[archi@db ~]$ sudo fail2ban-client unban 10.102.1.11
1
```
```
[archi@db ~]$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     3
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     1
   `- Banned IP list: