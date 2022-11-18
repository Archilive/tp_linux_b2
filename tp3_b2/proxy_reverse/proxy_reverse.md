# Module 1 : Reverse Proxy

Un reverse proxy est donc une machine que l'on place devant un autre service afin d'accueillir les clients et servir d'intermédiaire entre le client et le service.

L'utilisation d'un reverse proxy peut apporter de nombreux bénéfices :

- décharger le service HTTP de devoir effectuer le chiffrement HTTPS (coûteux en performances)
- répartir la charge entre plusieurs services
- effectuer de la mise en cache
- fournir un rempart solide entre un hacker potentiel et le service et les données importantes
- servir de point d'entrée unique pour accéder à plusieurs services web

## Sommaire

- [Module 1 : Reverse Proxy](#module-1--reverse-proxy)
  - [Sommaire](#sommaire)
- [I. Intro](#i-intro)
- [II. Setup](#ii-setup)
- [III. HTTPS](#iii-https)

# I. Intro

# II. Setup

🖥️ **VM `proxy.tp3.linux`**

**N'oubliez pas de dérouler la [📝**checklist**📝](#checklist).**

➜ **On utilisera NGINX comme reverse proxy**

- installer le paquet `nginx`

```
[archi@proxy ~]$ sudo dnf install nginx
...
Complete!
```
- démarrer le service `nginx`

```
[archi@proxy ~]$ sudo systemctl start nginx

[archi@proxy ~]$ sudo systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.

[archi@proxy ~]$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor pre>
     Active: active (running) since Thu 2022-11-17 11:21:16 CET; 9s ago
```
- utiliser la commande `ss` pour repérer le port sur lequel NGINX écoute
  
```
[archi@proxy ~]$ sudo ss -alntp | grep nginx
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=1086,fd=6),("nginx",pid=1085,fd=6))
LISTEN 0      511             [::]:80           [::]:*    users:(("nginx",pid=1086,fd=7),("nginx",pid=1085,fd=7))
```
- ouvrir un port dans le firewall pour autoriser le trafic vers NGINX

```
[archi@proxy ~]$ sudo firewall-cmd --add-port=80/tcp
success
[archi@proxy ~]$ sudo firewall-cmd --add-port=80/tcp --permanent
success
```
- utiliser une commande `ps -ef` pour déterminer sous quel utilisateur tourne NGINX

```
[archi@proxy ~]$ sudo ps -ef | grep nginx
nginx       1086    1085  0 11:21 ?        00:00:00 nginx: worker process

```
- vérifier que le page d'accueil NGINX est disponible en faisant une requête HTTP sur le port 80 de la machine

```
[archi@proxy ~]$ curl 10.102.1.13:80
<!doctype html>
<html>
  <head>
    <meta charset='utf-8'>
    <meta name='viewport' content='width=device-width, initial-scale=1'>
    <title>HTTP Server Test Page powered by: Rocky Linux</title>

```

➜ **Configurer NGINX**

- nous ce qu'on veut, c'pas une page d'accueil moche, c'est que NGINX agisse comme un reverse proxy entre les clients et notre serveur Web
- deux choses à faire :
  - créer un fichier de configuration NGINX
    - la conf est dans `/etc/nginx`
    - procédez comme pour Apache : repérez les fichiers inclus par le fichier de conf principal, et créez votre fichier de conf en conséquence

```
[archi@proxy ~]$ sudo cat /etc/nginx/conf.d/proxy.conf
server {
    # On indique le nom que client va saisir pour accéder au service
    # Pas d'erreur ici, c'est bien le nom de web, et pas de proxy qu'on veut ici !
    server_name web.tp2.linux;

    # Port d'écoute de NGINX
    listen 80;

    location / {
        # On définit des headers HTTP pour que le proxying se passe bien
        proxy_set_header  Host $host;
        proxy_set_header  X-Real-IP $remote_addr;
        proxy_set_header  X-Forwarded-Proto https;
        proxy_set_header  X-Forwarded-Host $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;

        # On définit la cible du proxying
        proxy_pass http://10.102.1.11:80;
    }

    # Deux sections location recommandés par la doc NextCloud
    location /.well-known/carddav {
      return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/caldav {
      return 301 $scheme://$host/remote.php/dav;
    }
}
*
```
  - NextCloud est un peu exigeant, et il demande à être informé si on le met derrière un reverse proxy
    - y'a donc un fichier de conf NextCloud à modifier
    - c'est un fichier appelé `config.php`

```
[archi@web ~]$ sudo cat /var/www/tp2_nextcloud/config/config.php
<?php
$CONFIG = array (
  'instanceid' => 'ochcha736bih',
  'passwordsalt' => 'M0xGRqB7uWL7pdra8l9jTCwPZT/zgE',
  'secret' => 'x5whgnJl0oj2Bo4QN8jWS2+jagUTB/V/CdVGA7s1uxT85qpK',
  'trusted_domains' =>
  array (
    0 => 'web.tp2.linux',
    1 => '10.102.1.13',
  ),
...
```


✨ **Bonus** : rendre le serveur `web.tp2.linux` injoignable sauf depuis l'IP du reverse proxy. En effet, les clients ne doivent pas joindre en direct le serveur web : notre reverse proxy est là pour servir de serveur frontal.

```
[archi@web ~]$ sudo cat /etc/httpd/conf.d/nextcloud.conf
<VirtualHost *:80>
# on indique le chemin de notre webroot
DocumentRoot /var/www/tp2_nextcloud/
# on précise le nom que saisissent les clients pour accéder au service
ServerName  web.tp2.linux

# on définit des règles d'accès sur notre webroot
<Directory /var/www/tp2_nextcloud/>
#    Require all granted
#   AllowOverride All
    require ip 10.102.1.13
    Options FollowSymLinks MultiViews
    <IfModule mod_dav.c>
    Dav off
    </IfModule>
</Directory>
</VirtualHost>
```

# III. HTTPS

Le but de cette section est de permettre une connexion chiffrée lorsqu'un client se connecte. Avoir le ptit HTTPS :)

Le principe :

- on génère une paire de clés sur le serveur `proxy.tp3.linux`
  - une des deux clés sera la clé privée : elle restera sur le serveur et ne bougera jamais
  - l'autre est la clé publique : elle sera stockée dans un fichier appelé *certificat*
    - le *certificat* est donné à chaque client qui se connecte au site

```
[archi@proxy ~]$ sudo mkdir /etc/nginx/certificate

[archi@proxy ~]$ cd /etc/nginx/certificate

[archi@proxy certificate]$ sudo openssl req -new -newkey rsa:4096 -x509 -sha256 -days 365 -nodes -out nginx-certificate.crt -keyout nginx.key      
...
...
[archi@proxy certificate]$ ls
nginx-certificate.crt  nginx.key

```
- on ajuste la conf NGINX
  - on lui indique le chemin vers le certificat et la clé privée afin qu'il puisse les utiliser pour chiffrer le trafic

```
[archi@proxy certificate]$ sudo cat /etc/nginx/conf.d/proxy.conf
...

server {
    # On indique le nom que client va saisir pour accéder au service
    # Pas d'erreur ici, c'est bien le nom de web, et pas de proxy qu'on veut ici !
    server_name web.tp2.linux;

    # Port d'écoute de NGINX
    listen 443 ssl;
    ssl_certificate /etc/nginx/certificate/nginx-certificate.crt;
    ssl_certificate_key /etc/nginx/certificate/nginx.key;

    location / {
        # On définit des headers HTTP pour que le proxying se passe bien
        proxy_set_header  Host $host;
        proxy_set_header  X-Real-IP $remote_addr;
        proxy_set_header  X-Forwarded-Proto https;
        proxy_set_header  X-Forwarded-Host $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;

        # On définit la cible du proxying
        proxy_pass http://10.102.1.11:80;
   }

    # Deux sections location recommandés par la doc NextCloud
    location /.well-known/carddav {
      return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/caldav {
      return 301 $scheme://$host/remote.php/dav;
    }
}

[archi@proxy certificate]$ sudo systemctl restart nginx

```
  - on lui demande d'écouter sur le port convetionnel pour HTTPS : 443 en TCP

```
[archi@proxy certificate]$ sudo firewall-cmd --add-port=443/tcp
success
[archi@proxy certificate]$ sudo firewall-cmd --add-port=443/tcp --permanent
success
```

Je vous laisse Google vous-mêmes "nginx reverse proxy nextcloud" ou ce genre de chose :)