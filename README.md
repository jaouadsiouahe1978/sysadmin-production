# Mon Laboratoire SysAdmin - Production Ready

## C'est quoi ce projet ?

Je construis une vraie infrastructure de production sur mon serveur personnel. Pas un jouet, du vrai travail de SysAdmin.

L'objectif : devenir **SysAdmin confirmé** en 2 ans en gérant une infrastructure réelle.

---

## 🖥️ Le Serveur
```
Marque      : Dell PowerEdge T310
CPU         : Intel Xeon X3440 (8 cores, 2.53 GHz)
RAM         : 19 Go
Disques     : 4 x 300 Go
OS          : Ubuntu 24.04 LTS
Réseau      : Dual NIC (192.168.1.135 + 192.168.1.136)
Localisation: Grenoble, France
```

---

## 📊 État Actuel

**Phase 1 : Foundation & Hardening** ✅ COMPLÉTÉE
- Sécurisation SSH (port 2222, clés RSA 4096)
- Firewall UFW configuré
- Fail2ban contre les attaques brute-force
- Audit logging (auditd)
- Kernel hardening
- 4 users créés (admin, docker_user, monitoring, backup_user)
- DNS local (dnsmasq)

**Phase 2 : Storage** ✅ COMPLÉTÉE
- RAID 1 sur 2 disques (miroir 278 GB)
- LVM avec 3 volumes logiques :
  - `/data` : 200 GB (données applicatives)
  - `/backups` : 50 GB (sauvegardes)
  - `/docker` : 28 GB (volumes Docker)

**Phase 3 : Applications** ⏳ EN COURS
- Docker installé
- Prêt pour déployer les services

---

## 🔒 Sécurité en Place

✓ SSH sur port 2222 (non-standard)
✓ Clés RSA 4096 bits
✓ Firewall UFW (deny incoming, allow spécifique)
✓ Fail2ban (protection SSH)
✓ Audit logging de tous les changements
✓ Kernel hardening appliqué
✓ Users avec permissions correctes

---

## 💾 Storage

### RAID 1 (Miroir)
- 2 disques en miroir (sdc + sdd)
- Protection contre la perte d'un disque
- ~63 minutes de sync au démarrage

### LVM (Volumes Logiques)
- Flexible : redimensionner sans reboot
- 3 volumes séparés pour différents usages
- Snapshots possibles pour backups

### Exemple : Comment ça marche ?
```
2 disques physiques
    ↓
RAID 1 (miroir automatique)
    ↓
LVM (découpe logique)
    ↓
3 volumes montés (/data, /backups, /docker)
```

Si un disque tombe en panne, RAID continue avec l'autre. Zéro perte de données.

---

## 📁 Structure du Projet
```
sysadmin-production/
├── README.md                          ← Toi es ici
├── docs/
│   ├── PHASE1-SEMAINE1.md            ← Hardening & SSH
│   └── PHASE1-SEMAINE2.md            ← RAID & LVM
├── logs/
│   ├── audit_initial.txt             ← État initial du serveur
│   └── audit_final.txt               ← État après Phase 1
└── troubleshooting/
    ├── ssh-socket-port-2222.md       ← Galère SSH
    ├── dnsmasq-port-53-conflict.md   ← Galère DNS
    └── lvm-snapshot-no-space.md      ← Galère LVM
```

---

## 🛠️ Outils Utilisés

| Outil | Rôle |
|-------|------|
| **SSH** | Accès sécurisé au serveur |
| **UFW** | Firewall simple |
| **Fail2ban** | Bloquer les attaques brute-force |
| **Auditd** | Logger tous les changements |
| **RAID** | Miroir des disques (protection) |
| **LVM** | Gestion volumes flexibles |
| **Dnsmasq** | DNS local |
| **Docker** | Containers pour les apps |

---

## 📈 Prochaines Étapes

**Phase 3** : Déployer des applications
- Nginx reverse proxy
- Nextcloud (stockage)
- Gitea (git personnel)
- PostgreSQL (database)

**Phase 4** : Monitoring
- Prometheus (métriques)
- Grafana (dashboards)
- AlertManager (alertes)
- Loki (logs centralisés)

**Phase 5** : Backups & Disaster Recovery
- Restic (backup automatique)
- S3/MinIO (stockage)
- Tests de restauration réguliers

---

## 📚 Ce que j'ai Appris

✅ Partitioning & RAID
✅ LVM (Logical Volume Manager)
✅ Firewall & Security
✅ SSH hardening
✅ User & permission management
✅ Logging & auditing
✅ Linux system administration
✅ Debugging & troubleshooting

---

## 🎯 Objectif Final

À la fin de 2 ans, avoir :
- Une infra stable 24/7
- Automatisation complète
- Monitoring et alertes
- Backups testés régulièrement
- Documentation professionnelle
- **Le titre de SysAdmin confirmé** 💪

---

## 📞 Contact & Questions

C'est un **projet d'apprentissage** personnel. N'hésite pas à fork et adapter pour ton lab !

---

**Dernière mise à jour** : 4 Décembre 2025, Grenoble 🇫🇷

**Status** : Phase 1 ✅ | Phase 2 ✅ | Phase 3 ⏳ | Phase 4 ⏳ | Phase 5 ⏳
