# <table><tr><td><p align="center">Configurer un serveur Samba comme membre d'un domaine Active Directory</p></table></tr></td>

## Environnement

- Un serveur windows 2022 comme contrôleur de domaine principale Active Directory → ici: WIN-KO477AGSO9G.home.lab (ip = 192.168.10.28)
- Le domaine: home.lab
- Le sous-réseau du lab: 192.168.10.0/24
- La passerelle: 192.168.10.254
- L'installation du service samba se fera sur un serveur Debian 13.1
- L'accès aux dossier partagé se fera sur un serveur Ubuntu 24.04

## Installation du serveur Debian 13 pour le service samba

- Installation de Debian 13 sans DE - avec SSH server
- Monter /srv sur une partition dédiée aux partages (optionnel)
- La partition sur laquelle seront définis les partages doit être ext4 ou xfs pour la prise en charge des acls et des quotas

## Post-installation

Ajouter l'utilisateur créé lors de l'installation au groupe `sudo`
```bash
usermod -aG sudo nom_de_l_utilisateur
```
La majorité des commandes dans cet article doivent être effectuées avec les privilèges super utilisateur, ici la connexion au serveur se fera avec un utilisateur membre du groupe `sudo`

### Configuration réseau

**Edition des fichiers de configuration**
```bash
sudo nano /etc/network/interfaces
```
```
allow-hotplug ens18
iface ens18 inet static
       address 192.168.10.245
       netmask 255.255.255.0
       gateway 192.168.10.254
       nameserver 192.168.10.28 9.9.9.9
```
```bash
sudo systemctl restart networking
```
```bash
sudo nano /etc/hosts
```
```
127.0.0.1 localhost
127.0.1.1 sambasrv.home.lab sambasrv
192.168.10.28   WIN-KO477AGSO9G.home.lab        WIN-KO477AGSO9G
```
```bash
sudo nano /etc/resolv.conf
```
```
domain home.lab
nameserver 192.168.10.28
nameserver 9.9.9.9
```

**Tester la résolution DNS**

```bash
nslookup WIN-KO477AGSO9G.home.lab
```
Doit retourner
```
Server:  192.168.10.28
Address: 192.168.10.28#53

Name: WIN-KO477AGSO9G.home.lab
Address: 192.168.10.28
```

### Synchronisation de l'heure avec le serveur AD

**Définition du serveur NTP**

```bash
sudo nano /etc/systemd/timesyncd.conf
```
```bash
[Time]
NTP=WIN-KO477AGSO9G.home.lab
FallbackNTP=0.debian.pool.ntp.org 1.debian.pool.ntp.org 2.debian.pool.ntp.org 3.debian.pool.ntp.org
RootDistanceMaxSec=500
```

```bash
sudo systemctl enable --now systemd-timesyncd
```
Vérification:
```bash
sudo systemctl status systemd-timesyncd
```

---

## Créer la configuration Samba

**Installation des paquets**

```bash
sudo apt install acl attr samba winbind libpam-winbind libnss-winbind krb5-config krb5-user dnsutils python3-setproctitle
```

**Configuration de krb5**

```bash
sudo nano /etc/krb5.conf
```
```
[libdefaults]
        default_realm = HOME.LAB
        dns_lookup_realm = false
        dns_lookup_kdc = true
```

> The Samba teams recommends to not set any further parameters in the `/etc/krb5.conf` file

**Création du dossier partagé**

