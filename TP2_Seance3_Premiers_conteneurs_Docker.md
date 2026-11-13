# INTRODUCTION AU CLOUD COMPUTING
## TP 2 — Premiers conteneurs avec Docker

**Niveau :** Licence 3 Sciences Informatiques  
**Séance :** 3 — Virtualisation : hyperviseurs, machines virtuelles et conteneurs  
**Environnement :** Windows 10, Multipass, Ubuntu Server 24.04.5 LTS et Docker  
**Groupe :** -ABE YEPI CHRIST-EMMANUEL
              -AKELY MAYLINE PHOEBE EDWINA
              -DANHO AKRE PRINCE PATRICK-JUNIOR

---

# Introduction

Le but de ce TP est de découvrir Docker et de comprendre le fonctionnement des conteneurs. Il s'agit d'installer Docker dans une machine virtuelle Ubuntu, de lancer et d'inspecter des conteneurs, de comparer les conteneurs avec une machine virtuelle, puis de créer une image Docker pour une API Python Flask et de la publier sur Docker Hub.

Contrairement à une machine virtuelle qui possède son propre système d'exploitation complet, un conteneur partage le noyau Linux de la machine hôte. Cela permet de lancer des applications de manière plus légère et plus rapide.

**Adaptation de l'environnement :** le sujet propose d'utiliser VirtualBox. Pour ce TP, nous avons utilisé Multipass sous Windows. La machine virtuelle `ubuntu-srv` a donc été conservée, mais les commandes de démarrage et l'accès réseau ont été adaptés à Multipass.

# Présentation de l'environnement

| Élément | Configuration |
|---|---|
| Système hôte | Windows 10 |
| Outil de virtualisation | Multipass |
| Machine virtuelle principale | ubuntu-srv |
| Système de la VM | Ubuntu 24.04.5 LTS |
| Processeur virtuel | 2 CPU |
| Mémoire RAM | Environ 1,9 Gio |
| Disque de la VM | 20 Go configurés |
| Partition principale (`/`) | 19 Go |
| Docker | 29.1.3 |
| Pilote de stockage | overlayfs |
| Gestionnaire de cgroups | systemd |
| Version des cgroups | v2 |
| Noyau Linux | 6.8.0-147-generic |

La machine virtuelle principale possède l'adresse IP `10.97.0.65`.

Une deuxième machine virtuelle appelée `web` existe également dans Multipass, avec l'adresse IP `10.97.0.162`. Elle est distincte du conteneur Nginx nommé `web`, utilisé dans ce TP.

---

# PARTIE 0 — Préparer la VM et installer Docker

## 0.1. Démarrage de la machine virtuelle

La machine virtuelle `ubuntu-srv` a été démarrée avec Multipass au lieu de VirtualBox.

Commande utilisée dans PowerShell :

```powershell
multipass list
multipass shell ubuntu-srv
```

La première commande permet de vérifier les machines virtuelles disponibles et leur état. La deuxième permet d'ouvrir un terminal dans la machine `ubuntu-srv`.

## 0.2. Mise à jour des paquets

Commande exécutée dans Ubuntu :

```bash
sudo apt update
```

Cette commande permet de mettre à jour la liste des paquets disponibles dans les dépôts Ubuntu.

**Résultat :** la mise à jour s'est terminée correctement. Environ 38,2 Mo de données ont été téléchargés en 30 secondes.

## 0.3. Installation de Docker

Commande utilisée :

```bash
sudo apt install -y docker.io docker-buildx docker-compose-v2
```

Cette commande installe Docker Engine ainsi que les outils Buildx et Docker Compose.

Les paquets ont été installés correctement.

Versions relevées :
- Docker Engine : `29.1.3-0ubuntu3~24.04.2`
- Docker Buildx : `0.30.1`
- Docker Compose : `2.40.3`

## 0.4. Autorisation de l'utilisateur à utiliser Docker

Commande exécutée :

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

La première commande ajoute l'utilisateur `ubuntu` au groupe `docker`. La deuxième permet de prendre en compte ce groupe dans le terminal actuel.

Vérification :

```bash
docker version
```

La commande a fonctionné sans utiliser `sudo`. Le client et le serveur Docker affichent la version `29.1.3`.

## 0.5. Vérification de la configuration Docker

Commande utilisée :

```bash
docker info | grep -E "Server Version|Storage Driver|Cgroup|Kernel Version"
```

Résultat obtenu :

```text
Server Version: 29.1.3
Storage Driver: overlayfs
Cgroup Driver: systemd
Cgroup Version: 2
Kernel Version: 6.8.0-147-generic
```

### Réponses aux questions

**Q1. Quelle version de Docker est installée ? Quel noyau Docker utilise-t-il ?**

La version installée est Docker `29.1.3`. Docker utilise le noyau Linux `6.8.0-147-generic`.

**Q2. Être membre du groupe docker revient presque à être administrateur de la machine. Pourquoi ? Quelle précaution cela impose-t-il sur un serveur partagé ?**

