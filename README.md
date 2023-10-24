# DNS-Data-NAS-Service
Mise en place d'un serveur NAS / Backup

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
    - rsync pour la synchronisation des sauvegardes

## Choix pour la simulation:
- Environnement: VirtualBox 7.0.12 + Extension Pack
- Debian 12.2 x64
- Hardware:
    - vCPU: 1
    - RAM: 4096 Mo
    - Disque système: 10 Gio (VDI)
- Paquets installés: utilitaires usuels du système
- Pas d'utilisateur root
- Un utilisateur -> Kaman kaman G:kaman,sudo

L'installation de la première machine étant terminée on ajoute l'ensembles
des disques:
- 7 * 3Go pour le RAID 6
- 3 * 1Go pour le LVM (en standbye)

![ConfigNasVB](./pictures/ConfigNasVB.jpg "Configuration de la machine virtuelle NAS")

Une fois terminer l'installation de cette première machine, on la clone intégralement
en veillant à changer les adresses MAC et en conserver le nom et les UUIDs des disques.