```bash
sudo mkdir -p /srv/shares/public
```
**Configuration du service samba**
```bash
sudo nano /etc/samba/smb.conf
```
```
[global]

    # Identification du domaine
    workgroup = HOME
    realm = HOME.LAB
    security = ADS
    netbios name = SAMBASRV

    # Backend IDMAP (AUTORID)
    idmap config * : backend = autorid
    idmap config * : range = 10000-999999
    idmap config * : rangesize = 100000

    # Paramètres d'authentification
    kerberos method = secrets and keytab
    dedicated keytab file = /etc/krb5.keytab
    winbind offline logon = false
    winbind enum users = yes
    winbind enum groups = yes
    winbind nss info = rfc2307

    # Performance et Logs
    server string = %h samba files server
    log file = /var/log/samba/log.%m
    log level = 1
    max log size = 1000
    socket options = TCP_NODELAY SO_KEEPALIVE SO_RCVBUF=131072 SO_SNDBUF=131072

    # Paramètres de sécurité
    client use spnego = yes
    client ntlmv2 auth = yes
    server min protocol = SMB2_10
    client min protocol = SMB2_10
    server smb encrypt = desired

    # Divers
    template shell = /bin/bash
    template homedir = /home/%U
    
    # ACLS
    vfs objects = acl_xattr
    map acl inherit = yes

# Définition des partages

[Public]
    path = /srv/shares/public
    comment = Partage Public
    read only = no
    
```

```bash
sudo smbcontrol all reload-config
sudo systemctl enable --now winbind smbd nmbd
```

---

## Rejoindre le domaine avec Samba

**Sur le DC windows server**

Créer un OU que l'on nommera "LinServers" dans cet exemple

<img width="1384" height="743" alt="6a995744f7edefd8f2702efadcfe76dd" src="https://github.com/user-attachments/assets/03ba1afe-7b34-401e-aa0f-878f95538881" />

**Sur le serveur debian**

```bash
sudo net ads join -S 192.168.10.28 -U "Administrator" HOME createcomputer="OU=LinServers,DC=home,DC=lab"
```

Configurer les permissions sur le dossier partagé

```bash
sudo chmod 2770 /srv/shares/public
sudo chown root:"HOME\domain users" /srv/shares/public
```

Vérifier l'intégration AD

```bash
wbinfo -u  # Liste les utilisateurs AD
wbinfo -g  # Liste les groupes AD
getent passwd  # Affiche les utilisateurs (locaux + AD)
getent group   # Affiche les groupes (locaux + AD)
```

Tester l'authentification

```bash
sudo smbclient -L localhost -U administrator
```

Vérifier les services
```bash
sudo systemctl status winbind smbd nmbd
```

_Optionnel: Configurer pam pour la création du dossier /home/user à la 1ère connexion_

```bash
sudo nano /etc/pam.d/common-session
```
_Ajouter cette ligne_
```
session required        pam_mkhomedir.so        skel=/etc/skel umask=0077
```

### Tests d'accès au partage

**Depuis un client linux**
```bash
smbclient //SAMBASRV/Public -U "HOME\Administrator"
```

**Depuis un client windows**

File Explorer &rarr; Network &rarr; SAMBASRV &rarr; Public

<img width="671" height="658" alt="4ad880d46175428d652affeb7fd8dd76" src="https://github.com/user-attachments/assets/f77f6f15-b60a-43b7-836a-d604681f47b8" />

---

## Gérer les permissions depuis le DC windows server

Start → Computer Management → Action → Connect to another computer = SAMBASRV

**Ne PAS faire de modification dans "share permissions"**

<img width="1412" height="836" alt="473e7bbda4e6969fae39c6e13dbd4080" src="https://github.com/user-attachments/assets/94764f50-5991-4052-b9a2-4cd30bfa66f7" />

**Les permissions se gèrent depuis "security" → "advanced"**

<img width="801" height="637" alt="220dd881d5e5f10fb5e7daede882a105" src="https://github.com/user-attachments/assets/74c128c1-4c4a-4fd0-abd9-70a2f0831145" />

---
`L'opération suivante est optionnelle, attention aux valeurs incrémentées qui pourraient altérer le bon fonctionnement de la machine!!!`

---

## Optimisations des performances pour samba

```bash
sudo nano /etc/sysctl.conf
```
```
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
```
```bash
sudo sysctl -p
```

### Détails

#### Paramètres Réseau

**net.core.rmem_max = 16777216**

