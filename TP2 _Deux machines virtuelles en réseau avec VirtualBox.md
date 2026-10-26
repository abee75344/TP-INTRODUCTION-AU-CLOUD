# COMPTE RENDU TP2 — Deux machines virtuelles en réseau avec VirtualBox

**Cours :** Introduction au Cloud Computing · L3 SI RIST & BDGL
**Date :** 18 octobre 2026
**Étudiants :** ABE YEPI CHRIST-EMMANUEL · AKELY MAYLINE PHOEBE EDWINA · DANHO AKRE PRINCE PATRICK-JUNIOR
**Environnement :** Windows, VirtualBox (commandes `VBoxManage` dans PowerShell), Ubuntu Server 26.04.1 LTS


**Objectif :** cloner une image Ubuntu en deux VM (`web` et `client`), les relier dans un réseau NAT privé `tp-cloud`, y publier un site avec Nginx, le protéger avec `ufw`, mesurer les ressources, puis nettoyer.

| Machine | Adresse IP | Rôle |
|---|---|---|
| `web` | 10.10.0.10/24 | Serveur Nginx, pare-feu `ufw` |
| `client` | 10.10.0.20/24 | Poste de test |
| Routeur NAT | 10.10.0.1 | Passerelle, DNS, redirections de port |

Image de départ : VM `ubuntu-srv` (Ubuntu Server, utilisateur `etudiant`, OpenSSH). Réseau cible : NAT VirtualBox `tp-cloud`, sous-réseau `10.10.0.0/24`, sans DHCP.

---

# PARTIE 1 — Créer les VM par clonage, les personnaliser et s'y connecter en SSH

### Étape 1.1 — Cloner l'image en VM `web`

```powershell
VBoxManage controlvm ubuntu-srv acpipowerbutton
VBoxManage list runningvms
Measure-Command { VBoxManage clonevm ubuntu-srv --name web --register } | Select-Object TotalSeconds
```

*Sortie :*

```text
TotalSeconds
------------
  35,1559528
```

*But : éteindre proprement la VM source (un clone ne peut pas être créé si la source est verrouillée), vérifier qu'elle ne tourne plus, puis la cloner en `web` (`--register` l'enregistre dans VirtualBox) en chronométrant l'opération.*

### Étape 1.2 — Cloner l'image en VM `client`

```powershell
VBoxManage clonevm ubuntu-srv --name client --register
```

*Durée : ** 45,128***

*But : créer la seconde VM de la même façon.*

### Étape 1.3 — Régler les ressources des clones

```powershell
VBoxManage modifyvm web --cpus 4 --memory 5048
VBoxManage modifyvm client --cpus 4 --memory 5048
```

*But : fixer le nombre de vCPU et la RAM (en Mo) de chaque VM. Nous avons choisi 4 vCPU et 5048 Mo, au lieu des 1 vCPU et 1 Go proposés par le sujet.*

### Étape 1.4 — Créer le réseau NAT `tp-cloud`

```powershell
VBoxManage natnetwork add --netname tp-cloud --network "10.10.0.0/24" --enable --dhcp off
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-web:tcp:[127.0.0.1]:2222:[10.10.0.10]:22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-client:tcp:[127.0.0.1]:2223:[10.10.0.20]:22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "http-web:tcp:[127.0.0.1]:8080:[10.10.0.10]:80"
VBoxManage natnetwork list
```

*Sortie (`natnetwork list`) :*

```text
Name:         tp-cloud
Enabled:      Yes
Network:      10.10.0.0/24
Gateway:      10.10.0.1
DHCP Server:  No
IPv6:         No
IPv6 Prefix:  fd17:625c:f037:2::/64
IPv6 Default: No
Port-forwarding (ipv4)
        http-web:tcp:[127.0.0.1]:8080:[10.10.0.10]:80
        ssh-client:tcp:[127.0.0.1]:2223:[10.10.0.20]:22
        ssh-web:tcp:[127.0.0.1]:2222:[10.10.0.10]:22
loopback mappings (ipv4)
        127.0.0.1=2
```


*But : créer un réseau privé sans DHCP (adresses fixes), puis ouvrir trois redirections de port depuis l'hôte : 2222 → SSH de `web`, 2223 → SSH de `client`, 8080 → HTTP de `web`. `natnetwork list` vérifie la configuration.*

### Étape 1.5 — Brancher les cartes réseau

```powershell
VBoxManage modifyvm web    --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage modifyvm client --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage showvminfo web | Select-String "Memory size|Number of CPUs|NIC 1"
```

