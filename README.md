# DNS-Data-NAS-Service
Mise en place d'un serveur NAS / Backup

## Décryptage du message subliminal
**NAS is not a backup !**

Je vous rejoins complètement dans cette analyse!!!

*Je me demande du coup pourquoi je fais ce travail...*
Bon pour ce code au départ je suis parti un peu loin en pensant à un Vigenère...
Mais non juste un XOR suffisait, comme quoi l'idée la plus simple est souvent la
meilleure.

```python
from operator import xor

l=['16','0E','01','78','26','21','78','21','3D','2C','6F','33','78','2D','33','3B','24','27','28','6F','73']
key = 'XOR' * 7
msg = ''
for k in range(len(l)):
     msg += chr(xor(int(l[k],16),int(ord(key[k]))))
print(msg)
```

## Les besoins
Serveur NAS:
- OS: Debian 12.2 (installation minimale)
- Stockage:
    - Boot 60 Go SSD
    - Stockage Raid 6: 7 * 3To
    - LVM: 3 * 1To
- Services:
    - Samba / CIFS
    - NFS
- Groupes:
    - Admin admin (rw)
    - User  user (r)
- Utilisateurs:    
    - Jean-Luc EDDIE (User)
    - Amin ALLYANT (Admin)
    - Medhi CAUTPUTMA (User)
    - Celestin LIRRTIRY (Admin)

Serveur de Backup:
- même configuration que le serveur NAS
- Services complémentaires:
    - rsync pour la gestion des sauvegardes

En analysant les besoins, on peut rapidement remarquer où sont les points de
défaillance critique:
- le système d'exploitation n'est pas protégé (2 JBOSS en raid 1 auraient été mieux);
- le raid 6 tolère 2 disques en échec au maximum et ramène la dimension du stockage à 15Go au lieu des 21
initialement prévus;
- les 3 disques de 1To en bonus ne servent à rien car non intégrables au raid par la suite. Si le client vient
à s'en servir (et il le fera) il n'aura aucune solution de secours hormis ce deuxième NAS-Backup qui ne sert à rien
puisque lui aussi aura vieilli en même temps que le premier (hmm, obsolescence programmée quand tu nous tiens).
- pas de redondance de la carte réseau avec failover en cas de défaut d'une des deux interfaces.

La conclusion de ce merveilleux travail, avant même de l'avoir commencé, c'est que le client finira par se retourner
contre notre prometteuse petite PME, et tout cela parce que Paul nous demande de faire de la m....

Bon, c'est le chef! Et puisque l'on obéit au chef, alors au travail!!!

## Choix pour la simulation et installation du laboratoire:
- Environnement: VirtualBox 7.0.12 + Extension Pack
- Debian 12.2 x64
- Hardware:
    - vCPU: 1
    - RAM: 4096 Mo
    - Disque système: 10 Gio (VDI)
- Paquets installés: utilitaires usuels du système
- Pas d'utilisateur root
- Un utilisateur -> Kaman kaman G:kaman,sudo

L'installation de la première machine étant terminée on ajoute l'ensemble
des disques:
- 7 * 3Go pour le RAID 6
- 3 * 1Go pour le LVM

![ConfigNasVB](./pictures/ConfigNasVB.jpg "Configuration de la machine virtuelle NAS")

Une fois terminée l'installation de cette première machine, on la clone intégralement
en veillant à changer les adresses MAC et en conserver le nom et les UUIDs des disques.

On finit enfin par installer un serveur ssh sur chaque machine pour gérer l'installation
depuis notre poste Contrôleur:
```bash
sudo apt install ssh -y &&
systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Tue 2023-10-24 16:57:11 CEST; 2min 7s ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 550 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 582 (sshd)
      Tasks: 1 (limit: 4645)
     Memory: 9.2M
        CPU: 126ms
     CGroup: /system.slice/ssh.service
             └─582 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"
```

Super! Une bonne chose de faite. On installe à présent deux/trois outils indispensables
pour se sentir à la maison.

```bash
sudo apt install -y vim screen bash-completion gdisk mdadm lvm2 xfsprogs xfsdump acl attr rsync
```

## On passe à l'action: configuration du RAID 6

- Analyse de la disposition des disques:
```bash
lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   10G  0 disk
├─sda1   8:1    0    9G  0 part /
├─sda2   8:2    0    1K  0 part
└─sda5   8:5    0  975M  0 part [SWAP]
sdb      8:16   0    3G  0 disk
sdc      8:32   0    3G  0 disk
sdd      8:48   0    3G  0 disk
sde      8:64   0    3G  0 disk
sdf      8:80   0    3G  0 disk
sdg      8:96   0    3G  0 disk
sdh      8:112  0    3G  0 disk
sdi      8:128  0    1G  0 disk
sdj      8:144  0    1G  0 disk
sdk      8:160  0    1G  0 disk
sr0     11:0    1 1024M  0 rom
```

