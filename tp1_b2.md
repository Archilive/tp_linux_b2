# TP1 : (re)Familiaration avec un système GNU/Linux

Dans ce TP, on va passer en revue des éléments de configurations élémentaires du système.

Vous pouvez effectuer ces actions dans la première VM. On la clonera ensuite avec toutes les configurations pré-effectuées.

Au menu :

- gestion d'utilisateurs
  - sudo
  - SSH et clés
- configuration réseau
- gestion de partitions
- gestion de services

## Sommaire

- [TP1 : (re)Familiaration avec un système GNU/Linux](#tp1--refamiliaration-avec-un-système-gnulinux)
  - [Sommaire](#sommaire)
  - [0. Préparation de la machine](#0-préparation-de-la-machine)
  - [I. Utilisateurs](#i-utilisateurs)
    - [1. Création et configuration](#1-création-et-configuration)
    - [2. SSH](#2-ssh)
  - [II. Partitionnement](#ii-partitionnement)
    - [1. Préparation de la VM](#1-préparation-de-la-vm)
    - [2. Partitionnement](#2-partitionnement)
  - [III. Gestion de services](#iii-gestion-de-services)
  - [1. Interaction avec un service existant](#1-interaction-avec-un-service-existant)
  - [2. Création de service](#2-création-de-service)
    - [A. Unité simpliste](#a-unité-simpliste)
    - [B. Modification de l'unité](#b-modification-de-lunité)

## 0. Préparation de la machine

> **POUR RAPPEL** pour chacune des opérations, vous devez fournir dans le compte-rendu : comment réaliser l'opération ET la preuve que l'opération a été bien réalisée

🌞 **Setup de deux machines Rocky Linux configurées de façon basique.**

- **un accès internet (via la carte NAT)**
  - carte réseau dédiée
  - route par défaut

```
[archi@node1 ~]$ sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s8
[archi@node1 ~]$ ip a | grep enp0s8
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.101.1.11/24 brd 10.101.1.255 scope global noprefixroute enp0s8
```

- **un accès à un réseau local** (les deux machines peuvent se `ping`) (via la carte Host-Only)
  - carte réseau dédiée (host-only sur VirtualBox)
  - les machines doivent posséder une IP statique sur l'interface host-only

```
[archi@node1 ~]$ ping 10.101.1.12
PING 10.101.1.12 (10.101.1.12) 56(84) bytes of data.
64 bytes from 10.101.1.12: icmp_seq=1 ttl=64 time=0.624 ms
64 bytes from 10.101.1.12: icmp_seq=2 ttl=64 time=0.999 ms
64 bytes from 10.101.1.12: icmp_seq=3 ttl=64 time=0.915 ms
64 bytes from 10.101.1.12: icmp_seq=4 ttl=64 time=0.256 ms
^C
--- 10.101.1.12 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3061ms
rtt min/avg/max/mdev = 0.256/0.698/0.999/0.290 ms

```
- **les machines doivent avoir un nom**
  - référez-vous au mémo
  - les noms que doivent posséder vos machines sont précisés dans le tableau plus bas

```
[archi@node1 ~]$ sudo nano /etc/hostname
[sudo] password for archi:
"""Node1.tp1.b2"""
```

- **utiliser `1.1.1.1` comme serveur DNS**
  - référez-vous au mémo
  - vérifier avec le bon fonctionnement avec la commande `dig`
    - avec `dig`, demander une résolution du nom `ynov.com`
    - mettre en évidence la ligne qui contient la réponse : l'IP qui correspond au nom demandé
    - mettre en évidence la ligne qui contient l'adresse IP du serveur qui vous a répondu

```
[archi@node1 ~]$ sudo nano /etc/resolv.conf
[archi@node1 ~]$ dig ynov.com

;; ANSWER SECTION:
ynov.com.               300     IN      A       104.26.11.233
ynov.com.               300     IN      A       104.26.10.233
ynov.com.               300     IN      A       172.67.74.226

;; Query time: 28 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Mon Nov 14 12:07:21 CET 2022
;; MSG SIZE  rcvd: 85


```

- **les machines doivent pouvoir se joindre par leurs noms respectifs**
  - fichier `/etc/hosts`
  - assurez-vous du bon fonctionnement avec des `ping <NOM>`
```
[archi@node1 ~]$ sudo nano /etc/hosts
[sudo] password for archi:
[archi@node1 ~]$ ping node1.tp1
PING node1 (10.101.1.11) 56(84) bytes of data.
64 bytes from node1 (10.101.1.11): icmp_seq=1 ttl=64 time=0.044 ms
64 bytes from node1 (10.101.1.11): icmp_seq=2 ttl=64 time=0.080 ms
64 bytes from node1 (10.101.1.11): icmp_seq=3 ttl=64 time=0.073 ms
^C
--- node1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2076ms
rtt min/avg/max/mdev = 0.044/0.065/0.080/0.015 ms

  ```

- **le pare-feu est configuré pour bloquer toutes les connexions exceptées celles qui sont nécessaires**
  - commande `firewall-cmd`

Pour le réseau des différentes machines (ce sont les IP qui doivent figurer sur les interfaces host-only):

| Name               | IP            |
|--------------------|---------------|
| 🖥️ `node1.tp1.b2` | `10.101.1.11` |
| 🖥️ `node2.tp1.b2` | `10.101.1.12` |
| Votre hôte         | `10.101.1.1`  |

## I. Utilisateurs

[Une section dédiée aux utilisateurs est dispo dans le mémo Linux.](../../cours/memos/commandes.md#gestion-dutilisateurs).

### 1. Création et configuration

🌞 **Ajouter un utilisateur à la machine**, qui sera dédié à son administration

- précisez des options sur la commande d'ajout pour que :
  - le répertoire home de l'utilisateur soit précisé explicitement, et se trouve dans `/home`
  - le shell de l'utilisateur soit `/bin/bash`
- prouvez que vous avez correctement créé cet utilisateur
  - et aussi qu'il a le bon shell et le bon homedir

    ```
    [archi@node1 ~]$ sudo useradd toto -m -s /bin/bash -u 2000
    [sudo] password for archi:

    [archi@node1 ~]$ sudo cat /etc/passwd | grep toto
    toto:x:2000:2000::/home/toto:/bin/bash
    ```

🌞 **Créer un nouveau groupe `admins`** qui contiendra les utilisateurs de la machine ayant accès aux droits de `root` *via* la commande `sudo`.

Pour permettre à ce groupe d'accéder aux droits `root` :

- il faut modifier le fichier `/etc/sudoers`
- on ne le modifie jamais directement à la main car en cas d'erreur de syntaxe, on pourrait bloquer notre accès aux droits administrateur
- la commande `visudo` permet d'éditer le fichier, avec un check de syntaxe avant fermeture
- ajouter une ligne basique qui permet au groupe d'avoir tous les droits (inspirez vous de la ligne avec le groupe `wheel`)


    ```

    [archi@node1 ~]$ sudo groupadd admins
    [archi@node1 ~]$ sudo visudo /etc/sudoers
    ```

🌞 **Ajouter votre utilisateur à ce groupe `admins`**

> Essayez d'effectuer une commande avec `sudo` peu importe laquelle, juste pour tester que vous avez le droit d'exécuter des commandes sous l'identité de `root`. Vous pouvez aussi utiliser `sudo -l` pour voir les droits `sudo` auquel votre utilisateur courant a accès.

    ```
    [archi@node1 ~]$ sudo usermod -aG admins toto
    [archi@node1 ~]$ su toto
    Password:
    [toto@node1 archi]$ sudo ls

    We trust you have received the usual lecture from the local System
    Administrator. It usually boils down to these three things:

        #1) Respect the privacy of others.
        #2) Think before you type.
        #3) With great power comes great responsibility.

    [sudo] password for toto:
    [toto@node1 archi]$

    ```

---


1. Utilisateur créé et configuré
2. Groupe `admins` créé
3. Groupe `admins` ajouté au fichier `/etc/sudoers`
4. Ajout de l'utilisateur au groupe `admins`

### 2. SSH

[Une section dédiée aux clés SSH existe dans le cours.](../../cours/SSH/README.md)

Afin de se connecter à la machine de façon plus sécurisée, on va configurer un échange de clés SSH lorsque l'on se connecte à la machine.

🌞 **Pour cela...**

- il faut générer une clé sur le poste client de l'administrateur qui se connectera à distance (vous :) )
  - génération de clé depuis VOTRE poste donc
  - sur Windows, on peut le faire avec le programme `puttygen.exe` qui est livré avec `putty.exe`
- déposer la clé dans le fichier `/home/<USER>/.ssh/authorized_keys` de la machine que l'on souhaite administrer
  - vous utiliserez l'utilisateur que vous avez créé dans la partie précédente du TP
  - on peut le faire à la main
  - ou avec la commande `ssh-copy-id`

    ```
    Victor@LAPTOP-INFPUA9P MINGW64 ~ (master)
    $ ssh-keygen -t rsa -b 4096
    Generating public/private rsa key pair.
    Enter file in which to save the key (/c/Users/Victor/.ssh/id_rsa):
    /c/Users/Victor/.ssh/id_rsa already exists.
    Overwrite (y/n)? y
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:
    Your identification has been saved in /c/Users/Victor/.ssh/id_rsa
    Your public key has been saved in /c/Users/Victor/.ssh/id_rsa.pub
    The key fingerprint is:
    SHA256:jOIT77EN476OEuw3Heox0ECiLuzTmxXTIMpIQuB7ea0 Victor@LAPTOP-INFPUA9P
    The key's randomart image is:
    +---[RSA 4096]----+
    |+..              |
    |+o               |
    |oo.. .           |
    |B ooo +o         |
    |oB.o++.oS        |
    |o =o.+=          |
    | + o=E=.         |
    |  + BB.*         |
    |   *o+B..        |
    +----[SHA256]-----+

    Victor@LAPTOP-INFPUA9P MINGW64 ~ (master)
    $ ssh-copy-id archi@10.101.1.11
    /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/c/Users/Victor/.ssh/id_rsa.pub"
    /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
    /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
    archi@10.101.1.11's password:

    Number of key(s) added: 1

    Now try logging into the machine, with:   "ssh 'archi@10.101.1.11'"
    and check to make sure that only the key(s) you wanted were added.


    ```

🌞 **Assurez vous que la connexion SSH est fonctionnelle**, sans avoir besoin de mot de passe.


    Victor@LAPTOP-INFPUA9P MINGW64 ~ (master)
    $ ssh archi@10.101.1.11
    Last login: Mon Nov 14 12:45:18 2022 from 10.101.1.1
    [archi@node1 ~]$


## II. Partitionnement

[Il existe une section dédiée au partitionnement dans le cours](../../cours/part/)

### 1. Préparation de la VM

⚠️ **Uniquement sur `node1.tp1.b2`.**

Ajout de deux disques durs à la machine virtuelle, de 3Go chacun.


    [archi@node1 ~]$ lsblk
    sdb           8:16   0    3G  0 disk
    sdc           8:32   0    3G  0 disk

    [archi@node1 ~]$ sudo pvcreate /dev/sdb
    [sudo] password for archi:
    Physical volume "/dev/sdb" successfully created.
    [archi@node1 ~]$ sudo pvcreate /dev/sdc
    Physical volume "/dev/sdc" successfully created.
    

### 2. Partitionnement

⚠️ **Uniquement sur `node1.tp1.b2`.**

🌞 **Utilisez LVM** pour...

- agréger les deux disques en un seul *volume group*


```
[archi@node1 ~]$ sudo vgcreate data /dev/sdb
  Volume group "data" successfully created

[archi@node1 ~]$ sudo vgextend data /dev/sdc
  Volume group "data" successfully extended
```
- créer 3 *logical volumes* de 1 Go chacun

```
[archi@node1 ~]$ sudo lvcreate -L 1G data -n data_1
  Logical volume "data_1" created.
[archi@node1 ~]$ sudo lvcreate -L 1G data -n data_2
  Logical volume "data_2" created.
[archi@node1 ~]$ sudo lvcreate -L 1G data -n data_3
  Logical volume "data_3" created.
[archi@node1 ~]$ sudo lvs
  Devices file sys_wwid t10.ATA_____VBOX_HARDDISK___________________________VB38ed5419-97b5c6ce_ PVID 644qgeiF7cImZAeo3DZt1CwbXwPAe0Zk last seen on /dev/sda2 not found.
  LV     VG   Attr       LSize Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  data_1 data -wi-a----- 1.00g                                                  
  data_2 data -wi-a----- 1.00g                                                  
  data_3 data -wi-a----- 1.00g    
```
- formater ces partitions en `ext4`

```
[archi@node1 ~]$ mkfs -t ext4 /dev/data/data_1
mke2fs 1.46.5 (30-Dec-2021)
mkfs.ext4: Permission denied while trying to determine filesystem size
[archi@node1 ~]$ sudo mkfs -t ext4 /dev/data/data_1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: 3ea97169-e86a-4506-8904-1458deb48221
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done

[archi@node1 ~]$ sudo mkfs -t ext4 /dev/data/data_2

[archi@node1 ~]$ sudo mkfs -t ext4 /dev/data/data_3

```
- monter ces partitions pour qu'elles soient accessibles aux points de montage `/mnt/part1`, `/mnt/part2` et `/mnt/part3`.

```
[archi@node1 ~]$ sudo mkdir /mnt/part1
[archi@node1 ~]$ sudo mount /dev/data/data_1 /mnt/part1/
[archi@node1 ~]$ sudo mkdir /mnt/part2
[archi@node1 ~]$ sudo mount /dev/data/data_1 /mnt/part2/
[archi@node1 ~]$ sudo mkdir /mnt/part3
[archi@node1 ~]$ sudo mount /dev/data/data_1 /mnt/part3/

```

🌞 **Grâce au fichier `/etc/fstab`**, faites en sorte que cette partition soit montée automatiquement au démarrage du système.

```
[archi@node1 ~]$ sudo nano /etc/fstab
[archi@node1 ~]$ sudo cat /etc/fstab

#
/dev/data/data_1 /mnt/part1 ext4 defaults 0 0
/dev/data/data_2 /mnt/part2 ext4 defaults 0 0
/dev/data/data_3 /mnt/part3 ext4 defaults 0 0

```

✨**Bonus** : amusez vous avez les options de montage. Quelques options intéressantes :

- `noexec`
- `ro`
- `user`
- `nosuid`
- `nodev`
- `protect`

## III. Gestion de services

Au sein des systèmes GNU/Linux les plus utilisés, c'est *systemd* qui est utilisé comme gestionnaire de services (entre autres).

Pour manipuler les services entretenus par *systemd*, on utilise la commande `systemctl`.

On peut lister les unités `systemd` actives de la machine `systemctl list-units -t service`.

**Référez-vous au mémo pour voir les autres commandes `systemctl` usuelles.**

## 1. Interaction avec un service existant

⚠️ **Uniquement sur `node1.tp1.b2`.**

Parmi les services système déjà installés sur Rocky, il existe `firewalld`. Cet utilitaire est l'outil de firewalling de Rocky.

🌞 **Assurez-vous que...**

- l'unité est démarrée
- l'unitée est activée (elle se lance automatiquement au démarrage)

```
[archi@node1 ~]$ sudo systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor>
     Active: active (running) since Mon 2022-11-14 15:55:51 CET; 30min ago

[archi@node1 ~]$ sudo systemctl enable firewalld
```

## 2. Création de service

### A. Unité simpliste

⚠️ **Uniquement sur `node1.tp1.b2`.**

🌞 **Créer un fichier qui définit une unité de service** 

- le fichier `web.service`
- dans le répertoire `/etc/systemd/system`

```
[archi@node1 ~]$ sudo cat /etc/systemd/system/web.service
[Unit]
Description=Very simple web service

[Service]
ExecStart=/usr/bin/python3 -m http.server 8888

[Install]
WantedBy=multi-use
```

Le but de cette unité est de lancer un serveur web sur le port 8888 de la machine. **N'oubliez pas d'ouvrir ce port dans le firewall.**

```
[archi@node1 ~]$ sudo firewall-cmd --add-port=8888/tcp --permanent
success
```

Une fois l'unité de service créée, il faut demander à *systemd* de relire les fichiers de configuration :

```bash
[archi@node1 ~]$ sudo systemctl daemon-reload

```

Enfin, on peut interagir avec notre unité :

```bash
$ sudo systemctl status web
$ sudo systemctl start web
$ sudo systemctl enable web
```

```
[archi@node1 ~]$ sudo systemctl status web
○ web.service - Very simple web service
     Loaded: loaded (/etc/systemd/system/web.service; disabled; vendor preset: >
     Active: inactive (dead)
lines 1-3/3 (END)

[archi@node1 ~]$ sudo systemctl start web

[archi@node1 ~]$ sudo systemctl enable web
Created symlink /etc/systemd/system/multi-user.target.wants/web.service → /etc/systemd/system/web.service.

[archi@node1 ~]$ sudo systemctl status web
● web.service - Very simple web service
     Loaded: loaded (/etc/systemd/system/web.service; enabled; vendor preset: d>
     Active: active (running) since Mon 2022-11-14 16:48:24 CET; 10s ago
   Main PID: 1170 (python3)
      Tasks: 1 (limit: 5896)
     Memory: 9.2M
        CPU: 168ms
     CGroup: /system.slice/web.service
             └─1170 /usr/bin/python3 -m http.server 8888



```

🌞 **Une fois le service démarré, assurez-vous que pouvez accéder au serveur web**

- avec un navigateur depuis votre PC
- ou la commande `curl` depuis l'autre machine (je veux ça dans le compte-rendu :3)
- sur l'IP de la VM, port 8888

```
Victor@LAPTOP-INFPUA9P MINGW64 ~ (master)
$ curl http://10.101.1.11:8888
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   975  100   975    0     0   327k      0 --:--:-- --:--:-- --:--:--  476k<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href="afs/">afs/</a></li>
<li><a href="bin/">bin@</a></li>
<li><a href="boot/">boot/</a></li>
<li><a href="dev/">dev/</a></li>
<li><a href="etc/">etc/</a></li>
<li><a href="home/">home/</a></li>
<li><a href="lib/">lib@</a></li>
<li><a href="lib64/">lib64@</a></li>
<li><a href="media/">media/</a></li>
<li><a href="mnt/">mnt/</a></li>
<li><a href="opt/">opt/</a></li>
<li><a href="proc/">proc/</a></li>
<li><a href="root/">root/</a></li>
<li><a href="run/">run/</a></li>
<li><a href="sbin/">sbin@</a></li>
<li><a href="srv/">srv/</a></li>
<li><a href="sys/">sys/</a></li>
<li><a href="tmp/">tmp/</a></li>
<li><a href="usr/">usr/</a></li>
<li><a href="var/">var/</a></li>
</ul>
<hr>
</body>
</html>
```

### B. Modification de l'unité

🌞 **Préparez l'environnement pour exécuter le mini serveur web Python**

- créer un utilisateur `web`

  ```
  [archi@node1 meow]$ sudo useradd web -m -s /bin/bash -u 2001
  [archi@node1 meow]$ sudo passwd web
  Changing password for user web.
  New password:
  BAD PASSWORD: The password is shorter than 8 characters
  Retype new password:
  passwd: all authentication tokens updated successfully.
  ```
- créer un dossier `/var/www/meow/`
  ```
  [archi@node1 ~]$ sudo mkdir /var/www
  [sudo] password for archi:
  [archi@node1 ~]$ sudo mkdir /var/www/meow/
  [archi@node1 ~]$ cd /var/www/meow/
  ```
- créer un fichier dans le dossier `/var/www/meow/` (peu importe son nom ou son contenu, c'est pour tester)
  ```
  [archi@node1 meow]$ sudo nano meow
  ```
- montrez à l'aide d'une commande les permissions positionnées sur le dossier et son contenu

  ```
  [archi@node1 meow]$ ls -all
  total 4
  drwxr-xr-x. 2 web  web  24 Nov 14 21:42 .
  drwxr-xr-x. 3 root root 18 Nov 14 21:33 ..
  -rw-r--r--. 1 web  web  27 Nov 14 21:41 index.html

  ```

> Pour que tout fonctionne correctement, il faudra veiller à ce que le dossier et le fichier appartiennent à l'utilisateur `web` et qu'il ait des droits suffisants dessus.

🌞 **Modifiez l'unité de service `web.service` créée précédemment en ajoutant les clauses**

- `User=` afin de lancer le serveur avec l'utilisateur `web` dédié
- `WorkingDirectory=` afin de lancer le serveur depuis le dossier créé au dessus : `/var/www/meow/`
- ces deux clauses sont à positionner dans la section `[Service]` de votre unité

  ```
  [archi@node1 meow]$ sudo cat /etc/systemd/system/web.service
  [Unit]
  Description=Very simple web service

  [Service]
  User=web
  WorkingDirectory=/var/www/meow
  ExecStart=/usr/bin/python3 -m http.server 8888

  [Install]
  WantedBy=multi-user.target
  ```

  ```
  [archi@node1 ~]$ sudo systemctl daemon-reload
  [sudo] password for archi:
  [archi@node1 ~]$ sudo systemctl restart web.service
  [archi@node1 ~]$ sudo systemctl enable web.service
  [archi@node1 ~]$ sudo systemctl status web.service
  ● web.service - Very simple web service
      Loaded: loaded (/etc/systemd/system/web.service; enabled; vendor preset: d>
      Active: active (running) since Mon 2022-11-14 21:50:40 CET; 14s ago
    Main PID: 1201 (python3)
        Tasks: 1 (limit: 5896)
      Memory: 9.0M
          CPU: 39ms
      CGroup: /system.slice/web.service
              └─1201 /usr/bin/python3 -m http.server 8888

  ```

🌞 **Vérifiez le bon fonctionnement avec une commande `curl`**

  ```
  Victor@LAPTOP-INFPUA9P MINGW64 ~ (master)
  $ curl http://10.101.1.11:8888
    % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                  Dload  Upload   Total   Spent    Left  Speed
  100    27  100    27    0     0  12838      0 --:--:-- --:--:-- --:--:-- 27000Archilive test Web Service

  ```