Un utilisateur membre du groupe `docker` peut lancer des conteneurs et accéder à certaines ressources de la machine hôte. Il peut notamment monter des répertoires de l'hôte dans un conteneur et accéder ainsi à des fichiers importants.

Cela peut lui permettre d'obtenir des privilèges très élevés sur la machine. Sur un serveur partagé, il faut donc limiter l'accès au groupe `docker` aux personnes de confiance.

---

# PARTIE 1 — Lancer et inspecter des conteneurs

## 1.1. Premier conteneur : hello-world

Commande exécutée :

```bash
docker run hello-world
```

Cette commande télécharge et exécute l'image officielle `hello-world` si elle n'est pas déjà disponible localement.

**Résultat obtenu :**

Le message suivant a été affiché :

```text
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

Docker a donc réussi à télécharger l'image, à créer le conteneur, à l'exécuter et à afficher son message.

Vérification des images :

```bash
docker images
```

L'image `hello-world` apparaît avec une taille d'environ `25.9 kB`.

Vérification des conteneurs :

```bash
docker ps -a
```

Le conteneur `hello-world` apparaît avec l'état `Exited (0)`. Cela signifie qu'il a terminé son exécution normalement.

## 1.2. Lancement d'un serveur web Nginx

Commande utilisée :

```bash
docker run -d --name web -p 8080:80 nginx
```

Explication :
- `docker run` crée et lance un conteneur.
- `-d` lance le conteneur en arrière-plan.
- `--name web` donne le nom `web` au conteneur.
- `-p 8080:80` associe le port 8080 de la VM au port 80 du conteneur.
- `nginx` indique l'image utilisée.

Vérification :

```bash
docker ps
```

Le conteneur `web` apparaît comme actif (`Up`) avec la redirection :

```text
0.0.0.0:8080->80/tcp
```

Test de la page :

```bash
curl http://localhost:8080
```

Résultat : la page d'accueil par défaut de Nginx s'affiche avec le message :

```html
<h1>Welcome to nginx!</h1>
```

## 1.3. Inspection du système d'exploitation

Pour connaître le système vu depuis le conteneur :

```bash
docker exec web cat /etc/os-release
```

Le conteneur utilise Debian GNU/Linux 13.7.

Pour connaître le système de la VM :

```bash
cat /etc/os-release
```

La machine virtuelle utilise Ubuntu 24.04.5 LTS.

Vérification du noyau :

```bash
docker exec web uname -r
uname -r
```

Dans les deux cas, le noyau affiché est :

```text
6.8.0-147-generic
```

Le conteneur utilise donc une distribution différente de celle de la VM, mais partage le même noyau Linux.

## 1.4. Observation des journaux, processus et ressources

Commandes utilisées :

```bash
docker logs --tail 5 web
docker top web
ps aux | grep "nginx: master" | grep -v grep
docker stats --no-stream web
```

Ces commandes permettent respectivement :
- d'afficher les dernières lignes des journaux ;
- de voir les processus exécutés dans le conteneur ;
- de rechercher le processus principal Nginx depuis la VM ;
- d'observer la consommation de ressources du conteneur.

Résultats observés :
- Nginx possède un processus principal (`master`) et deux processus secondaires (`worker`).
- La consommation mémoire observée était d'environ `3.402 MiB`.
- Le conteneur utilisait environ `0 %` du processeur lors de la mesure.
- Le nombre de processus affiché était de `3`.

## 1.5. Modification de la page d'accueil dans le conteneur

Commande utilisée :

```bash
docker exec web sh -c 'echo "<h1>Bienvenue sur mon serveur Docker !</h1>" > /usr/share/nginx/html/index.html'
```

Cette commande remplace le contenu de la page d'accueil directement dans le conteneur.

Vérification :

```bash
curl http://localhost:8080
```

Résultat obtenu :

```html
<h1>Bienvenue sur mon serveur Docker !</h1>
```

Ensuite, le conteneur a été supprimé :

```bash
docker rm -f web
```

Puis recréé à partir de la même image :

```bash
docker run -d --name web -p 8080:80 nginx
```

Nouvelle vérification :

```bash
curl http://localhost:8080
```

La page d'accueil originale de Nginx est réapparue.

Cette expérience montre que les modifications effectuées directement dans la couche du conteneur disparaissent lorsqu'on supprime celui-ci.

## 1.6. Utilisation d'un volume

Création du répertoire sur la VM :

```bash
mkdir -p ~/site
```

Création de la page HTML dans ce répertoire :

```bash
echo '<h1>Mon site Docker avec un volume persistant !</h1>' > ~/site/index.html
```

L'option `-v` permet de monter le répertoire de la VM dans le conteneur.

Commande utilisée :

```bash
docker rm -f web
docker run -d --name web -p 8080:80 -v ~/site:/usr/share/nginx/html:ro nginx
```

Vérification :

```bash
curl http://localhost:8080
```

Résultat obtenu :

```html
<h1>Mon site Docker avec un volume persistant !</h1>
```

Le fichier reste stocké dans la VM. Il n'est donc pas supprimé avec le conteneur.

L'option `:ro` signifie que le montage est en lecture seule depuis le conteneur.

### Réponses aux questions

**Q1. D'après le message de hello-world, quelles sont les quatre étapes réalisées par Docker ?**

Les quatre étapes principales sont :
1. Docker recherche l'image nécessaire localement ou la télécharge depuis un registre.
2. Docker crée un conteneur à partir de cette image.
3. Docker exécute le conteneur.
4. Le conteneur affiche son message dans le terminal.

**Q2. Le conteneur web et la VM ont-ils la même distribution ? Le même noyau ?**

Non, ils n'ont pas la même distribution :
- la VM utilise Ubuntu 24.04.5 LTS ;
- le conteneur utilise Debian GNU/Linux 13.7.

Cependant, ils utilisent le même noyau Linux `6.8.0-147-generic`. Cela s'explique par le fait que les conteneurs partagent le noyau de la machine hôte.

**Q3. Les processus Nginx du conteneur apparaissent-ils dans ps aux sur la VM ? Qu'est-ce que cela montre ?**

Oui, les processus Nginx apparaissent dans la liste des processus de la VM.

Cela montre qu'un conteneur n'est pas une machine virtuelle indépendante avec son propre noyau. Ses processus sont exécutés sur le noyau Linux de la machine hôte, tout en étant isolés dans leur environnement.

**Q4. Retracez le chemin d'une requête du navigateur de l'hôte jusqu'à Nginx.**

Avec l'environnement Multipass utilisé, le chemin est :

```text
Navigateur Windows
       |
       v