Parfait notre disque système est sda. Nous allons configurer
les disques sd{b..h} pour le RAID et les disques sd{i..k} pour
LVM.

```bash
# Un petit script bash pour faire le job
for k in {b..h}; do
    sgdisk --new=1:0:0 --typecode=1:fd00 --change-name=1:"RAID" /dev/sd$k
done

for k in {i..k}; do
    sgdisk --new=1:0:0 --typecode=1:8e00 --change-name=1:"LVM" /dev/sd$k
done
```

Enfin on vérifie le travail que l'on conserve comme preuve d'un
travail vraiment bien fait:
```bash
for k in {b..k}; do
    echo '--------------------------------------' >> verif_disks.log
    echo "-     DISK: /dev/sd$k" >> verif_disks.log
    echo '--------------------------------------' >> verif_disks.log
    sgdisk -i 1 /dev/sd$k >> verif_disks.log
done
```
[verif_disks.log](./reports/verif_disks.log)

Je vous l'accorde, le script pour la partie vérification aurait pu être un peu plus soigné.
Mais le JOB est fait.

- Création de la grappe:
```bash
mdadm --create /dev/md0 --level=6 --raid-device=7 /dev/sd{b..h}1
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

# Et on vérifie encore une fois que tout s'est bien passé
cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdh1[6] sdg1[5] sdf1[4] sde1[3] sdd1[2] sdc1[1] sdb1[0]
      15705600 blocks super 1.2 level 6, 512k chunk, algorithm 2 [7/7] [UUUUUUU]

unused devices: <none>

mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Wed Oct 25 17:52:09 2023
        Raid Level : raid6
        Array Size : 15705600 (14.98 GiB 16.08 GB)
     Used Dev Size : 3141120 (3.00 GiB 3.22 GB)
      Raid Devices : 7
     Total Devices : 7
       Persistence : Superblock is persistent

       Update Time : Wed Oct 25 17:53:08 2023
             State : clean
    Active Devices : 7
   Working Devices : 7
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : nas:0  (local to host nas)
              UUID : c02d27cf:97e82f81:5a10cc14:975c159d
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       17        0      active sync   /dev/sdb1
       1       8       33        1      active sync   /dev/sdc1
       2       8       49        2      active sync   /dev/sdd1
       3       8       65        3      active sync   /dev/sde1
       4       8       81        4      active sync   /dev/sdf1
       5       8       97        5      active sync   /dev/sdg1
       6       8      113        6      active sync   /dev/sdh1
```

- En finir avec cette partie:

Le client n'ayant pas tranché pour le système de fichiers, nous partons sur un XFS. On pourra faire un dump
du système de fichiers par la suite pour l'envoyer directement sur un serveur de bandes au cas où notre
client et Paul deviendraient plus raisonnables.
```bash
# Formatage XFS
mkfs.xfs /dev/md0
log stripe unit (524288 bytes) is too large (maximum is 256KiB)
log stripe unit adjusted to 32KiB
meta-data=/dev/md0               isize=512    agcount=16, agsize=245504 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=3926400, imaxpct=25
         =                       sunit=128    swidth=640 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# Récupération de l'UUID du disque md0
blkid /dev/md0
/dev/md0: UUID="9848edc3-b8e9-44fe-bbc0-10ae8fb467e2" BLOCK_SIZE="512" TYPE="xfs"

# Création des deux répertoires pour le montage des partitions
mkdir -p /mnt/exports/{raid,lvm}

# Ecriture du fichier /etc/fstab pour le montage automatique
echo 'UUID=9848edc3-b8e9-44fe-bbc0-10ae8fb467e2 /mnt/raid xfs defaults 1 2' | tee -a /etc/fstab
UUID=9848edc3-b8e9-44fe-bbc0-10ae8fb467e2 /mnt/exports/raid xfs defaults 1 2

# On recharge le montage automatique
systemctl daemon-reload
mount -a
mount | grep md0
/dev/md0 on /mnt/exports/raid type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,sunit=1024,swidth=5120,noquota)
```


## Le JBOD à présent
Précédemment nous avions déjà préparé les disques, il ne reste plus qu'à fabriquer le tas de .....