- Valeur= 16Mo
- Signification: Taille maximale des buffers de réception (read) par socket
- Impact: Permet de recevoir plus de données en une fois avant de devoir traiter

**net.core.wmem_max = 16777216**

- Valeur= 16Mo
- Signification: Taille maximale des buffers d'envoi (write) par socket
- Impact: Permet d'envoyer plus de données en une fois, réduisant les appels système

**net.ipv4.tcp_rmem = 4096 87380 16777216**

- 4096 (4 Ko) : Taille minimale de buffer de réception
- 87380 (85 Ko) : Taille par défaut
- 16777216 (16 Mo) : Taille maximale (doit match rmem_max)
- Comportement : Le kernel ajuste dynamiquement entre min et max

**net.ipv4.tcp_wmem = 4096 65536 16777216**

- 4096 (4 Ko) : Taille minimale de buffer d'envoi
- 65536 (64 Ko) : Taille par défaut
- 16777216 (16 Mo) : Taille maximale (doit match wmem_max)

#### Paramètres Mémoire (Page Cache)

**vm.dirty_ratio = 10**

- Signification: Pourcentage de RAM sale (modifiée mais pas écrite sur disque) avant écriture forcée
- Exemple: Sur 16 Go RAM → écriture forcée à 1.6 Go de données sales
- Impact: Réduit les écritures disque fréquentes

**vm.dirty_background_ratio = 5**

- Signification: Pourcentage de RAM sale avant que le kernel lance l'écriture en background
- Exemple: Sur 16 Go RAM → début écriture à 0.8 Go de données sales
- Impact: Écriture asynchrone, moins bloquant

**Attention aux valeurs**

Adaptez à votre mémoire RAM:

|RAM totale|  dirty_ratio| dirty_background_ratio|
|---------:|------------:|----------------------:|
|4 Go      | 15          | 10                    |
|8 Go      | 10          | 5                     |
|16 Go     | 5           | 2                     |
|32 Go+    | 2           | 1                     |

---

##  Gestion des quotas d'espace disque par utilisateurs ou par groupes

**A savoir:**
- La limite _soft_ est une limite d'avertissement au-delà de laquelle l'utilisateur pourra continuer à écrire sur le disque jusqu'à la limite _hard_ et durant la période de _grâce_
- Les quotas définis pour un groupe s'appliquent de manière collective à l'ensemble des utilisateurs de ce groupe (quota partagé)
- Les quotas définis en inodes agissent sur le nombre de fichiers

### Sur le serveur Debian

_Tout comme pour la gestion des acls, le système de fichier doit pouvoir gérer les quotas, dans ce lab, nous utilisons ext4_

**Installation des paquets**
```bash
sudo apt install quota
```

**Ajouter la prise en charge des quotas au montage de la partition:**

```bash
sudo nano /etc/fstab
```
Définir les options _usrquota_ et _grpquota_ sur le montage concerné
```
/dev/sdb1       /srv    ext4    defaults,usrquota,grpquota       0       0
```
Remonter la partition
```bash
sudo mount -o remount /srv
```
Initialiser les quotas
```bash
quotacheck -cug /srv
```

**Connaître la valeur d'un bloc en octets sur notre partition**

- Identifier le périphérique:
```bash
df | grep /srv
```
ici:
```
/dev/sdb1       16401276    2120  15544016   1% /srv
```
- Afficher la taille d'un bloc en octets:
```bash
sudo tune2fs -l /dev/sdb1 | grep -i "block size"
```
ici:
```
Block size:               4096
```

### Pour fixer un quota à un utilisateur de l'Active Directory

**Identifier l'utilisateur sur lequel nous souhaitons appliquer un quota**

```bash
wbinfo -u
```

Exemple de sortie

```
HOME\administrator
HOME\guest
HOME\krbtgt
HOME\cronuser
HOME\test.user
```

**Configurer les limites _soft_ et _hard_ en octets pour l'utilisateur _test.user_**