Adresse de la VM : 10.97.0.65:8080
       |
       v
Port 8080 de la VM Ubuntu
       |
       v
Port 80 du conteneur Nginx
       |
       v
Page HTML retournée
```

La redirection Docker est réalisée avec `-p 8080:80`.

Contrairement au sujet initial, qui prévoit une redirection VirtualBox, l'accès a été effectué avec l'adresse IP de la VM Multipass.

**Q5. Pourquoi la page modifiée a-t-elle disparu après docker rm ? Que change l'option -v ?**

La modification a disparu parce qu'elle avait été faite directement dans le système de fichiers du conteneur. Lorsqu'on supprime le conteneur, cette modification est supprimée avec lui.

L'option `-v` permet de monter un répertoire de la VM dans le conteneur. Le fichier reste alors stocké sur la VM et peut être réutilisé même si le conteneur est supprimé et recréé.

---

# PARTIE 2 — Comparer conteneur et VM

## 2.1. Téléchargement de l'image Alpine

Commande utilisée :

```bash
docker pull alpine
```

L'image Alpine a été téléchargée correctement.

Vérification :

```bash
docker images
```

Tailles observées :
- Alpine : `13 MB`
- Nginx : `243 MB`
- hello-world : `25.9 kB`

## 2.2. Mesure du temps de démarrage d'Alpine

Le sujet demande d'exécuter deux fois la commande `time` et de conserver la deuxième mesure.

Première exécution :

```bash
time docker run --rm alpine echo "Bonjour depuis Alpine"
```

Résultat :

```text
real    0m0.730s
```

Deuxième exécution :

```bash
time docker run --rm alpine echo "Bonjour depuis Alpine"
```

Résultat :

```text
real    0m0.654s
```

La mesure conservée est donc :

**0,654 seconde.**

La commande `time` permet de mesurer le temps nécessaire à l'exécution d'une commande :
- `real` : temps réellement écoulé ;
- `user` : temps processeur utilisé en mode utilisateur ;
- `sys` : temps processeur utilisé par le système.

## 2.3. Mesure du démarrage de la VM

Commande utilisée :

```bash
systemd-analyze
```

Résultat obtenu :

```text
Startup finished in 103ms (firmware)
+ 1.056s (loader)
+ 3.065s (kernel)
+ 7min 4.903s (userspace)
= 7min 9.128s
graphical.target reached after 1min 1.973s in userspace
```

Le temps total affiché est de `7 minutes et 9,128 secondes`.

Cette mesure concerne le démarrage de la machine virtuelle et non celui d'un conteneur.

## 2.4. Mesure de la mémoire

Commande utilisée :

```bash
free -h
```

Résultat obtenu :

```text
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       442Mi       262Mi       992Ki       1.3Gi       1.4Gi
Swap:             0B          0B          0B
```

La VM dispose d'environ :
- 1,9 Gio de mémoire totale ;
- 442 Mio de mémoire utilisée ;
- 262 Mio de mémoire libre ;
- 1,3 Gio de mémoire utilisée par les caches et les buffers ;
- 1,4 Gio de mémoire disponible.

## 2.5. Vérification du système Alpine

Commande utilisée :

```bash
docker run --rm alpine cat /etc/os-release
```

Résultat obtenu :

```text
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.4
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
```

## 2.6. Tableau comparatif

| Mesure | VM ubuntu-srv | Conteneur nginx | Conteneur alpine |
|---|---|---|---|
| Taille sur disque | Partition principale : 19 Go ; disque configuré : 20 Go | 243 MB | 13 MB |
| Temps de démarrage | 7 min 9,128 s selon `systemd-analyze` | Non mesuré séparément | 0,654 s |
| Mémoire utilisée | 442 MiB | Environ 3,402 MiB | Non mesurée |
| Système | Ubuntu 24.04.5 LTS | Debian GNU/Linux 13.7 | Alpine Linux 3.24.4 |
| Noyau | 6.8.0-147-generic | Partagé avec la VM | Partagé avec la VM |

**Remarque :** les 19 Go correspondent à la taille de la partition principale de la VM, et non à la taille exacte du dossier de la VM demandé dans le sujet. La taille du dossier de la VM du TP1 n'a pas été relevée dans les résultats disponibles. Les mesures de mémoire et de démarrage ne sont pas toutes directement comparables, car elles ne portent pas sur les mêmes éléments.

### Réponses aux questions

**Q1. Dans quel rapport le conteneur est-il plus léger et plus rapide que la VM ? Vos mesures confirment-elles le tableau du cours ?**

L'image Alpine occupe environ 13 MB, alors que la partition principale de la VM dispose de 19 Go. Le conteneur est donc beaucoup plus léger qu'un système complet.

Le démarrage d'Alpine a pris 0,654 seconde, contre 429,128 secondes affichées par `systemd-analyze` pour le démarrage de la VM.

Le rapport calculé à partir de ces deux durées est d'environ 656. Il faut cependant préciser que les deux mesures ne représentent pas exactement la même opération : l'une mesure l'exécution d'un conteneur, l'autre le démarrage complet de la VM.

Les résultats confirment globalement que les conteneurs sont plus légers et peuvent démarrer beaucoup plus rapidement qu'une machine virtuelle.

**Q2. Pourquoi le premier time docker run est-il plus long que le second ?**

Le premier lancement peut nécessiter le téléchargement de l'image et la préparation des différentes couches.

Au deuxième lancement, l'image est déjà présente localement et les couches sont disponibles. Docker n'a donc pas besoin de les télécharger à nouveau.

Dans nos mesures, le premier lancement a pris 0,730 seconde et le deuxième 0,654 seconde. La différence observée est faible, mais le deuxième lancement a bien été plus rapide.

**Q3. Citez un cas où vous choisiriez quand même une VM plutôt qu'un conteneur.**

Je choisirais une machine virtuelle lorsque je dois installer un système d'exploitation complet avec son propre noyau, tester un autre système d'exploitation ou exécuter des services qui nécessitent un environnement système indépendant.

---

# PARTIE 3 — Conteneuriser l'API fil rouge

## 3.1. Création du dossier du projet

Commande utilisée :

```bash
mkdir -p ~/api
cd ~/api
```

Le dossier `api` a été créé dans le répertoire personnel de l'utilisateur `ubuntu`.

## 3.2. Création du fichier app.py

Le fichier `app.py` contient le code de l'API Flask.

Version initiale :

```python
import os
import socket
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def accueil():
    return jsonify(
        message="API fil rouge",
        binome=os.environ.get("BINOME", "inconnu"),
        conteneur=socket.gethostname(),
    )

