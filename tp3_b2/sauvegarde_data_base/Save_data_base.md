# Module 3 : Sauvegarde de base de données

Dans cette partie le but va être d'écrire un script `bash` qui récupère le contenu de la base de données utilisée par NextCloud, afin d'être en mesure de restaurer les données plus tard si besoin.

Le script utilisera la commande `mysqldump` qui permet de récupérer le contenu de la base de données sous la forme d'un fichier `.sql`.

Ce fichier `.sql` on pourra ensuite le compresser et le placer dans un dossier dédié afin de l'archiver.

Une fois le script fonctionnel, on créera alors un service qui permet de déclencher l'exécution de ce script dans de bonnes conditions.

Enfin, un *timer* permettra de déclencher l'exécution du *service* à intervalles réguliers.

## I. Script dump

➜ **Créer un utilisateur DANS LA BASE DE DONNEES**

- inspirez-vous des commandes SQL que je vous ai données au TP2
- l'utilisateur doit pouvoir se connecter depuis `localhost`
- il doit avoir les droits sur la base de données `nextcloud` qu'on a créé au TP2
- l'idée est d'avoir un utilisateur qui est dédié aux dumps de la base
  - votre script l'utilisera pour se connecter à la base et extraire les données


```bash
[archi@db ~]$ sudo mysql -u root -p
[sudo] password for archi:
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 4
Server version: 10.5.16-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> create user 'backup'@'127.0.0.1' identified by 'toto';
Query OK, 0 rows affected (0.003 sec)
MariaDB [(none)]> grant all on nextcloud.* to  'backup'@'127.0.0.1';
Query OK, 0 rows affected (0.002 sec)
MariaDB [(none)]> flush privileges;

MariaDB [(none)]> CREATE USER 'backup'@'10.102.1.12' IDENTIFIED BY 'toto';
Query OK, 0 rows affected (0.004 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* TO 'backup'@'10.102.1.12';
Query OK, 0 rows affected (0.003 sec)
MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)

```

➜ **Ecrire le script `bash`**

- il s'appellera `tp3_db_dump.sh`
- il devra être stocké dans le dossier `/srv` sur la machine `db.tp2.linux`
- le script doit commencer par un *shebang* qui indique le chemin du programme qui exécutera le contenu du script
  - ça ressemble à ça si on veut utiliser `/bin/bash` pour exécuter le contenu de notre script :

```
#!/bin/bash
```

- le script doit contenir une commande `mysqldump`
  - qui récupère le contenu de la base de données `nextcloud`
  - en utilisant l'utilisateur précédemment créé
- le fichier `.sql` produit doit avoir **un nom précis** :
  - il doit comporter le nom de la base de données dumpée
  - il doit comporter la date, l'heure la minute et la seconde où a été effectué le dump
  - par exemple : `db_nextcloud_2211162108.sql`
- enfin, le fichier `sql` doit être compressé
  - au format `.zip` ou `.tar.gz`
  - le fichier produit sera stocké dans le dossier `/srv/db_dumps/`
  - il doit comporter la date, l'heure la minute et la seconde où a été effectué le dump

> On utilise la notation américaine de la date `yymmdd` avec l'année puis le mois puis le jour, comme ça, un tri alphabétique des fichiers correspond à un tri dans l'ordre temporel :)

```bash
[archi@db ~]$ sudo mkdir /srv/db_dumps

[archi@db ~]$ sudo cat /srv/tp3_db_dump.sh
#!/bin/sh

username=backup
password=toto
database=nextcloud
date="$(date +'%d_%m_%Y_%H_%M')"
filename="db_"$database"_$date".sql
backupfolder="/srv/db_dumps"

pathbackup="$backupfolder/$filename"

mysqldump --user=$username --password="toto" $database > $filename
zip /srv/db_dumps/$filename.zip $filename
```

## II. Clean it

On va rendre le script un peu plus propre vous voulez bien ?

➜ **Utiliser des variables** déclarées en début de script pour stocker les valeurs suivantes :

- utilisateur de la base de données utiliser pour dump
- son password
- le nom de la base
- l'IP à laquelle la commande `mysqldump` se connecte
- le nom du fichier `.tar.gz` ou `.zip` produit par le script

