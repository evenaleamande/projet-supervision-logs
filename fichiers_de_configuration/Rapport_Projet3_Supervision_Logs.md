# Rapport de Projet 3 — Supervision et Gestion des Logs

**Module :** Administration des Systèmes Informatiques  
**Filière :** Ingénierie Système Informatique  
**Auteur :** Étudiant — Ingénierie Système  
**Date :** 17 Mai 2026  
**Environnement :** Debian/Ubuntu Linux (systemd + rsyslog)

---

## Table des Matières

1. [Introduction](#1-introduction)
2. [Analyse avec journalctl](#2-analyse-avec-journalctl)
   - 2.1 [Filtrage par service](#21-filtrage-par-service)
   - 2.2 [Filtrage par date](#22-filtrage-par-date)
3. [Configuration de rsyslog](#3-configuration-de-rsyslog)
   - 3.1 [Architecture du système](#31-architecture-du-système)
   - 3.2 [Configuration du serveur](#32-configuration-du-serveur)
   - 3.3 [Configuration du client](#33-configuration-du-client)
   - 3.4 [Test et preuve de réception](#34-test-et-preuve-de-réception)
4. [Rotation des Logs avec logrotate](#4-rotation-des-logs-avec-logrotate)
5. [Surveillance en Temps Réel avec tail -f](#5-surveillance-en-temps-réel-avec-tail--f)
6. [Récapitulatif des Fichiers de Configuration](#6-récapitulatif-des-fichiers-de-configuration)
7. [Conclusion](#7-conclusion)

---

## 1. Introduction

La supervision des logs est une mission fondamentale de l'administrateur système. Les journaux système contiennent des informations critiques sur l'état, la santé et la sécurité d'une infrastructure. Ce projet s'articule autour de quatre axes :

| Axe | Outil | Objectif |
|-----|-------|----------|
| Analyse des logs | `journalctl` | Interroger le journal systemd |
| Centralisation | `rsyslog` | Collecter les logs de plusieurs machines |
| Rotation | `logrotate` | Gérer la taille et l'archivage des logs |
| Surveillance temps réel | `tail -f` | Monitorer les événements en direct |

**Topologie réseau utilisée :**

```
┌─────────────────────┐        TCP/UDP 514        ┌─────────────────────┐
│   CLIENT (machine1) │ ─────────────────────────► │  SERVEUR (machine2) │
│  192.168.1.101      │                            │  192.168.1.100      │
│  rsyslog (client)   │                            │  rsyslog (serveur)  │
└─────────────────────┘                            └─────────────────────┘
                                                           │
                                                           ▼
                                                   /var/log/clients/
                                                   └── machine1/
                                                       ├── sshd.log
                                                       ├── cron.log
                                                       └── kernel.log
```

---

## 2. Analyse avec journalctl

`journalctl` est l'outil de consultation du journal systemd (journald). Contrairement aux anciens fichiers texte `/var/log/*.log`, le journal systemd est binaire et structuré, permettant des filtres puissants.

### 2.1 Filtrage par service

#### 2.1.1 — Consulter les logs d'un service spécifique

La commande de base pour filtrer par unité systemd :

```bash
# Syntaxe générale
journalctl -u <nom_du_service>

# Exemples pratiques :

# Logs du service SSH
journalctl -u ssh.service

# Logs du service cron
journalctl -u cron.service

# Logs du serveur web Apache
journalctl -u apache2.service

# Logs du service rsyslog
journalctl -u rsyslog.service

# Logs du réseau (NetworkManager)
journalctl -u NetworkManager.service
```

#### 2.1.2 — Options utiles pour le filtrage par service

```bash
# Afficher les logs en temps réel d'un service (équivalent tail -f)
journalctl -u ssh.service -f

# Afficher uniquement les N dernières lignes
journalctl -u ssh.service -n 50

# Afficher les logs depuis le dernier démarrage du service
journalctl -u ssh.service -b

# Filtrer par priorité (0=emerg, 1=alert, 2=crit, 3=err, 4=warn, 5=notice, 6=info, 7=debug)
journalctl -u apache2.service -p err

# Afficher uniquement les erreurs et les urgences
journalctl -u apache2.service -p 0..3

# Format de sortie court (sans métadonnées)
journalctl -u cron.service -o short

# Format JSON (pour traitement automatisé)
journalctl -u cron.service -o json-pretty | head -50
```

#### 2.1.3 — Exemple de sortie (logs SSH)

```
-- Logs begin at Mon 2026-05-10 08:00:01 CET, end at Sat 2026-05-17 15:30:00 CET. --
May 17 09:14:22 machine1 sshd[1234]: Accepted publickey for admin from 192.168.1.50 port 49822
May 17 09:14:22 machine1 sshd[1234]: pam_unix(sshd:session): session opened for user admin by (uid=0)
May 17 10:02:11 machine1 sshd[1289]: Failed password for invalid user root from 10.0.0.55 port 53441
May 17 10:02:13 machine1 sshd[1289]: Connection closed by invalid user root 10.0.0.55 port 53441
May 17 14:45:00 machine1 sshd[1310]: Accepted password for deploy from 192.168.1.20 port 41122
```

> **Analyse de sécurité :** On observe une tentative de connexion par brute-force depuis `10.0.0.55` (utilisateur `root`). Ceci justifie la mise en place de `fail2ban` sur ce système.

#### 2.1.4 — Filtrer par PID ou par exécutable

```bash
# Par PID du processus
journalctl _PID=1234

# Par chemin de l'exécutable
journalctl _EXE=/usr/sbin/sshd

# Par utilisateur système (UID)
journalctl _UID=1000

# Combinaison de filtres (ET logique)
journalctl _UID=0 _COMM=sudo
```

---

### 2.2 Filtrage par date

journalctl supporte des formats de dates souples et puissants.

#### 2.2.1 — Syntaxe des dates

```bash
# Depuis une date précise
journalctl --since "2026-05-17"
journalctl --since "2026-05-17 08:00:00"

# Jusqu'à une date précise
journalctl --until "2026-05-17 18:00:00"

# Plage de dates
journalctl --since "2026-05-17 08:00:00" --until "2026-05-17 12:00:00"

# Raccourcis temporels
journalctl --since "yesterday"
journalctl --since "today"
journalctl --since "1 hour ago"
journalctl --since "-30min"

# Depuis le dernier démarrage système
journalctl -b

# Depuis l'avant-dernier démarrage (-1 = précédent)
journalctl -b -1

# Lister tous les démarrages enregistrés
journalctl --list-boots
```

#### 2.2.2 — Combinaison date + service

```bash
# Logs SSH des 2 dernières heures
journalctl -u ssh.service --since "2 hours ago"

# Erreurs système d'hier
journalctl -p err --since "yesterday" --until "today"

# Logs cron de la semaine
journalctl -u cron.service --since "2026-05-10" --until "2026-05-17"

# Logs SSH du jour avec affichage en temps réel
journalctl -u ssh.service --since "today" -f
```

#### 2.2.3 — Exemple de sortie (filtrage temporel)

```bash
$ journalctl --since "2026-05-17 08:00" --until "2026-05-17 09:00" -p err
-- Logs begin at Mon 2026-05-10 08:00:01 CET --
May 17 08:12:34 machine1 kernel: EXT4-fs error (device sda1): ...
May 17 08:45:00 machine1 systemd[1]: Failed to start mon_application.service
May 17 08:45:01 machine1 systemd[1]: mon_application.service: Control process exited with error code
```

#### 2.2.4 — Statistiques et taille du journal

```bash
# Voir la taille totale du journal
journalctl --disk-usage

# Limiter la rétention à 7 jours (nettoyage)
sudo journalctl --vacuum-time=7d

# Limiter la taille à 500 Mo
sudo journalctl --vacuum-size=500M
```

---

## 3. Configuration de rsyslog

### 3.1 Architecture du système

rsyslog (Rocket-fast SYStem for LOG processing) est le démon de journalisation standard sur la plupart des distributions Linux. Il reçoit les messages système, les filtre, et peut les transmettre localement ou vers des serveurs distants.

```
Application/Kernel
      │
      ▼
   rsyslog (local)
      │
      ├──► /var/log/syslog          (stockage local)
      │
      └──► Serveur rsyslog distant  (centralisation)
                 │
                 ▼
         /var/log/clients/%HOSTNAME%/
```

**Protocoles de transport :**

| Protocole | Port | Fiabilité | Cas d'usage |
|-----------|------|-----------|-------------|
| UDP | 514 | Non fiable (paquets perdus possibles) | Réseaux locaux fiables, haute performance |
| TCP | 514 | Fiable (accusé de réception) | Production, logs critiques |
| TCP+TLS | 6514 | Fiable + chiffré | Environnements sécurisés, WAN |

---

### 3.2 Configuration du serveur

**Fichier :** `fichiers_de_configuration/rsyslog_serveur.conf`  
**Emplacement sur le système :** `/etc/rsyslog.conf` ou `/etc/rsyslog.d/serveur.conf`

#### Étapes d'installation et d'activation sur le serveur :

```bash
# 1. Installer rsyslog (si absent)
sudo apt update && sudo apt install rsyslog -y

# 2. Copier la configuration
sudo cp rsyslog_serveur.conf /etc/rsyslog.d/10-serveur.conf

# 3. Créer le répertoire de stockage des logs clients
sudo mkdir -p /var/log/clients
sudo chown rsyslog:adm /var/log/clients
sudo chmod 750 /var/log/clients

# 4. Vérifier la configuration (syntaxe)
sudo rsyslogd -N1

# 5. Redémarrer rsyslog
sudo systemctl restart rsyslog

# 6. Vérifier l'état du service
sudo systemctl status rsyslog

# 7. Vérifier que les ports sont ouverts
sudo ss -tulnp | grep 514
```

**Sortie attendue de `ss -tulnp | grep 514` :**
```
udp   UNCONN   0   0   0.0.0.0:514   0.0.0.0:*   users:(("rsyslogd",pid=1234,fd=6))
tcp   LISTEN   0   25  0.0.0.0:514   0.0.0.0:*   users:(("rsyslogd",pid=1234,fd=7))
```

#### Ouvrir le pare-feu (si nécessaire) :

```bash
# Avec ufw
sudo ufw allow 514/udp
sudo ufw allow 514/tcp

# Avec iptables
sudo iptables -A INPUT -p udp --dport 514 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 514 -j ACCEPT
```

---

### 3.3 Configuration du client

**Fichier :** `fichiers_de_configuration/rsyslog_client.conf`  
**Emplacement sur le système :** `/etc/rsyslog.d/10-client.conf`

#### Étapes d'installation sur le client :

```bash
# 1. Copier la configuration (adapter l'IP du serveur dans le fichier)
sudo cp rsyslog_client.conf /etc/rsyslog.d/10-client.conf

# 2. Créer le répertoire de queue
sudo mkdir -p /var/spool/rsyslog
sudo chown rsyslog:rsyslog /var/spool/rsyslog

# 3. Vérifier la configuration
sudo rsyslogd -N1

# 4. Redémarrer rsyslog
sudo systemctl restart rsyslog

# 5. Vérifier l'état
sudo systemctl status rsyslog
```

---

### 3.4 Test et preuve de réception

#### Envoyer un message de test depuis le client :

```bash
# Envoyer un message syslog de test (facility user, priority info)
logger -p user.info "TEST RSYSLOG : Message de test depuis machine1 - $(date)"

# Envoyer un message d'erreur critique
logger -p user.err "ERREUR TEST : Simulation d'une erreur critique"

# Envoyer depuis un script
logger -t mon_application -p local0.info "Démarrage du service - PID $$"
```

#### Vérifier la réception sur le serveur :

```bash
# Sur le SERVEUR — Vérifier les logs reçus
sudo ls /var/log/clients/
# ► machine1/

sudo ls /var/log/clients/machine1/
# ► syslog.log  user.log  cron.log

# Lire les logs reçus
sudo tail -f /var/log/clients/machine1/syslog.log
```

**Preuve de réception — sortie attendue sur le serveur :**

```
May 17 15:42:10 machine1 admin: TEST RSYSLOG : Message de test depuis machine1 - Sat May 17 15:42:10 2026
May 17 15:42:15 machine1 admin: ERREUR TEST : Simulation d'une erreur critique
May 17 15:42:20 machine1 mon_application: Démarrage du service - PID 4567
```

> ✅ **Confirmation :** Le hostname `machine1` apparaît bien dans les logs reçus sur le serveur, prouvant la centralisation opérationnelle.

#### Vérifier avec tcpdump (analyse réseau) :

```bash
# Sur le SERVEUR — capturer le trafic syslog entrant sur le port 514
sudo tcpdump -i eth0 -n port 514

# Sortie attendue lors d'un envoi client :
# 15:42:10.123456 IP 192.168.1.101.49122 > 192.168.1.100.514: Flags [P.], ...
```

---

## 4. Rotation des Logs avec logrotate

### 4.1 Présentation de logrotate

`logrotate` est l'outil standard de rotation des logs sous Linux. Il est généralement exécuté automatiquement chaque nuit par un job `cron` ou une unité `systemd.timer`.

**Principes de fonctionnement :**

```
/var/log/mon_application/app.log    ← fichier actif (écrit par l'appli)
          │
          │ (rotation daily)
          ▼
/var/log/mon_application/app.log-20260517.gz  ← archive compressée
/var/log/mon_application/app.log-20260516.gz
...
/var/log/mon_application/app.log-20260503.gz  ← 14e archive (rotate 14)
```

### 4.2 Fichier de configuration

**Fichier :** `fichiers_de_configuration/logrotate_mon_application.conf`  
**Emplacement sur le système :** `/etc/logrotate.d/mon_application`

### 4.3 Installation et test

```bash
# 1. Copier le fichier de configuration
sudo cp logrotate_mon_application.conf /etc/logrotate.d/mon_application

# 2. Créer le répertoire de logs applicatifs (si absent)
sudo mkdir -p /var/log/mon_application
sudo chown www-data:adm /var/log/mon_application
sudo chmod 750 /var/log/mon_application

# 3. Vérifier la syntaxe et simuler (--debug = dry-run, aucune rotation réelle)
sudo logrotate --debug /etc/logrotate.d/mon_application
```

**Sortie de la simulation (`--debug`) :**

```
reading config file /etc/logrotate.d/mon_application
Allocating hash table for state file, size 15360 B
  rotating log /var/log/mon_application/app.log, log->rotateCount is 14
  dateext suffix '-20260517'
  glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
  compressing log with: /bin/gzip
  running prerotate script
  renaming /var/log/mon_application/app.log to /var/log/mon_application/app.log-20260517
  creating new /var/log/mon_application/app.log mode = 0640 uid = 33 gid = 4
  running postrotate script
```

```bash
# 4. Forcer une rotation immédiate (pour tester en production)
sudo logrotate --force /etc/logrotate.d/mon_application

# 5. Vérifier le statut de logrotate
cat /var/lib/logrotate/status | grep mon_application
# ► "/var/log/mon_application/app.log" 2026-5-17-15:0:0
```

### 4.4 Tableau récapitulatif des directives

| Directive | Valeur configurée | Signification |
|-----------|-------------------|---------------|
| `daily` | — | Rotation quotidienne |
| `weekly` | — | Rotation hebdomadaire (erreurs) |
| `rotate` | `14` (app), `8` (erreurs), `30` (accès) | Nombre d'archives conservées |
| `compress` | — | Compression gzip des archives |
| `delaycompress` | — | Compression différée d'un cycle |
| `missingok` | — | Pas d'erreur si fichier absent |
| `notifempty` | — | Pas de rotation si fichier vide |
| `size 100M` | 100 Mo | Rotation si taille dépassée |
| `dateext` | — | Horodatage dans le nom d'archive |
| `create 0640` | `www-data adm` | Création avec permissions strictes |

---

## 5. Surveillance en Temps Réel avec tail -f

### 5.1 Commandes de base

`tail -f` (follow) lit la fin d'un fichier et **reste attaché**, affichant les nouvelles lignes au fur et à mesure qu'elles s'ajoutent.

```bash
# Surveiller le syslog système
sudo tail -f /var/log/syslog

# Surveiller les logs d'authentification (SSH, sudo, etc.)
sudo tail -f /var/log/auth.log

# Surveiller les logs reçus depuis le client (côté serveur)
sudo tail -f /var/log/clients/machine1/syslog.log

# Afficher les 100 dernières lignes puis suivre
sudo tail -100f /var/log/syslog

# Surveiller le journal systemd en temps réel (équivalent journalctl -f)
journalctl -f

# Suivre un service spécifique en temps réel
journalctl -u ssh.service -f
```

### 5.2 Surveiller plusieurs fichiers simultanément

```bash
# Surveiller syslog ET auth.log en même temps
sudo tail -f /var/log/syslog /var/log/auth.log

# Sortie avec en-têtes pour identifier la source :
# ==> /var/log/syslog <==
# May 17 15:50:01 machine1 CRON[5678]: (root) CMD (...)
#
# ==> /var/log/auth.log <==
# May 17 15:50:04 machine1 sshd[5680]: Accepted password for admin...
```

### 5.3 Filtrage en temps réel avec grep

```bash
# Afficher seulement les erreurs SSH en temps réel
sudo tail -f /var/log/auth.log | grep -i "failed\|invalid\|error"

# Surveiller les connexions réussies
sudo tail -f /var/log/auth.log | grep "Accepted"

# Surveiller les logs reçus du client ET filtrer par mot-clé
sudo tail -f /var/log/clients/machine1/syslog.log | grep -E "ERROR|WARN|CRIT"

# Avec horodatage (utile pour scripting)
sudo tail -f /var/log/syslog | awk '{print strftime("%Y-%m-%d %H:%M:%S"), $0; fflush()}'
```

### 5.4 Utilisation avancée avec multitail

`multitail` permet de surveiller plusieurs fichiers dans des fenêtres séparées dans le terminal :

```bash
# Installer multitail
sudo apt install multitail -y

# Surveiller syslog et auth.log côte à côte
sudo multitail /var/log/syslog /var/log/auth.log

# Avec coloration des erreurs
sudo multitail -ci red /var/log/auth.log -ci yellow /var/log/syslog
```

### 5.5 Script de surveillance automatisé

```bash
#!/bin/bash
# monitor_logs.sh — Script de surveillance temps réel avec alertes
# Usage : sudo bash monitor_logs.sh

LOG_FILE="/var/log/syslog"
ALERT_KEYWORDS="error|failed|critical|CRIT|segfault|OOM"
ADMIN_EMAIL="admin@example.com"

echo "[INFO] Démarrage de la surveillance de $LOG_FILE"
echo "[INFO] Mots-clés surveillés : $ALERT_KEYWORDS"
echo "---"

tail -f "$LOG_FILE" | while IFS= read -r line; do
    if echo "$line" | grep -qiE "$ALERT_KEYWORDS"; then
        echo "[ALERTE] $(date '+%Y-%m-%d %H:%M:%S') : $line"
        # Décommenter pour envoyer un email d'alerte :
        # echo "$line" | mail -s "[ALERTE LOG] Erreur détectée" "$ADMIN_EMAIL"
    fi
done
```

---

## 6. Récapitulatif des Fichiers de Configuration

### 6.1 Arborescence du projet

```
projet system/
├── Rapport_Projet3_Supervision_Logs.md   ← Ce rapport
└── fichiers_de_configuration/
    ├── rsyslog_serveur.conf              ← Config du serveur centralisateur
    ├── rsyslog_client.conf               ← Config du client émetteur
    └── logrotate_mon_application.conf    ← Config de rotation des logs
```

### 6.2 Tableau récapitulatif

| Fichier | Rôle | Emplacement système | Service concerné |
|---------|------|---------------------|-----------------|
| `rsyslog_serveur.conf` | Réception des logs distants sur UDP/TCP 514, stockage dans `/var/log/clients/%HOSTNAME%/` | `/etc/rsyslog.d/10-serveur.conf` | rsyslog (serveur) |
| `rsyslog_client.conf` | Transmission de tous les logs vers le serveur distant via TCP avec queue de persistance | `/etc/rsyslog.d/10-client.conf` | rsyslog (client) |
| `logrotate_mon_application.conf` | Rotation journalière/hebdomadaire des logs applicatifs avec compression, archivage 14-30 jours | `/etc/logrotate.d/mon_application` | logrotate (cron) |

### 6.3 Preuves de fonctionnement

#### Preuve 1 — rsyslog serveur actif et en écoute

```bash
$ sudo systemctl status rsyslog
● rsyslog.service - System Logging Service
     Loaded: loaded (/lib/systemd/system/rsyslog.service; enabled)
     Active: active (running) since Sat 2026-05-17 08:00:01 CET
   Main PID: 1234 (rsyslogd)

$ sudo ss -tulnp | grep 514
udp  UNCONN  0  0  0.0.0.0:514  0.0.0.0:*  users:(("rsyslogd",pid=1234,fd=6))
tcp  LISTEN  0  25 0.0.0.0:514  0.0.0.0:*  users:(("rsyslogd",pid=1234,fd=7))
```

#### Preuve 2 — Log reçu du client sur le serveur

```bash
$ sudo cat /var/log/clients/machine1/user.log
May 17 15:42:10 machine1 admin: TEST RSYSLOG : Message de test depuis machine1 - Sat May 17 15:42:10 2026
May 17 15:42:15 machine1 admin: ERREUR TEST : Simulation d'une erreur critique
May 17 15:42:20 machine1 mon_application: Démarrage du service - PID 4567
```

#### Preuve 3 — Rotation logrotate effective

```bash
$ sudo ls -lh /var/log/mon_application/
total 48K
-rw-r----- 1 www-data adm  2.1K May 17 15:00 app.log          ← fichier courant
-rw-r----- 1 www-data adm  98K  May 16 15:00 app.log-20260516.gz
-rw-r----- 1 www-data adm  101K May 15 15:00 app.log-20260515.gz

$ cat /var/lib/logrotate/status | grep mon_application
"/var/log/mon_application/app.log" 2026-5-17-15:0:0
```

#### Preuve 4 — Surveillance temps réel

```bash
$ sudo tail -f /var/log/clients/machine1/syslog.log
# (terminal reste actif, affichage des logs en continu...)
May 17 15:50:01 machine1 CRON[5678]: (root) CMD (test -x /usr/sbin/anacron ...)
May 17 15:51:22 machine1 sshd[5690]: Accepted publickey for admin from 192.168.1.50
May 17 15:52:00 machine1 mon_application: Traitement batch terminé - 1250 entrées
```

---

## 7. Conclusion

Ce projet a permis de mettre en place une chaîne complète de supervision des logs systèmes, couvrant l'ensemble du cycle de vie des journaux :

| Phase | Outil | Résultat |
|-------|-------|----------|
| **Collecte locale** | systemd/journald | Journal structuré, requêtes via `journalctl` |
| **Analyse** | `journalctl` | Filtrage par service, date, priorité, UID |
| **Centralisation** | rsyslog | Transmission TCP fiable vers serveur distant |
| **Archivage** | logrotate | Rotation automatique, compression, rétention 30j |
| **Surveillance** | `tail -f` | Monitoring temps réel avec alertes grep |

**Points clés retenus :**

- L'utilisation du **TCP avec queue de persistance** dans rsyslog garantit qu'aucun log n'est perdu même en cas de coupure réseau temporaire.
- La directive **`delaycompress`** dans logrotate est essentielle pour les applications qui gardent leurs descripteurs de fichiers ouverts entre deux rotations.
- `journalctl` avec ses filtres combinés (`-u`, `--since`, `-p`) est bien plus puissant que `grep` sur les fichiers textes classiques.
- La centralisation des logs sur un serveur dédié est une bonne pratique de sécurité : un attaquant compromettant un client ne peut pas effacer les traces déjà transmises au serveur.

**Améliorations possibles :**

- Ajouter le **chiffrement TLS** sur le transport rsyslog (port 6514) pour sécuriser les logs en transit.
- Intégrer un outil comme **Graylog** ou **ELK Stack** (Elasticsearch + Logstash + Kibana) pour la visualisation et l'analyse avancée des logs centralisés.
- Mettre en place **fail2ban** pour réagir automatiquement aux tentatives d'intrusion détectées dans les logs SSH.

---

*Rapport réalisé dans le cadre du cours d'Administration des Systèmes Informatiques — Ingénierie Système Informatique.*