@app.get("/sante")
def sante():
    return jsonify(statut="ok")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

L'API possède deux routes au départ :
- `/` : affiche le message de l'API, le nom du binôme et le nom du conteneur ;
- `/sante` : permet de vérifier si l'API fonctionne.

La fonction `os.environ.get()` permet de récupérer la variable d'environnement `BINOME`.

La fonction `socket.gethostname()` permet de récupérer le nom d'hôte du conteneur.

## 3.3. Création de requirements.txt

Commande utilisée :

```bash
echo "flask>=3,<4" > requirements.txt
```

Ce fichier indique que le projet utilise Flask dans une version supérieure ou égale à 3 et inférieure à 4.

## 3.4. Création du Dockerfile

Fichier `Dockerfile` :

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

Explication des instructions :
- `FROM` définit l'image de base Python.
- `WORKDIR` définit le répertoire de travail dans le conteneur.
- `COPY requirements.txt .` copie le fichier des dépendances.
- `RUN` installe Flask.
- `COPY . .` copie les fichiers du projet.
- `EXPOSE 5000` indique le port utilisé par l'application.
- `CMD` définit la commande exécutée au démarrage du conteneur.

## 3.5. Création du fichier .dockerignore

Commande utilisée :

```bash
printf '__pycache__/\n*.pyc\n.git\n' > .dockerignore
```

Ce fichier permet d'exclure certains éléments du contexte de construction de l'image, notamment les fichiers Python temporaires et le dossier Git.

## 3.6. Construction de l'image api:1.0

Commande utilisée :