```bash
sudo edquota "HOME\\test.user"
```
Cette commande ouvre l'éditeur par défaut pour définir la configuration
```
Disk quotas for user HOME\test.user (uid 111156):
  Filesystem                   blocks       soft       hard     inodes     soft     hard
  /dev/sdb1                         0          0          0          0        0        0
```

- Filesystem = nom du périphérique concerné par les quotas appliqués
- blocks = blocs déjà utilisés par l'utilisateur (ou le groupe)
- soft = limite _soft_ en octets
- hard = limite _hard_ en octets
- inodes = inodes déjà utilisés par l'utilisateur (ou le groupe)
- soft = limite _soft_ en inodes
- hard = limite _hard_ en inodes

Pour définir une limite _soft_ à 1Go et une limite _hard_ à 1,25Go
```
Disk quotas for user HOME\test.user (uid 111156):
  Filesystem                   blocks       soft       hard     inodes     soft     hard
  /dev/sdb1                         0      262144    327680          0        0        0
```
Pour vérifier la configuration définie:
```bash
sudo repquota -auv
```
Doit retourner
```
*** Report for user quotas on device /dev/sdb1
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
User            used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      52       0       0              8     0     0       
HOME\test.user --       0  262144  327680              0     0     0
```

### Pour fixer un quota à un groupe de l'Active Directory

**Identifier le groupe sur lequel nous souhaitons appliquer un quota**

```bash
wbinfo -g
```
Exemple de sortie
```
HOME\domain computers
HOME\domain controllers
HOME\schema admins
HOME\enterprise admins
HOME\cert publishers
HOME\domain admins
HOME\domain users
HOME\domain guests
HOME\group policy creator owners
HOME\ras and ias servers
HOME\allowed rodc password replication group
HOME\denied rodc password replication group
HOME\read-only domain controllers
HOME\enterprise read-only domain controllers
HOME\cloneable domain controllers
HOME\protected users
HOME\key admins
HOME\enterprise key admins
HOME\dnsadmins
HOME\dnsupdateproxy
HOME\access-denied assistance users
```

**Configurer les limites _soft_ et _hard_ en octets pour le groupe**

```bash
sudo edquota -g "HOME\\domain users"
```
Cette commande ouvre l'éditeur par défaut pour définir la configuration
```
Disk quotas for group HOME\domain users (gid 110513):
  Filesystem                   blocks       soft       hard     inodes     soft     hard
  /dev/sdb1                         0          0          0          0        0        0
```
Pour définir une limite _soft_ à 5Go et une limite _hard_ à 6Go
```
Disk quotas for group HOME\domain users (gid 110513):
  Filesystem                   blocks       soft       hard     inodes     soft     hard
  /dev/sdb1                         0      1310720     1572864       0        0        0
```
Pour vérifier la configuration définie
```bash
sudo repquota -agv
```
Doit retourner
```
*** Report for group quotas on device /dev/sdb1
Block grace time: 7days; Inode grace time: 7days
                        Block limits                File limits
Group           used    soft    hard  grace    used  soft  hard  grace
----------------------------------------------------------------------
root      --      40       0       0              5     0     0       
HOME\domain users --      0  1310720  1572864              0     0     0
```

**Configurer la période de grâce**

```bash
sudo edquota -t
```
Cette commande ouvre l'éditeur par défaut pour définir la configuration
```
Grace period before enforcing soft limits for users:
Time units may be: days, hours, minutes, or seconds
  Filesystem             Block grace period     Inode grace period
  /dev/sdb1                     7days                  7days
```
**La configuration de la période de grâce est valable pour l'ensemble des éléments concernés (groupes, utilisateurs, blocs et inodes) et des systèmes de fichiers pour lesquels la politique de quotas est activée.**

---

## Utiliser le partage _Public_ sur un serveur Ubuntu 24.04

### Joindre le serveur Ubuntu au domaine AD

**Installation des paquets**

```bash
sudo apt install sssd sssd-ad sssd-tools samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user cifs-utils realmd adcli
```

**Configuration DNS**

```bash
sudo nano /etc/hosts
```

