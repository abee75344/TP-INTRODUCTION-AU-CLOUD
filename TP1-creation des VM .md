# TP 1 — Installation de VirtualBox et création d'une VM Ubuntu


**Cours :** Introduction au Cloud · L3 SI RIST & BDGL
**Date de réalisation :** 12 octobre 2026
**Étudiant(s) :**
*-Etudiant 1: ABE YEPI CHRIST-EMMANUEL*
*-Etudiante 2: AKELY MAYLINE PHOEBE EDWINA*
*-Etudiant 3: DANHO AKRE PRINCE PATRICK-JUNIOR*
**Professeur :**
Dr BAKAYOKO IBRAHIMA
**Statut du compte rendu :** Parties 1 à 6 et comparaison Multipass / VirtualBox terminées 


---

## Partie 1 — Préparer l'hôte et installer VirtualBox

### Actions réalisées
1. Vérification de la virtualisation matérielle dans le Gestionnaire des tâches (Performance → Processeur → Virtualisation : **Activé**).

2. Vérification de l'état de l'hyperviseur Windows (Hyper-V / WSL 2) :
   ```powershell
   bcdedit /enum | findstr hypervisorlaunchtype
   ```
   *(aucun résultat affiché)*

   **But :** vérifier si l'hyperviseur Windows (Hyper-V) est configuré pour se lancer au démarrage, car il peut entrer en conflit avec VirtualBox.

   ```powershell
   systeminfo | findstr /C:"Hyper-V"
   ```
   *Configuration requise pour Hyper-V : Un hyperviseur a été détecté. Les fonctionnalités nécessaires à Hyper-V ne seront pas affichées.*

   **But :** savoir si un hyperviseur est déjà actif sur l'hôte et si les fonctionnalités Hyper-V sont disponibles.