```bash
pvcreate /dev/sd{i..k}1
vgcreate vg_nas /dev/sd{i..k}1
lvcreate -l 100%FREE -n lv_nas vg_nas
lvdisplay
--- Logical volume ---
  LV Path                /dev/vg_nas/lv_nas
  LV Name                lv_nas
  VG Name                vg_nas
  LV UUID                mGCoGF-i2VU-jKox-GP45-mWE6-KNk7-dlffDh
  LV Write Access        read/write
  LV Creation host, time nas, 2023-10-25 19:42:11 +0200
  LV Status              available
  # open                 0
  LV Size                <2,99 GiB
  Current LE             765
  Segments               3
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:0

# On formate en XFS
mkfs.xfs /dev/vg_nas/lv_nas
meta-data=/dev/vg_nas/lv_nas     isize=512    agcount=4, agsize=195840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=783360, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# Et on monte
blkid /dev/vg_nas/lv_nas
/dev/vg_nas/lv_nas: UUID="6571e3e1-dff6-4c33-96a8-c45db148c3f6" BLOCK_SIZE="512" TYPE="xfs"

echo 'UUID=6571e3e1-dff6-4c33-96a8-c45db148c3f6 /mnt/exports/lvm xfs defaults 1 2' | tee -a /etc/fstab
UUID=6571e3e1-dff6-4c33-96a8-c45db148c3f6 /mnt/exports/lvm xfs defaults 1 2

systemctl daemon-reload
mount -a
mount | grep lvm
/dev/mapper/vg_nas-lv_nas on /mnt/exports/lvm type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
```

Voilà enfin un système de fichiers parfaitement opérationnel:
```bash
Sys. de fichiers          Taille Utilisé Dispo Uti% Monté sur
udev                        1,9G       0  1,9G   0% /dev
tmpfs                       392M    688K  391M   1% /run
/dev/sda1                   8,9G    1,4G  7,0G  17% /
tmpfs                       2,0G       0  2,0G   0% /dev/shm
tmpfs                       5,0M       0  5,0M   0% /run/lock
/dev/md0                     15G    140M   15G   1% /mnt/exports/raid
tmpfs                       392M       0  392M   0% /run/user/1000
/dev/mapper/vg_nas-lv_nas   3,0G     54M  2,9G   2% /mnt/exports/lvm
```

## NFS & SAMBA/CIFS
Alors là c'est le pompon!!! Quitte à continuer à faire des cochonneries je vais tranquillement commencer par
me faire un café. Je reviens dans deux minutes...

...

...

...

J'espère que je n'ai pas été trop long.
Avant de faire quoi que ce soit nous allons rapidement créer nos petits utilisateurs à partir d'un fichier csv
préparé aux petits oignons:
```csv
NOM,PRENOM,GROUPE
EDDIE,Jeanluc,User
ALLYANT,Amin,Admin
CAUTPUTMA,Medhi,User
LIRRTIRY,Celestin,Admin
```

On ajoute les deux groupes sur le système:
```bash
groupadd -g 10001 Admin
groupadd -g 10002 User
```

Puis passage à la moulinette par notre script CreateUser. Pour le test, le mot de passe de tous les utilisateurs
est générique: **abcd**.

D'autre part, les utilisateurs Admin/User sont configurés de la manière suivante:
- pas de shell de connexion,
- pas de groupe personnel,
- pas de répertoire personnel.
```bash
#!/bin/bash

NEWUSERFILE=$1

# Le fichier csv est composé de 3 champs
# Nom,Prénom,Role
#

for ENTRY in $(cat $NEWUSERFILE.csv | grep -v ^NOM)
do
    FIRSTNAME=$(echo $ENTRY | cut -d, -f2)
	LASTNAME=$(echo $ENTRY | cut -d, -f1)
	ROLE=$(echo $ENTRY | cut -d, -f3)
	
    echo $FIRSTNAME $LASTNAME $PASSWORD $ROLE

	# Nettoyage des caractères accentués pour la génération du nom d'utilisateur
	NORMFIRST=$(echo $FIRSTNAME | sed 'y/àâçéèëêïîöôùüûÀÇÉÈËÊÏÎÖÔÙÜÛ/aaceeeeiioouuuACEEEEIIOOUUU/')
	NORMLAST=$(echo $LASTNAME | sed 'y/àâçéèëêïîöôùüûÀÇÉÈËÊÏÎÖÔÙÜÛ/aaceeeeiioouuuACEEEEIIOOUUU/')
    
    # Normalisation du nom d'utilisateur    
	FIRSTINITIAL=$(echo $NORMFIRST | cut -c 1 | tr 'A-Z' 'a-z')
	LOWERLASTNAME=$(echo $NORMLAST | tr -d \' | tr 'A-Z' 'a-z')
	ACCTNAME=$FIRSTINITIAL$LOWERLASTNAME
	
    # Tests pour les doublons
	id $ACCTNAME
    if [ $? -eq 0 ]; then
        continue
	else
		# Création du compte et affectation au groupe
		echo "Création de l'utilisateur $ACCTNAME $ROLE"
        if [ "$ROLE" = "Admin" ] ; then
            useradd -c "$FIRSTNAME $LASTNAME" -g Admin -s /usr/sbin/nologin $ACCTNAME
        else
            useradd -c "$FIRSTNAME $LASTNAME" -g User -s /usr/sbin/nologin $ACCTNAME
        fi
		echo "$ACCTNAME:abcd" | chpasswd
        echo "$ACCTNAME $PASSWD $ROLE" >> create.txt # pour pouvoir nettoyer le système durant les tests et après par la même occasion.
	fi
done
exit 0
```