```bash
docker build -t api:1.0 .
```

Cette commande construit une image Docker appelée `api` avec le tag `1.0`.

La construction s'est terminée correctement après environ 91,5 secondes.

Vérification :

```bash
docker images api
```

Résultat observé :

| Élément | Valeur |
|---|---|
| Repository | api |
| Tag | 1.0 |
| Image ID | 9a578a6dafcb |
| Taille disque affichée | 198 MB |
| Content size | 48,3 MB |

## 3.7. Inspection des couches de l'image

Commande utilisée :

```bash
docker history api:1.0
```

Cette commande affiche les différentes couches de l'image.

Les instructions issues de notre Dockerfile sont :
1. `WORKDIR /app`
2. `COPY requirements.txt .`
3. `RUN pip install --no-cache-dir -r requirements.txt`
4. `COPY . .`
5. `EXPOSE 5000`
6. `CMD ["python", "app.py"]`

Le reste des couches provient de l'image de base `python:3.12-slim`.

## 3.8. Lancement de l'API

Commande utilisée :

```bash
docker run -d --name api -p 5000:5000 -e BINOME="ABE-AKELY-DANHO" api:1.0
```

Cette commande :
- lance le conteneur en arrière-plan ;
- lui donne le nom `api` ;
- expose le port 5000 ;
- transmet le nom du binôme avec la variable d'environnement `BINOME`.

Vérification de la route principale :

```bash
curl http://localhost:5000
```

Résultat obtenu :

```json
{"binome":"ABE-AKELY-DANHO","conteneur":"b91f8fb1515a","message":"API fil rouge"}
```

Vérification de la route santé :

```bash
curl http://localhost:5000/sante
```

Résultat :

```json
{"statut":"ok"}
```

Les journaux ont également été consultés avec :

```bash
docker logs api
```

## 3.9. Lancement de trois copies de la même image

Le sujet demande de lancer trois copies de la même image sur trois ports différents.

Commandes utilisées :

```bash
for p in 5001 5002 5003; do
  docker run -d --name api-$p -p $p:5000 -e BINOME="ABE-AKELY-DANHO" api:1.0
done
```

Interrogation des trois conteneurs :

```bash
for p in 5001 5002 5003; do
  curl -s http://localhost:$p
  echo
done
```

Résultats observés :

Les trois conteneurs ont renvoyé le même nom de binôme, mais chacun avait un identifiant de conteneur différent.

Identifiants observés :
- `c91fe15c0ec6`
- `e900a3e25fb1`
- `26aa88aa55b9`

Suppression des copies temporaires :

```bash
docker rm -f api-5001 api-5002 api-5003
```

Les trois conteneurs ont été supprimés correctement.

## 3.10. Ajout de la route /version

Le fichier `app.py` a été modifié pour ajouter une troisième route :

```python
@app.get("/version")
def version():
    return jsonify(version="1.1")
```

Cette route renvoie la version de l'API.

La nouvelle version du fichier contient donc trois routes :
- `/`
- `/sante`
- `/version`

## 3.11. Construction de la nouvelle image api:1.1

Commande utilisée :

```bash
docker build -t api:1.1 .
```

La construction s'est terminée correctement.

Résultat important : Docker a affiché plusieurs lignes `CACHED`.

Les instructions suivantes ont été réutilisées depuis le cache :
- `WORKDIR /app`
- `COPY requirements.txt .`
- `RUN pip install --no-cache-dir -r requirements.txt`

L'instruction `COPY . .` a été exécutée à nouveau pour intégrer le nouveau code.

Remplacement de l'ancien conteneur :

```bash
docker rm -f api
docker run -d --name api -p 5000:5000 api:1.1
```

Vérification de la nouvelle route :

```bash
curl -s http://localhost:5000/version
```

Résultat obtenu :

```json
{"version":"1.1"}
```

La version 1.1 de l'API fonctionne correctement.

### Réponses aux questions

**Q1. Quelle est la taille de l'image api:1.0 ? Combien de couches docker history affiche-t-il, et lesquelles viennent de votre Dockerfile ?**

La taille affichée pour `api:1.0` est de 198 MB sur disque, avec un content size de 48,3 MB.

Le Dockerfile contient six instructions principales :
- WORKDIR
- COPY requirements.txt
- RUN pip install
- COPY .
- EXPOSE
- CMD

Les autres couches proviennent de l'image de base Python.

**Q2. Pourquoi copie-t-on requirements.txt et installe-t-on les dépendances avant de copier le code ?**

On copie d'abord `requirements.txt` et on installe les dépendances avant de copier le code pour profiter du cache Docker.

Si seul le code de l'application change, Docker peut conserver les couches correspondant à l'installation des dépendances.

Dans notre cas, lors de la construction de `api:1.1`, les instructions `WORKDIR`, `COPY requirements.txt` et `RUN pip install` ont été réutilisées grâce au cache. Seule la copie du code a été exécutée à nouveau.

Cela permet de gagner du temps lors des reconstructions.

