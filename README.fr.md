# Boîte à outils de serveur Minecraft

Exploitez un dossier de serveur Minecraft depuis Bash et Docker : un jar de serveur principal avec, en option, des services annexes comme Velocity dans la même console, plus des outils pour les mises à jour, les sauvegardes et Dynmap.

La configuration vit dans `.env`. Les fichiers du dépôt sont faits pour rester génériques ; copiez `.env.example` en `.env`, puis ne décommentez que les valeurs que ce serveur surcharge réellement.

## Fichiers

| Fichier | Rôle |
| --- | --- |
| `start.sh` | Lance le jar du serveur principal et les jars de services dans une seule console, avec des logs partagés et des commandes de lanceur. |
| `mc-common.sh` | Journalisation, analyse des booléens et chargement du `.env` partagés. |
| `mc-updater.sh` | Installateur/metteur à jour manuel pour les jars de serveur Fabric, les jars PaperMC/Velocity, Modrinth, CurseForge et les jars par URL directe. |
| `mc-backup.sh` | Sauvegardes complètes ou des mondes seuls, restauration, save-off/save-on et redémarrage optionnel du serveur. |
| `mc-dynmap.sh` | Outils d'envoi web Dynmap et de réinitialisation des tables SQL. |
| `Dockerfile` | Conteneur Java/tmux générique. Il ne connaît ni Velocity ni une topologie de serveur particulière. |
| `docker-compose.yml` | Monte un dossier de serveur sur `/minecraft`, expose les ports configurés et lance `START_COMMAND`. |
| `.env.example` | Catalogue de surcharges essentiellement commenté. À copier en `.env`, puis décommenter/modifier les valeurs. |

## Prérequis

L'usage local demande Bash plus les outils utilisés par les commandes que vous lancez :

```bash
chmod +x start.sh mc-*.sh
```

```bash
# updater
curl jq

# backups
tar gzip zip unzip

# dynmap upload / SQL reset
openssh-client mysql-client
```

L'usage Docker demande Docker Compose. L'image inclut déjà Java, tmux, curl, jq, zip/unzip, tar/gzip et les outils client SSH.

## Partir de zéro

1. Copiez les fichiers de la boîte à outils dans un dossier de serveur vide.

2. Créez votre configuration locale :

```bash
cp .env.example .env
```

3. Ouvrez `.env`. Les réglages les plus courants sont en haut et commentés avec leurs valeurs par défaut ou des exemples :

```env
# MC_GAME_VERSION=1.21.6
# MC_SERVER_LOADER=fabric
# MC_UPDATER_PLATFORMS=fabric,velocity,paper
# SERVER_JAVA_OPTS=-Xms2G -Xmx4G
# MINECRAFT_PORT=25565
# DOCKER_BASE_IMAGE=ghcr.io/graalvm/jdk-community:25-ol9
```

Décommentez les lignes que vous voulez surcharger. Une ligne laissée commentée signifie que la valeur par défaut du script est utilisée.

4. Choisissez une topologie et décommentez/modifiez ces lignes.

Pour un seul jar de serveur, sans services :

```env
START_SERVER_ON_BOOT=true
START_SERVICES_ON_BOOT=false
SERVICES_CONFIG=
SERVER_WORKING_DIR=.
SERVER_ARGS=nogui
```

Pour Velocity plus un backend à démarrage à la demande lancé par AutoServer :

```env
START_SERVER_ON_BOOT=false
START_SERVICES_ON_BOOT=true
SERVER_WORKING_DIR=.
SERVER_ARGS=nogui
MC_HEALTHCHECK_PORT=25565
```

Vous pouvez laisser `SERVICES_CONFIG` commenté jusqu'à l'installation de Velocity. Le metteur à jour le remplira avec le vrai nom de fichier de Velocity téléchargé.

Dans cette configuration, Velocity écoute sur `25565` et le serveur Minecraft backend écoute en général sur un autre port, par exemple `25566`.

Le reste de `.env.example` est regroupé par outil : lanceur, Docker, metteur à jour, sauvegardes, Dynmap et chargement partagé du `.env`. Ces sections du bas sont surtout des réglages avancés.

## Installer les jars, les mods et les plugins

Le metteur à jour est manuel. Il n'installe ou ne met à jour des fichiers que lorsque vous le lancez.

Installez un jar de serveur Fabric et écrivez le nom de fichier téléchargé dans `SERVER_JAR` du `.env` :

```bash
./mc-updater.sh add fabric-server . --mc 1.21.6 --env-key SERVER_JAR
```

Si `SERVER_JAR` est encore commenté dans `.env`, le metteur à jour active cette ligne du modèle au lieu d'ajouter un doublon.

Installez Velocity depuis PaperMC dans `velocity/` :

```bash
./mc-updater.sh add papermc velocity velocity
```

Cela conserve le vrai nom de fichier PaperMC, active `SERVICES_CONFIG` s'il est commenté, ajoute le service `velocity` s'il manque, et met à jour le nom du jar du service lors des prochains `./mc-updater.sh update`.