```bash
./CreateUser user.csv
Jeanluc EDDIE User
id: « jeddie » : utilisateur inexistant
Création de l'utilisateur jeddie User
Amin ALLYANT Admin
id: « aallyant » : utilisateur inexistant
Création de l'utilisateur aallyant Admin
Medhi CAUTPUTMA User
id: « mcautputma » : utilisateur inexistant
Création de l'utilisateur mcautputma User
Celestin LIRRTIRY Admin
id: « clirrtiry » : utilisateur inexistant
Création de l'utilisateur clirrtiry Admin
```

### Installation du service NFS

Comme à notre habitude un petit **apt install** s'impose:

```bash
apt install -y nfs-kernel-server nfs-common nfs4-acl-tools
systemctl status nfs-kernel-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; preset: enabled)
     Active: active (exited) since Wed 2023-10-25 22:18:27 CEST; 34s ago
   Main PID: 19114 (code=exited, status=0/SUCCESS)
        CPU: 17ms
```
Bien, on a un service actif, moins bien il doit être configuré pour répondre aux différentes
versions de NFS. On vérifie et on essaye d'arrange cela.

```bash
cat /proc/fs/nfsd/version
-2 +3 +4 +4.1 +4.2
```

**Hmm sur Debian je ne trouve pas encore l'option pour
désactiver les versions V3,V4,V4.1... chiant**

Quoiqu'il en soit on va préparer les dossiers d'exports que nos chers amis linuxiens
puissent *mounter* les systèmes de fichiers sur leurs machines.

Edition de /etc/exports:
```txt
# /etc/exports
/mnt/exports/raid       192.168.56.0/24(rw,sync,no_root_squash,no_subtree_check,fsid=0,acl)
/mnt/exports/lvm        192.168.56.0/24(rw,sync,no_root_squash,no_subtree_check,fsid=0,acl)
```

Modification des droits sur les dossiers d'exportation NFS
```bash
chown nobody:nogroup /mnt/exports
chmod -R 755 /mnt/exports
chown -R nobody:Admin /mnt/exports/{raid,lvm}
chmod -R 2775 /mnt/exports/{raid,lvm}
```
Le umask par défaut sur un système GNU/Linux est 022. En principe,
pour le modifier il suffit de changer la valeur dans **/etc/login.defs**
pour le mettre à 002. Nos utilisateurs n'ayant pas de groupe personnel
dans le système et étant directement affecté au groupe *Admin* ou *User*
on va pouvoir rapidement gérer l'ensemble des droits sur les partages sans se
prendre la tête avec une gestion poussée des ACLs.

**Remarque:** on ajoute aux fichiers
*/etc/pam.d/{common-session,common-session-noninteractive}* **session optional pam_umask.so umask=0002**

Enfin on relance tout le bouzin et on regarde:
```bash
systemctl restart nfs-server
systemctl status nfs-server
● nfs-server.service - NFS server and services
     Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; preset: enabled)
    Drop-In: /run/systemd/generator/nfs-server.service.d
             └─order-with-mounts.conf
     Active: active (exited) since Sat 2023-10-28 22:29:06 CEST; 9s ago
    Process: 2599 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
    Process: 2600 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
   Main PID: 2600 (code=exited, status=0/SUCCESS)
        CPU: 12ms

oct. 28 22:29:06 nas systemd[1]: Starting nfs-server.service - NFS server and services...
oct. 28 22:29:06 nas systemd[1]: Finished nfs-server.service - NFS server and services.

showmount -e
Export list for nas:
/mnt/exports/lvm  192.168.56.0/24
/mnt/exports/raid 192.168.56.0/24
```

**Great!!!**

### Installation de SAMBA
```bash
apt install -y samba samba-client cifs-utils
```

On conserve tous les paramètres par défaut et on crée nos deux
partages:
```smb.conf
[RAID]
        comment = NAS RAID 6
        path = /mnt/raid
        browseable = yes
        read only = yes
        guest ok = no
        valid users = @Admin @User
        write list = @Admin
        create mask = 2664
        directory mask = 2775
        force directory mode = 775
        map acl inherit = yes
        inherit acls = yes

[LVM]
        comment = NAS LVM JBOD
        path = /mnt/lvm
        browseable = yes
        read only = yes
        guest ok = no
        valid users = @Admin @User
        write list = @Admin
        create mask = 2664
        directory mask = 2775
        force directory mode = 775
        map acl inherit = yes
        inherit acls = yes
```