**Q3. Que renvoie le champ conteneur pour les trois copies de l'étape 5 ? Quelle caractéristique du cloud cette expérience illustre-t-elle ?**

Les trois copies renvoient des identifiants différents :
- `c91fe15c0ec6`
- `e900a3e25fb1`
- `26aa88aa55b9`

Cela montre qu'une même image peut être exécutée plusieurs fois dans des conteneurs indépendants.

Cette expérience illustre la scalabilité horizontale : on peut lancer plusieurs instances d'une application pour répartir la charge et augmenter la capacité du service.

**Q4. Pourquoi passer le nom du binôme par une variable d'environnement plutôt que de l'écrire dans le code ? Quel lien avec les secrets ?**

Une variable d'environnement permet de modifier une information sans changer le code de l'application ni reconstruire l'image.

La même image peut ainsi être utilisée avec différentes configurations.

Pour les mots de passe et les clés d'API, il faut éviter de les écrire directement dans le code ou de les intégrer dans une image Docker. Ils doivent être transmis de manière sécurisée au moment de l'exécution, à l'aide de mécanismes adaptés de gestion des secrets.

---

# PARTIE 4 — Publier l'image et nettoyer

## 4.1. Publication sur Docker Hub

Le registre choisi est Docker Hub.

Identifiants utilisés pour le dépôt :
- Compte Docker Hub : `ayce1324`
- Nom du dépôt : `api-fil-rouge`
- Version publiée : `1.1`

Le problème de résolution DNS rencontré lors de la connexion à Docker Hub a été corrigé temporairement avec les commandes suivantes :

```bash
sudo resolvectl dns eth0 1.1.1.1#one.one.one.one
sudo resolvectl dnsovertls eth0 yes
```

La résolution DNS a ensuite été vérifiée avec :

```bash
resolvectl query auth.docker.io
```

La connexion au registre a été testée avec :

```bash
curl -I --max-time 15 https://registry-1.docker.io/v2/
```

Le serveur a répondu avec le code HTTP `401`. Cette réponse est normale pour une requête au registre sans authentification préalable.

Connexion à Docker Hub :

```bash
docker login
```

La connexion a réussi.

## 4.2. Attribution du nom Docker Hub à l'image

Commande utilisée :

```bash
docker tag api:1.1 ayce1324/api-fil-rouge:1.1
```

Cette commande ajoute un nom permettant d'identifier l'image sur Docker Hub.

Vérification :

```bash
docker images
```

Les deux noms correspondaient à la même image locale.

Identifiant observé :

```text
9ad81c214cc1
```

## 4.3. Publication de l'image

Commande utilisée :

```bash
docker push ayce1324/api-fil-rouge:1.1
```

La publication s'est terminée correctement.

Digest obtenu :

```text
sha256:9ad81c214cc16ae45e086f69a83eb48299a7079af4e8ef7444c7f4ca59703f8f
```

L'image est disponible à l'adresse :

**https://hub.docker.com/r/ayce1324/api-fil-rouge**

La visibilité du dépôt a été vérifiée sur Docker Hub. Le dépôt est **public**.

## 4.4. Vérification du téléchargement depuis Docker Hub

Pour vérifier que l'image était bien disponible sur le registre, le conteneur et les images locales ont été supprimés.

Commandes utilisées :

```bash
docker rm -f api
docker rmi api:1.1 ayce1324/api-fil-rouge:1.1
```

Téléchargement depuis Docker Hub :

```bash
docker pull ayce1324/api-fil-rouge:1.1
```

Le téléchargement a réussi.

Le digest obtenu correspondait à celui de la publication :

```text
sha256:9ad81c214cc16ae45e086f69a83eb48299a7079af4e8ef7444c7f4ca59703f8f
```

L'image a ensuite été lancée :

```bash
docker run -d --name api -p 5000:5000 ayce1324/api-fil-rouge:1.1
```

Vérification :

```bash
curl -s http://localhost:5000/version
```

Résultat :

```json
{"version":"1.1"}
```

La route `/version` fonctionnait bien après téléchargement de l'image depuis Docker Hub.

Lors d'un premier test de la route principale sans variable d'environnement, le résultat était :

```json
{"binome":"inconnu","conteneur":"f8eb8f98535f","message":"API fil rouge"}
```

Cela est normal, car la variable `BINOME` n'avait pas été transmise lors de ce lancement.

Le conteneur a donc été relancé avec la variable correcte :

```bash
docker rm -f api
docker run -d --name api -p 5000:5000 -e BINOME="ABE-AKELY-DANHO" ayce1324/api-fil-rouge:1.1
```

Résultat de la vérification :

```bash
curl -s http://localhost:5000
```

```json
{"binome":"ABE-AKELY-DANHO","conteneur":"d5d19a57b945","message":"API fil rouge"}
```

Cela confirme que l'image publiée sur Docker Hub fonctionne correctement et que la variable d'environnement est bien prise en compte.

