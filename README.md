# Systeme

Le systeme est un raspberry PI 4  

## Raspberry Pi OS Lite

- Télécharger l'image depuis: https://www.raspberrypi.org/software/
- Enregistrer l'image sur une carte SD puis démarer le raspberry.

> A ce stade il faut un écran et un clavier pour se connecter au raspebrry.

```
login: pi
passwor: raspberry
```

## Configuration du raspberry

- Executer la commande `sudo raspi-config`

  - Saisir les options de localisation

    > L1 Locale > ajouter fr_FR.UTF8 UTF8 (tab)<br>
    > L2 Time Zone > Europe\Paris<br>
    > L3 Keyboard > ajouter le francais (azerty)

  - Changer le mot de passe pour l'utilisateur pi

    > 1 - Systeme options > S3 - Password

  - Mettre à jour raspi

    > 8 - Update

- Changer le mot de passe <b>root</b> `sudo passwd root`

- Etape optionnelle, edition du style de la console `sudo dpkg-reconfigure console-setup`

  > UTF-8<br> 
  > Guess optimal character set<br>
  > Terminus<br>
  > 10x20.

- Prépartion et mise à jour du système :

  ```
  sudo apt update
  sudo apt upgrade -y  
  sudo rm -f /etc/systemd/network/99-default.link
  sudo reboot
  ```

## Overclock du RPI4

Editer le fichier `sudo nano noot/config.txt` et remplacer les blocs suivants:

```
#uncomment to overclock the arm. 700 MHz is the default.
arm_freq=1750
over_voltage=2
#gpu_freq=750
```

et

```
[all]
gpu_mem=256
```
> Sauvegarder et quitter `CTRL + X`, puis `Y`, et `ENTER`.

Rédémarer le raspberry `sudo reboot`

> L'augmentation de la memoire du GPU permet d'améliorer l'acceleration materiel du transcodage video (pour jellyfin).

## Partition sur la carte SD

> TODO - Optionnel, ce point est utilise si l'on souhaite stocker les configurations de dockers et des applications dans un volume dédié.

# OpenMediaVault

OpenMediavault (OMV) est une solution de stockage en réseau (NAS) basée sur Debian.

## Installation

- Execution d'un script d'installation (environ 20 minutes)

  ```
  wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
  ```

- Connection à l'interface web. Entrer l'adresse d'OMV dans le navigateur

  > L'adresse est indiquée à la fin de l'installation ou apres le boot de la machine. 

    ```
    login: admin
    passwor: openmediavault
    ```

> Si la connexion est possible, c'est que OMV est installé correctement et qu'il est possible de ce connecter à la machine en SSH. Dans le cas d'unne réinstallation l'on réinstalle OMV, pour etablir une nouvelle connexion SSH, il faut changer les clefs SSH  `ssh-keygen -R @adresse`.

## Configuration et réglage de base

Apres le premier démarage, configurer les parametres <b>systeme</b>.

- Dans la section <b>Parametres généraux</b> 

  - Modifier le mot de passe administrateur.

- Dans la section <b>Date & Heure</b>

  - Régler le Fuseau horaire sur <b>Europe/Paris</b>.

- Dans la section <b>Réseau</b>

  - Indiquer le nom de l'hote et le domaine. Par exemple:

    ```
    hote: omv
    domaine: home
    ```  

  - Désactiver l'IPV6

  - Activer le DHCP (si gestion des baux statiques via le routeur), sinon activer une IP statique.

    ```
    adresse: 192.168.0.101
    masque: 255.255.255.0
    passerelle: 192.168.0.254
    dns: 192.168.0.254 
    ```

- Dans la section <b>Gestionnaire de mises à jours</b>
  - Verifier et installer tous les paquets disponibles
  - Redémarrer la machine

- Dans la section <b>Stockage</b>
  - Créer un systeme de fichiers
  - Afficher les points de montage  pour recuperer facilement les paths vers les volumes dockers

## Configuration de Docker

- Dans la section <b>OMV-extras</b> (précédemment installés par le script d’installation) :
  - Installer portainer. Cela va installer automatiquement docker, docker-compose et portainer.
  - Activer le mode <b>avancé</b> de portainer et enregistrer.
  - Lancer portainer et indiquer le mot de passe adimistrateur.
    - Choisir « Local » pour la gestion de l’environnement Docker.

