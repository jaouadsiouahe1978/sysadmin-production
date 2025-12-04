# Phase 1 - Semaine 1 : Hardening & Sécurisation SSH

## Ce qu'on va faire cette semaine

Sécuriser ton serveur. Imagine que ton serveur c'est une maison :
- Tu changes la serrure de la porte (SSH)
- Tu mets une alarme (Firewall)
- Tu engages un gardien (Fail2ban)
- Tu installes des caméras (Audit logging)

Sans ça, n'importe qui sur Internet peut essayer de rentrer dans ta maison.

---

## Jour 1-2 : Mises à Jour Système

### Pourquoi c'est important ?

Les mises à jour = patchs de sécurité. Sans elles, les hackers utilisent des failles connues pour entrer.

**Analogie réelle** : C'est comme réparer les trous d'une clôture. Si tu laisses les trous, les voleurs passent directement.

### Les Commandes

```bash
# 1. Récupère la liste des mises à jour disponibles
sudo apt update

# 2. Installe toutes les mises à jour
sudo apt upgrade -y

# 3. Installe les outils qu'on va utiliser
sudo apt install -y lvm2 mdadm fail2ban auditd openssh-server openssh-client
```

### Explication de chaque ligne

**`sudo apt update`**
- `sudo` = "fais ça en tant que super-utilisateur"
- `apt` = gestionnaire de packages (comme un app store Linux)
- `update` = récupère la liste des mises à jour (pas d'installation encore)

**❌ Erreur à ne pas faire**
```bash
sudo apt upgrade -y  # Sans update d'abord
```
Si tu fais `upgrade` sans `update` en premier, tu upgrades les vieilles versions. C'est comme essayer de mettre à jour une app sans vérifier quelle version existe.

**`sudo apt upgrade -y`**
- `upgrade` = installe les mises à jour
- `-y` = "oui à toutes les questions" (sinon il te demande pour chaque)

**💡 Conseil production**
Sur un serveur de production RÉEL, ne mets pas `-y` automatiquement. Regarde d'abord :
```bash
sudo apt upgrade    # Sans -y, il te montre ce qui va changer
# Tu vérifies, puis tu approuves
```

**`sudo apt install -y ...`**
Installe les packages qu'on va utiliser :
- `lvm2` = gestion de volumes (on en aura besoin pour le storage)
- `mdadm` = gestion RAID (miroir de disques)
- `fail2ban` = gardien qui bloque les tentatives d'accès échouées
- `auditd` = caméra qui enregistre tout ce qui se passe

---

## Jour 3 : SSH Hardening (La partie IMPORTANTE)

### Pourquoi SSH est critique

SSH = ta porte pour accéder au serveur. Si elle est mal sécurisée, les hackers rentrent. C'est le port #1 qu'on attaque.

**Analogie** : SSH c'est comme la porte d'entrée de ta maison. La serrure doit être bonne, sinon n'importe qui rentre.

### Étape 1 : Créer des clés SSH (sur ton PC)

**Sur TON PC LOCAL** (pas le serveur) :

```bash
# Génère une paire de clés RSA 4096 bits
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_sysadmin -N "MonMotDePasse123!"
```

**Explication**
- `ssh-keygen` = outil qui crée les clés
- `-t rsa` = type de chiffrement (RSA c'est solide)
- `-b 4096` = 4096 bits (très long, très sûr)
- `-f ~/.ssh/id_rsa_sysadmin` = fichier où sauvegarder
- `-N "MonMotDePasse123!"` = mot de passe pour la clé

**Ce que tu vas obtenir**
```
id_rsa_sysadmin      ← Clé PRIVÉE (à garder secret)
id_rsa_sysadmin.pub  ← Clé PUBLIQUE (on met sur le serveur)
```

**❌ Erreur production classique**
```bash
ssh-keygen -t rsa -b 1024  # Trop court ! Hackable
ssh-keygen -t rsa          # Défaut 2048 bits, c'est juste ok
```

**Utilise minimum 4096 bits en production.**

**💡 Conseil**
Choisis un mot de passe FORT pour ta clé :
- Minimum 12 caractères
- Majuscules + minuscules + chiffres + symboles
- Exemple : `MyServer!2025@Grenoble123`

### Étape 2 : Copier la clé publique sur le serveur

**Sur ton PC** :

```bash
# Copie ta clé publique sur le serveur
cat ~/.ssh/id_rsa_sysadmin.pub | ssh root@192.168.1.135 \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**Explication**
- `cat` = affiche le contenu
- `ssh` = envoie à distance
- `mkdir -p ~/.ssh` = crée le dossier s'il existe pas
- `cat >> ~/.ssh/authorized_keys` = ajoute la clé à la liste des clés autorisées
- `chmod 600` = permissions strictes (seulement root peut lire)

**Quoi faire**
Il va te demander le password root du serveur. Tape-le.

**❌ Erreur à ne pas faire**
```bash
chmod 777 ~/.ssh/authorized_keys  # N'IMPORTE QUI peut modifier !
```
Si les permissions sont loose, n'importe qui peut ajouter sa clé. C'est une grosse faille de sécurité.

**Bon** :
```bash
chmod 600 ~/.ssh/authorized_keys  # Seulement toi
```

### Étape 3 : Modifier SSH Config (port 2222)

**Sur le serveur** :

```bash
# Édite le fichier de config SSH
sudo nano /etc/ssh/sshd_config

# Cherche la ligne "#Port 22" et change-la en :
Port 2222

# Aussi ajoute (cherche ou ajoute) :
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password

# Sauvegarde : Ctrl+O, Enter, Ctrl+X
```

**Explication de chaque changement**

| Paramètre | Ancien | Nouveau | Pourquoi |
|-----------|--------|---------|---------|
| `Port` | 22 | 2222 | Port 22 c'est le premier qu'on attaque. 2222 c'est moins connu |
| `PasswordAuthentication` | yes | no | On utilise QUE les clés, pas de password |
| `PubkeyAuthentication` | yes | yes | Utiliser les clés publiques |
| `PermitRootLogin` | yes | prohibit-password | Pas de login root direct + pas de password |

**❌ Erreur classique**
```bash
Port 2222
PasswordAuthentication no
# Et tu oublies de copier ta clé avant ↑
# Résultat : LOCKED OUT ! Tu peux plus accéder au serveur !
```

**Ordre CRITIQUE** :
1. D'abord, copier la clé
2. Puis changer la config
3. Puis tester

### Étape 4 : Vérifier la config et redémarrer SSH

**Sur le serveur** :

```bash
# Vérifie qu'il n'y a pas d'erreur de syntaxe
sudo sshd -t

# Si OK, redémarre SSH
sudo systemctl restart ssh

# Teste la connexion AVANT de fermer
# Sur ton PC, OUVRE UNE NOUVELLE TERMINAL :
ssh -i ~/.ssh/id_rsa_sysadmin -p 2222 root@192.168.1.135
```

**❌ ATTENTION : Erreur de débutant**

```bash
sudo systemctl restart ssh
# Et tu fermes la connexion immédiatement
```

Si tu fermes avant de tester, tu seras peut-être locked out.

**Bon processus** :
```bash
sudo systemctl restart ssh          # Redémarre
# Reste connecté dans cette session
# Ouvre UNE AUTRE SESSION pour tester
ssh -i ~/.ssh/id_rsa_sysadmin -p 2222 root@192.168.1.135
# Si ça marche → OK, ferme l'ancienne
```

---

## Jour 4 : Firewall UFW

### Pourquoi un firewall ?

Sans firewall = tout est ouvert par défaut.
Avec firewall = tout est fermé, tu ouvres QUE ce que tu veux.

**Analogie** : C'est comme une porte avec un portier. Sans portier, n'importe qui rentre. Avec, il dit "non" à tout le monde sauf les VIP.

### Les Commandes

```bash
# 1. Refuse TOUT ce qui entre
sudo ufw default deny incoming

# 2. Autorise ce qui sort
sudo ufw default allow outgoing

# 3. IMPORTANT : Autorise SSH AVANT d'activer UFW
sudo ufw allow 2222/tcp

# 4. Autorise HTTP et HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 5. Autorise DNS
sudo ufw allow 53/udp
sudo ufw allow 53/tcp

# 6. Vérifie les règles AVANT d'activer
sudo ufw status verbose

# 7. Active le firewall
sudo ufw enable

# 8. Vérifie que c'est bien activé
sudo ufw status
```

**⚠️ ERREUR CRITIQUE EN PRODUCTION**

```bash
sudo ufw enable              # Active UFW
# Oups, j'ai oublié d'autoriser SSH
# LOCKED OUT ! Plus d'accès !
```

**Solution production** : Si tu te locked out, redémarre le serveur. UFW se désactive temporairement pendant le boot.

**Explications des règles**

| Règle | Pourquoi |
|-------|---------|
| `default deny incoming` | Refuse tout par défaut, c'est la base de la sécurité |
| `default allow outgoing` | Laisse le serveur faire des requêtes (apt, DNS, etc) |
| `allow 2222/tcp` | SSH, ta porte d'accès |
| `allow 80/tcp` | HTTP, pour le site web |
| `allow 443/tcp` | HTTPS, site web sécurisé |
| `allow 53/udp` + `tcp` | DNS, pour résoudre les noms |

---

## Jour 5 : Fail2ban (Gardien)

### Pourquoi Fail2ban ?

Des bots essaient des milliers de passwords par jour pour rentrer.
Fail2ban dit : "3 fois faux, t'es banni 1 heure".

**Analogie** : C'est comme une serrure à 3 tentatives. Après 3 faux codes, la porte se bloque 1h.

### Les Commandes

```bash
# 1. Installe fail2ban
sudo apt install -y fail2ban

# 2. Crée une config locale
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# 3. Édite la config pour SSH
sudo nano /etc/fail2ban/jail.local

# Cherche [sshd] et configure :
[sshd]
enabled = true
port = 2222                # Port où SSH écoute
logpath = /var/log/auth.log
maxretry = 3               # Max 3 tentatives
findtime = 600             # Dans cette fenêtre de 10 min
bantime = 3600             # Ban pendant 1h

# 4. Redémarre fail2ban
sudo systemctl restart fail2ban

# 5. Vérifie
sudo systemctl status fail2ban
```

**Explication des paramètres**

```
maxretry = 3      → 3 mauvais passwords
findtime = 600    → Dans 10 minutes (600 secondes)
bantime = 3600    → Ban pendant 3600 secondes = 1 heure
```

**Donc** : Si quelqu'un se trompe 3 fois en 10 minutes, il est banni 1h.

**❌ Erreur débutant**
```bash
maxretry = 100    # Trop tolérant ! Les bots ont le temps
```

**Bon**
```bash
maxretry = 3      # Strict, on bloque vite
```

---

## Jour 6 : Auditd (Caméra de sécurité)

### Pourquoi auditer ?

Si quelqu'un rentre dans le serveur, tu veux savoir QUOI il a fait.

**Analogie** : C'est comme des caméras de sécurité. Si quelqu'un rentre, on a des preuves.

### Les Commandes

```bash
# 1. Install auditd
sudo apt install -y auditd

# 2. Start et enable
sudo systemctl start auditd
sudo systemctl enable auditd

# 3. Ajoute des règles
sudo nano /etc/audit/rules.d/audit.rules

# Ajoute à la fin :
-w /etc/ssh/sshd_config -p wa -k ssh_config_changes
-w /var/log/auth.log -p wa -k auth_logs
-a exit,always -F arch=b64 -S execve -F uid=0 -k root_commands

# 4. Redémarre
sudo systemctl restart auditd

# 5. Vérifie
sudo ausearch -k ssh_config_changes
```

**Explication des règles**

```
-w /etc/ssh/sshd_config          → Surveille ce fichier
-p wa                             → write (w) et attribute change (a)
-k ssh_config_changes             → Clé pour identifier cette règle
```

Ça veut dire : "Si quelqu'un modifie la config SSH, enregistre-le".

---

## Jour 7 : DNS Local (Dnsmasq)

### Pourquoi DNS local ?

DNS = traduit "google.com" en adresse IP.
Avoir ton DNS local = répondre rapidement LOCALEMENT.

**Analogie** : C'est comme un annuaire téléphonique. Au lieu de chercher dans un grand annuaire, tu as une version locale.

### Les Commandes

```bash
# 1. Install dnsmasq
sudo apt install -y dnsmasq

# 2. Edit config
sudo nano /etc/dnsmasq.conf

# Ajoute à la fin :
listen-address=127.0.0.1,192.168.1.135
server=8.8.8.8
server=8.8.4.4
address=/sysadmin.local/192.168.1.135
address=/nextcloud.local/192.168.1.135
address=/gitea.local/192.168.1.135

# 3. Redémarre
sudo systemctl restart dnsmasq

# 4. Teste
nslookup sysadmin.local 192.168.1.135
```

**Explication**

```
listen-address=192.168.1.135   → Écoute sur cette IP locale
server=8.8.8.8                 → Utilise Google comme serveur upstream
address=/sysadmin.local/...    → Crée un enregistrement local
```

Maintenant, si tu fais :
```bash
ping sysadmin.local
```
Ça répond 192.168.1.135 au lieu de chercher sur Internet.

---

## ✅ Checkpoint Semaine 1

**Vérifie tout fonctionne** :

```bash
# SSH sur port 2222 ?
ssh -i ~/.ssh/id_rsa_sysadmin -p 2222 root@192.168.1.135 "echo SSH OK"

# Firewall actif ?
sudo ufw status

# Fail2ban tourne ?
sudo systemctl status fail2ban

# Auditd tourne ?
sudo systemctl status auditd

# DNS répond ?
nslookup sysadmin.local 192.168.1.135
```

Si tout dit OK → **SEMAINE 1 COMPLÉTÉE** ✅

---

## 💡 Résumé : Ce que tu as appris

1. **SSH Hardening** : Clés fortes, port custom, pas de passwords
2. **Firewall** : Deny incoming, allow spécifique
3. **Fail2ban** : Protéger contre brute-force
4. **Audit** : Enregistrer les changements
5. **DNS Local** : Résoudre localement

C'est les **FONDATIONS** de tout serveur sécurisé. Sans ça, n'importe quel hacker rentre.

---

**Prochaine semaine** : RAID + LVM (Storage)
