# Compte rendu TP2 : Premiers conteneurs avec Docker

**Module :** Docker et conteneurisation  
**Environnement :** Ubuntu sur VirtualBox  
**Equipe :** - ABE YEPI CHRIST-EMMANUEL
             - AKELY MAYLINE PHOEBE EDWINA
             - DANHO AKRE PRINCE PATRICK-JUNIOR

> Chaque manipulation est présentée avec la **commande saisie** (ligne commençant par `etudiant@ubuntusrv:~/api$` ou par `$` quand le dossier n'a pas d'importance) puis le **résultat affiché** dans le terminal.

---

# Sommaire et correspondance avec le barème

| Partie | Contenu | Critère du barème | Points |
|---|---|---|---:|
| [Partie 0](#partie-0--préparer-la-machine-virtuelle-et-installer-docker) | Machine virtuelle, Docker installé, questions de sécurité | Docker installé, questions de sécurité | 2 |
| [Partie 1](#partie-1--lancer-et-inspecter-des-conteneurs) | Conteneurs lancés et inspectés, volume | Conteneurs lancés et inspectés, volume compris | 5 |
| [Partie 2](#partie-2--comparer-un-conteneur-et-une-machine-virtuelle) | Tableau comparatif VM / conteneurs et questions | Mesures comparées et interprétées | 3 |
| [Partie 3](#partie-3--conteneuriser-lapi-fil-rouge) | Dockerfile, build, cache, tests de l'API | Dockerfile fonctionnel, image construite, cache expliqué | 6 |
| [Partie 4](#partie-4--publier-limage-et-nettoyer) | Publication sur Docker Hub, nettoyage | Image publiée et vérifiée, nettoyage fait | 2 |
| Ensemble du document | Clarté, captures, délai | Qualité du compte rendu | 2 |
| | | **Total** | **20** |

---

# Partie 0 : Préparer la machine virtuelle et installer Docker

## 0.1. Configuration de la machine virtuelle

Pour ce TP, nous avons travaillé sur la VM `Ubuntu_serveur`, qui tourne sous Ubuntu 26.04.1 LTS.

| Élément | Valeur |
|---|---|
| Nom de la VM | `Ubuntu_serveur` |
| Nom d'hôte | `ubuntusrv` |
| Utilisateur | `etudiant` |
| Système d'exploitation | Ubuntu 26.04.1 LTS |
| Version de Docker | 29.1.3 |
| Version du noyau Linux | 7.0.0-38-generic |

On se connecte à la VM en SSH : le port 2222 de la machine hôte est redirigé vers le port 22 de la VM.

**Commandes saisies depuis PowerShell :**

```powershell
$env:Path += ";C:\Program Files\Oracle\VirtualBox"
VBoxManage startvm "Ubuntu_serveur" --type headless
VBoxManage controlvm "Ubuntu_serveur" natpf1 "ssh,tcp,127.0.0.1,2222,,22"
VBoxManage controlvm "Ubuntu_serveur" natpf1 "web,tcp,127.0.0.1,8080,,8080"
VBoxManage controlvm "Ubuntu_serveur" natpf1 "api,tcp,127.0.0.1,5000,,5000"
ssh -p 2222 etudiant@127.0.0.1
```

Ces commandes démarrent la VM sans interface graphique, créent trois redirections de ports (SSH sur 2222, web sur 8080, API sur 5000), puis ouvrent la connexion SSH.

## 0.2. Questions

### Q1. Quelle version de Docker est installée ? Quel noyau utilise-t-il ?

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker version
Client:
 Version:           29.1.3
 API version:       1.52
 Go version:        go1.24.13
 Git commit:        29.1.3-0ubuntu4.1
 Built:             Wed Apr 29 16:40:20 2026
 OS/Arch:           linux/amd64
 Context:           default

Server:
 Engine:
  Version:          29.1.3
  API version:      1.52 (minimum version 1.44)
  Go version:       go1.24.13
  Git commit:       29.1.3-0ubuntu4.1
  Built:            Wed Apr 29 16:40:20 2026
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          2.2.2
  GitCommit:
 runc:
  Version:          1.4.0-0ubuntu1
  GitCommit:
 docker-init:
  Version:          0.19.0
  GitCommit:
```

Le client et le serveur (le démon Docker) sont tous les deux en version **29.1.3**, installés depuis les paquets Ubuntu (`29.1.3-0ubuntu4.1`). Docker s'appuie sur `containerd` 2.2.2 et `runc` 1.4.0 pour lancer réellement les conteneurs.

**Réponse :**

- Version de Docker : **29.1.3**
- Noyau (ligne *Kernel Version*) : **7.0.0-38-generic**

Docker n'a pas de noyau propre : il utilise celui de la VM.

### Q2. Pourquoi être membre du groupe `docker` revient-il presque à être administrateur ? Quelle précaution prendre sur un serveur partagé ?

**Pourquoi.** Le démon Docker (`dockerd`) tourne en **root**. Le groupe `docker` donne accès à sa socket (`/var/run/docker.sock`), donc le droit de lui envoyer n'importe quelle commande, sans `sudo` ni mot de passe. Un simple membre du groupe peut par exemple lancer un conteneur qui monte tout le système de fichiers de l'hôte :

```bash
docker run -it --rm -v /:/hote alpine chroot /hote sh
```

Il obtient alors un shell root sur l'hôte : il peut lire `/etc/shadow`, modifier `/etc/sudoers` ou ajouter une clé SSH. Les options `--privileged` ou `--pid=host` donnent des possibilités équivalentes. Il n'y a donc pas de vraie séparation des privilèges : être dans le groupe `docker`, c'est être root.

**Précautions sur un serveur partagé :**

- ne mettre dans le groupe `docker` que des administrateurs de confiance, comme on le ferait pour `sudo` ;
- proposer aux autres utilisateurs un mode **rootless** (Docker rootless ou Podman) ou activer `userns-remap`, pour que root dans le conteneur ne soit pas root sur l'hôte ;
- ne pas exposer la socket Docker sur le réseau sans TLS et ne pas la monter dans un conteneur ;
- vérifier régulièrement qui fait partie du groupe avec `getent group docker`.

---

# Partie 1 : Lancer et inspecter des conteneurs

## 1.1. Fonctionnement de Docker avec `hello-world`

**Commandes saisies :**

```bash
docker run hello-world
docker images
docker ps -a
```

`docker run hello-world` télécharge l'image si besoin, crée un conteneur, affiche un message de confirmation puis s'arrête. `docker images` montre l'image `hello-world` en local et `docker ps -a` montre le conteneur terminé.

Le message affiché explique les quatre étapes réalisées par Docker :

1. Le client Docker contacte le démon Docker.
2. Le démon télécharge l'image `hello-world` depuis le Docker Hub.
3. Le démon crée un nouveau conteneur à partir de cette image et exécute le programme qui produit le message.
4. Le démon renvoie la sortie au client Docker, qui l'affiche dans le terminal.

Cet exemple montre la différence entre une image, qui sert de modèle, et un conteneur, qui est une instance créée à partir de cette image.

## 1.2. Serveur web Nginx et comparaison avec la VM

**Commandes saisies :**

```bash
docker run -d --name web -p 8080:80 nginx
docker ps
curl -s http://localhost:8080 | head -n 5
```

Le conteneur `web` tourne en arrière-plan et son port 80 est publié sur le port 8080 de la VM. Le `curl` récupère la page d'accueil de Nginx.

**Comparaison des systèmes, commandes saisies :**

```bash
docker exec web cat /etc/os-release | head -n 2
cat /etc/os-release | head -n 2
docker exec web uname -r
uname -r
```

**Résultats observés sur la VM :**

```console
$ cat /etc/os-release | head -n 2
PRETTY_NAME="Ubuntu 26.04.1 LTS"
NAME="Ubuntu"
$ uname -r
7.0.0-38-generic
```

**Résultats observés dans le conteneur :**

```console
etudiant@ubuntusrv:~/api$ docker run -d --name web -p 8080:80 nginx
63eac01a44ae01122282f3f6bd2a3b2ddc562322d3ea56028ea8be7c82e3d262
etudiant@ubuntusrv:~/api$ docker exec web cat /etc/os-release | head -n 2
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
$ docker exec web uname -r
7.0.0-38-generic
```

La VM tourne sous Ubuntu 26.04.1 LTS, alors que le conteneur Nginx tourne sous **Debian GNU/Linux 13 (trixie)** : il a son propre système de fichiers et sa propre distribution. En revanche, les deux affichent exactement le même noyau, `7.0.0-38-generic` : le conteneur partage le noyau de la VM. Un conteneur n'est donc pas une machine virtuelle complète.

## 1.3. Journaux, processus et consommation

**Commandes saisies :**

```bash
docker logs --tail 5 web
docker top web
ps aux | grep "nginx: master" | grep -v grep
docker stats --no-stream web
```

**Résultat de `docker logs --tail 5 web` (extrait) :**

```console
2026/10/20 11:26:21 [notice] 1#1: start worker process 30
172.17.0.1 - - [20/Oct/2026:11:28:54 +0000] "GET / HTTP/1.1" 200 896 "-" "curl/8.18.0" "-"
10.0.2.2 - - [20/Oct/2026:11:29:54 +0000] "GET / HTTP/1.1" 200 896 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
```

On y voit le démarrage des processus de travail de Nginx, la requête faite avec `curl` depuis la VM (adresse `172.17.0.1`) et celle faite depuis le navigateur Windows de l'hôte (adresse `10.0.2.2`, celle que voit la VM à travers la redirection VirtualBox).

**Résultat de `docker top web` :**

```console
etudiant@ubuntusrv:~/api$ docker top web
UID      PID     PPID    C    STIME   TTY   TIME       CMD
root     11010   10987   0    09:11   ?     00:00:00   nginx: master process nginx -g daemon off;
syslog   11081   11010   0    09:11   ?     00:00:00   nginx: worker process
syslog   11082   11010   0    09:11   ?     00:00:00   nginx: worker process
```

La commande affiche les processus du conteneur tels que l'hôte les voit : un processus maître (PID 11010) lancé par `root`, et deux processus de travail (PID 11081 et 11082) qui en dépendent (PPID 11010). Ces processus ont de vrais numéros de PID dans la VM. Les processus de travail apparaissent sous le nom `syslog` parce que leur numéro d'utilisateur correspond, côté VM, à cet utilisateur.

**Mesure de la consommation, commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker stats --no-stream web
CONTAINER ID   NAME   CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O     PIDS
63eac01a44ae   web    0.00%     3.375MiB / 7.251GiB   0.05%     1.08kB / 126B   0B / 24.6kB   3
```

Le conteneur Nginx `web` utilise 3.375 MiB de mémoire (0.05 %), pour 3 processus : cela correspond exactement aux trois processus vus avec `docker top`.

**Recherche du processus dans la liste de la VM.** Notre première tentative a échoué, car nous avions oublié le mot `grep` avant le motif :

```console
etudiant@ubuntusrv:~/api$ ps aux | "nginx: master" | grep -v grep
nginx: master: command not found
```

Notre deuxième tentative ne renvoie aucune ligne, car nous avons écrit `masteur` au lieu de `master`, donc aucun processus ne correspond au motif :

```console
etudiant@ubuntusrv:~/api$ ps aux | grep "nginx: masteur" | grep -v grep
etudiant@ubuntusrv:~/api$
```

Entre-temps, nous avons supprimé les deux conteneurs de mesure. **Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker rm -f web alpine-mesure
web
alpine-mesure
```

Les deux conteneurs (`web` et `alpine-mesure`) sont supprimés. Comme le conteneur `web` n'existait plus, nous l'avons relancé avant de refaire la recherche, cette fois avec la bonne commande :

```console
etudiant@ubuntusrv:~/api$ docker run -d --name web -p 8080:80 nginx
7d1c58663d50a73cf4a12e29c5c11b3a9e318ace64842482555c6165fb8b019b
etudiant@ubuntusrv:~/api$ ps aux | grep "nginx: master" | grep -v grep
root       11346  0.3  0.1  14976  9020 ?        Ss   09:46   0:00 nginx: master process nginx -g daemon off;
```

Cette fois, le processus maître de Nginx apparaît **directement dans la liste des processus de la VM** : il a le PID 11346 et appartient à `root`. Le conteneur est un processus comme un autre pour la VM.

Les processus du conteneur sont visibles depuis la VM : un conteneur est un simple processus de l'hôte, isolé par des mécanismes du noyau Linux (*namespaces* et *cgroups*), et non une machine virtuelle avec son propre noyau.

## 1.4. Trajet d'une requête HTTP vers Nginx

Quand le navigateur de la machine hôte ouvre `http://127.0.0.1:8080`, la requête passe par trois étapes :

1. Le navigateur s'adresse au port 8080 de la machine hôte.
2. VirtualBox redirige ce port vers le port 8080 de la VM.
3. Docker redirige le port 8080 de la VM vers le port 80 du conteneur `web`, grâce à l'option `-p 8080:80` du `docker run`.

## 1.5. Éphémérité des données et persistance avec un volume

**Commandes saisies, première partie : modifier la page dans le conteneur, puis le supprimer.**

```bash
docker exec web sh -c 'echo "<h1>Binôme TP2</h1>" > /usr/share/nginx/html/index.html'
curl -s http://localhost:8080
docker rm -f web
docker run -d --name web -p 8080:80 nginx
curl -s http://localhost:8080 | grep -o "<title>.*</title>"
```

Après la modification, le `curl` renvoie notre page personnalisée. Mais après `docker rm -f web` et un nouveau `docker run`, la page modifiée a disparu et c'est la page d'accueil par défaut de Nginx qui est servie. L'espace d'écriture d'un conteneur est éphémère : il est lié à son cycle de vie.

**Commandes saisies, deuxième partie : monter un dossier de la VM avec `-v`.**

```bash
mkdir -p ~/site
echo "<h1>Page servie depuis un volume</h1>" > ~/site/index.html
docker rm -f web
docker run -d --name web -p 8080:80 -v ~/site:/usr/share/nginx/html:ro nginx
curl -s http://localhost:8080
```

L'option `-v` monte le dossier `~/site` de la VM à la place du dossier de contenu de Nginx (en lecture seule avec `:ro`). Les données restent stockées en dehors du conteneur, donc elles survivent à sa suppression.

## 1.6. Bilan de la partie

Cette partie nous a permis de découvrir les images et les conteneurs, de comprendre que le noyau est partagé, d'observer les processus de l'hôte, et de voir comment fonctionnent la publication des ports et la persistance des données.

---

# Partie 2 : Comparer un conteneur et une machine virtuelle

## 2.1. Mesures sur la VM Ubuntu

### Temps de démarrage

**Commande saisie :**

```bash
systemd-analyze
```

**Résultat :**

```console
graphical.target reached after 18.263s in userspace.
```

| Mesure | Valeur |
|---|---:|
| Démarrage du noyau | 6.047 s |
| Initialisation de l'initrd | 10.661 s |
| Espace utilisateur (*userspace*) | 18.265 s |
| Temps total de démarrage | 34.973 s |
| Cible graphique atteinte après | 18.263 s dans l'espace utilisateur |

### Mémoire

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ free -h
               total        used        free      shared  buff/cache   available
Mem:           7,3Gi       627Mi       5,3Gi       1,1Mi       1,6Gi       6,6Gi
Swap:             0B          0B          0B
```

| Mesure mémoire | Valeur |
|---|---:|
| Mémoire totale | 7.3 Gi |
| Mémoire utilisée | 627 Mi |
| Mémoire libre | 5.3 Gi |
| Mémoire partagée | 1.1 Mi |
| Buffers/cache | 1.6 Gi |
| Mémoire disponible | 6.6 Gi |
| Swap total | 0 B |
| Swap utilisé | 0 B |
| Swap libre | 0 B |

### Espace disque de la VM

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              1,5G  1,2M  1,5G   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   19G  3,9G   14G  23% /
tmpfs                              3,7G     0  3,7G   0% /dev/shm
tmpfs                              3,7G     0  3,7G   0% /tmp
none                               1,0M     0  1,0M   0% /run/credentials/systemd-journald.service
none                               1,0M     0  1,0M   0% /run/credentials/systemd-resolved.service
/dev/sda2                          2,0G  100M  1,7G   6% /boot
none                               1,0M     0  1,0M   0% /run/credentials/systemd-networkd.service
tmpfs                              743M  8,0K  743M   1% /run/user/1000
none                               1,0M     0  1,0M   0% /run/credentials/getty@tty6.service
none                               1,0M     0  1,0M   0% /run/credentials/getty@tty1.service
```

Le système de fichiers principal (`/`) fait 19 Go, dont 3,9 Go utilisés (23 %). La partition `/boot` fait 2,0 Go avec 100 Mo utilisés (6 %).

### Taille du dossier de la VM

**Commande saisie depuis PowerShell (sur la machine hôte) et résultat :**

```console
PS C:\Users\HP> "{0:N2} Go" -f ((Get-ChildItem "$env:USERPROFILE\VirtualBox VMs\Ubuntu_serveur" -Recurse | Measure-Object Length -Sum).Sum / 1GB)
7,56 Go
```

Le dossier de la VM `Ubuntu_serveur` occupe **7,56 Go** sur le disque de la machine hôte.

## 2.2. Mesure de l'image Alpine

**Téléchargement, commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker pull alpine
Using default tag: latest
latest: Pulling from library/alpine
Digest: sha256:71e9b7bae160ba167fa17414401b94cbcdaf3c32feeab9298bd794a2a0614c35
Status: Image is up to date for alpine:latest
docker.io/library/alpine:latest
```

Docker indique que l'image est déjà à jour : elle était déjà présente en local, donc rien n'a été retéléchargé.

**Une erreur de frappe, puis la bonne commande :**

```console
etudiant@ubuntusrv:~/api$ docker image alpine
docker: unknown command: docker image alpine

Usage:  docker image

Run 'docker image --help' for more information
etudiant@ubuntusrv:~/api$ docker images alpine

IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
alpine:latest   71e9b7bae160           13MB         3.94MB
```

| Élément | Valeur |
|---|---|
| Image | `alpine:latest` |
| ID de l'image | `71e9b7bae160` |
| Taille sur disque (*DISK USAGE*) | 13 MB |
| Taille du contenu (*CONTENT SIZE*) | 3.94 MB |

### Système de l'image Alpine

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker run --rm alpine cat /etc/os-release
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.24.3
PRETTY_NAME="Alpine Linux v3.24"
HOME_URL="https://alpinelinux.org/"
BUG_REPORT_URL="https://gitlab.alpinelinux.org/alpine/aports/-/issues"
```

Le conteneur tourne bien sous **Alpine Linux v3.24** (version 3.24.3), alors que la VM est sous Ubuntu 26.04.1 LTS : le conteneur a sa propre distribution mais, comme on l'a vu, il partage le noyau de la VM.

### Mémoire d'un conteneur Alpine

Pour mesurer la mémoire d'Alpine, il faut un conteneur qui reste en vie quelques instants. Nous avons lancé `sleep 300` (une pause de 5 minutes). Notre première tentative a échoué, car nous avions oublié le mot `run` :

```console
etudiant@ubuntusrv:~/api$ docker -d --name alpine-mesure alpine sleep 300
unknown shorthand flag: 'd' in -d

Usage:  docker [OPTIONS] COMMAND [ARG...]

Run 'docker --help' for more information
```

**Commande correcte et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker run -d --name alpine-mesure alpine sleep 300
b9239000ba145f87f4da57b11344f0aadc7bf4047109711073f8cbe153458322
etudiant@ubuntusrv:~/api$ docker stats --no-stream alpine-mesure
CONTAINER ID   NAME            CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O   PIDS
b9239000ba14   alpine-mesure   0.00%     352KiB / 7.251GiB   0.00%     726B / 126B   0B / 0B     1
```

| Mesure | Valeur observée |
|---|---:|
| CPU | 0.00 % |
| Mémoire utilisée | 352 KiB |
| Limite mémoire affichée | 7.251 GiB |
| Pourcentage mémoire | 0.00 % |
| Réseau reçu / envoyé | 726 B / 126 B |
| Entrées-sorties disque | 0 B / 0 B |
| Nombre de processus | 1 |

Un conteneur Alpine qui fait une simple pause n'utilise que **352 KiB** de mémoire, avec un seul processus.

Alpine est très compacte quand on la compare à une distribution complète installée dans une VM.

## 2.3. Temps d'exécution d'Alpine

**Commande saisie** (lancée deux fois) :

```bash
time docker run --rm alpine echo "Bonjour depuis un conteneur"
```

**Résultat des deux essais :**

```console
Bonjour depuis un conteneur
```

suivi des temps mesurés :

| Mesure | Premier essai | Deuxième essai |
|---|---:|---:|
| `real` | 0m2,290s | 0m1,962s |
| `user` | 0m0,010s | 0m0,039s |
| `sys` | 0m0,075s | 0m0,081s |

`real` est le temps total écoulé. `user` et `sys` sont les temps CPU passés respectivement en espace utilisateur et dans le noyau.

## 2.4. Image et conteneur Nginx

### Première tentative : une erreur de commande

Nous avons d'abord tapé `docker image nginx`, qui a produit une erreur : la commande `docker image` attend une sous-commande (par exemple `ls` ou `inspect`).

```console
etudiant@ubuntusrv:~/api$ docker image nginx
docker: unknown command: docker image nginx

Usage:  docker image

Run 'docker image --help' for more information
```

### Téléchargement de l'image

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
8334177aa35b: Pull complete
d5f3a4724daf: Pull complete
dd00b11a22aa: Pull complete
9a2c3eb3312e: Pull complete
24bf03b6a474: Pull complete
a856bfc2a772: Pull complete
ee284e8d8c11: Download complete
6542b5f86ffd: Download complete
Digest: sha256:4c41e2d0a3a55b325ed572cd0200bdb72af1433c3daa2bca6ef8eea68ce504a2
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
```

### Taille de l'image

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker images nginx

IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:latest   4c41e2d0a3a5        243MB         66.8MB
```

| Élément | Valeur |
|---|---|
| Image | `nginx:latest` |
| ID de l'image | `4c41e2d0a3a5` |
| Taille sur disque (*DISK USAGE*) | 243 MB |
| Taille du contenu (*CONTENT SIZE*) | 66.8 MB |

### Lancement du conteneur

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker run -d --name nginx-mesure -p 8080:80 nginx
e136a7f19a048f4ce46e19cfc848557eaa7b2409bcea10edea9292b1ad35ba38
```

### Consommation de ressources

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker stats --no-stream nginx-mesure
CONTAINER ID   NAME           CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O     PIDS
e136a7f19a04   nginx-mesure   0.00%     3.504MiB / 7.251GiB   0.05%     946B / 126B   0B / 24.6kB   3
```

| Mesure | Valeur observée |
|---|---:|
| CPU | 0.00 % |
| Mémoire utilisée | 3.504 MiB |
| Limite mémoire affichée | 7.251 GiB |
| Pourcentage mémoire | 0.05 % |
| Réseau reçu | 946 B |
| Réseau envoyé | 126 B |
| Entrées-sorties disque | 0 B / 24.6 kB |
| Nombre de processus | 3 |

### Suppression du conteneur

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker rm -f nginx-mesure
nginx-mesure
```

### Système de l'image Nginx

Le résultat de `docker exec web cat /etc/os-release | head -n 2` (voir la partie 1.2) montre que l'image Nginx est basée sur **Debian GNU/Linux 13 (trixie)**.

## 2.5. Tableau comparatif

| Mesure | VM Ubuntu | Conteneur/image Alpine | Conteneur/image Nginx |
|---|---|---|---|
| Système | Ubuntu 26.04.1 LTS | Alpine Linux v3.24 | Debian GNU/Linux 13 (trixie) |
| Noyau | Linux 7.0.0-38-generic | Noyau partagé avec la VM | Noyau partagé avec la VM |
| Taille | Dossier de la VM : 7,56 Go | 13 MB sur disque ; contenu de 3.94 MB | 243 MB sur disque ; contenu de 66.8 MB |
| Temps mesuré | 34.973 s au total | 2.290 s puis 1.962 s pour la commande `echo` | Temps de démarrage non relevé |
| Mémoire | 627 Mi utilisés par la VM | 352 KiB pour le conteneur pendant la mesure | 3.504 MiB pour le conteneur pendant la mesure |

## 2.6. Questions

### Q1. Dans quel rapport le conteneur est-il plus léger et plus rapide que la VM ? Les mesures confirment-elles le tableau du cours ?

**Taille.** Le dossier de la VM occupe 7,56 Go, alors que l'image Alpine pèse 13 MB sur disque (3.94 MB de contenu). L'image Alpine est donc environ 600 fois plus petite que la VM (7,56 Go ≈ 7 700 MB, divisés par 13 MB).

**Mémoire.** La VM utilise 627 Mi, alors qu'un conteneur Alpine en pause n'utilise que 352 KiB (environ 1 800 fois moins) et un conteneur Nginx 3.504 MiB (environ 180 fois moins). Ces conteneurs ne font presque rien pendant la mesure, donc l'écart est surtout indicatif.

**Vitesse.** La VM met 34.973 s à démarrer, alors que nos deux exécutions de la commande Alpine ont pris 2.290 s et 1.962 s, soit environ 15 à 18 fois moins.

Il faut toutefois être prudent : pour la vitesse, nous comparons le démarrage complet d'une VM avec l'exécution d'une simple commande dans un conteneur, ce ne sont pas tout à fait les mêmes opérations. Pour la taille, la VM contient un système complet et ses données, alors qu'Alpine est volontairement minimale. Les mesures vont néanmoins dans le sens du cours : un conteneur est beaucoup plus léger et beaucoup plus rapide à lancer qu'une VM.

### Q2. Pourquoi le premier `time docker run` est-il plus long que le second ?

Le premier essai a duré 2.290 s, contre 1.962 s pour le second. L'image Alpine avait déjà été téléchargée avec `docker pull alpine` avant les mesures, donc le téléchargement n'explique pas forcément l'écart. Nos résultats montrent seulement que le deuxième essai est plus court, sans qu'on puisse en identifier la cause avec certitude. Si l'image n'avait pas été présente, le téléchargement aurait bien sûr allongé le premier essai.

### Q3. Citez un cas où vous choisiriez quand même une VM plutôt qu'un conteneur.

On choisirait une VM pour faire tourner un système d'exploitation complètement différent de celui de l'hôte, ou quand on a besoin d'une isolation matérielle forte assurée par un hyperviseur.

## 2.7. Bilan de la partie

Nous avons comparé plusieurs mesures entre la VM Ubuntu et les conteneurs Alpine et Nginx. L'image Alpine est très compacte et le conteneur Nginx consomme très peu de mémoire. En revanche, il nous manque certaines valeurs, donc nous ne pouvons pas calculer tous les rapports demandés.

---

# Partie 3 : Conteneuriser l'API fil rouge

## 3.1. Présentation de l'application

L'API fil rouge est une petite application Python écrite avec Flask. Nous avons créé le dossier de travail avec les fichiers `app.py` et `requirements.txt` :

```bash
mkdir -p ~/api && cd ~/api
```

Puis nous avons regardé le contenu du dossier et affiché le code.

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ ls
Dockerfile  app.py  requirements.txt
```

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ cat app.py
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

La route `/` renvoie un JSON avec trois champs :

- `message` : le texte « API fil rouge » ;
- `binome` : la valeur de la variable d'environnement `BINOME`, ou `inconnu` si elle n'est pas définie ;
- `conteneur` : le nom d'hôte, obtenu avec `socket.gethostname()`.

La route `/sante` renvoie `{"statut":"ok"}` et sert à vérifier que l'application répond. L'application écoute sur `0.0.0.0:5000`, ce qui la rend joignable depuis l'extérieur du conteneur grâce au port publié par Docker.

## 3.2. Dockerfile

Le Dockerfile part de l'image `python:3.12-slim` :

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

Le fichier `requirements.txt` contient la dépendance Flask. Le fichier `.dockerignore` exclut notamment :

```text
__pycache__/
*.pyc
.git
```

## 3.3. Construction de l'image

### Vérifications avant le build

Avant de construire l'image, nous avons fait deux erreurs en tapant `docker image` sans sous-commande, puis nous avons essayé d'afficher l'historique d'une image qui n'existait pas encore.

```console
etudiant@ubuntusrv:~/api$ docker image api
docker: unknown command: docker image api

Usage:  docker image

Run 'docker image --help' for more information
```

```console
etudiant@ubuntusrv:~/api$ docker image
Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Download an image from a registry
  push        Upload an image to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE

Run 'docker image COMMAND --help' for more information on a command.
```

```console
etudiant@ubuntusrv:~/api$ docker history api:1.0
Error response from daemon: No such image: api:1.0
```

Ce dernier message est normal : l'image `api:1.0` n'avait pas encore été construite.

### Première tentative de build : une erreur

```console
etudiant@ubuntusrv:~/api$ docker build -t api:1.0
ERROR: docker: 'docker buildx build' requires 1 argument

Usage:  docker buildx build [OPTIONS] PATH | URL | -

Run 'docker buildx build --help' for more information
```

Il manquait le chemin du contexte de construction, c'est-à-dire le point à la fin de la commande.

### Build réussi

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker build -t api:1.0 .
[+] Building 5.0s (11/11) FINISHED                                          docker:default
 => [internal] load build definition from Dockerfile                                  0.1s
 => => transferring dockerfile: 195B                                                  0.0s
 => [internal] load metadata for docker.io/library/python:3.12-slim                   2.2s
 => [auth] library/python:pull token for registry-1.docker.io                         0.0s
 => [internal] load .dockerignore                                                     0.1s
 => => transferring context: 64B                                                      0.0s
 => [1/5] FROM docker.io/library/python:3.12-slim@sha256:cb33f7f771ecec2ef1225cf39b159591a040391bd73e4c01f00fdfc4c47611d2   0.2s
 => => resolve docker.io/library/python:3.12-slim@sha256:cb33f7f771ecec2ef1225cf39b159591a040391bd73e4c01f00fdfc4c47611d2  0.2s
 => [internal] load build context                                                     0.0s
 => => transferring context: 125B                                                     0.0s
 => CACHED [2/5] WORKDIR /app                                                         0.0s
 => CACHED [3/5] COPY requirements.txt .                                              0.0s
 => CACHED [4/5] RUN pip install --no-cache-dir -r requirements.txt                   0.0s
 => CACHED [5/5] COPY . .                                                             0.0s
 => exporting to image                                                                1.8s
 => => exporting layers                                                               0.0s
 => => exporting manifest sha256:ceceeea61c5825ededef229997522e7c72c136168981c5d74bad88bc6f8cfff7   0.0s
 => => exporting config sha256:cad5f1e2f150ae5a0020c22d7a1c2c8d5e2f6f534e9c753365188d5968c05f9d     0.0s
 => => exporting attestation manifest sha256:68444b22cd48993fb32518b99d47fd8c1b7a4ca3623ba6a7e851c80cd3ad5ac3  0.2s
 => => exporting manifest list sha256:a57011b278a1d0e38e532e2ba8da9e37f2115eca9972bec7966a1f2ac7b11f78         0.1s
 => => naming to docker.io/library/api:1.0                                            0.0s
 => => unpacking to docker.io/library/api:1.0                                         1.2s
```

La construction a pris 5.0 secondes. Les étapes suivantes étaient marquées `CACHED` :

- `WORKDIR /app` ;
- `COPY requirements.txt .` ;
- `RUN pip install --no-cache-dir -r requirements.txt` ;
- `COPY . .`.

Docker réutilise ainsi les étapes déjà construites tant que leurs entrées n'ont pas changé. Les durées les plus longues sont le chargement des métadonnées de l'image Python (2.2 s), l'export de l'image (1.8 s) et son déballage (1.2 s).

## 3.4. Taille et historique de l'image `api:1.0`

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker images api

IMAGE     ID             DISK USAGE   CONTENT SIZE   EXTRA
api:1.0   a57011b278a1        198MB         48.3MB
```

| Élément | Valeur |
|---|---|
| Image | `api:1.0` |
| ID de l'image | `a57011b278a1` |
| Taille sur disque (*DISK USAGE*) | 198 MB |
| Taille du contenu (*CONTENT SIZE*) | 48.3 MB |

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker history api:1.0
IMAGE          CREATED                  CREATED BY                                      SIZE      COMMENT
a57011b278a1   3 hours ago              CMD ["python" "app.py"]                         0B        buildkit.dockerfile.v0
<missing>      3 hours ago              EXPOSE [5000/tcp]                               0B        buildkit.dockerfile.v0
<missing>      3 hours ago              COPY . . # buildkit                             20.5kB    buildkit.dockerfile.v0
<missing>      3 hours ago              RUN /bin/sh -c pip install --no-cache-dir -r…   15.3MB    buildkit.dockerfile.v0
<missing>      3 hours ago              COPY requirements.txt . # buildkit              12.3kB    buildkit.dockerfile.v0
<missing>      3 hours ago              WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      Less than a second ago   CMD ["python3"]                                 0B        buildkit.dockerfile.v0
<missing>      Less than a second ago   RUN /bin/sh -c set -eux; for src in idle3 p…    16.4kB    buildkit.dockerfile.v0
<missing>      Less than a second ago   RUN /bin/sh -c set -eux; savedAptMark="$(a…     41.4MB    buildkit.dockerfile.v0
<missing>      Less than a second ago   ENV PYTHON_SHA256=c2c4321961fab0fb999d66e0ce…   0B        buildkit.dockerfile.v0
<missing>      Less than a second ago   ENV PYTHON_VERSION=3.12.15                      0B        buildkit.dockerfile.v0
<missing>      Less than a second ago   ENV GPG_KEY=7169605F62C751356D054A26A821E680…   0B        buildkit.dockerfile.v0
<missing>      Less than a second ago   RUN /bin/sh -c set -eux; apt-get update; a…     4.92MB    buildkit.dockerfile.v0
<missing>      Less than a second ago   ENV LANG=C.UTF-8                                0B        buildkit.dockerfile.v0
<missing>      Less than a second ago   ENV PATH=/usr/local/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      About an hour ago        # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.7MB    debuerreotype 0.17
```

La sortie compte **16 entrées**. Celles qui viennent de notre Dockerfile sont :

1. `WORKDIR /app` ;
2. `COPY requirements.txt .` ;
3. `RUN pip install --no-cache-dir -r ...` ;
4. `COPY . .` ;
5. `EXPOSE 5000/tcp` ;
6. `CMD ["python" "app.py"]`.

Les autres entrées viennent de l'image de base Python/Debian (Python 3.12.15 sur Debian « trixie », architecture amd64). Les mentions `<missing>` sont normales : seules la dernière couche porte un identifiant d'image local.

### Pourquoi copier `requirements.txt` et installer les dépendances avant de copier le code ?

C'est pour profiter du cache. `requirements.txt` change beaucoup moins souvent que le code. Si on ne modifie que le code, Docker réutilise les couches d'installation des dépendances au lieu de tout refaire. Les lignes `CACHED` du build en sont un bon exemple.

## 3.5. Lancement du conteneur API

Avant de lancer le conteneur, nous avons vérifié qu'aucun conteneur n'existait déjà.

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Seul l'en-tête s'affiche : il n'y a aucun conteneur, ni en cours ni arrêté.

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker run -d --name api-test -p 5000:5000 api:1.0
bc4e4586ba56367c61c726daecb53699317c8a31c48275518fec5469a1cfd381
```

Le conteneur tourne en arrière-plan, s'appelle `api-test` et son port 5000 est publié sur le port 5000 de l'hôte. Docker affiche son identifiant complet.

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ docker ps
CONTAINER ID   IMAGE     COMMAND           CREATED         STATUS         PORTS                                         NAMES
bc4e4586ba56   api:1.0   "python app.py"   8 seconds ago   Up 8 seconds   0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp   api-test
```

| Champ | Valeur |
|---|---|
| ID court | `bc4e4586ba56` |
| Image | `api:1.0` |
| Commande | `python app.py` |
| État | `Up 8 seconds` |
| Redirection IPv4 | `0.0.0.0:5000->5000/tcp` |
| Redirection IPv6 | `[::]:5000->5000/tcp` |
| Nom du conteneur | `api-test` |

## 3.6. Test de la route `/`

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ curl http://localhost:5000/
{"binome":"inconnu","conteneur":"bc4e4586ba56","message":"API fil rouge"}
```

- `binome` vaut `inconnu` parce que nous n'avons pas défini la variable `BINOME` au lancement ;
- `conteneur` vaut `bc4e4586ba56`, le nom d'hôte renvoyé par `socket.gethostname()`, qui correspond à l'ID court du conteneur ;
- `message` contient bien `API fil rouge`.

On remarque aussi que les clés sont affichées dans l'ordre alphabétique (`binome`, `conteneur`, `message`).

## 3.7. Test de la route `/sante`

**Commande saisie et résultat :**

```console
etudiant@ubuntusrv:~/api$ curl http://localhost:5000/sante
{"statut":"ok"}
```

Les deux tests montrent que l'application Flask fonctionne dans le conteneur et que la redirection du port 5000 est correcte.

Nous avons aussi consulté les journaux du conteneur avec :

```bash
docker logs api
```

Cette commande affiche les messages de Flask et les requêtes HTTP reçues par l'application.

## 3.8. Variable d'environnement `BINOME`

Pour afficher le nom du binôme dans la réponse, on définit la variable au lancement du conteneur :

```bash
docker run -d --name api -p 5000:5000 -e BINOME="Binome-TP2" api:1.0
curl -s http://localhost:5000
curl -s http://localhost:5000/sante
```

**Résultats :**

```console
{"binome":"Binome-TP2","conteneur":"a19c19429988","message":"API fil rouge"}
{"statut":"ok"}
```

Cette fois, le champ `binome` affiche bien `Binome-TP2`, et le champ `conteneur` a changé (`a19c19429988`) parce que ce nouveau conteneur a un nouvel identifiant.

Une variable d'environnement permet de garder la configuration séparée du code source. En revanche, les secrets comme les mots de passe ou les clés ne doivent pas être écrits en dur dans le code ou dans l'image : il faut les gérer avec des mécanismes adaptés.

## 3.9. Lancement de trois copies de l'API

Nous avons lancé trois conteneurs à partir de la même image, sur les ports 5001, 5002 et 5003, puis interrogé chacun :

```bash
for p in 5001 5002 5003; do docker run -d --name api-$p -p $p:5000 -e BINOME="Binome-TP2" api:1.0; done
for p in 5001 5002 5003; do curl -s http://localhost:$p; echo; done
docker rm -f api-5001 api-5002 api-5003
```

Les valeurs du champ `conteneur` renvoyées par les trois copies sont :

| Port | Valeur du champ `conteneur` |
|---|---|
| 5001 | `71bc8b8f46f1` |
| 5002 | `402984b32771` |
| 5003 | `e3382f736d28` |

Chaque copie renvoie un nom d'hôte différent : les trois conteneurs sont bien des instances distinctes de la même image. Cette expérience illustre le passage à l'échelle horizontal (*scaling*) : on peut multiplier les instances d'une même image très facilement. Les conteneurs de test sont ensuite supprimés avec `docker rm -f`.

## 3.10. Ajout de la route `/version`

Le sujet demande d'ajouter une route `/version` qui renvoie :

```json
{"version":"1.1"}
```

Nous avons ensuite reconstruit l'image avec le tag `api:1.1`, relancé le conteneur et testé la route :

```bash
docker build -t api:1.1 .
docker rm -f api && docker run -d --name api -p 5000:5000 api:1.1
curl -s http://localhost:5000/version
docker history api:1.1
```

La reconstruction réutilise les couches en cache : seules les couches qui suivent la modification du code sont refaites. `docker history api:1.1` permet de le vérifier.


## 3.11. Bilan de la partie

Dans cette partie, nous avons construit une image à partir d'une application Flask, examiné ses couches et vu le cache de construction à l'œuvre. Le conteneur `api-test` a démarré sans problème et les routes `/` et `/sante` répondent correctement. La variable `BINOME` permet de personnaliser la réponse sans toucher au code, et le lancement de trois instances montre qu'une même image peut servir à faire tourner plusieurs conteneurs.

---

# Partie 4 : Publier l'image et nettoyer

## 4.1. Publication sur Docker Hub

L'image publiée est :

```text
ayce1324/api-fil-rouge:1.1
```

**Commandes saisies** pour nous connecter, étiqueter l'image et la publier :

```bash
docker login -u ayce1324
docker tag api:1.1 ayce1324/api-fil-rouge:1.1
docker push ayce1324/api-fil-rouge:1.1
```

**Résultats :**

```console
Login Succeeded
...
1.1: digest: sha256:412dda176344bdc59c4d8b865f375e4b90ae10418df681e574d2d9ea1c7281c3 size: 856
```

`docker login` demande le mot de passe (ou un jeton d'accès personnel) et confirme par `Login Succeeded`. Le `docker push` envoie toutes les couches puis affiche le digest de l'image publiée.

**Vérification depuis le registre :** nous avons supprimé les images locales puis relancé l'application à partir de l'image publiée :

```bash
docker rm -f api
docker rmi ayce1324/api-fil-rouge:1.1 api:1.1
docker run -d --name api -p 5000:5000 ayce1324/api-fil-rouge:1.1
curl -s http://localhost:5000/version
```

Comme l'image n'existait plus en local, Docker l'a téléchargée depuis le Docker Hub (toutes les couches en `Pull complete`) avant de lancer le conteneur. Cela prouve que l'image publiée est bien utilisable par n'importe qui. Le dépôt est public, et le tag `1.1` identifie clairement une version précise.

## 4.2. Nettoyage des conteneurs et des images

**Commandes saisies :**

```bash
docker rm -f $(docker ps -aq)
docker system df
docker image prune -a -f
docker system df
docker pull alpine
docker images
```

**Résultat de `docker image prune -a -f` :**

```console
Total reclaimed space: 36.97MB
```

**Résultat de `docker system df` relevé après le nettoyage :**

| Type | Total | Actif | Taille | Récupérable |
|---|---:|---:|---:|---:|
| Images | 0 | 0 | 299.3 MB | 299.3 MB (100 %) |
| Conteneurs | 0 | 0 | 0 B | 0 B |
| Volumes locaux | 0 | 0 | 0 B | 0 B |
| Cache de construction | 19 | 0 | 308.1 MB | 308.1 MB |

**Résultat de `docker images` après avoir retéléchargé Alpine :**

| Élément | Valeur |
|---|---|
| Image | `alpine:latest` |
| ID | `71e9b7bae160` |
| Taille sur disque | 13 MB |
| Taille du contenu | 3.94 MB |

Nettoyer permet de libérer de la place en supprimant les conteneurs et images dont on n'a plus besoin. Il faut quand même vérifier ce qu'on supprime, au cas où quelque chose serait encore utile.

## 4.3. Questions

### Q1. Quelle est l'adresse complète de l'image publiée ? Est-elle publique ou privée ?

L'image est `ayce1324/api-fil-rouge:1.1`, sur le Docker Hub. Le dépôt est public.

### Q2. Pourquoi utiliser un numéro de version comme `1.1` plutôt que toujours `latest` ?

Un tag précis identifie exactement une version de l'application, garantit la traçabilité et permet de la récupérer à l'identique à chaque déploiement. Le tag `latest`, lui, peut changer au fil des publications, ce qui provoque des effets de bord.

### Q3. Quel espace disque la commande de nettoyage a-t-elle libéré ? Pourquoi nettoyer aussi les images dans un registre distant ?

`docker image prune -a -f` a libéré **36.97 MB** en local. Dans le cloud, nettoyer les registres évite d'accumuler d'anciennes versions d'images devenues inutiles et réduit les coûts de stockage.

## 4.4. Bilan de la partie

Cette partie nous a fait revoir la publication d'une image sur Docker Hub, l'intérêt des tags de version et les commandes pour nettoyer les ressources Docker locales.

---


# Conclusion générale

Ce TP nous a fait découvrir les bases de Docker et le cycle de vie d'un conteneur.

En comparant la VM Ubuntu avec les images Alpine et Nginx, nous avons pu observer des tailles, des temps d'exécution et une consommation de mémoire. Alpine est très compacte et un conteneur Nginx consomme très peu de mémoire quand il tourne. Nous avons aussi vérifié que le conteneur partage le noyau de la VM (`7.0.0-38-generic` des deux côtés).

Nous avons ensuite conteneurisé une application Flask avec l'image `api:1.0`. L'historique de l'image nous a montré qu'elle est faite de plusieurs couches, certaines venant de l'image de base et d'autres de notre Dockerfile, et les étapes `CACHED` nous ont montré l'intérêt du cache. Le conteneur `api-test` a démarré correctement, les routes `/` et `/sante` répondent comme prévu, la variable `BINOME` permet de changer la configuration sans modifier le code, et trois instances de la même image ont tourné en parallèle avec des identifiants différents.

Enfin, la publication de `ayce1324/api-fil-rouge:1.1` sur Docker Hub, sa récupération depuis le registre et le nettoyage des ressources ont bouclé le cycle : construire, publier et gérer des images Docker.

**En résumé, ce TP nous a appris à construire, lancer, tester, versionner et publier une application conteneurisée, tout en gardant un regard critique sur les limites de nos mesures.**

