# Tp 5 de linux : Api minecraft server deploiment

#### collaborateur : Yanice Bousseria, Aymeric Lavaud, Victor Dalamel de Bournet


## Sommaire

- [Tp 5 de linux : Api minecraft server deploiment](#tp-5-de-linux--api-minecraft-server-deploiment)
      - [collaborateur : Yanice Bousseria, Aymeric Lavaud, Victor Dalamel de Bournet](#collaborateur--yanice-bousseria-aymeric-lavaud-victor-dalamel-de-bournet)
  - [Sommaire](#sommaire)
  - [Déroulement](#déroulement)
    - [Choix de la solution](#choix-de-la-solution)
    - [Mise en place de la solution](#mise-en-place-de-la-solution)
    - [Maîtrise de la solution](#maîtrise-de-la-solution)
    - [Amélioration de la solution](#amélioration-de-la-solution)
 
 ## Déroulement

### Choix de la solution

Pour le tp 5, nous avons choisi de faire une api capable de lancer rapidement des serveurs minecraft heberger sur un serveur (Notre machine, une vm ou meme un serveur qu'on loue).
Une fois le service executé, on à accès au serveur et jouer à Minecraft !


### Mise en place de la solution
Pour mettre en place me service nous avons créé un script, pour l'executer, il suffit de lancer la commande suivante :
```bash
bash script
```
![alt text](https://miro.medium.com/max/688/1*aqOwM4T47u7riL32Fn235Q.png)

Voici le repo dans lequel vous pouvez avoir accès au code de notre api   
https://github.com/Ciremy/express-docker



### Maîtrise de la solution

Une fois en place, posez-vous les questions pour comprendre ce qui a été mis en place, plus en détail. Réfléchissez avec les outils et concepts qu'on a vus en cours :

- combien de programmes y a-t-il ?  
-> Notre service dispose d'un seul programme, Docker et une api
  - sous quelle identité, quel user, tournent chacun de ces processus ?  
-> Pour le processus docker, l'identité du user est "docker"
- sur quel(s) port(s) écoute la solution ?  
-> Les ports qui écoutent la solution sont du port 25565 jusqu’au port 25575
- comment on gère le sycle de vie de l'app ?  
-> Une fois que l'on arrete un processus, il meurt.

- l'architecture et/ou la conf de la solution respectent-elles les bonnes pratiques élémentaires ?
  - gestion correcte des utilisateurs et des permissions
-> Oui, le user à pour seul accès le docker 

### Amélioration de la solution

↓  ↓ Afin d’améliorer le service, nous avons ajouté différentes routes à notre API ↓  ↓


les routes :

```bash
get /launch
````
Cette commande lance un conteneur.

```bash
post /close
```
Cette commande permet de recevoir un objet {"server": "number"} et ferme le conteneur correspondant. 

```bash
get /closeAll
```
Cette commande ferme tout les conteneurs.

```bash
get /purge
```
Cette commande permet purger les data.   


Voila, votre service de serveur minecraft est maintenant à portée de main  
![alt text](https://miro.medium.com/max/415/1*JS8H0spN34TK4LXVQAWUdQ.png)