```bash
# Déclaration d'une variable toto qui contient la string "tata"
toto="tata"

# Appel de la variable toto
# Notez l'utilisation du dollar et des double quotes
echo "$toto"
```
```bash
[archi@db ~]$ sudo cat /srv/tp3_db_dump.sh
#!/bin/sh

user='backup'
password='toto'
dbname='nextcloud'
datesave=$(date '+%y%m%d%H%M%S')
name="${dbname}_${datesave}"
outputpath="/srv/db_dumps/${name}.sql"

# Dump

echo "Backup started for database - ${dbname}."
mysqldump -u ${user} -p${password} --skip-lock-tables --databases ${dbname} > $outputpath
if [[ $? == 0 ]]
then
        gzip -c $outputpath > "${outputpath}.gz"
        rm -f $outputpath
        echo "Backup successfully completed."
else
        echo "Backup failed."
        rm -f outputpath
        exit 1
fi
```
---

➜ **Commentez le script**

- au minimum un en-tête sous le shebang
  - date d'écriture du script
  - nom/pseudo de celui qui l'a écrit
  - un résumé TRES BREF de ce que fait le script

```
[archi@db ~]$ sudo cat /srv/tp3_db_dump.sh
#!/bin/sh
# Last Update 22/11/2022
# This script will dump the database and save it to a file
# Editor : Victor Dalamel de Bournet

```
---


➜ **Environnement d'exécution du script**

- créez un utilisateur sur la machine `db.tp2.linux`
  - il s'appellera `db_dumps`
  - son homedir sera `/srv/db_dumps/`
  - son shell sera `/usr/bin/nologin`
```bash
[archi@db ~]$ sudo useradd db_dumps -m -d /srv/db_dumps/ -s /usr/bin/nologin
useradd: Warning: missing or non-executable shell '/usr/bin/nologin'
useradd: warning: the home directory /srv/db_dumps/ already exists.
useradd: Not copying any file from skel directory into it.
```
- cet utilisateur sera celui qui lancera le script
- le dossier `/srv/db_dumps/` doit appartenir au user `db_dumps`
```
[archi@db ~]$ sudo chown db_dumps:db_dumps /srv/tp3_db_dump.sh
[archi@db ~]$ sudo chown db_dumps:db_dumps /srv/db_dumps/
[archi@db ~]$ ls -al /srv/
total 4
drwxr-xr-x.  3 root     root      44 Nov 21 12:19 .
dr-xr-xr-x. 18 root     root     235 Sep 30 10:26 ..
drwxr-xr-x.  2 db_dumps db_dumps  51 Nov 21 12:39 db_dumps
-rw-r--r--.  1 db_dumps db_dumps 428 Nov 24 08:47 tp3_db_dump.sh
```

- pour tester l'exécution du script en tant que l'utilisateur `db_dumps`, utilisez la commande suivante :

```
[archi@db ~]$ sudo -u db_dumps /srv/tp3_db_dump.sh

[archi@db ~]$ ls -al
total 52
drwxr-xr-x. 2 db_dumps db_dumps    43 Nov 23 11:49 .
drwxr-xr-x. 3 root     root        44 Nov 23 11:46 ..
-rw-r--r--. 1 db_dumps db_dumps 52285 Nov 23 11:49 nextcloud_221115091918.sql.gz
```


---

## III. Service et timer

➜ **Créez un *service*** système qui lance le script

- inspirez-vous du *service* créé à la fin du TP1
- la seule différence est que vous devez rajouter `Type=oneshot` dans la section `[Service]` pour indiquer au système que ce service ne tournera pas à l'infini (comme le fait un serveur web par exemple) mais se terminera au bout d'un moment
- vous appelerez le service `db-dump.service`

```
[archi@db ~]$ sudo nano /etc/systemd/system/db-dump.service
[archi@db ~]$ sudo cat /etc/systemd/system/db-dump.service
[Unit]
Description=Dump the nextcloud database

[Service]
ExecStart=/srv/tp3_db_dump.sh
Type=oneshot
User=db_dumps
WorkingDirectory=/srv/db_dumps

[Install]
WantedBy=multi-user.target
```
```
[archi@db ~]$ sudo chmod 754 /srv/tp3_db_dump.sh
[archi@db ~]$ ls -al /srv/
total 4
drwxr-xr-x.  3 root     root      44 Nov 21 12:19 .
dr-xr-xr-x. 18 root     root     235 Sep 30 10:26 ..
drwxr-xr-x.  2 db_dumps db_dumps 137 Nov 24 09:03 db_dumps
-rwxr-xr--.  1 db_dumps db_dumps 692 Nov 24 09:06 tp3_db_dump.sh
```
- assurez-vous qu'il fonctionne en utilisant des commandes `systemctl`

