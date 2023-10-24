# DNS-Data-NAS-Service
Mise en place d'un serveur NAS / Backup

## Décryptage du message subliminal
**NAS is not a backup !**
Je vous rejoins complètement dans cet analyse!!!

*Je me demande du coup pourquoi je fais se travaille...
Bon pour ce code au départ je suis parti un peu loin en pensant à un Vigenère...
Mais non juste un XOR suffisait.*

```python
from operator import xor

l=['16','0E','01','78','26','21','78','21','3D','2C','6F','33','78','2D','33','3B','24','27','28','6F','73']         47 key='XOR'*7
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
    - rsync pour la synchronisation des sauvegardes

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

L'installation de la première machine étant terminée on ajoute l'ensembles
des disques:
- 7 * 3Go pour le RAID 6
- 3 * 1Go pour le LVM (en standbye)

![ConfigNasVB](./pictures/ConfigNasVB.jpg "Configuration de la machine virtuelle NAS")

Une fois terminer l'installation de cette première machine, on la clone intégralement
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
Super une bonne chose de faite.