Création des utilisateurs dans la base
```
for k in jeddie aallyant mcautputma clirrtiry; do
    echo -e "abcd\nabcd" | pdbedit -a $k
done

Unix username:        jeddie
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-1275815848-2222129501-753046093-1002
Primary Group SID:    S-1-5-21-1275815848-2222129501-753046093-513
Full Name:            Jeanluc EDDIE
Home Directory:       \\NAS\jeddie
HomeDir Drive:
Logon Script:
Profile Path:         \\NAS\jeddie\profile
Domain:               NAS
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          mer., 06 févr. 2036 16:06:39 CET
Kickoff time:         mer., 06 févr. 2036 16:06:39 CET
Password last set:    jeu., 26 oct. 2023 01:14:24 CEST
Password can change:  jeu., 26 oct. 2023 01:14:24 CEST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
Unix username:        aallyant
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-1275815848-2222129501-753046093-1003
Primary Group SID:    S-1-5-21-1275815848-2222129501-753046093-513
Full Name:            Amin ALLYANT
Home Directory:       \\NAS\aallyant
HomeDir Drive:
Logon Script:
Profile Path:         \\NAS\aallyant\profile
Domain:               NAS
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          mer., 06 févr. 2036 16:06:39 CET
Kickoff time:         mer., 06 févr. 2036 16:06:39 CET
Password last set:    jeu., 26 oct. 2023 01:15:23 CEST
Password can change:  jeu., 26 oct. 2023 01:15:23 CEST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
Unix username:        mcautputma
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-1275815848-2222129501-753046093-1004
Primary Group SID:    S-1-5-21-1275815848-2222129501-753046093-513
Full Name:            Medhi CAUTPUTMA
Home Directory:       \\NAS\mcautputma
HomeDir Drive:
Logon Script:
Profile Path:         \\NAS\mcautputma\profile
Domain:               NAS
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          mer., 06 févr. 2036 16:06:39 CET
Kickoff time:         mer., 06 févr. 2036 16:06:39 CET
Password last set:    jeu., 26 oct. 2023 01:15:57 CEST
Password can change:  jeu., 26 oct. 2023 01:15:57 CEST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
Unix username:        clirrtiry
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-1275815848-2222129501-753046093-1005
Primary Group SID:    S-1-5-21-1275815848-2222129501-753046093-513
Full Name:            Celestin LIRRTIRY
Home Directory:       \\NAS\clirrtiry
HomeDir Drive:
Logon Script:
Profile Path:         \\NAS\clirrtiry\profile
Domain:               NAS
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          mer., 06 févr. 2036 16:06:39 CET
Kickoff time:         mer., 06 févr. 2036 16:06:39 CET
Password last set:    jeu., 26 oct. 2023 01:16:16 CEST
Password can change:  jeu., 26 oct. 2023 01:16:16 CEST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
```

Enfin on teste les paramètres et on relance le serveur:
```bash
testparm
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
Weak crypto is allowed by GnuTLS (e.g. NTLM as a compatibility fallback)

Server role: ROLE_STANDALONE

Press enter to see a dump of your service definitions

# Global parameters
[global]
        log file = /var/log/samba/log.%m
        logging = file
        map to guest = Bad User
        max log size = 1000
        obey pam restrictions = Yes
        pam password change = Yes
        panic action = /usr/share/samba/panic-action %d
        passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
        passwd program = /usr/bin/passwd %u
        server role = standalone server
        unix password sync = Yes
        usershare allow guests = Yes
        idmap config * : backend = tdb


[homes]
        browseable = No
        comment = Home Directories
        create mask = 0700
        directory mask = 0700
        valid users = %S


[printers]
        browseable = No
        comment = All Printers
        create mask = 0700
        path = /var/tmp
        printable = Yes


[print$]
        comment = Printer Drivers
        path = /var/lib/samba/printers


[RAID]
        comment = NAS RAID 6
        create mask = 02664
        directory mask = 02775
        force directory mode = 0775
        inherit acls = Yes
        map acl inherit = Yes
        path = /mnt/exports/raid
        read only = No
        valid users = @Admin @User
        write list = @Admin


[LVM]
        comment = NAS LVM JBOD
        create mask = 02664
        directory mask = 02775
        force directory mode = 0775
        inherit acls = Yes
        map acl inherit = Yes
        path = /mnt/exports/lvm
        read only = No
        valid users = @Admin @User
        write list = @Admin


systemctl restart smbd.service
systemctl status smbd
● smbd.service - Samba SMB Daemon
     Loaded: loaded (/lib/systemd/system/smbd.service; enabled; preset: enabled)
     Active: active (running) since Sat 2023-10-28 22:13:41 CEST; 7s ago
       Docs: man:smbd(8)
             man:samba(7)
             man:smb.conf(5)
    Process: 2449 ExecCondition=/usr/share/samba/is-configured smb (code=exited, status=0/SUCCESS)
    Process: 2452 ExecStartPre=/usr/share/samba/update-apparmor-samba-profile (code=exited, status=0/SUCCESS)
   Main PID: 2461 (smbd)
     Status: "smbd: ready to serve connections..."
      Tasks: 4 (limit: 4640)
     Memory: 6.7M
        CPU: 203ms
     CGroup: /system.slice/smbd.service
             ├─2461 /usr/sbin/smbd --foreground --no-process-group
             ├─2463 /usr/sbin/smbd --foreground --no-process-group
             ├─2464 /usr/sbin/smbd --foreground --no-process-group
             └─2465 /usr/sbin/smbd --foreground --no-process-group

oct. 28 22:13:41 nas systemd[1]: Starting smbd.service - Samba SMB Daemon...
oct. 28 22:13:41 nas systemd[1]: Started smbd.service - Samba SMB Daemon.
oct. 28 22:13:41 nas smbd[2465]: pam_unix(samba:session): session opened for user clirrtiry(uid=1004) by (uid=0)
```
Comme nous pouvons le voir, notre utilisateur clirrtiry n'a pas pu attendre et il s'est déjà connecté à notre
super serveur SAMBA. **Un coquin ce clirrtiry**