```
127.0.0.1 localhost
127.0.1.1       ubuntusrv.home.lab     ubuntusrv
192.168.10.28   WIN-KO477AGSO9G.home.lab        WIN-KO477AGSO9G
```

```bash
sudo nano /etc/resolv.conf
```

```
nameserver 192.168.10.28
nameserver 192.168.10.254
options edns0 trust-ad
search home.lab
```
Vérification
```bash
nslookup WIN-KO477AGSO9G.home.lab
```

**Synchronisation de l'heure avec le serveur AD**

```bash
sudo nano /etc/systemd/timesyncd.conf
```

```
[Time]
NTP=WIN-KO477AGSO9G.home.lab
FallbackNTP=ntp.ubuntu.com
RootDistanceMaxSec=15
```

```bash
sudo systemctl enable --now systemd-timesyncd
```
Vérification
```bash
sudo systemctl status systemd-timesyncd
```

**Configuration de sssd**

```bash
sudo nano /etc/sssd/sssd.conf
```

```
[sssd]
domains = home.lab
config_file_version = 2
services = nss, pam

[domain/home.lab]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = HOME.LAB
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u
ad_domain = home.lab
use_fully_qualified_names = False
ldap_id_mapping = True
access_provider = ad
cache_credentials = true
enumerate = true
```

```bash
sudo systemctl restart sssd
```
**Configuration de krb5**
```bash
sudo nano /etc/krb5.conf
```
```
[logging]
   default = FILE:/var/log/krb5libs.log
   kdc = FILE:/var/log/krb5kdc.log
   admin_server = FILE:/var/log/kadmind.log

[libdefaults]
   default_realm = HOME.LAB
   dns_lookup_realm = false
   dns_lookup_kdc = false
   ticket_lifetime = 24h
   renew_lifetime = 7d
   forwardable = true

[realms]
   HOME.LAB = {
     kdc = WIN-KO477AGSO9G.home.lab
     admin_server = WIN-KO477AGSO9G.home.lab
   }

[domain_realm]
    .home.lab = HOME.LAB
    home.lab = HOME.LAB
```
**Joindre le serveur AD avec realm**

```bash
sudo realm -v discover home.lab
sudo realm join -U administrator@HOME.LAB home.lab --computer-ou="OU=LinServers,DC=home,DC=lab"
```

_Optionnel: pour la création du dossier personnel des utilisateurs à la 1ère connexion_

```bash
sudo pam-auth-update --enable mkhomedir
```

Vérifier la jonction

```bash
sudo realm list
```

Tester l'authentification kerberos

```bash
kinit Administrator@HOME.LAB
klist
```

Vérifier la résolution des utilisateurs AD

```bash
getent passwd HOME\\Administrator
```

**Mise en place du point de montage _Public_ au démarrage du serveur avec fstab**

Création du dossier utilisé pour le montage

```bash
sudo mkdir /mnt/public
```

Configuration du fichier fstab

```bash
sudo nano /etc/fstab
```
On ajoute cette ligne
```bash
//sambasrv.home.lab/public /mnt/public cifs _netdev,sec=krb5,cruid=0,multiuser,vers=3.1.1,x-systemd.after=krb-boot.service   0 0
```

**Options:**

- _netdev #_attend le réseau avant le montage_
- sec=krb5 #_authentification kerberos_
- cruid=0 #_défini le propriétaire du cache pour l'authentification (root)_
- vers=3.1.1 #_version du protocole SMB utilisé_
- x-systemd.after=krb-boot.service #_le service d'obtention de ticket kerberos doit être lancé_