```bash
[archi@db ~]$ sudo systemctl start db-dump
[archi@db ~]$ sudo systemctl status db-dump
○ db-dump.service - Dump the nextcloud database
     Loaded: loaded (/etc/systemd/system/db-dump.service; disabled; vendor preset: disabled)
     Active: inactive (dead) since Thu 2022-11-24 09:28:37 CET; 5s ago
TriggeredBy: ● db-dump.timer
    Process: 1745 ExecStart=/srv/tp3_db_dump.sh (code=exited, status=0/SUCCESS)
   Main PID: 1745 (code=exited, status=0/SUCCESS)
        CPU: 16ms

Nov 24 09:28:37 db.tp2.linux systemd[1]: Starting Dump the nextcloud database...
Nov 24 09:28:37 db.tp2.linux tp3_db_dump.sh[1745]: Backup started for database - nextcloud.
Nov 24 09:28:37 db.tp2.linux tp3_db_dump.sh[1745]: Backup successfully completed.
Nov 24 09:28:37 db.tp2.linux systemd[1]: db-dump.service: Deactivated successfully.
Nov 24 09:28:37 db.tp2.linux systemd[1]: Finished Dump the nextcloud database.
```


➜ **Créez un *timer*** système qui lance le *service* à intervalles réguliers

- le fichier doit être créé dans le même dossier
- le fichier doit porter le même nom
- l'extension doit être `.timer` au lieu de `.service`
- ainsi votre fichier s'appellera `db-dump.timer`
- la syntaxe est la suivante :

```systemd
[archi@db ~]$ sudo nano /etc/systemd/system/db-dump.timer
[archi@db ~]$ sudo cat /etc/systemd/system/db-dump.timer
[Unit]
Description=Run service X

[Timer]
OnCalendar=*-*-* 4:00:00

[Install]
WantedBy=timers.target
```

> [La doc Arch est cool à ce sujet.](https://wiki.archlinux.org/title/systemd/Timers)

- une fois le fichier créé :

```bash
# demander au système de lire le contenu des dossiers de config
[archi@db ~]$ sudo systemctl daemon-reload

[archi@db ~]$ sudo systemctl start db-dump.timer

[archi@db ~]$ sudo systemctl enable db-dump.timer
Created symlink /etc/systemd/system/timers.target.wants/db-dump.timer → /etc/systemd/system/db-dump.timer.

[archi@db ~]$ sudo systemctl status db-dump.timer
● db-dump.timer - Run service X
     Loaded: loaded (/etc/systemd/system/db-dump.timer; enabled; vendor preset: disabled)
     Active: active (waiting) since Thu 2022-11-24 09:14:53 CET; 13s ago
      Until: Thu 2022-11-24 09:14:53 CET; 13s ago
    Trigger: Fri 2022-11-25 04:00:00 CET; 18h left
   Triggers: ● db-dump.service

Nov 24 09:14:53 db.tp2.linux systemd[1]: Started Run service X.
# il apparaîtra quand on demande au système de lister tous les timers
$ sudo systemctl list-timers
```

➜ **Tester la restauration des données** sinon ça sert à rien :)

**Pour récupérer les données comment faire ?**
- Première étape récupérer la dernière sauvegarde qu'on a fait avec le service :
- 
```bash
[archi@db ~]$ ls -lt /srv/db_dumps/ | head -2
total 304
-rw-r--r--. 1 db_dumps db_dumps  52285 Nov 23 11:56 nextcloud_221123115607.sql.gz
```

- Deuxième étape on unzip le fichier :
```bash
[archi@db ~]$ sudo gzip -d /srv/db_dumps/nextcloud_221123115607.sql.gz
```

- Ensuite on se connecte à la base de données en local :
```bash
[archi@db ~]$ mysql -u root -p nextcloud < /srv/db_dumps/db_nextcloud_221123115607.sql
```

- Enfin on fait un dump sur notre base de donnée avec le fichier de la backup qu'on a unzip :
```bash
[archi@db ~]$ mysql -u dump -p nextcloud < /srv/db_dumps/db_nextcloud_221123115607.sql
```

**Et voila on a maintenant restaurer les données sur notre base de données !**