Installez un jar de serveur Paper au lieu de Fabric :

```bash
./mc-updater.sh add papermc paper . --filename paper.jar --env-key SERVER_JAR
```

Installez Fabric API dans `mods/` :

```bash
./mc-updater.sh add modrinth fabric-api mods
```

Installez un plugin Velocity depuis Modrinth, par exemple AutoServer :

```bash
./mc-updater.sh add modrinth autoserver velocity/plugins --loader velocity --ignore-game-version
```

Installez un projet CurseForge par son identifiant numérique :

```bash
CURSEFORGE_API_KEY=your-api-key
./mc-updater.sh add curseforge 238222 mods
```

Installez n'importe quel jar par URL directe :

```bash
./mc-updater.sh add url my-plugin https://example.com/plugin.jar plugins --filename my-plugin.jar
```

Gérez les entrées suivies :

```bash
./mc-updater.sh list
./mc-updater.sh update
./mc-updater.sh update fabric-api
./mc-updater.sh disable fabric-api
./mc-updater.sh enable fabric-api
./mc-updater.sh remove fabric-api
```

Acceptez l'EULA avant le premier démarrage du serveur :

```bash
printf 'eula=true\n' > eula.txt
```

## Console du lanceur

Pour démarrer en local :

```bash
./start.sh
```

Les commandes du lanceur commencent par `!` :

```text
!help
!start
!stop
!restart
!status
!services
!start-service velocity
!stop-service velocity
!send say hello
!exit
```

Quand le serveur principal tourne, la saisie console normale est envoyée à Minecraft. Pour envoyer une commande Minecraft qui commence par `!`, tapez-la sous la forme `!!command`.

Des scripts externes peuvent piloter le lanceur en cours avec les mêmes commandes :

```bash
./start.sh start
./start.sh stop
./start.sh send "say hello"
```

## AutoServer de Velocity

Pour le démarrage à la demande du backend, configurez AutoServer pour appeler le lanceur :

```toml
[servers.host]
workingDirectory = "../"
start = "bash start.sh start"
stop = "bash start.sh stop"
startupDelay = 10
shutdownDelay = 30
autoShutdownDelay = 1800
preserveQuotes = true
```

Velocity doit pointer vers le port du backend :

```toml
[servers]
host = "127.0.0.1:25566"
try = ["host"]
```

Le `server.properties` du backend doit correspondre :

```properties
server-port=25566
online-mode=false
```

## Docker

Compiler et démarrer :

```bash
docker compose up -d --build
```

Suivre les logs combinés :

```bash
docker compose logs -f minecraft
```

Se rattacher à la console tmux en direct :

```bash
docker compose exec minecraft attach
```

Pour se détacher sans arrêter le serveur : appuyez sur `Ctrl+B`, puis `D`.

Arrêter la pile :

```bash
docker compose down
```

Les valeurs Docker utiles du `.env` sont déjà listées en haut de `.env.example` ; décommentez celles dont vous avez besoin :

```env
SERVER_WORKING_DIR=.
MINECRAFT_PORT=25565
MINECRAFT_CONTAINER_PORT=25565
MC_HEALTHCHECK_ENABLED=true
MC_HEALTHCHECK_PORT=25565
START_COMMAND=./start.sh
DOCKER_BASE_IMAGE=ghcr.io/graalvm/jdk-community:25-ol9
```

Utilisez `DOCKER_BASE_IMAGE` quand vous avez besoin d'une autre version de Java, mais gardez-la compatible avec l'étape d'installation de paquets du `Dockerfile`.

## Sauvegardes

Configurez les chemins de sauvegarde dans `.env` :

```env
SERVER_WORKING_DIR=.
MC_BACKUP_DIR=./backups
MC_BACKUP_WORLDS=world,world_nether,world_the_end
```

Lancez :

```bash
./mc-backup.sh full
./mc-backup.sh worlds
./mc-backup.sh worlds --no-stop
./mc-backup.sh restart
./mc-backup.sh restore ./backups/example.zip
```

Les sauvegardes de mondes à chaud utilisent `save-all`, `save-off`, puis `save-on`. Si l'archivage échoue, le script réactive la sauvegarde avant de rendre la main.

## Dynmap

Configurez l'envoi web et la réinitialisation SQL dans `.env` :

```env
DYNMAP_WEB_SOURCE=dynmap/web
DYNMAP_REMOTE_HOST=user@example.com
DYNMAP_REMOTE_WEB_LOCATION=/home/user/.nginx/html/dynmap
DYNMAP_DB_HOST=example.com
DYNMAP_DB_USER=dynmap
DYNMAP_DB_NAME=dynmap
```

Lancez :

```sh
./mc-dynmap.sh upload-web
./mc-dynmap.sh reset-sql
./mc-dynmap.sh sync
```

`sync` lance d'abord `reset-sql`, puis envoie les fichiers web.