*Sortie :*

```text
Number of CPUs:  4
NIC 1:           MAC: 080027E9B66F, Attachment: NAT Network 'tp-cloud',
                 Cable connected: on, Trace: off (file: none), Type: 82540EM,
                 Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny,
                 Bandwidth group: none
```

*But : connecter la carte réseau 1 de chaque VM au réseau `tp-cloud` et vérifier les réglages de `web`. La ligne `Memory size` n'apparaît pas dans notre relevé 

### Étape 1.6 — Personnaliser `web` (nom, identifiant de machine, clés SSH)

```bash
sudo hostnamectl set-hostname web
sudo sed -i 's/ubuntu-srv/web/g' /etc/hosts
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A
```

*Sortie :*

```text
$ sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
Initializing machine ID from random generator.
$ sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A
ssh-keygen: generating new host keys: RSA ECDSA ED25519
```

*But : un clone est une copie exacte de l'original. On change donc son nom d'hôte, son identifiant de machine et ses clés SSH d'hôte (copiés avec l'image). Le prompt affiche encore `ubuntu-srv` : le nouveau nom n'apparaît qu'à la prochaine session.*

### Étape 1.7 — Donner une adresse fixe à `web`

Fichier `/etc/netplan/99-tp.yaml` :

```bash
sudo tee /etc/netplan/99-tp.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.10.0.10/24]
      routes:
        - to: default
          via: 10.10.0.1
      nameservers:
        addresses: [10.10.0.1, 1.1.1.1]
EOF
sudo chmod 600 /etc/netplan/99-tp.yaml
sudo netplan apply
```

*Sortie (première tentative) :*

```text
$ sudo netplan apply
Cannot find unique matching interface for enp0s3
```

*Diagnostic et correction :*

```bash
ip -br link
ls /etc/netplan/
sudo cat /etc/netplan/00-installer-config.yaml
sudo mv /etc/netplan/00-installer-config.yaml /etc/netplan/00-installer-config.yaml.bak
sudo netplan apply
ip -4 addr show enp0s3
```

```text
$ ip -br link
lo       UNKNOWN   00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp0s3   UP        08:00:27:e9:b6:6f <BROADCAST,MULTICAST,UP,LOWER_UP>
$ ls /etc/netplan/
00-installer-config.yaml  99-tp.yaml
$ ip -4 addr show enp0s3
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    altname enx080027e9b66f
    inet 10.10.0.10/24 brd 10.10.0.255 scope global enp0s3
       valid_lft forever preferred_lft forever
```