## 4.5. Nettoyage des conteneurs et des images

Avant le nettoyage, la commande suivante a été exécutée :

```bash
docker system df
```

Résultat observé :

```text
TYPE            TOTAL     ACTIVE    SIZE       RECLAIMABLE
Images          5         3         352.7MB    291.4MB (82%)
Containers      3         2         315.4kB    4.096kB (1%)
Local Volumes   0         0         0B         0B
Build Cache     12        0         110.3MB    36.86kB
```

Suppression des conteneurs :

```bash
docker rm -f $(docker ps -aq)
```

Trois conteneurs ont été supprimés.

Après cette suppression, `docker system df` affichait :

```text
Images          5    0    352.3MB    86.98MB (24%)
Containers      0    0    0B         0B
Local Volumes   0    0    0B         0B
Build Cache     12   0    110.3MB    36.86kB
```

Suppression des images inutilisées :

```bash
docker image prune -a -f
```

Docker a supprimé les images locales inutilisées, notamment :
- `ayce1324/api-fil-rouge:1.1`
- `alpine:latest`
- `api:1.0`
- `hello-world:latest`
- `nginx:latest`

Espace récupéré annoncé par Docker :

```text
Total reclaimed space: 40.94MB
```

Vérification finale :

```bash
docker system df
```

Résultat obtenu :

```text
Images          0    0    149.7MB    149.7MB (100%)
Containers      0    0    0B         0B
Local Volumes   0    0    0B         0B
Build Cache     12   0    110.3MB    110.3MB (100%)
```

La sortie finale indique qu'il n'y a plus d'image ni de conteneur répertorié. Le cache de construction est encore présent.

La commande `docker image prune -a -f` a annoncé une récupération de `40.94 MB`. Il ne faut pas confondre cette valeur avec la taille affichée dans la colonne `SIZE` de la dernière commande `docker system df`.

Docker a été conservé dans la VM pour les séances suivantes, comme demandé dans le sujet.

## 4.6. Vérification de l'accès depuis le navigateur Windows

Après le nettoyage, un dernier test a été réalisé pour vérifier l'accès à l'API depuis Windows.

L'image a été téléchargée à nouveau depuis Docker Hub et lancée dans Ubuntu :

```bash
docker run -d --name api -p 5000:5000 -e BINOME="ABE-AKELY-DANHO" ayce1324/api-fil-rouge:1.1
```

Le conteneur était actif et le test local a réussi :

```bash
curl http://localhost:5000
```

Résultat :

```json
{"binome":"ABE-AKELY-DANHO","conteneur":"aa883acd4f52","message":"API fil rouge"}
```

Les journaux ont confirmé que Flask écoutait sur toutes les interfaces :

```text
* Running on all addresses (0.0.0.0)
* Running on http://127.0.0.1:5000
* Running on http://172.17.0.2:5000
```

Depuis Google Chrome sous Windows, l'adresse suivante a été testée :

```text
http://10.97.0.65:5000
```

La page a fonctionné et a affiché la réponse JSON de l'API.

La route `/version` a également été testée depuis Ubuntu :

```bash
curl -i http://localhost:5000/version
```

Résultat :

```text
HTTP/1.1 200 OK
Server: Werkzeug/3.1.9 Python/3.12.16
Content-Type: application/json

{"version":"1.1"}
```

La route `/version` a finalement été affichée correctement dans le navigateur Windows après une nouvelle tentative avec l'adresse exacte :

```text
http://10.97.0.65:5000/version
```

Cette vérification confirme que l'API est accessible depuis l'hôte Windows et que la version 1.1 fonctionne.

Après le test, le conteneur a été supprimé :

```bash
docker rm -f api
```

Résultat :

```text
api
```

Le conteneur a donc bien été supprimé. L'image reste disponible sur Docker Hub.

### Réponses aux questions

**Q1. Indiquez l'adresse complète de votre image publiée. Est-elle publique ou privée ?**

Adresse complète du dépôt :

https://hub.docker.com/r/ayce1324/api-fil-rouge

Nom de l'image :

```text
ayce1324/api-fil-rouge:1.1
```

Le dépôt a été vérifié sur Docker Hub et il est public.

**Q2. Pourquoi étiqueter l'image avec un numéro de version (1.1) plutôt que de toujours utiliser latest ?**

Le numéro de version permet de reconnaître précisément la version d'une application.

Il facilite le suivi des modifications, le déploiement d'une version précise et le retour à une ancienne version en cas de problème.

Le tag `latest` peut changer lorsqu'une nouvelle image est publiée. Il ne permet donc pas de savoir exactement quelle version est utilisée.

**Q3. Quel espace disque docker image prune a-t-il libéré ? Dans le cloud, pourquoi faut-il aussi nettoyer les registres ?**

La commande `docker image prune -a -f` a annoncé :

```text
Total reclaimed space: 40.94MB
```

Dans le cloud, il faut nettoyer les registres pour supprimer les anciennes images qui ne sont plus utilisées. Cela permet de limiter l'espace de stockage occupé, de réduire les coûts et de faciliter la gestion des versions.