Création d'un script pour kinit
```bash
sudo nano /usr/local/bin/kinit-machine.sh
```
```bash
#!/bin/bash
/usr/bin/kinit -k UBUNTUSRV\$@HOME.LAB
```
Le rendre exécutable
```bash
sudo chmod +x /usr/local/bin/kinit-machine.sh
```
Tester l'obtention du ticket via le script
```bash
sudo bash /usr/local/bin/kinit-machine.sh
sudo klist
```
Doit retourner une sortie similaire à
```
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: UBUNTUSRV$@HOME.LAB

Valid starting       Expires              Service principal
09/29/2025 18:42:06  09/30/2025 04:42:06  krbtgt/HOME.LAB@HOME.LAB
	renew until 10/06/2025 18:42:06
```
Création d'un service systemd pour l'obtention du ticket kerberos au démarrage
```bash
sudo nano /etc/systemd/system/krb-boot.service
```
```
[Unit]
Description=Acquire Kerberos ticket at boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/kinit-machine.sh
ExecStartPost=/bin/sleep 5
TimeoutSec=60

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable krb-boot.service
```
Exécution automatique du script toutes les 8h (validité du ticket = 10h par défaut)
```bash
sudo crontab -e
```
On ajoute cette ligne
```
0 */8 * * * /usr/local/bin/kinit-machine.sh
```
**Monter le partage samba immédiatement et redémarrer le serveur si possible pour vérifications**

Montage immédiat
```bash
sudo mount -av
```
Doit retourner
```
mount.cifs kernel mount options: ip=192.168.10.245,unc=\\sambasrv.home.lab\public,sec=krb5,multiuser,vers=3.1.1,cruid=0,user=root,pass=********
/mnt/public              : successfully mounted
```
Redémarrer le serveur si possible
```bash
sudo systemctl reboot
```
Vérification de la disponibilité du montage après le redémarrage du serveur
```bash
df | grep public
```
Doit retourner
```
//sambasrv.home.lab/public        1310720       12   1310708   1% /mnt/public
```

**Vérifier la connexion, l'accès au partage et les permissions avec un utilisateur de l'AD selon la configuration des acls**

_Pour cet exemple, un utilisateur nommé test.user a été créé dans l'AD (membre de Domain Users par défaut)_

```bash
su - test.user@home.lab
Password: 
Creating directory '/home/test.user@home.lab'.
```
On crée un fichier test.txt à la racine du dossier _public_
```bash
cd /mnt/public
touch test.txt
ls -l
```
Doit retourner
```
-rwxr-xr-x 1 test.user@home.lab domain users@home.lab 0 Sep 29 19:26 test.txt
```
Notre utilisateur a les permissions en écriture à la racine du partage _public_

---

## Utiliser le partage _Public_ sur un serveur REHL 10
_Ici nous utiliserons le root (sudo -i) pour l'ensemble des commandes_
### Ajouter le serveur AD dans les dns de la machine

1. Définir le hostname de la machine
```bash
hostnamectl set-hostname rhel-srv
```
2. Renseigner les serveurs DNS
```bash
vim /etc/resolv.conf
```
```
search home.lab
nameserver 192.168.10.254
nameserver 192.168.10.28
```

**Vérification:**
```bash
nslookup WIN-KO477AGSO9G.home.lab
Server:		192.168.10.254
Address:	192.168.10.254#53

Name:	WIN-KO477AGSO9G.home.lab
Address: 192.168.10.28
```

3. Installation des paquets nécessaires

```bash
dnf install sssd realmd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation openldap-clients python3-policycoreutils
```
4. Rejoindre le domaine avec realm

```bash
realm join --user=Administrator --computer-ou="OU=LinServers,DC=home,DC=lab" home.lab
```

**Vérification:**
```bash
realm list
```
Doit renvoyer
```
home.lab
  type: kerberos
  realm-name: HOME.LAB
  domain-name: home.lab
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  required-package: sssd-common
  required-package: oddjob
  required-package: oddjob-mkhomedir
  required-package: sssd-ad
  required-package: adcli
  required-package: samba-common-tools
  login-formats: %U@home.lab
  login-policy: allow-realm-logins
  ```

---

_Source principale:_
- <https://wiki.samba.org/index.php/Setting_up_a_Share_Using_Windows_ACLs>
- https://wiki.debian.org/fr/Quota