*But : définir une adresse IPv4 fixe (le réseau n'a pas de DHCP), la route par défaut vers le routeur NAT et les serveurs DNS. L'interface `enp0s3` existait bien (`ip -br link`), mais le fichier de l'installateur `00-installer-config.yaml` la liait à l'adresse MAC de la VM d'origine (`08:00:27:cb:20:ac`). VirtualBox attribue une nouvelle MAC à chaque clone (`08:00:27:e9:b6:6f` pour `web`) : la condition ne correspondait plus à aucune carte. On désactive donc ce fichier en le renommant en `.bak`, car netplan ne lit que les fichiers `.yaml`.*

### Étape 1.8 — Personnaliser `client`

Les étapes 1.6 et 1.7 ont été refaites sur `client` (nom `client`, adresse `10.10.0.20/24`), puis vérifiées :

```bash
ip -4 addr show enp0s3
ip route
ping -c 2 10.10.0.1
ls /etc/ssh/ssh_host_*
```

*Sortie :*

```text
$ ip -4 addr show enp0s3
    inet 10.10.0.20/24 brd 10.10.0.255 scope global enp0s3
$ ip route
default via 10.10.0.1 dev enp0s3 proto static
10.10.0.0/24 dev enp0s3 proto kernel scope link src 10.10.0.20
$ ping -c 2 10.10.0.1
2 packets transmitted, 2 received, 0% packet loss, time 3490ms
rtt min/avg/max/mdev = 0.549/0.906/1.264/0.357 ms
$ ls /etc/ssh/ssh_host_*
/etc/ssh/ssh_host_ecdsa_key    /etc/ssh/ssh_host_ed25519_key    /etc/ssh/ssh_host_rsa_key
/etc/ssh/ssh_host_ecdsa_key.pub  /etc/ssh/ssh_host_ed25519_key.pub  /etc/ssh/ssh_host_rsa_key.pub
```

*But : vérifier l'adresse, la route par défaut, l'accès à la passerelle (0 % de perte) et la présence des six fichiers de clés SSH (3 privées, 3 publiques). Le nom d'hôte de `client` n'avait en réalité pas été changé : l'erreur n'a été vue qu'à l'étape suivante (voir étape 1.9 et tableau des difficultés).*

### Étape 1.9 — Se connecter en SSH depuis l'hôte

```powershell
ssh-keygen -R "[127.0.0.1]:2222"
ssh-keygen -R "[127.0.0.1]:2223"
ssh -p 2222 etudiant@127.0.0.1 "hostnamectl; lsb_release -a; uname -r"
ssh -p 2223 etudiant@127.0.0.1 "hostname; ip -4 -br addr"
```

*Sortie (`web`, port 2222) :*

```text
Static hostname: web
      Icon name: computer-vm
        Chassis: vm
     Machine ID: 55211e66f8f740e3a4f93c914c9079b2
 Virtualization: oracle
Operating System: Ubuntu 26.04.1 LTS
         Kernel: Linux 7.0.0-38-generic
   Architecture: x86-64
Hardware Vendor: innotek GmbH
 Hardware Model: VirtualBox
Distributor ID: Ubuntu
Description:    Ubuntu 26.04.1 LTS
Release:        26.04
Codename:       resolute
7.0.0-38-generic
```

*Sortie (`client`, port 2223, première tentative) :*

```text
ubuntu-srv
lo      UNKNOWN   127.0.0.1/8
enp0s3  UP        10.10.0.20/24
```

*Correction du nom de `client` :*

```bash
sudo hostnamectl set-hostname client
sudo sed -i 's/ubuntu-srv/client/g' /etc/hosts
hostnamectl
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
```

```text
$ hostnamectl
 Static hostname: client
      Machine ID: 65a5dfe8bfb845878a7dc58cb829b557
$ sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
Initializing machine ID from random generator.
```


*But : `ssh-keygen -R` efface les anciennes empreintes enregistrées pour ces ports (celles de `ubuntu-srv`), ce qui évite l'alerte `REMOTE HOST IDENTIFICATION HAS CHANGED`. Les deux connexions passent par les redirections de port de l'étape 1.4. Le nom de `client` a ensuite été corrigé et son identifiant de machine régénéré, pour garantir qu'il diffère de celui de `web`.*

### Réponses aux questions de la partie 1

1. **Combien de temps a pris le clonage de `web` ? Comparaison avec l'installation d'Ubuntu et un serveur physique ?** **35,16 secondes**. L'installation d'Ubuntu au TP précédent a duré environ **[À COMPLÉTER : durée de votre installation]**. Un serveur physique demande en plus une commande, une livraison, un montage en baie, un câblage puis l'installation : de plusieurs jours à plusieurs semaines. Le clonage d'une image est de loin la méthode la plus rapide, et c'est le principe du lancement d'une instance dans le cloud.
2. **Pourquoi changer le nom d'hôte, l'identifiant de machine et les clés SSH d'un clone ? Quel risque avec les mêmes clés sur 50 serveurs ?**
   - Le **nom d'hôte** identifie la machine sur le réseau et dans les journaux : deux machines au même nom prêtent à confusion.
   - L'**identifiant de machine** (`/etc/machine-id`) identifie la machine de façon unique (journaux systemd, identifiant DHCP, outils de gestion) : des doublons provoquent des conflits.
   - Les **clés SSH d'hôte** prouvent l'identité du serveur au client. Si 50 serveurs partagent la même clé privée, la compromission d'un seul permet d'usurper les 49 autres (attaque de l'homme du milieu), et un client ne peut plus les distinguer.
   - C'est le rôle de **cloud-init** dans les instances cloud réelles.
3. **Version d'Ubuntu, noyau, vCPU et RAM de `web` ? Quel service cloud joue le rôle de `ubuntu-srv` ?** Ubuntu **26.04.1 LTS** (« resolute »), noyau **Linux 7.0.0-38-generic**, **4 vCPU**, RAM **5048 Mo** (valeur configurée avec `modifyvm`, **[À VÉRIFIER]** avec `VBoxManage showvminfo web | Select-String "Memory size"`). `ubuntu-srv` joue le rôle d'une **image de machine** : une **AMI** sur AWS EC2, une image de la galerie de calcul sur Azure, une image de disque sur Google Compute Engine.

---

# PARTIE 2 — Relier les VM en réseau

### Étape 2.1 — Tester la connexion de `client` vers `web`

```bash
ping -c 4 10.10.0.10
```

*Résultat :* 4 paquets reçus sur 4, 0 % de perte, min/moy/max = 1,467 / 5,897 / 14,513 ms.

*But : vérifier que `web` répond et mesurer le délai.*

### Étape 2.2 — Tester la passerelle et Internet

```bash
ping -c 2 10.10.0.1
ping -c 2 1.1.1.1
```

*Résultat :* passerelle `10.10.0.1` : 2 reçus sur 2, environ 1 ms ; Internet (`1.1.1.1`) : 2 reçus sur 2, environ 12 ms.

*But : vérifier que `client` joint le routeur NAT, puis l'extérieur à travers lui.*

### Étape 2.3 — Voir la route par défaut et les serveurs DNS

```bash
ip route
resolvectl status | grep "DNS Servers"
```

*Résultat :* `default via 10.10.0.1 dev enp0s3` et `10.10.0.0/24 dev enp0s3 ... src 10.10.0.20` ; serveurs DNS `10.10.0.1 1.1.1.1`.

*But : afficher le réseau local, la passerelle par défaut et les serveurs DNS de la VM.*

### Étape 2.4 — Donner un nom à `web`

```bash
echo "10.10.0.10 web" | sudo tee -a /etc/hosts
```

*But : associer le nom `web` à son adresse dans `/etc/hosts` (`tee -a` ajoute sans écraser).*

### Étape 2.5 — Tester par le nom

```bash
ping -c 2 web
```

*Résultat :* 2 reçus sur 2, environ 2,3 ms de moyenne.

*But : vérifier que le nom `web` est résolu et que la machine répond.*

### Étape 2.6 — Pinger `web` depuis l'hôte Windows

```powershell
ping -n 2 10.10.0.10
```

*Sortie :*

```text
PS> ping -n 2 10.10.0.10
Envoi d'une requête 'Ping'  10.10.0.10 avec 32 octets de données :
Délai d'attente de la demande dépassé.
Délai d'attente de la demande dépassé.
Statistiques Ping pour 10.10.0.10:
    Paquets : envoyés = 2, reçus = 0, perdus = 2 (perte 100%)
```


*But : montrer que l'hôte n'a pas d'accès direct au réseau privé `tp-cloud`.*

### Réponses aux questions de la partie 2

1. **Même sous-réseau ?** Oui. `web` a l'adresse `10.10.0.10/24` et `client` `10.10.0.20/24`. Le masque `/24` (255.255.255.0) fixe les 24 premiers bits comme partie réseau : les deux adresses appartiennent à `10.10.0.0/24` (hôtes de `10.10.0.1` à `10.10.0.254`). Elles communiquent directement, sans routeur entre elles.
2. **Passerelle par défaut de `client`, son rôle, son équivalent dans un VPC / VNet ?** **`10.10.0.1`** (`default via 10.10.0.1`). Elle reçoit tout le trafic destiné à des adresses hors du sous-réseau, ici vers Internet, et assure le NAT et le DNS. Dans un VPC AWS, c'est le routeur du VPC avec sa route par défaut vers une Internet Gateway ou une NAT Gateway ; dans un VNet Azure, la passerelle système du sous-réseau et sa route par défaut vers Internet.
3. **Temps de ping moyen de `client` vers `web` ? Pourquoi si faible ?** **5,9 ms** (min 1,47 ms, max 14,5 ms). Le premier paquet est le plus lent, probablement parce qu'il inclut la résolution ARP de `10.10.0.10` ; les suivants sont de l'ordre de 1 à 5 ms. Les deux VM tournent sur le même PC : les paquets ne traversent ni câble ni routeur physique, ils sont transmis en mémoire par VirtualBox.
4. **Le ping depuis Windows aboutit-il ? Comment l'hôte joint-il `web` en SSH ?** Non : 100 % de perte. `tp-cloud` est un réseau privé géré par VirtualBox : Windows n'a ni interface ni route vers `10.10.0.0/24`, et le NAT n'accepte aucun trafic entrant sans règle explicite. C'est l'équivalent d'un sous-réseau privé du cloud, inaccessible depuis Internet. L'hôte atteint pourtant `web` en SSH grâce à la **redirection de port** `ssh-web:tcp:[127.0.0.1]:2222:[10.10.0.10]:22` : en se connectant à `127.0.0.1:2222`, Windows envoie le trafic TCP à VirtualBox, qui le transfère vers `10.10.0.10:22`. Seul le trafic TCP explicitement configuré est redirigé ; le ping (ICMP) n'est pas concerné. C'est le principe d'une règle de groupe de sécurité ou d'un load balancer qui expose un seul port d'une instance privée.

---

# PARTIE 3 — Publier une page web et la protéger

### Étape 3.1 — Installer Nginx sur `web`

```powershell
ssh -p 2222 etudiant@127.0.0.1
```

```bash
sudo apt update && sudo apt install -y nginx
systemctl status nginx --no-pager
```

*Sortie (extrait) :*

```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-18 19:08:54 UTC
   Main PID: 2211 (nginx)
      Tasks: 5 (limit: 2734)
```


*But : installer Nginx et vérifier qu'il tourne. `enabled` signifie qu'il redémarre avec la VM ; le processus maître est accompagné de 4 processus `worker`, un par vCPU.*

### Étape 3.2 — Créer notre page d'accueil

```bash
echo "<h1>Binome PATRICK - YEPI - MAYLINE : notre premier VM en reseau</h1>" | sudo tee /var/www/html/index.html
```

*But : remplacer la page d'accueil de Nginx par notre texte (`sudo tee` pour écrire dans `/var/www/html`).*

### Étape 3.3 — Afficher la page depuis `client` et depuis l'hôte

```powershell
ssh -p 2223 etudiant@127.0.0.1 "curl -s http://web"
```

*Sortie :*

```text
<h1>Binome PATRICK - YEPI - MAYLINE : notre premier VM en reseau</h1>
```


*But : `client` joint `web` par son nom sur le port 80 (`-s` : sans barre de progression). Le navigateur de l'hôte affiche la même page sur `http://127.0.0.1:8080`, grâce à la redirection de port `http-web`.*

### Étape 3.4 — Activer le pare-feu (SSH seul autorisé)

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

*Sortie :*

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
```

*But : autoriser SSH **avant** d'activer le pare-feu pour ne pas perdre l'accès à la VM. Le trafic entrant non autorisé est ensuite refusé par défaut (`deny (incoming)`).*

### Étape 3.5 — Tester avec le port 80 fermé

```powershell
ssh -p 2223 etudiant@127.0.0.1 "curl -m 5 http://web; echo code retour: `$?"
```

*Sortie :*

```text
curl: (28) Connection timed out after 5012 milliseconds
code retour: 28
```


*But : constater que le pare-feu bloque le trafic web alors que Nginx tourne toujours (`-m 5` : abandon après 5 secondes ; `$?` : code retour de curl, protégé par un accent grave dans PowerShell).*

### Étape 3.6 — Ouvrir le port 80

```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw status numbered
```

*Sortie :*

```text
$ sudo ufw allow 'Nginx HTTP'
Rule added
Rule added (v6)
$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] OpenSSH                    ALLOW IN    Anywhere
[ 2] Nginx HTTP                 ALLOW IN    Anywhere
[ 3] OpenSSH (v6)               ALLOW IN    Anywhere (v6)
[ 4] Nginx HTTP (v6)            ALLOW IN    Anywhere (v6)
```

*But : autoriser le trafic web (port 80) à traverser le pare-feu et afficher les règles avec leur numéro.*

### Étape 3.7 — Retester l'accès web

```powershell
ssh -p 2223 etudiant@127.0.0.1 "curl -s http://web"
```

*Sortie :*

```text
<h1>Binome PATRICK - YEPI - MAYLINE : notre premier VM en reseau</h1>
```

*But : vérifier que la page est de nouveau servie.*

### Étape 3.8 — Voir les ports en écoute

```bash
sudo ss -tlnp
```

*Sortie :*

```text
State   Recv-Q  Send-Q  Local Address:Port   Process
LISTEN  0       4096    127.0.0.54:53        users:(("systemd-resolve",pid=626,fd=19))
LISTEN  0       4096    0.0.0.0:22           users:(("sshd",pid=1521,fd=3),("systemd",pid=1,fd=105))
LISTEN  0       511     0.0.0.0:80           users:(("nginx",pid=2216,fd=5), ... )
LISTEN  0       4096    127.0.0.53%lo:53     users:(("systemd-resolve",pid=626,fd=17))
LISTEN  0       4096    [::]:22              users:(("sshd",pid=1521,fd=4),("systemd",pid=1,fd=106))
LISTEN  0       511     [::]:80              users:(("nginx",pid=2216,fd=6), ... )
```


*But : lister les ports TCP en écoute (`-t` TCP, `-l` en écoute, `-n` numérique, `-p` processus) et le programme associé.*

### Réponses aux questions de la partie 3

1. **Que renvoie `curl` quand le port 80 est fermé ? Pourquoi le navigateur reste-t-il bloqué ?** Le message **`curl: (28) Connection timed out after 5012 milliseconds`** et le code retour **28** (délai dépassé, interrompu par `-m 5`). Avec la politique `deny (incoming)`, `ufw` **ignore silencieusement** les paquets : la demande de connexion (SYN) ne reçoit ni refus ni message d'erreur. Le client ne sait pas si le serveur est lent, absent ou filtré : il retransmet et attend jusqu'au délai maximal, d'où le navigateur qui reste bloqué. Un port fermé sans pare-feu répond au contraire immédiatement `Connection refused`. Un groupe de sécurité AWS ou un NSG Azure se comporte de la même façon : le trafic non autorisé est ignoré sans réponse.
2. **Sur quel port et quelles adresses Nginx écoute-t-il ?** Sur le **port 80**, sur **`0.0.0.0:80`** (toutes les adresses IPv4 de la machine, notamment `10.10.0.10` et `127.0.0.1`) et sur **`[::]:80`** (toutes les adresses IPv6). Les autres services en écoute sont `sshd` sur le port 22 et `systemd-resolved` sur le port 53, en local seulement. Nginx écoute sur toutes les interfaces : c'est donc le pare-feu, et non Nginx, qui décide qui peut l'atteindre.
3. **`ufw` et groupe de sécurité du cloud : deux ressemblances et une différence ?**
   - *Ressemblance 1 :* les deux fonctionnent par **règles d'autorisation** sur des ports et protocoles : tout le trafic entrant est refusé par défaut, on ouvre explicitement le nécessaire (SSH, HTTP).
   - *Ressemblance 2 :* le trafic non autorisé est **ignoré sans réponse** (délai dépassé, `curl` code 28), et les règles se modifient à chaud, sans redémarrer la machine ni le service.
   - *Différence :* `ufw` s'exécute **à l'intérieur de la VM**, géré par l'utilisateur de la VM, et peut être modifié ou désactivé par quelqu'un qui en prend le contrôle. Un groupe de sécurité s'applique **à l'extérieur de la VM**, au niveau du réseau virtuel du fournisseur, se gère par la console ou l'API, protège l'instance même si l'OS invité est compromis, et peut s'appliquer à plusieurs instances.
4. **Lien avec le cours : tableau « Qui gère quoi ? »** Dans un IaaS, le fournisseur gère le matériel et la virtualisation, le client gère tout ce qui est au-dessus.

| Couche | Qui s'en est chargé dans ce TP |
|---|---|
| Matériel physique (serveur, stockage, réseau physique) | Le PC de l'hôte (notre propre machine) |
| Virtualisation (hyperviseur) | **VirtualBox** |
| Matériel virtuel (vCPU, RAM, disque, carte réseau) | **VirtualBox**, à partir de nos réglages (`modifyvm`) |
| Réseau virtuel (réseau NAT, passerelle, redirections de port) | **VirtualBox** (`natnetwork`), configuré par nous |
| Système d'exploitation | **Nous** : Ubuntu dans `ubuntu-srv`, nom d'hôte, identifiant de machine, clés SSH, adresse IP (`netplan`) |
| Sécurité du système | **Nous** : pare-feu `ufw`, règles ouvertes, accès SSH |
| Middleware et application | **Nous** : installation de Nginx et publication de la page |
| Données | **Nous** : contenu de `index.html` |

VirtualBox a fourni la machine virtuelle et son réseau ; tout ce qui est à partir du système d'exploitation est resté à notre charge, comme avec une instance EC2 ou une VM Azure.

---

# PARTIE 4 — Mesurer et nettoyer

### Étape 4.1 — Mesurer les ressources vues depuis `web`

```powershell
ssh -p 2222 etudiant@127.0.0.1 "free -h; df -h /; nproc"
```

*Sortie :*

```text
              total        used        free      shared  buff/cache   available
Mem:           4,2Gi       479Mi       3,3Gi       1,1Mi       732Mi       3,8Gi
Swap:          2,0Gi          0B       2,0Gi
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   12G  5,2G  5,5G  49% /
4
```


*But : voir la mémoire, le disque racine et le nombre de processeurs de la VM : 4 processeurs, 4,2 Gio de RAM, 2 Gio de swap, 12 Go de système de fichiers dont 5,2 Go utilisés.*

### Étape 4.2 — Mesurer côté hôte

Dans le Gestionnaire des tâches (onglet Détails), six processus `VirtualBoxVM.exe` tournent ; deux consomment l'essentiel de la mémoire :

| Processus | Mémoire |
|---|---|
| `VirtualBoxVM.exe` (PID 10564), VM `web` | 1 875 148 Ko, soit environ 1,79 Gio |
| `VirtualBoxVM.exe` (PID 12300), VM `client` | 1 110 284 Ko, soit environ 1,06 Gio |
| Quatre autres processus `VirtualBoxVM.exe` | de 1 084 à 1 204 Ko chacun |


*But : comparer la mémoire configurée, celle vue par la VM et celle réellement consommée sur l'hôte.*

### Étape 4.3 — Mesurer la place des disques

```powershell
VBoxManage list hdds
```

*Résultat :* taille réelle du fichier `.vdi` de `ubuntu-srv` (même type que les disques des clones) : **6,55 Go**. La commande du TP sur `$HOME\VirtualBox VMs` renvoie `ubuntu-srv : 0,00 Go` : les disques virtuels ne se trouvent pas dans ce dossier.

*But : comparer la capacité annoncée du disque virtuel (25 Go) à la place occupée sur l'hôte. Le `.vdi` d'un clone n'a pas été mesuré avant suppression : **[À COMPLÉTER]** si possible.*

### Étape 4.4 — Arrêter, redémarrer et supprimer

```powershell
VBoxManage controlvm web acpipowerbutton
VBoxManage controlvm client acpipowerbutton
VBoxManage list runningvms
VBoxManage startvm web --type headless
VBoxManage controlvm web poweroff
VBoxManage unregistervm web --delete
VBoxManage unregistervm client --delete
VBoxManage natnetwork remove --netname tp-cloud
VBoxManage list vms
```

*Sortie :*

```text
Waiting for VM "web" to power on...
VM "web" has been successfully started.
"ubuntu-srv" {15f6c946-a136-4034-bb10-debfe52dbea0}
"ubuntu-srv2" {deff2e04-b1cf-4010-81b3-e875a5b690c1}
```


*But : éteindre proprement les deux VM, redémarrer `web` sans fenêtre (mode headless, comme une instance cloud accessible seulement en SSH), la couper brutalement, puis supprimer les deux VM et le réseau. `ubuntu-srv`, image de base des prochains TP, est conservée ; `ubuntu-srv2` n'a pas été touchée.*

### Réponses aux questions de la partie 4

1. **La VM `web` voit-elle 1 Go de RAM ? Combien en consomme-t-elle sur l'hôte ? Comment expliquer l'écart ?** Non : la RAM de `web` a été réglée à 5048 Mo (pas 1 Go) et `free -h` affiche **4,2 Gio**. L'écart avec la valeur configurée correspond à de la mémoire réservée par le noyau et le firmware, que `free` ne compte pas. Sur l'hôte, les processus des VM utilisent environ **1,79 Gio** pour `web` et **1,06 Gio** pour `client`, bien moins que la mémoire configurée : VirtualBox n'alloue la RAM de l'hôte qu'à la demande, une page n'occupant de place réelle que lorsque l'invité l'a utilisée. La VM utilise 479 Mio et son cache 732 Mio, soit environ 1,2 Gio ; le reste de l'écart vient des pages utilisées puis libérées par l'invité qui restent allouées côté hôte, et de la mémoire consommée par VirtualBox lui-même (émulation des périphériques, mémoire vidéo). Ce principe de sur-allocation permet à un fournisseur de faire tourner plus de VM que de RAM physique. `web` consomme plus que `client` car il a fait plus de choses (installation de paquets, Nginx).
2. **Le disque d'un clone fait 25 Go : quelle place occupe-t-il sur l'hôte ? Pourquoi ?** Le `.vdi` occupe **6,55 Go**, soit environ un quart des 25 Go annoncés. Le disque est **alloué dynamiquement** : le fichier ne grossit qu'à mesure que la VM y écrit. La taille réelle correspond donc à ce qui a été écrit (ici 5,2 Go utilisés sur `/`, plus d'autres partitions et des blocs écrits puis libérés), et non à la capacité annoncée. Le fournisseur peut ainsi promettre plus de stockage qu'il n'en occupe réellement.
3. **Différence entre `acpipowerbutton`, `poweroff` et `unregistervm --delete` ?**
   - **`acpipowerbutton`** simule l'appui sur le bouton d'alimentation : l'invité reçoit un signal ACPI et s'éteint **proprement** (arrêt des services, écriture des données).
   - **`poweroff`** coupe la VM **brutalement**, comme si on débranchait la prise : risque de fichiers corrompus.
   - **`unregistervm --delete`** **supprime** la VM de VirtualBox et ses fichiers (configuration et disque), de façon irréversible. Sans `--delete`, elle est seulement retirée de la liste. Dans le cloud, c'est l'équivalent de « terminer » l'instance.
4. **Dans le cloud, une VM arrêtée est-elle encore facturée ?** Le **calcul** (vCPU, RAM) n'est généralement plus facturé, mais le **disque** attaché (volume EBS sur AWS, disque managé sur Azure) et ses sauvegardes le restent, ainsi que l'**adresse IP publique** réservée tant qu'elle existe (AWS facture les IPv4 publiques, utilisées ou non). Le détail dépend du fournisseur : sur Azure, il faut que la VM soit « désallouée » pour ne plus payer le calcul. Pour ne plus rien payer, il faut **supprimer** l'instance et ses ressources associées (tarifs à vérifier sur la grille du fournisseur).

---

## Principales difficultés

| Étape | Problème | Cause | Solution |
|---|---|---|---|
| 1.4 | `NATNetwork server does not exist` | Les commandes `natnetwork modify` ont été lancées avant la création de `tp-cloud` | Créer d'abord le réseau avec `natnetwork add`, puis lancer les `modify` |
| 1.6 | `rm: cannot remove 'f ...': No such file or directory`, puis prompt `>` | Apostrophe `'f` à la place de `-f`, puis `-f` oublié (`rm f`) | `Ctrl+C`, puis retaper : `sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup` |
| 1.6 | `Could not save your private key ... Permission denied` | `sudo` oublié devant `ssh-keygen -A` : il ne s'applique qu'à la commande qui le suit, et `etudiant` ne peut pas écrire dans `/etc/ssh` | `sudo ssh-keygen -A`, puis `ls /etc/ssh/ssh_host_*`. Sans ces clés, SSH ne fonctionnerait pas |
| 1.7 | `Cannot find unique matching interface for enp0s3` | Le fichier `00-installer-config.yaml` liait `enp0s3` à la MAC de la VM d'origine, différente de celle du clone | Renommer ce fichier en `.bak`, puis `sudo netplan apply` |
| 1.9 | `hostname` renvoie `ubuntu-srv` sur `client` | Le nom d'hôte n'avait pas été changé (commande non exécutée ou mal saisie) | `sudo hostnamectl set-hostname client`, vérification avec `hostnamectl`, nouvel identifiant de machine. Le `grep client /etc/hosts` n'affiche aucune ligne : le `sed` n'avait rien à remplacer |
| 1.9 | `Connection refused` sur le port 2223 | La VM `client` n'était pas encore prête (SSH pas encore démarré) ; la redirection `ssh-client` existait bien | Attendre le démarrage complet, puis relancer `ssh`. Sinon : `sudo systemctl status ssh` puis `sudo systemctl restart ssh` |
| 2.1 | `Destination Host Unreachable` en pingant `10.10.0.10` (réponse venant de `10.10.0.20`, 100 % de perte) | `web` avait été éteinte avec `poweroff` ; la passerelle répondait toujours, donc le réseau de `client` fonctionnait | Démarrer `web`, attendre ~30 s, vérifier avec `VBoxManage list runningvms` que les deux VM tournent. Les deux VM doivent être allumées pour communiquer |


## Conclusion

Ce TP montre comment un fournisseur IaaS lance des instances : une image de base clonée en 35 secondes, personnalisée (nom, identifiant, clés SSH, adresse), puis placée dans un réseau virtuel privé. Les deux VM communiquent dans le même sous-réseau `10.10.0.0/24` avec un délai de quelques millisecondes, mais l'hôte ne les atteint que par les redirections de port, comme un sous-réseau privé du cloud. `ufw` joue le rôle d'un groupe de sécurité : tout est refusé par défaut, et on n'ouvre que SSH puis HTTP. Les mesures montrent enfin que la mémoire et le disque sont alloués à la demande, ce qui permet la sur-allocation. Les difficultés rencontrées (conflit netplan dû à l'adresse MAC, clés SSH, nom d'hôte oublié, VM éteinte) rappellent pourquoi un clone doit être personnalisé, rôle que remplit cloud-init dans le cloud.

---

**Reste à compléter :** durée de l'installation d'Ubuntu (TP précédent) · durée du clonage de `client` · RAM de `web` (`showvminfo`) · taille du `.vdi` d'un clone · captures d'écran.