## Configuration du système de réplication
Apparement, rsync est le choix du chef! Donc on part là-dessus.
Deux options s'offrent à nous. Faisons nous le mirroir en mode
PUSH ou PULL? D'autre part à quelle fréquence? Est-ce le client 
qui décidera  de ce point ou bien est-ce que je dois prendre 
une initiative?

Bon, puisque l'on est obligé de trancher rapidement, je fais partir
sur un mode PULL avec une fréquence de 1 minute pour les essais.

Néanmoins, le client va être déçu car si pour une raison quelconque
il souhaite récupérer un fichier perdu accidentellement et bien
ce n'est pas en allant voir son NAS-(BACKUP) qu'il va pouvoir le
retrouver. Cela lui permettra juste d'être redondant en cas de défaillance
majeur du NAS principal.

- Création d'un utilisateur de sauvegarde *backuprsync* appartenant au groupe *sudo*
avec accès à la commande **sudo** sans mot de passe. Cet utilisateur n'aura pas besoin
de mot de passe pour se connecter, donc on ne le définit pas.

```bash
useradd -m -u 20000 -G sudo -c "Backup Admin" -s /bin/bash backuprsync
ssh-keygen -t ed25519 -C "BackupRsync" -N "" -f saversync
mkdir ~backuprsync/.ssh
cp  saversync* ~backuprsync/.ssh/.
chown -R backuprsync:backuprsync ~backuprsync/.ssh
```

On a généré une paire de clefs SSH que l'on a mis sur les deux
machines pour l'utilisateur *backuprsync* pour que rsync puisse se connecter sans mot
passe. Evidemment, on a copié la clef publique dans le fichier authorized_keys.
Enfin on a mis la commande sudo sans mot de passe pour tous les membres du groupe
**sudo**. En fonction de la politique de sécurité il sera toujours temps d'effectuer
un petit réglage sur ce point.

```sudo
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```

- Passons aux scripts de sauvegarde et de restauration:
```bash
#!/bin/bash
#
# filename: BackupNas
#
# Description: sauvegarde NAS ---> NAS-BACKUP à exécuter depuis nas-backup
#

PORT=22
USERNAME=backuprsync
KEY=/home/$USERNAME/.ssh/saversync
rep1=/mnt/exports/raid/
rep2=/mnt/exports/lvm/

DATE=$(date +"%Y%m%d-%H%M")
REPORT=/home/backuprsync/report-save-$DATE

for k in $rep{1..2}; do
    echo "/*********************************************\" >> $REPORT
    echo -e "\t\t$k" >> $REPORT
    echo "\*********************************************/" >> $REPORT
    rsync -arlEv --delete --force --stats \
        --rsync-path="sudo rsync" \
        -e "ssh -i $KEY -p $PORT" \
        $USERNAME@nas:$k $k >> $REPORT
    echo -e "\n\n\n" >> $REPORT
done
exit 0
```

```bash
#!/bin/bash
#
# filename: RestoreNas
#
# Description: restauration NAS-BACKUP ---> NAS à exécuter depuis nas
#

PORT=22
USERNAME=backuprsync
KEY=/home/$USERNAME/.ssh/saversync
rep1=/mnt/exports/raid/
rep2=/mnt/exports/lvm/

DATE=$(date +"%Y%m%d-%H%M")
REPORT=/home/backuprsync/report-restore-$DATE

for k in $rep{1..2}; do
    echo "/*********************************************\ >> $REPORT
    echo -e "\t\t$k >> $REPORT"
    echo "\*********************************************/" >> $REPORT
    rsync -av --delete --force --stats \
        --rsync-path="sudo rsync" \
        -e "ssh -i $KEY -p $PORT" \
        $USERNAME@nas-backup:$k $k >> $REPORT
    echo -e "\n\n\n" >> $REPORT
done
exit 0
```

## TEST RSYNC (à remettre au client)
Paramètres pour les tests:
- nas: 192.168.56.101/24
- nas-backup: 192.168.56.103/24

