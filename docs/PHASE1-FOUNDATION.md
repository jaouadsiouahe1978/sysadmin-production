# Projet SysAdmin en Production - Phase 1 - Fondations

## 1. Audit initial du serveur

➡️ **Pourquoi c'est important**
- Connaître son environnement c'est la base !
- Permet d'identifier le matériel, les ressources dispo
- Anticipe les éventuels problèmes (disques trop petits, RAM insuffisante...)

🏗️ **Étapes clés**
1. `lshw` - liste le matériel (CPU, RAM, disques, BIOS...)
2. `lsblk` - liste les disques et partitions
3. `df -h` - espace disque utilisé/libre
4. `free -m` - RAM utilisée/libre
5. `ip a` - config réseau (IPs, interfaces...)
6. `sudo lshw -C network` - détails des cartes réseau

⚠️ **Pièges à éviter**
- Ne pas vérifier la compatibilité Linux du matériel
- Ne pas anticiper les besoins en ressources (ex : prévoir assez d'espace disque)

## 2. Mise à jour complète du système

➡️ **Pourquoi c'est crucial en prod**
- Corrige les failles de sécurité
- Apporte de nouvelles fonctionnalités, optimisations
- Assure la compatibilité avec les nouvelles applis
- Un système non patché = porte ouverte aux hackers !

🏗️ **Commandes clés**
1. `sudo apt update` - met à jour la liste des paquets dispo
2. `sudo apt upgrade` - installe les nouvelles versions
3. `sudo apt autoremove` - supprime les paquets inutilisés
4. `sudo reboot` - redémarre si noyau mis à jour

⚠️ **Bonnes pratiques**
- Le faire régulièrement (au moins tous les mois)
- Tester d'abord sur environnement de dev/test
- Monitorer si pas de régression après update

## 3. Génération de clés SSH sécurisées

➡️ **Pourquoi c'est indispensable**
- Mot de passe = faible (brute force, keyloggers...)
- Clé SSH = très forte, quasi impossible à casser
- Permet l'automatisation des tâches (connexions sans saisie)

🏗️ **Étapes de génération**
1. `ssh-keygen -t rsa -b 4096` - génère clé RSA 4096 bits
2. Passphrase forte pour sécuriser la clé privée
3. Copier `id_rsa.pub` sur serveur dans `~/.ssh/authorized_keys`

⚠️ **Règles d'or**
- Taille minimum 2048 bits (4096 recommandé)
- Passphrase d'au moins 20 caractères (phrases > mots)
- Ne JAMAIS partager sa clé privée
- Désactiver la connexion par mot de passe

## 4. Hardening SSH

➡️ **Les enjeux**
- SSH est LA porte d'entrée du serveur
- Doit être verrouillée au maximum car cible N°1 des hackers
- Un serveur exposé avec SSH mal conçu = pwned en quelques minutes

🏗️ **Mesures de durcissement**
1. Changer le port par défaut (22 -> 2222)
2. Désactiver l'accès root
3. Limiter les users pouvant se connecter
4. Mettre en place une Whitelist par IP
5. Configurer le timeout des sessions inactives 
6. Utiliser une version à jour d'OpenSSH (8.2+)

⚠️ **Ne jamais faire**
- Laisser le port 22 par défaut
- Autoriser l'accès root direct
- Ouvrir à tout le monde

## 5. Fail2ban

➡️ **Concept**
- Protège contre les attaques par brute force
- Bloque une IP après X tentatives infructueuses
- Un essentiel absolu sur tout serveur exposé

🏗️ **Mise en place**
1. `sudo apt install fail2ban` 
2. Copier `jail.conf` en `jail.local`
3. Adapter la section `[sshd]` :
```
   [sshd]
   enabled = true
   port = 2222
   logpath = /var/log/auth.log
   maxretry = 3
   findtime = 600
   bantime = 3600
```

⚠️ **Points clés** 
- Ne JAMAIS utiliser les valeurs par défaut
- Bien surveiller les logs auth.log et fail2ban.log

## 6. Firewall UFW

➡️ **Rôle essentiel**
- Un serveur sans firewall = comme une maison sans porte 
- Bloque tout le trafic indésirable entrant et sortant
- Agit comme un gardien qui ne laisse passer que le nécessaire

🏗️ **Configuration de base**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing  
sudo ufw allow 2222/tcp   # SSH
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS
sudo ufw enable
```

⚠️ **Erreurs à ne pas commettre**
- Oublier d'ouvrir le port SSH avant d'activer le firewall (= locked out)
- Laisser des ports non nécessaires ouverts
- Ne pas surveiller régulièrement les logs UFW

## 7. Configuration de base de LVM

➡️ **Avantages de LVM**
- Gestion souple des disques/partitions (redimensionner à chaud)
- Snapshots (sauvegarde instantanée)
- Stripping/Mirroring (RAID)
- Priorité sur un serveur de prod !

🏗️ **Création de volumes logiques**
```bash
pvcreate /dev/sdb
vgcreate vg_data /dev/sdb
lvcreate -L 20G -n lv_appli vg_data
mkfs.ext4 /dev/vg_data/lv_appli
mkdir /appli
echo '/dev/vg_data/lv_appli /appli ext4 defaults 0 0' >> /etc/fstab
mount -a
```

⚠️ **À savoir**
- Prévoir des volumes dédiés pour différents usages (/appli, /data, /backup...)
- Bien dimensionner au départ, on peut agrandir mais réduire c'est très délicat
- Penser aux snapshots avant toute modif importante

## 8. Serveur Nextcloud en production

➡️ **Une application concrète**
- Déployer un vrai service Nextcloud utilisable en prod
- Accessible depuis Internet de façon sécurisée
- Base de données, cache Redis, stack Docker complète
- Aperçu des tâches classiques d'un SysAdmin

🏗️ **Installation via Docker Compose**
```bash
apt install -y docker.io docker-compose
systemctl enable docker
mkdir -p /docker/nextcloud/  
cd /docker/nextcloud/
wget https://raw.githubusercontent.com/nextcloud/docker/master/docker-compose.yml
docker-compose up -d
```

⚠️ **Passage en prod**
- Reverse proxy HTTPS avec Nginx et Let's Encrypt
- Nom de domaine dédié 
- Sécuriser la configuration Nextcloud
- Tests de montée en charge

## En résumé

On a vu dans cette phase les bases indispensables pour tout serveur :
- ✅ Faire un audit complet
- ✅ Mettre à jour et sécuriser
- ✅ Configurer les accès (SSH) et gérer les clés  
- ✅ Mettre en place un firewall strict
- ✅ Installer un service anti-bruteforce
- ✅ Utiliser LVM pour la souplesse
- ✅ Déployer une appli réelle sécurisée

Ces étapes constituent le minimum vital pour tout serveur de prod Linux.

La prochaine étape va être de monter en compétence en se mettant dans la peau d'un SysAdmin en condition réelle !

**Let's go pour la Phase 2 ! 💪**