3. Téléchargement et installation de VirtualBox pour Windows (options par défaut, installation du pilote réseau acceptée).
4. Vérification de l'installation :
   ```powershell
   $env:Path += ";C:\Program Files\Oracle\VirtualBox"
   ```
   **But :** ajouter le dossier d'installation de VirtualBox au `PATH` de la session PowerShell, pour appeler `VBoxManage` sans taper son chemin complet.

   ```powershell
   VBoxManage --version
   7.2.20r175154
   ```
   **But :** afficher la version de VirtualBox installée (branche 7.2, comme indiqué dans l'énoncé).

   ```powershell
   VBoxManage list hostinfo
   ```
   ```
   Host Information:

Host time: 2026-10-14T23:12:00.464000000Z
Processor online count: 12
Processor count: 12
Processor online core count: 10
Processor core count: 10
Processor supports HW virtualization: yes
Processor supports PAE: yes
Processor supports long mode: yes
Processor supports nested paging: yes
Processor supports unrestricted guest: no
Processor supports nested HW virtualization: no
Processor supports virt. vmsave/vmload: no
Processor#0 speed: unknown
Processor#0 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#1 speed: unknown
Processor#1 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#2 speed: unknown
Processor#2 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#3 speed: unknown
Processor#3 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#4 speed: unknown
Processor#4 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#5 speed: unknown
Processor#5 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#6 speed: unknown
Processor#6 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#7 speed: unknown
Processor#7 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#8 speed: unknown
Processor#8 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#9 speed: unknown
Processor#9 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#10 speed: unknown
Processor#10 description: 13th Gen Intel(R) Core(TM) i7-1355U
Processor#11 speed: unknown
Processor#11 description: 13th Gen Intel(R) Core(TM) i7-1355U
Memory size: 16027 MByte
Memory available: 3670 MByte
Operating system: Windows 11
Operating system version: 10.0.26200.9457
```
   **But :** afficher les caractéristiques de l'hôte (processeur, prise en charge de la virtualisation matérielle, mémoire, système) pour dimensionner la VM.

5. Exploration de l'interface : Fichier → Paramètres → Général (dossier des machines) et Fichier → Outils → Gestionnaire de réseau.


### Réponses aux questions

**1. Hyperviseur de type 1 et de type 2**
-D'abord un hyperviseur est un logiciel sous jacent qui gère les machines virtuelles

- Un hyperviseur de **type 1** (bare-metal) s'exécute directement sur le matériel, sans système d'exploitation hôte. Exemples : KVM, VMware ESXi, Xen, Microsoft Hyper-V. Un fournisseur cloud comme AWS utilise Nitro, basé sur KVM.

- Un hyperviseur de **type 2** (hébergé) s'exécute comme une application au-dessus d'un système d'exploitation hôte.

- **VirtualBox est de type 2** : il tourne sur Windows.

**2. Pourquoi la virtualisation matérielle (VT-x / AMD-V) est indispensable**
Ces extensions du processeur permettent d'exécuter le code de la VM de façon isolée et efficace, directement sur le processeur. Sans elles, VirtualBox ne peut pas faire tourner une VM 64 bits.

**3. Version de VirtualBox et ressources de l'hôte**
Version installée : **7.2.20r175154**. 
-Nombre de cœurs et RAM de l'hôte renvoyés par `VBoxManage list hostinfo` : **Nombre de coeurs-Processor online count: 12**
**Memory size: 16027 MByte** .

---

## Partie 2 — Télécharger et vérifier l'image ISO Ubuntu

### Actions réalisées
1. Téléchargement d'Ubuntu Server LTS (amd64) : fichier **`ubuntu-26.04.1-live-server-amd64.iso`**, stocké dans `D:\ubuntu\`.
2. Récupération du fichier `SHA256SUMS` de la même version sur `releases.ubuntu.com`.
3. Calcul et comparaison de l'empreinte :
   ```powershell
   cd $HOME\Downloads
   Get-FileHash (Get-ChildItem .\*live-server-amd64.iso) -Algorithm SHA256
   Select-String -Path .\SHA256SUMS -Pattern "live-server-amd64"
   ```

| Source | Empreinte SHA-256 |
|---|---|
| Fichier téléchargé (`Get-FileHash`) | `CC8A95CDE20F6CED61A322420DE00F10CC3C90CED545DAA46CB9C1A117F1D927` |
| `SHA256SUMS` (ligne 5, `ubuntu-26.04.1-live-server-amd64.iso`) | `cc8a95cde20f6ced61a322420de00f10cc3c90ced545daa46cb9c1a117f1d927` |

**But :** `Get-FileHash` calcule l'empreinte SHA-256 de l'ISO téléchargée ; `Select-String` extrait du fichier `SHA256SUMS` la ligne correspondante, afin de comparer les deux valeurs et valider l'intégrité et l'authenticité de l'image.

**Résultat : les empreintes sont identiques** (la casse est la seule différence, sans importance).

4. Taille du fichier ISO : **2,73 Gio** (valeur affichée par VirtualBox dans le lecteur optique).

### Remarque
`Select-String` a renvoyé deux lignes : la ligne 2 (`ubuntu-26.04-live-server-amd64.iso`) et la ligne 5 (`ubuntu-26.04.1-live-server-amd64.iso`). Seule la ligne 5 correspond à mon fichier. La première concerne une autre image que je n'ai pas téléchargée. Le nom de mon fichier diffère donc de celui indiqué dans l'énoncé (`26.04.1` au lieu de `26.04`).

### Réponses aux questions

**1. Les empreintes sont-elles identiques ? Pourquoi vérifier ?**
Oui, elles sont identiques. Vérifier l'empreinte garantit que le fichier n'a pas été corrompu pendant le téléchargement et qu'il n'a pas été modifié ou piégé par un tiers. Il faut aussi que le fichier `SHA256SUMS` provienne du site officiel.

**2. Ubuntu Desktop et Ubuntu Server**
- Desktop inclut une interface graphique et des applications bureautiques.
- Server fonctionne en ligne de commande uniquement : il est plus léger (moins de RAM, de disque et de CPU) et expose moins de composants, donc une surface d'attaque réduite.
- On choisit Server pour un serveur cloud pour ces raisons, et parce qu'il s'administre à distance (SSH, automatisation).

**3. Signification de LTS**
*Long Term Support* : version à support prolongé. Elle reçoit **5 ans** de mises à jour de sécurité standard (extensibles avec Ubuntu Pro).

---

## Partie 3 — Créer et configurer la machine virtuelle

### Paramètres retenus

| Paramètre | Valeur |
|---|---|
| Nom | `ubuntu-srv` |
| Dossier de la VM | `D:\ubuntu` |
| Image ISO | `ubuntu-26.04.1-live-server-amd64.iso` |
| Type / version | Linux / Ubuntu (64-bit) |
| Installation sans surveillance | Désactivée (« Skip Unattended Installation » cochée) |
| Mémoire vive | 2048 Mo |
| Processeurs | 2 |
| Disque dur | VDI, 25 Go, alloué dynamiquement (`ubuntu-srv.vdi`) |
| Carte réseau 1 | NAT |

### Redirection de port SSH

| Nom | Protocole | IP hôte | Port hôte | IP invité | Port invité |
|---|---|---|---|---|---|
| ssh | TCP | 127.0.0.1 | 2222 | _(vide)_ | 22 |

### Vérification en ligne de commande
```powershell
VBoxManage showvminfo ubuntu-srv
```
Extrait :
```
NIC 1: MAC: 080027CB20AC, Attachment: NAT, Cable connected: on ...
NIC 1 Rule(0): name = ssh, protocol = tcp, host ip = 127.0.0.1, host port = 2222, guest ip = , guest port = 22
NIC 2 à 8 : disabled
```

**But :** `VBoxManage showvminfo` affiche la configuration complète de la VM (mémoire, CPU, disque, cartes réseau, règles de redirection de port) pour vérifier en ligne de commande que les paramètres saisis sont corrects.
Le récapitulatif de VirtualBox confirme : 2048 Mo, 2 processeurs, disque `ubuntu-srv.vdi` de 25,00 Gio, ISO attachée dans le lecteur optique, interface 1 en NAT.


**Taille du fichier `.vdi`** :

- Taille actuelle du fichier de base `ubuntu-svr.vdi` (propriétés du fichier dans `D:\ubuntu\ubuntu-srv\`), mesurée après l'installation d'Ubuntu Server : **6,55 Go** pour une capacité maximale de **25 Go**. Le fichier n'occupe donc qu'une partie de la capacité annoncée, ce qui illustre l'allocation dynamique. Son nom (`ubuntu-srvr.vdi`) n'a pas changé lors du renommage de la VM.

- Taille juste après la création du disque, avant l'installation : **2,00 Mo**, mesurée sur un disque neuf identique (VDI, 25 Go, alloué dynamiquement) créé en ligne de commande puis supprimé :
  ```powershell

  VBoxManage createmedium disk --filename "D:\ubuntu\test-disque.vdi" --size 25600 --format VDI
  Medium created. UUID: b30bb49a-5a41-4d76-b461-97b3e5bbaaf9
  "{0:N2} Mo" -f ((Get-Item "D:\ubuntu\test-disque.vdi").Length / 1MB)
  2,00 Mo
  VBoxManage closemedium disk "D:\ubuntu\test-disque.vdi" --delete
  ```

  **But :**
  - `VBoxManage createmedium disk ...` : créer un disque virtuel VDI de 25 Go (25600 Mo) alloué dynamiquement.
  - `"{0:N2} Mo" -f (...)` : mesurer la taille réelle du fichier `.vdi` sur le disque de l'hôte, en Mo.
  - `VBoxManage closemedium disk ... --delete` : détacher puis supprimer le disque de test, devenu inutile.
  Un disque de 25 Go ne pèse donc que **2 Mo** à sa création : le fichier ne grossit qu'au fur et à mesure des écritures.



### Réponses aux questions

**1. Disque dynamique et disque fixe**
- Un disque **alloué dynamiquement** est un fichier `.vdi` qui grossit au fur et à mesure que la VM y écrit des données. Juste après sa création, il est très petit.
- Un disque de **taille fixe** réserve immédiatement les 25 Go sur l'hôte. Il est un peu plus rapide, mais occupe tout l'espace dès le départ.
- Taille du `.vdi` juste après création : **2,00 Mo** pour une capacité de 25 Go. Après installation du système, le fichier de base pèse **6,55 Go** : il a grossi avec les données écrites.

**2. Pourquoi désactiver l'installation sans surveillance**
Pour réaliser soi-même chaque étape de l'installation (langue, clavier, réseau, stockage, profil, SSH) et en comprendre les choix, comme sur un vrai serveur.

**3. NAT et redirection 2222 → 22**
En mode NAT, la VM est derrière un routeur virtuel de VirtualBox et possède une adresse privée (`10.0.2.15`) non joignable directement depuis l'hôte. La redirection envoie toute connexion vers `127.0.0.1:2222` sur l'hôte vers le port 22 (SSH) de la VM.

---

## Partie 4 — Installer Ubuntu Server

### Déroulement
La VM a été démarrée avec **Démarrer → Start with GUI** (équivalent du « Démarrage normal »), puis **Try or Install Ubuntu Server**.

| Écran de l'installateur | Choix |
|---|---|
| Langue | Français |
| Mise à jour de l'installateur | Continuer sans mettre à jour |
| Clavier | French (AZERTY) |
| Type d'installation | Ubuntu Server (pas la version minimisée) |
| Réseau | `enp0s3` en DHCPv4 : adresse `10.0.2.15` |
| Proxy / miroir | Par défaut |
| Stockage | Tout le disque, avec LVM (choix par défaut) |
| Profil | Serveur `ubuntu-srv`, utilisateur `etudiant`, mot de passe retenu |
| Ubuntu Pro | Passé |
| SSH | « Installer le serveur OpenSSH » coché |
| Snaps proposés | Aucun |

L'installateur a affiché la fin du processus (installation du noyau, de GRUB, d'`openssh-server`, de `cloud-init`, puis téléchargement des mises à jour de sécurité). Des avertissements `soft lockup - CPU stuck for 34s` sont apparus pendant cette phase : ils signalent une lenteur du processeur virtuel (hyperviseur de type 2 partageant les ressources de l'hôte) et n'ont pas empêché l'installation de se terminer.

Après redémarrage, la connexion sur la console avec `etudiant` a fonctionné :
```
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-38-generic x86_64)
IPv4 address for enp0s3: 10.0.2.15
```

### Réponses aux questions

**1. Durée de l'installation et comparaison avec le cloud**
Durée : 25:26:03. L'installation à partir d'une ISO est longue (de l'écran de langue aux mises à jour de sécurité), alors qu'une instance cloud démarre en une à deux minutes à partir d'une image déjà préparée.

**2. Adresse IP obtenue sur `enp0s3` et origine**
`10.0.2.15`, attribuée par le **serveur DHCP intégré au réseau NAT de VirtualBox**.

**3. Pourquoi installer OpenSSH dès l'installation**
Sans serveur SSH, il n'y aurait aucun moyen d'administrer la machine à distance : seule la console de l'hyperviseur serait disponible. Sur un serveur cloud, qui n'a ni écran ni clavier physique, l'absence de SSH ferait perdre l'accès à la machine.

---

## Comparaison Multipass et VirtualBox

Cette section compare la VM Ubuntu Server créée avec **VirtualBox** (Parties 1 à 4) à une VM Ubuntu créée avec **Multipass**.

### Version de Multipass

```powershell
PS C:\Users\DELL> multipass version
multipass   1.16.4+win
```
**But :** afficher la version de Multipass installée sur l'hôte Windows (**1.16.4**, build Windows).

### Informations de la VM Multipass

Informations affichées à la connexion à la VM :

```
System load:    0.15 0.13 0.10
Disk usage:     2.2GiB out of 6.7GiB
Memory usage:   375.6MiB out of 1.9GiB
```
**But :** résumer l'état de la VM à la connexion : charge moyenne du processeur (sur 1, 5 et 15 minutes), espace disque utilisé et mémoire vive utilisée.

```
$ lsb_release -a
Distributor ID: Ubuntu
Description:    Ubuntu 26.04.1 LTS
Release:        26.04
Codename:       resolute
```
**But :** afficher la distribution Linux de la VM (nom, version, description, nom de code).

```
$ uname -r
7.0.0-34-generic
```
**But :** afficher la version du noyau Linux en cours d'exécution.

### Temps de création et image utilisée

- **Temps de création de la VM Multipass : environ 5 minutes.**
- Multipass fournit déjà des **images prêtes à être téléchargées** (Ubuntu LTS notamment) : il n'est **pas nécessaire de télécharger une image ISO**, ni de **vérifier une empreinte SHA256** à la main (Partie 2), ni de dérouler l'installateur (Partie 4). L'image est téléchargée puis la VM démarre directement, déjà configurée (utilisateur, SSH, `cloud-init`). L'intégrité de l'image est contrôlée automatiquement par l'outil.
- Avec VirtualBox, il faut télécharger l'ISO (2,73 Gio), vérifier son empreinte, créer la VM, régler ses paramètres, puis installer le système à la main (durée relevée en Partie 4 : 25:26).

### Tableau comparatif

| Critère | VirtualBox | Multipass |
|---|---|---|
| Version utilisée | 7.2.20r175154 | 1.16.4+win |
| Nature | Hyperviseur de type 2 avec interface graphique et ligne de commande (`VBoxManage`) | Outil en ligne de commande pour créer et gérer des VM Ubuntu ; il s'appuie sur un hyperviseur de l'hôte |
| Source du système | ISO téléchargée (2,73 Gio) + vérification SHA256 manuelle | Image Ubuntu prête à l'emploi, sans ISO ni SHA256 manuel |
| Installation du système | Manuelle (langue, clavier, stockage, profil, SSH) | Aucune, VM prête au démarrage |
| Temps de création | ≈ 25 min (25:26 relevé en Partie 4) | ≈ 5 min |
| Système obtenu | Ubuntu 26.04.1 LTS, noyau 7.0.0-38-generic | Ubuntu 26.04.1 LTS, noyau 7.0.0-34-generic |
| Mémoire | 2048 Mo configurés (1,6 Gi visibles) | 1,9 GiB (375,6 MiB utilisés) |
| Disque | 25 Go (VDI dynamique) ; `/` : 12 Go, 5,4 Go utilisés | 6,7 GiB ; 2,2 GiB utilisés |
| Accès SSH | À configurer (NAT, redirection 2222 → 22, serveur OpenSSH) | Fourni par l'outil, sans configuration |
| Instantanés et clones | Oui (`VBoxManage snapshot`, `clonevm`), très complets | Oui, mais plus simples |
| Réseau | NAT, pont, réseau hôte, etc., entièrement configurable | Préconfiguré, peu de réglages |
| Systèmes invités | Nombreux (Linux, Windows, etc.) | Ubuntu uniquement |
| Contrôle et pédagogie | Maximal : on voit chaque étape | Limité : tout est automatisé |

### Conclusion

Les deux outils ont produit la même version d'Ubuntu (26.04.1 LTS), mais avec des approches très différentes.

**Multipass est la solution la plus adaptée à la virtualisation d'un serveur Ubuntu dans un contexte cloud.** Elle a créé la VM en environ 5 minutes contre environ 25 pour VirtualBox, n'exige ni ISO ni vérification SHA256, et fournit une machine déjà configurée et accessible en SSH. Elle est légère (375,6 MiB de RAM utilisés, 2,2 GiB de disque, charge quasi nulle) et se rapproche du fonctionnement d'une instance cloud (image préparée, démarrage rapide, pilotage en ligne de commande), ce qui la rend idéale pour tester, développer ou déployer rapidement.

**VirtualBox reste préférable** lorsqu'il faut un contrôle fin : installer un autre système que Ubuntu (Windows, autre distribution), configurer précisément le réseau, gérer des instantanés et des clones avancés, ou comprendre en détail chaque étape de l'installation, ce qui était l'objectif pédagogique de ce TP.

En résumé : **Multipass pour la rapidité, la légèreté et la proximité avec le cloud ; VirtualBox pour la flexibilité, le contrôle et l'apprentissage.**


toutefoispour la tenu du tp , nous avons continuer avec la VM sur VirtualBox 
---

## Partie 5 — Premiers pas, réseau et accès SSH 

### 5.1 Identification du système et des ressources

```
$ lsb_release -a
No LSB modules are available
Distributor ID: Ubuntu
Description: Ubuntu 26.04.1 LTS
Release: 26.04
Codename: resolute
```
**But :** afficher des informations détaillées sur la distribution Linux utilisée (nom, version, description, nom de code).

```
$ uname -r
7.0.0-38-generic
```
**But :** afficher uniquement la version du noyau (kernel) Linux en cours d'exécution.

```
$ hostnamectl
Static hostname: unbuntu-srvr
Icon name: computer-vm
Chassis: vm
Virtualization: oracle
Operating System: Ubuntu 26.04.1 LTS
Kernel: Linux 7.0.0-38-generic
Architecture: x86-64
Hardware Vendor: innotek GmbH
Hardware Model: VirtualBox
```
**But :** afficher des informations complètes sur le système : nom de la machine (hostname), système d'exploitation, version du noyau, architecture, et type de virtualisation.

Le système sait qu'il est virtualisé par VirtualBox (`Virtualization: oracle`, `Chassis: vm`).

```
$ nproc
2
```
**But :** afficher le nombre d'unités de traitement (cœurs de processeur) disponibles pour le système.

```
$ free -h
          total   used   free   shared  buff/cache  available
Mem:      1,6Gi   367Mi  886Mi  1,1Mi   535Mi       1,2Gi
Swap:     2,0Gi   0B     2,0Gi
```
**But :** afficher l'utilisation de la mémoire vive (RAM) et de la zone d'échange (swap) dans un format lisible par un humain (Go, Mo).

```
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5,4G  5,4G  50% /
```
**But :** afficher l'espace disque utilisé et disponible sur la partition racine (`/`), au format lisible par un humain.

```
$ lsblk
sda                       25G  disk
├─sda1                     1M  part
├─sda2                     2G  part  /boot
└─sda3                    23G  part
  └─ubuntu--vg-ubuntu--lv 11,5G lvm  /
```
**But :** afficher la liste de tous les périphériques de stockage (disques, partitions, volumes LVM) détectés par le système, sous forme d'arborescence.

Le disque virtuel fait 25 Go, mais l'installateur n'a alloué que 11,5 Go au volume logique `/` (comportement par défaut de LVM). L'espace restant est disponible dans le groupe de volumes et peut être récupéré avec `lvextend`.

### 5.2 Réseau et accès à Internet

**Table de routage :**
```
$ip -4 addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 85953sec preferred_lft 85953sec

$ ip route

default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.1.1 via 10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
```

**But :**
- `ip -4 addr show` : afficher les adresses IPv4 des interfaces réseau (adresse, masque, durée du bail DHCP).
- `ip route` : afficher la table de routage (passerelle par défaut, réseau local) pour savoir par où sortent les paquets.

| Élément | Valeur |
|---|---|
| Adresse IP / masque | `10.0.2.15/24` |
| Passerelle | `10.0.2.2` |
| Serveur DNS | `192.168.1.1` (IPv4) et `fd17:625c:f037:2::3` (IPv6), relevés avec `resolvectl dns` |

**Test de connectivité :**
```
$ ping -c 3 ubuntu.com
PING ubuntu.com (185.125.190.20) 56(84) bytes of data.
64 bytes from website-content-cache-1.ps5.canonical.com (185.125.190.20): icmp_seq=1 ttl=64 time=120 ms
64 bytes from website-content-cache-1.ps5.canonical.com (185.125.190.20): icmp_seq=2 ttl=64 time=116 ms
64 bytes from website-content-cache-1.ps5.canonical.com (185.125.190.20): icmp_seq=3 ttl=64 time=121 ms

--- ubuntu.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 3085ms
rtt min/avg/max/mdev = 116.051/119.125/121.084/2.200 ms

```

**But :** `ping -c 3` envoie 3 paquets ICMP vers `ubuntu.com` pour tester la résolution DNS et la connectivité vers Internet (temps de réponse, pertes de paquets).
La résolution DNS fonctionne (le nom a été traduit en `185.125.188.232`), mais les paquets ICMP ne reçoivent pas de réponse. Le NAT de VirtualBox ne transmet pas toujours le trafic ICMP selon la configuration de l'hôte, ce qui n'indique pas une coupure d'Internet. Le test de référence est une vraie connexion TCP/HTTPS.


Résultat de `sudo apt update && sudo apt upgrade -y` : les quatre dépôts (`resolute`, `resolute-security`, `resolute-updates`, `resolute-backports`) ont répondu (`Atteint : 1` à `4`), ce qui confirme que `apt` accède bien aux miroirs Ubuntu. Le résumé final indique : « Tous les paquets sont à jour. Mise à niveau de : 0. Installation de : 0, Supprimé : 0. Non mis à jour : 0 ». `apt` suggère aussi `sudo apt autoremove` pour supprimer d'anciens paquets du noyau 7.0.0-30 devenus inutiles.

**But :** `sudo apt update && sudo apt upgrade -y` : actualiser la liste des paquets depuis les dépôts Ubuntu (`update`), puis installer toutes les mises à jour disponibles (`upgrade`) ; cela confirme aussi que la VM accède bien aux miroirs.

### 5.3 Serveur SSH

```
$ systemctl status ssh --no-pager

 ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-10-14 22:43:16 UTC; 1h 35min ago
Invocation: e5efff52101943eebefed71a4a3e0485
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 1006 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 1039 (sshd)
      Tasks: 1 (limit: 1888)
     Memory: 4.6M (peak: 8.5M)
        CPU: 262ms
     CGroup: /system.slice/ssh.service
             └─1039 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Oct. 14 22:43:15 ubuntu-srv systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Oct. 14 22:43:16 ubuntu-srv sshd[1039]: Server listening on 0.0.0.0 port 22.
Oct. 14 22:43:16 ubuntu-srv systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
Oct. 14 22:43:16 ubuntu-srv sshd[1039]: Server listening on :: port 22.
Oct. 14 22:45:22 ubuntu-srv sshd-session[1413]: Accepted publickey for etudiant from 10.0.2.2 port 58052 ssh2: ED25519 SHA256:VwT...
Oct. 14 22:45:22 ubuntu-srv sshd-session[1413]: pam_unix(sshd:session): session opened for user etudiant(uid=1000) by etudiant(ui
Hint: Some lines were ellipsized, use -l to show in full.

```

**But :** `systemctl status ssh` afficher l'état du service SSH (actif ou non, activé au démarrage, processus, derniers journaux) ; `--no-pager` évite la pagination de la sortie.

Le service SSH est **actif (`active (running)`)** et écoute sur le port 22 en IPv4 (`0.0.0.0`) et en IPv6 (`::`). Il a démarré automatiquement au démarrage de la VM. La mention `disabled` concerne l'unité `ssh.service` elle-même : sur cette version d'Ubuntu, le service est déclenché par `ssh.socket` (`TriggeredBy: ssh.socket`), ce qui explique qu'il démarre malgré tout.

Vérification des ports en écoute :
```
$ sudo ss -tlnp | grep :22
[sudo] password: 
LISTEN 0      4096            0.0.0.0:22            0.0.0.0:*    users:(("sshd",pid=1039,fd=3),("systemd",pid=1,fd=100))
LISTEN 0      4096               [::]:22               [::]:*    users:(("sshd",pid=1039,fd=4),("systemd",pid=1,fd=101))

```

**But :** `sudo ss -tlnp | grep :22` lister les ports TCP en écoute (`-t` TCP, `-l` en écoute, `-n` numérique, `-p` processus) et filtrer le port 22 pour vérifier que `sshd` attend bien les connexions.
Le processus `sshd` (PID 1110) écoute sur le port **22** en IPv4 (`0.0.0.0`) et en IPv6 (`[::]`), joignable depuis toutes les interfaces. Le résolveur DNS local `systemd-resolved` écoute sur `127.0.0.53` et `127.0.0.54`, port 53 (`domain`).

_Note : dans ma première tentative, la commande avait été tapée `ss -tlmp` (option `m` au lieu de `n`). Sans l'option `-n`, `ss` affiche les noms de services (`ssh`, `domain`) au lieu des numéros de port, ce qui explique que `grep :22` n'affichait rien._

### 5.4 Connexion SSH depuis l'hôte et copie de fichier

**Connexion depuis PowerShell sur l'hôte (via la redirection de port) :**
```powershell
PS C:\Users\DELL> ssh -p 2222 etudiant@127.0.0.1
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ED25519 key fingerprint is SHA256:jWcp0VRrgWBzP047xZT5uvDcul6SVO8i/uM9Y5x5XhI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[127.0.0.1]:2222' (ED25519) to the list of known hosts.
etudiant@127.0.0.1's password:
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-38-generic x86_64)
...
IPv4 address for enp0s3: 10.0.2.15
etudiant@unbuntu-srv:~$
```

**But :** `ssh -p 2222 etudiant@127.0.0.1` ouvrir depuis l'hôte une session sécurisée sur la VM, via la redirection de port NAT (`2222` → `22`).
La connexion SSH depuis l'hôte vers la VM a **réussi** : la redirection `127.0.0.1:2222` → port 22 de la VM fonctionne, et le prompt `etudiant@...` confirme que je suis bien sur la VM (le nom d'hôte contient encore une faute de frappe, voir la Partie 4). À la première connexion, SSH a demandé de valider l'empreinte ED25519 du serveur, puis l'a enregistrée dans la liste des hôtes connus (`known_hosts`).

**Copie d'un fichier avec `scp` :**
```powershell

PS C:\Users\DELL> "Bonjour depuis l'hôte" | Out-File -Encoding utf8 bonjour.txt

PS C:\Users\DELL> scp -P 2222 .\bonjour.txt etudiant@127.0.0.1:~
etudiant@127.0.0.1's password:
bonjour.txt        100%   27     8.8KB/s   00:00
```

**But :**
- `"Bonjour depuis l'hôte" | Out-File -Encoding utf8 bonjour.txt` : créer sur l'hôte un petit fichier texte de test.
- `scp -P 2222 .\bonjour.txt etudiant@127.0.0.1:~` : copier ce fichier de l'hôte vers le dossier personnel de la VM, à travers SSH.
Le transfert a **réussi** : le fichier `bonjour.txt` (27 octets) a été copié de l'hôte vers le dossier personnel de `etudiant` sur la VM, à travers la même redirection de port (`-P 2222` en majuscule pour `scp`). Les 27 octets correspondent au texte (22 octets, le « ô » en prenant 2 en UTF-8), à la fin de ligne Windows (2 octets) et à la marque d'ordre des octets (BOM, 3 octets) ajoutée par `Out-File -Encoding utf8`.

Vérification du contenu avec `ssh -p 2222 etudiant@127.0.0.1 "cat ~/bonjour.txt"` : la commande affiche le texte « Bonjour depuis l'hôte » (l'accent peut apparaître modifié selon l'encodage de la console, ce qui n'a pas d'incidence). Le fichier est donc bien arrivé intact sur la VM, et la commande SSH distante fonctionne sans ouvrir de session interactive.

**But :** `ssh -p 2222 etudiant@127.0.0.1 "cat ~/bonjour.txt"` exécuter une commande à distance sur la VM et afficher le contenu du fichier copié, sans ouvrir de session interactive.

_Note : lors de ma première tentative, le fichier avait été créé sous le nom `bonjours.txt` (avec un « s »), d'où l'erreur `No such file or directory` de `scp` qui cherchait `bonjour.txt`._

### 5.5 Bonus — authentification par clé

**Génération de la paire de clés sur l'hôte :**
```powershell
PS C:\Users\DELL> ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (C:\Users\DELL/.ssh/id_ed25519):

Enter passphrase (empty for no passphrase):

Your identification has been saved in C:\Users\DELL/.ssh/id_ed25519
Your public key has been saved in C:\Users\DELL/.ssh/id_ed25519.pub
```

**But :** `ssh-keygen -t ed25519` générer une paire de clés (privée et publique) de type ED25519 pour s'authentifier sans mot de passe.
L'emplacement par défaut a été conservé et la passphrase laissée vide.

**Copie de la clé publique dans la VM :**
```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 etudiant@127.0.0.1 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
etudiant@127.0.0.1's password:
```

**But :** envoyer le contenu de la clé publique dans le fichier `~/.ssh/authorized_keys` de la VM (en créant le dossier `~/.ssh` si besoin), pour autoriser cette clé à se connecter.
Le mot de passe a été demandé une dernière fois pour cette copie.

**Connexion par clé :**
```powershell
PS C:\Users\DELL> ssh -p 2222 etudiant@127.0.0.1
Welcome to Ubuntu 26.04.1 LTS (GNU/Linux 7.0.0-38-generic x86_64)
```

**But :** se reconnecter à la VM pour vérifier que l'authentification par clé fonctionne, sans saisie de mot de passe.
Cette fois, la connexion s'est établie **sans aucune demande de mot de passe** : l'authentification par clé fonctionne. La clé privée reste sur l'hôte (`id_ed25519`), seule la clé publique (`id_ed25519.pub`) a été déposée dans `~/.ssh/authorized_keys` de la VM.

### Problèmes rencontrés et corrections
| Problème | Cause | Correction |
|---|---|---|
| `lsb_release` : commande introuvable | Fautes de frappe, puis paquet non installé | Utilisation de `cat /etc/os-release` (ou installation de `lsb-release`) |
| `curl: (60) SSL: no alternative certificate subject name matches 'unbuntu.com'` | Nom de domaine mal orthographié | Commande à retaper avec `ubuntu.com` |
| `ping` sans réponse | ICMP non relayé par le NAT | Test de la connectivité avec `curl` et `apt` |
| `resolvectl status \| grep "DNS servers"` n'affiche rien | `grep` est sensible à la casse : la sortie contient « DNS Servers » avec un S majuscule | Utilisation de `resolvectl dns` (ou `grep -i dns`) |

### Réponses aux questions

**1. IP, masque, passerelle, DNS ; rôle de `10.0.2.2`**
IP `10.0.2.15`, masque `/24` (`255.255.255.0`), passerelle `10.0.2.2`, DNS `192.168.1.1` (le DNS de mon réseau local, transmis à la VM par le DHCP de VirtualBox). L'adresse `10.0.2.2` correspond au **routeur virtuel du NAT de VirtualBox** : c'est par lui que la VM atteint Internet, et c'est lui qui traduit ses adresses privées vers l'adresse de l'hôte.

**2. Pourquoi se connecter à `127.0.0.1:2222` et non à `10.0.2.15:22`**
L'adresse `10.0.2.15` est une adresse privée interne au réseau NAT de VirtualBox, non routable depuis l'hôte. Seule la redirection de port (`127.0.0.1:2222` → port 22 de la VM) permet à l'hôte d'atteindre le service SSH.

**3. Message affiché à la première connexion SSH (empreinte)**
Le message indique que l'authenticité de l'hôte ne peut pas être établie et affiche l'**empreinte de la clé publique du serveur** (ici ED25519, `SHA256:jWcp0VRrgWBzP047xZT5uvDcul6SVO8i/uM9Y5x5XhI`). Il demande de confirmer que c'est bien le serveur attendu. Une fois acceptée, la clé est enregistrée dans `known_hosts`, et SSH avertira si elle change lors d'une connexion ultérieure. Ce mécanisme protège contre l'**attaque de l'homme du milieu** : un tiers qui se ferait passer pour le serveur présenterait une autre clé, donc une autre empreinte.

**4. Avantages de l'authentification par clé par rapport au mot de passe**
- **Sécurité** : la clé privée ne circule jamais sur le réseau, contrairement à un mot de passe qui peut être deviné, volé ou testé par force brute. Une clé ED25519 est pratiquement impossible à deviner.
- **Confort et automatisation** : plus de mot de passe à saisir, ce qui permet les scripts, les déploiements automatisés et les outils (Git, Ansible, `scp`) sans intervention humaine.
- **Contrôle** : on peut révoquer l'accès d'une machine en supprimant sa clé publique de `authorized_keys`, sans changer un mot de passe partagé.

Les fournisseurs cloud (AWS, Azure) l'imposent par défaut parce que les serveurs sont exposés sur Internet et subissent en permanence des tentatives de connexion par force brute sur les mots de passe. Imposer une clé supprime cette surface d'attaque dès la création de l'instance.

---

## Partie 6 — Instantané, clone et arrêt 

### Problème rencontré avant de commencer
```
$ VBoxManage list vms
"ubuntu-svr" {15f6c946-a136-4034-bb10-debfe52dbea0}
$ VBoxManage snapshot ubuntu-srv take apres-installation ...
VBoxManage.exe: error: Could not find a registered machine named 'ubuntu-srv'
```

**But :**
- `VBoxManage list vms` : lister les VM enregistrées dans VirtualBox (nom et UUID) pour vérifier leur nom exact.
- `VBoxManage snapshot ubuntu-srv take ...` : tenter de prendre un instantané ; il échoue car le nom de la VM est incorrect.
La VM est enregistrée sous le nom **`ubuntu-srvr`** (avec deux « r ») et non `ubuntu-srv`, donc les commandes de l'énoncé échouent. Correction : VM éteinte, puis renommage.

### 6.1 Renommage de la VM et instantané
```powershell
PS C:\Users\DELL> VBoxManage modifyvm ubuntu-srvr --name ubuntu-srv
PS C:\Users\DELL> VBoxManage snapshot ubuntu-srv take apres-installation --description "Ubuntu installé, OpenSSH actif"
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
Snapshot taken. UUID: 126d1152-dd78-474b-863a-748bc261f890
PS C:\Users\DELL> VBoxManage snapshot ubuntu-srv list
   Name: apres-installation (UUID: 126d1152-dd78-474b-863a-748bc261f890) *
   Description: Ubuntu installé, OpenSSH actif
```

**But :**
- `VBoxManage modifyvm ubuntu-srvr --name ubuntu-srv` : renommer la VM (VM éteinte) pour correspondre au nom attendu.
- `VBoxManage snapshot ubuntu-srv take apres-installation --description ...` : prendre un instantané de l'état propre de la VM.
- `VBoxManage snapshot ubuntu-srv list` : lister les instantanés de la VM et repérer l'état courant (`*`).
Le renommage a fonctionné et l'instantané `apres-installation` a été pris sur la VM propre (éteinte). L'étoile `*` indique que c'est l'état courant de la VM. La barre de titre de VirtualBox affiche d'ailleurs `ubuntu-srv (apres-installation)`.

La VM a ensuite été redémarrée :
```powershell
PS C:\Users\DELL> VBoxManage startvm ubuntu-srv --type headless
Waiting for VM "ubuntu-srv" to power on...
VM "ubuntu-srv" has been successfully started.
```

**But :** `VBoxManage startvm ubuntu-srv --type headless` démarrer la VM sans interface graphique (en arrière-plan).

### 6.2 Modification visible avant la restauration
Premier essai : `sudo apt install -y htop` a répondu que **`htop` est déjà installé** (version 3.4.1-5build2) : le paquet fait partie de l'image Ubuntu Server de base,  le fichier `~/avant-restauration.txt` a été créé avec `touch avant-restauration.txt` depuis le dossier personnel et la VM VirtuelBox fut eteinte(sudo poweroff). 

**But :** `sudo apt install -y htop` installer (ici vérifier la présence de) `htop` ; `touch avant-restauration.txt` créer un fichier témoin, pour constater plus tard qu'il disparaît après la restauration ; `sudo poweroff` éteindre proprement la VM, condition nécessaire pour restaurer un instantané.

### Problèmes rencontrés
| Problème | Cause | Correction |
|---|---|---|
| `sudo`, `apt`, `touch` non reconnus dans PowerShell | Commandes Linux tapées sur l'hôte Windows (`PS C:\Users\DELL>`) au lieu de la VM | Les lancer dans la VM (`etudiant@ubuntu-srv`) |
| `Authentication failed, try again` | Mot de passe `sudo` mal saisi au premier essai | Nouvelle saisie |
| `VBoxManage snapshot ubuntu-srv restore` : `Cannot delete the current state of the running machine (machine state: Running)`, puis `startvm` : `already locked by a session` | La VM était encore **allumée** : un instantané ne se restaure que sur une VM éteinte. `which tree; ls ~` montrait d'ailleurs `/usr/bin/tree` et `avant-restauration.txt` toujours présents | Éteindre la VM (`sudo poweroff` ou `acpipowerbutton`), vérifier `VBoxManage list runningvms`, puis restaurer |
| `touch: impossible de toucher '/avant-restauration.txt': Permission denied` | Chemin `/avant-restauration.txt` (racine, interdite à `etudiant`) au lieu de `~/avant-restauration.txt` | Utiliser `touch avant-restauration.txt` depuis le dossier personnel |


## 6.3 — Restauration de l'instantané

```powershell
PS C:\Windows\system32> VBoxManage snapshot ubuntu-srv restore apres-installation
Restoring snapshot 'apres-installation' (126d1152-dd78-474b-863a-748bc261f890)
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%

PS C:\Windows\system32> VBoxManage startvm ubuntu-srv --type headless
Waiting for VM "ubuntu-srv" to power on...
VM "ubuntu-srv" has been successfully started.

PS C:\Windows\system32> ssh -p 2222 etudiant@127.0.0.1 "which htop; ls ~"
kex_exchange_identification: read: Connection reset
Connection reset by 127.0.0.1 port 2222
```
**But :**
- `VBoxManage snapshot ubuntu-srv restore apres-installation` : ramener la VM à l'état de l'instantané (les modifications faites depuis sont perdues).
- `VBoxManage startvm ubuntu-srv --type headless` : redémarrer la VM restaurée, sans interface graphique.
- `ssh ... "which htop; ls ~"` : vérifier à distance que `avant-restauration.txt` a bien disparu. Ici la connexion a été réinitialisée, car SSH n'était pas encore prêt juste après le démarrage.

## 6.4 — Clone `ubuntu-clone` sur le port 2223

```powershell
# VBoxManage controlvm ubuntu-srv acpipowerbutton
(aucun résultat affiché)

# VBoxManage clonevm ubuntu-srv --name ubuntu-clone --register
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%

# VBoxManage modifyvm ubuntu-clone --natpf1 delete ssh
(aucun résultat affiché)

# VBoxManage modifyvm ubuntu-clone --natpf1 "ssh,tcp,127.0.0.1,2223,22"
(aucun résultat affiché)

# VBoxManage startvm ubuntu-clone --type headless
Waiting for VM "ubuntu-clone" to power on...
VM "ubuntu-clone" has been successfully started

# ssh -p 2223 etudiant@127.0.0.1 "sudo hostnamectl set-hostname ubuntu-clone; hostname"

# VBoxManage list runningvms
"ubuntu-clone" {f95251c4-587b-4ce4-a8a1-55a66d279f10}
```
**But :**
- `controlvm ubuntu-srv acpipowerbutton` : demander à la VM source un arrêt propre avant de la cloner.
- `clonevm ubuntu-srv --name ubuntu-clone --register` : créer une copie indépendante de la VM et l'enregistrer dans VirtualBox.
- `modifyvm ubuntu-clone --natpf1 delete ssh` : supprimer la règle de redirection SSH héritée du clone (port 2222 déjà utilisé par la VM d'origine).
- `modifyvm ubuntu-clone --natpf1 "ssh,tcp,127.0.0.1,2223,22"` : créer une nouvelle règle SSH sur le port hôte 2223, pour éviter le conflit.
- `startvm ubuntu-clone --type headless` : démarrer le clone sans interface graphique.
- `ssh -p 2223 ... hostnamectl set-hostname ubuntu-clone; hostname` : changer le nom d'hôte du clone pour le distinguer de la VM d'origine, puis l'afficher.
- `list runningvms` : vérifier quelles VM sont en cours d'exécution.

## 6.5 — Comparaison de l'espace occupé et suppression du clone

```powershell
PS C:\Program Files\Oracle\VirtualBox> Get-ChildItem "$HOME\VirtualBox VMs" -Directory | ForEach-Object { "{0} : {1:N2} Go" -f $_.Name, ((Get-ChildItem $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum / 1GB) }
ubuntu_svr : 3,58 Go
ubuntu-clone : 12,63 Go
ubuntu-svr : 0,00 Go
VUbuntu : 15,30 Go
```
**But :** calculer et afficher, en Go, l'espace disque occupé par chaque dossier de VM, pour comparer la taille de la VM d'origine et celle du clone.

```powershell
PS C:\Program Files\Oracle\VirtualBox> .\VBoxManage controlvm ubuntu-clone acpipowerbutton

PS C:\Program Files\Oracle\VirtualBox> .\VBoxManage unregistervm ubuntu-clone --delete
0%...10%...20%...30%...40%...50%...60%...70%...80%...90%...100%
```
**But :** éteindre proprement le clone (`acpipowerbutton`), puis le retirer de VirtualBox et supprimer ses fichiers du disque (`unregistervm --delete`).

## réponses aux questions

**1.différence entre un instantané et un clone **
Le snapshot fige l'état d'une VM (disque différentiel) pour revenir en arrière. Le clone est une nouvelle VM indépendante. Un snapshot n'est *pas une sauvegarde*, car il dépend du disque parent et reste sur le même hôte.

**2.les équivalents cloud de l'instantané et du clone** 
snapshot de disque (EBS, etc.) pour l'instantané, image (AMI, etc.) pour le clone.

**3.différence entre acpipowerbutton et poweroff dans VBoxManage controlvm**
le premier demande un arrêt propre à l'OS invité, le second coupe brutalement, avec un risque de corruption.
**4.Pourquoi démarrer une VM en mode headless ? Quel lien avec une instance cloud ?**
VM sans interface graphique, pilotée par SSH ou script, comme une instance cloud.

**5.•	Pourquoi faut-il changer le port hôte et le nom d'hôte du clone ?**
deux VM ne peuvent pas partager le même port hôte (conflit), et un hostname identique crée des confusions d'identité sur le réseau.

---

## Bilan 

| Partie | État |
|---|---|
| 1 — VirtualBox installé, notions d'hyperviseur | Terminée |
| 2 — ISO téléchargée et empreinte vérifiée | Terminée |
| 3 — VM créée et configurée | Terminée |
| 4 — Ubuntu Server installé | Terminée  |
| 5 — Réseau et SSH, bonus clé SSH | Terminée |
| 6 — Instantané et clone | Terminée |
| Comparaison Multipass et VirtualBox | Terminée |