Modification des fichiers hosts en conséquence:
```bash
# /etc/hosts
192.168.56.101 nas.laplateforme.lan nas
192.168.56.103 nas-backup.laplateforme.lan nas-backup
```

On génère à l'aide d'un script des fichiers et répertoires sur les volumes RAID et LVM
```bash
#!/bin/bash
#
# Description: génère des dossiers et des fichiers quelconques
#
rep1=/mnt/exports/raid/
rep2=/mnt/exports/lvm/

for k in range $rep{1..2}; do
    mkdir -p $k/dir{1..10}/dir{1..10}
    touch $k/dir{1..10}/dir{1..10}/file{1..10}
    touch $k/dir{1..10}/file{1..10}
    touch $k/file{1..10}
done
exit 0
```

Enfin on effectue 5 tests:
1. synchronisation complète des deux volumes.
[Report 20231027-2012](./reports/report-save-20231027-2012)
2. on efface tous les répertoires du volume RAID
et l'on crée 10 nouveaux fichiers.
[Report 20231027-2015](./reports/report-save-20231027-2015)
3. on supprime l'intégralité du contenu des deux volumes.
[Report 20231027-2016](./reports/report-save-20231027-2016)
4. on programme le crontab de *backuprsync* et on joue avec les fichiers
des deux volumes (programmation du contrab toutes les minutes)
```txt
*/1 * * * * /home/backuprsync/bin/BackupNas
```
L'ensemble des éléments de ce test sont dans l'archive [cron-save](./reports/cron-save.tar.bz2).

5. restauration complète du NAS après désastre
[Report 10231027-2129](./reports/report-restore-20231027-2129)

## TEST RAID (à remettre au client)
Pour ce test on ne va pas passer par 4 chemins. On rend directement deux disques
défaillants et on regarde si le RAID fait encore son travail.

```bash
# Retrait /dev/sdb1
mdadm --manage /dev/md0 --fail /dev/sdb1 --remove /dev/sdb1
mdadm: set /dev/sdb1 faulty in /dev/md0
mdadm: hot removed /dev/sdb1 from /dev/md0

# Retrait /dev/sdc1
mdadm --manage /dev/md0 --fail /dev/sdc1 --remove /dev/sdc1
mdadm: set /dev/sdc1 faulty in /dev/md0
mdadm: hot removed /dev/sdc1 from /dev/md0

# Vérification
cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] [linear] [multipath] [raid0] [raid1] [raid10]
md0 : active raid6 sdf1[4] sdg1[5] sde1[3] sdd1[2] sdh1[6]
      15705600 blocks super 1.2 level 6, 512k chunk, algorithm 2 [7/5] [__UUUUU]

unused devices: <none>

lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                 8:0    0   10G  0 disk
├─sda1              8:1    0    9G  0 part  /
├─sda2              8:2    0    1K  0 part
└─sda5              8:5    0  975M  0 part  [SWAP]
sdb                 8:16   0    3G  0 disk
└─sdb1              8:17   0    3G  0 part
sdc                 8:32   0    3G  0 disk
└─sdc1              8:33   0    3G  0 part
sdd                 8:48   0    3G  0 disk
└─sdd1              8:49   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sde                 8:64   0    3G  0 disk
└─sde1              8:65   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdf                 8:80   0    3G  0 disk
└─sdf1              8:81   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdg                 8:96   0    3G  0 disk
└─sdg1              8:97   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdh                 8:112  0    3G  0 disk
└─sdh1              8:113  0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdi                 8:128  0    1G  0 disk
└─sdi1              8:129  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdj                 8:144  0    1G  0 disk
└─sdj1              8:145  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdk                 8:160  0    1G  0 disk
└─sdk1              8:161  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sr0                11:0    1 1024M  0 rom



# Remettre les deux disques
mdadm /dev/md0 --add /dev/sdb1
mdadm: added /dev/sdb1

mdadm /dev/md0 --add /dev/sdc1
mdadm: added /dev/sdc1

cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] [linear] [multipath] [raid0] [raid1] [raid10]
md0 : active raid6 sdc1[8] sdb1[7] sdf1[4] sdg1[5] sde1[3] sdd1[2] sdh1[6]
      15705600 blocks super 1.2 level 6, 512k chunk, algorithm 2 [7/7] [UUUUUUU]

unused devices: <none>


lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                 8:0    0   10G  0 disk
├─sda1              8:1    0    9G  0 part  /
├─sda2              8:2    0    1K  0 part
└─sda5              8:5    0  975M  0 part  [SWAP]
sdb                 8:16   0    3G  0 disk
└─sdb1              8:17   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdc                 8:32   0    3G  0 disk
└─sdc1              8:33   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdd                 8:48   0    3G  0 disk
└─sdd1              8:49   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sde                 8:64   0    3G  0 disk
└─sde1              8:65   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdf                 8:80   0    3G  0 disk
└─sdf1              8:81   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdg                 8:96   0    3G  0 disk
└─sdg1              8:97   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdh                 8:112  0    3G  0 disk
└─sdh1              8:113  0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdi                 8:128  0    1G  0 disk
└─sdi1              8:129  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdj                 8:144  0    1G  0 disk
└─sdj1              8:145  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdk                 8:160  0    1G  0 disk
└─sdk1              8:161  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sr0                11:0    1 1024M  0 rom
```