# Conteneurs

## Structure de fichiers

Le repertoire `appdata` contient les fichiers de configuration de chaques conteneurs.

```
- System
- Tank
  - appdata
  - cloud
  - medias
    - videos
      - séries
      - films
    - musiques
    - photos
  - downloads
```

## Environement

Le fichiers si dessous contient les varaibles d'environement communes à tous les conteneurs.

.env.base
```
PUID_admin=998             
PGID_admin=100              
TZ=Europe/Paris
```
- Recuperer le PUID et le PGID de l'utilisateur docker (admin) via la command `id admin` et mettre a jour si besoin

  > La commande `cat /etc/passwd` permet d'afficher la liste des utilisateurs avec PUID et GUID

- Récuperer le point de montage

## Services

### Reverse Proxy, Swag

Creation pour tous les sous domaines (wildcard)

- Renseigner la variable d'environnement `${TANK}` pour le point de montage
- Renseigner la variable d'environnement `${DOMAIN}`

#### Configuration DNS 

Cas du fournisseur OVH, dans le conteneur swag editer le fichier `config/dns-conf/ovh.ini`

> Pour un fournisseur différent, il faut éditer et suivre les instructions du fichier correspondant et surcharger les variable d'environement utilisée dans .env.proxy

- Pour généerer les clé API, se connecter a l'adresse suivante `https://eu.api.ovh.com/createToken/`. Cette page va fournir 3 clés. Il faut à remplacer les clés du fichiers `ovh.ini` par celle fourni par l'API. Indiquer les information suivantes
  - Renseignez l'identifiant client, le mot de passe et un nom de d'application.
  - Script name `swag`
  - Script description `rproxy`
  - Validity `Unlinimied`
  - Rights `GET /domain/zone/*`
  - Rights `PUT /domain/zone/*`
  - Rights `POST /domain/zone/*`
  - Rights `DELETE /domain/zone/*`

## Cloud

- Editer une stack <b>cloud</b> avec le fichier docker-compose suivant.
  - Mettre à jour le fichier docker-compose et deployer la stack.

```
version: "3.8"
services:

  nextcloud:
    image: ghcr.io/linuxserver/nextcloud
    container_name: nextcloud
    environment:
      - PUID=${PUID_admin}
      - PGID=${PGID_admin}
      - TZ=${TZ}
      - NEXTCLOUD_ADMIN_USER=fred
      - MYSQL_HOST=nextcloud-db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=nextcloud
    volumes:
      - ${TANK}/appdata/nextcloud/config:/config
      - ${TANK}/cloud/data:/data       
      - ${TANK}/medias:/medias
      - ${TANK}/downloads:/downloads         
    depends_on:
      - nextcloud-db
    ports:
      - 450:443
    networks:
      - gobieIO
    restart: unless-stopped
    
  mariadb:
    image: ghcr.io/linuxserver/mariadb
    container_name: nextcloud-db
    environment:
      - PUID=${PUID_admin}
      - PGID=${PGID_admin}
      - TZ=${TZ}
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=admin
      - MYSQL_ROOT_PASSWORD=nextcloud
      - MYSQL_PASSWORD=nextcloud
    ports:
      - 3306:3306
    volumes:
      - ${TANK}/cloud/db:/config    # Remplacer /disk par le moint de montage
    restart: unless-stopped
```
<hr>

## Medias

- Editer une stack <b>media</b> avec le fichier docker-compose suivant.
  - Mettre à jour le fichier docker-compose et deployer la stack.

```
version: "3.8"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin
    container_name: jellyfin
    devices:
      - /dev/vchiq:/dev/vchiq
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/London
      - JELLYFIN_PublishedServerUrl=192.168.0.101 #optional
    volumes:
      - /disk/appdata/jellyfin:/config  # Remplacer /disk par le moint de montage
      - /disk/media:/data/media         # Remplacer /disk par le moint de montage
    ports:
      - 8096:8096
    restart: unless-stopped
```

### Jellyfin

```
sudo docker exec -it jellyfin /bin/bash
usermod -aG video jellyfin
exit
```

<hr> 