---

## 6. Problèmes rencontrés et solutions apportées

Au cours de ce TP, quelques difficultés techniques ont été rencontrées. Elles ont été résolues progressivement.

### 6.1. Adaptation de VirtualBox à Multipass

Le TP utilise VirtualBox, mais nous avons utilisé Multipass pour créer et gérer la machine virtuelle Ubuntu.

**Solution :** les manipulations ont été adaptées à Multipass. La VM `ubuntu-srv` a été utilisée pour installer Docker et exécuter les conteneurs.

### 6.2. Confusion entre le terminal Windows et le terminal Ubuntu

Certaines commandes Multipass ont été exécutées dans le terminal Ubuntu. Le message `multipass: command not found` est alors apparu.

**Solution :** les commandes Multipass ont été exécutées dans Windows PowerShell, tandis que les commandes Docker et Linux ont été exécutées dans le terminal Ubuntu.

### 6.3. Problème de résolution DNS

Lors du téléchargement et de la publication des images Docker, des difficultés de résolution DNS ont empêché certaines connexions aux serveurs Docker Hub.

**Solution :** le DNS a été configuré temporairement avec Cloudflare et DNS-over-TLS :

```bash
sudo resolvectl dns eth0 1.1.1.1#one.one.one.one
sudo resolvectl dnsovertls eth0 yes
```

Après cette configuration, la résolution DNS et les connexions à Docker Hub ont fonctionné.

### 6.4. Problème avec le caractère spécial `!` dans une commande

Lors de la création du fichier HTML du site Nginx, l'utilisation du caractère `!` dans une commande `echo` avec des guillemets doubles a provoqué une erreur liée à l'historique de Bash.

**Solution :** les guillemets simples ont été utilisés pour encadrer le texte HTML.

### 6.5. Difficulté lors de la vérification de l'API

Lors des premiers tests, certaines requêtes vers l'API ont échoué temporairement. La route `/version` a également renvoyé une erreur 404 lors d'un premier essai.

**Solution :** le conteneur a été vérifié, les journaux consultés et la présence de la route dans `app.py` contrôlée. Après vérification, les routes `/` et `/version` ont répondu correctement.

### 6.6. Difficulté à retrouver la taille exacte du dossier de la VM

La commande `multipass get local.instances` a renvoyé une erreur, car `local.instances` n'est pas une clé de configuration reconnue par Multipass.

**Solution :** les informations disponibles avec `multipass info ubuntu-srv` ont été utilisées. Elles indiquent un disque de 19,3 Gio, dont 2,6 Gio sont utilisés. Cette mesure correspond à l'utilisation du disque virtuel et non à la taille exacte du dossier de stockage de la VM sur Windows. Cette dernière mesure reste à effectuer.

### Bilan

Malgré ces difficultés, toutes les principales manipulations du TP ont été réalisées : installation de Docker, création et exécution des conteneurs, construction de l'API Flask, publication de l'image sur Docker Hub, téléchargement de l'image publiée et vérification de son fonctionnement depuis Windows.


# Conclusion générale

Ce TP nous a permis de découvrir Docker et de comprendre la différence entre une machine virtuelle et un conteneur.

Nous avons installé Docker dans une machine virtuelle Ubuntu avec Multipass, exécuté plusieurs conteneurs, utilisé Nginx, observé les processus et les ressources consommées, puis utilisé un volume pour conserver les fichiers en dehors du conteneur.

Nous avons également comparé les images Docker et les temps d'exécution d'un conteneur avec les mesures de démarrage de la machine virtuelle.

Dans la troisième partie, nous avons créé une API avec Python et Flask, écrit un Dockerfile, construit l'image `api:1.0`, puis ajouté une route `/version` pour construire la version `api:1.1`. Nous avons aussi lancé trois copies de la même image pour observer que chaque conteneur possède son propre identifiant.

Enfin, nous avons publié l'image sur Docker Hub, vérifié qu'elle pouvait être téléchargée à nouveau et exécutée, puis nettoyé les conteneurs et les images locales.

Le test final depuis le navigateur Windows a confirmé que l'API était accessible à travers le réseau Multipass.

Ce TP nous a donc permis de comprendre les principales étapes de la conteneurisation d'une application : création, construction, exécution, inspection, publication et nettoyage.

---

# Informations finales du projet

| Élément | Valeur |
|---|---|
| Nom du projet | API fil rouge |
| Langage | Python |
| Framework | Flask |
| Version de l'API | 1.1 |
| Image Docker | `ayce1324/api-fil-rouge:1.1` |
| Registre | Docker Hub |
| Visibilité | Publique |
| Port de l'API | 5000 |
| Route principale | `/` |
| Route santé | `/sante` |
| Route version | `/version` |
| Adresse du dépôt | https://hub.docker.com/r/ayce1324/api-fil-rouge |
| État du nettoyage | Terminé |