Encore un super JOB!!!

## TEST JBOD: ajout d'un disque (à remettre au client)

On ajoute un disque de 1Go que l'on va intégrer à notre tas de ... LVM

```bash
lsblk
NAME              MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINTS
sda                 8:0    0   10G  0 disk
├─sda1              8:1    0    9G  0 part  /
├─sda2              8:2    0    1K  0 part
└─sda5              8:5    0  975M  0 part  [SWAP]
sdb                 8:16   0    3G  0 disk
└─sdb1              8:17   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdc                 8:32   0    3G  0 disk
└─sdc1              8:33   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdd                 8:48   0    3G  0 disk
└─sdd1              8:49   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sde                 8:64   0    3G  0 disk
└─sde1              8:65   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdf                 8:80   0    3G  0 disk
└─sdf1              8:81   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdg                 8:96   0    3G  0 disk
└─sdg1              8:97   0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdh                 8:112  0    3G  0 disk
└─sdh1              8:113  0    3G  0 part
  └─md0             9:0    0   15G  0 raid6 /mnt/exports/raid
sdi                 8:128  0    1G  0 disk
└─sdi1              8:129  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdj                 8:144  0    1G  0 disk
└─sdj1              8:145  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdk                 8:160  0    1G  0 disk
└─sdk1              8:161  0 1023M  0 part
  └─vg_nas-lv_nas 253:0    0    3G  0 lvm   /mnt/exports/lvm
sdl                 8:176  0    1G  0 disk
sr0                11:0    1 1024M  0 rom


# On prépare le disque
sgdisk --new=1:0:0 --typecode=1:8e00 --change-name=1:"LVM" /dev/sdl
Creating new GPT entries in memory.
The operation has completed successfully.

partprobe
pvcreate /dev/sdl1
 Physical volume "/dev/sdl1" successfully created.

vgextend vg_nas /dev/sdl1
  Volume group "vg_nas" successfully extended

lvextend -l +100%FREE /dev/vg_nas/lv_nas
  Size of logical volume vg_nas/lv_nas changed from <2,99 GiB (765 extents) to 3,98 GiB (1020 extents).
  Logical volume vg_nas/lv_nas successfully resized.

xfs_growfs -m 100%/dev/vg_nas/lv_nas
meta-data=/dev/mapper/vg_nas-lv_nas isize=512    agcount=4, agsize=195840 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=783360, imaxpct=100
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 783360 to 1044480

df -h
Sys. de fichiers          Taille Utilisé Dispo Uti% Monté sur
udev                        1,9G       0  1,9G   0% /dev
tmpfs                       392M    2,2M  390M   1% /run
/dev/sda1                   8,9G    1,6G  6,8G  19% /
tmpfs                       2,0G       0  2,0G   0% /dev/shm
tmpfs                       5,0M       0  5,0M   0% /run/lock
/dev/mapper/vg_nas-lv_nas   4,0G     62M  3,9G   2% /mnt/exports/lvm
/dev/md0                     15G    141M   15G   1% /mnt/exports/raid
tmpfs                       392M       0  392M   0% /run/user/1000
```

Good JOB!!! 1Go de plus pour notre JBOD. Le client va vraiment être très content
de notre petite entreprise.

## TEST NFS / SAMBA / CIFS (à remettre au client)
### Méthodologie employée
Pour ce dernier test:
- mettre un interpréteur shell aux utilisateurs clirrtiry(Admin) et jeddie(User);
- créer deux répertoires que root monte avec les protocoles CIFS / NFS:
    - /mnt/cifs (option de montage: mount -t cifs -o username=jeddie,password=abcd,gid=10001,file_mode=2664,dir_mode=2775 "\\\192.168.56.101\raid" /mnt/cifs )
    - /mnt/nfs (option de montage: mount -t nfs 192.168.56.101:/mnt/exports/raid )
- root prendra les rôles alternativement de jeddie et clirrtiry pour faire les tests sur un le serveur.

### Compte rendu pour la gestion des droits
Le compte rendu pour la gestion des droits est directement fait dans la vidéo mise à disposition pour le client.
[Test: Gestion des droits utilisateurs](./reports/test-droits-users.mp4 "Droits utilisateur")


## En résumé
Un travail amusant. Le plus long restant le faire de faire ce foutu compte-rendu...
Merci d'avoir eu la patience et surtout pris le temps de lire ma prose pas toujours très à propos!!!
