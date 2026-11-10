# Administration d'un serveur Linux : lab virtuel

> Projet individuel : mise en place et administration d'un serveur Linux en environnement virtuel.

| | |
|---|---|
| **Auteur** | Smaoui Eya |
| **Date** | 9 novembre 2026 |
| **Environnement** | Ubuntu Desktop dans VirtualBox |

---

## Sommaire

1. [Objectif du projet](#1-objectif-du-projet)
2. [Architecture et réseau](#2-architecture-et-réseau)
3. [Préparation de la VM](#3-préparation-de-la-vm)
4. [Mise à jour et informations système](#4-mise-à-jour-et-informations-système)
5. [Utilisateurs et groupes](#5-utilisateurs-et-groupes)
6. [Permissions et dossier partagé](#6-permissions-et-dossier-partagé)
7. [SSH : installation, clé et durcissement](#7-ssh--installation-clé-et-durcissement)
8. [Pare-feu UFW](#8-pare-feu-ufw)
9. [Serveur web Apache](#9-serveur-web-apache)
10. [Fail2ban](#10-fail2ban)
11. [Supervision](#11-supervision)
12. [Sauvegarde automatique avec cron](#12-sauvegarde-automatique-avec-cron)
13. [Mises à jour automatiques de sécurité](#13-mises-à-jour-automatiques-de-sécurité)
14. [Vérification finale](#14-vérification-finale)
15. [Problèmes rencontrés et solutions](#15-problèmes-rencontrés-et-solutions)
16. [Conclusion](#16-conclusion)

---

## 1. Objectif du projet

Transformer une machine virtuelle Ubuntu en serveur configuré, sécurisé et surveillé, comme le ferait un administrateur système en entreprise.

**Compétences travaillées :** gestion des utilisateurs et des permissions, accès distant SSH, pare-feu, services web, protection contre le brute-force, supervision, sauvegardes automatisées, mises à jour de sécurité.

---

## 2. Architecture et réseau

| Élément | Valeur |
|---|---|
| Hyperviseur | VirtualBox |
| Système | Ubuntu Desktop |
| Nom d'hôte | `server` |
| Adaptateur 1 | NAT (`enp0s3`, `10.0.2.15`) : accès à Internet |
| Adaptateur 2 | Réseau privé hôte (`enp0s8`, `192.168.56.101`) : accès depuis le PC |

```text
 ┌────────────┐   Host-only 192.168.56.0/24   ┌──────────────────┐
 │  PC hôte   │ ◄───────────────────────────► │   VM Ubuntu      │
 │192.168.56.1│        SSH (22) / HTTP (80)   │ 192.168.56.101   │
 └────────────┘                               │        │ NAT     │
                                              └────────┼─────────┘
                                                       ▼ Internet
```

---

## 3. Préparation de la VM

- Adaptateur 1 en NAT, adaptateur 2 en réseau privé hôte.
- Snapshot initial : `00_etat_initial`.

![Configuration réseau VirtualBox](captures/01_vbox_config_reseau.png)

**Explication :** la machine virtuelle utilise deux cartes réseau avec des rôles différents. L'adaptateur 1, en mode NAT, permet à Ubuntu d'accéder à Internet pour télécharger les mises à jour et installer les paquets. L'adaptateur 2, en mode Host-only, crée un réseau privé entre le PC hôte et la VM. Ce deuxième réseau permet d'administrer le serveur à distance avec SSH et d'accéder à la page web depuis le PC, sans rendre ces services directement accessibles depuis le réseau local ou Internet.

---

## 4. Mise à jour et informations système

```bash
sudo apt update && sudo apt upgrade -y
hostnamectl
ip a
df -h
free -h
```

![Mise à jour du système](captures/02_maj_systeme.png)

![Informations système et IP](captures/03_infos_systeme_ip.png)

**Explication :** ces commandes permettent de maintenir Ubuntu à jour et de consulter l'état général de la machine. `apt update` actualise la liste des paquets disponibles, tandis que `apt upgrade` installe les mises à jour. `hostnamectl` affiche les informations sur le système et son nom d'hôte, `ip a` présente les interfaces réseau et leurs adresses IP, `df -h` indique l'espace utilisé et disponible sur les disques, et `free -h` affiche l'utilisation de la mémoire vive et du swap.

---

## 5. Utilisateurs et groupes

```bash
sudo groupadd devteam
sudo adduser alice
sudo adduser bob
sudo usermod -aG devteam alice
sudo usermod -aG devteam bob
sudo usermod -aG sudo alice
id alice
getent group devteam
sudo chage -M 90 bob
```

![Utilisateurs et groupes](captures/04_utilisateurs_groupes.png)

**Explication :** un groupe permet de gérer les droits de plusieurs utilisateurs plus facilement. `alice` et `bob` appartiennent au groupe `devteam`, ce qui leur permet de partager les ressources accordées à ce groupe. `alice` possède aussi les droits d'administration grâce au groupe `sudo`, alors que `bob` reste un utilisateur standard. La commande `chage -M 90 bob` fixe à 90 jours la durée maximale de validité du mot de passe de `bob`.

---

## 6. Permissions et dossier partagé

```bash
sudo mkdir -p /srv/projet
sudo chown root:devteam /srv/projet
sudo chmod 2770 /srv/projet
ls -ld /srv/projet

sudo adduser invite
sudo -u alice touch /srv/projet/test_alice.txt
sudo -u bob ls /srv/projet
sudo -u invite ls /srv/projet     # doit être refusé
```

![Permissions du dossier partagé](captures/05_permissions_projet.png)

**Explication :** le dossier `/srv/projet` appartient à `root` et au groupe `devteam`. La permission `2770` donne tous les droits au propriétaire et au groupe, mais aucun droit aux autres utilisateurs. Le chiffre `2` active le bit setgid : les nouveaux fichiers et dossiers créés à l'intérieur héritent du groupe `devteam`. L'utilisateur `invite`, qui ne fait pas partie de ce groupe, ne peut donc pas accéder au dossier.

---

## 7. SSH : installation, clé et durcissement

### 7.1 Installation

```bash
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
systemctl status ssh
```

![Statut du service SSH](captures/06_ssh_status.png)

### 7.2 Connexion depuis le PC

```bash
ip a
ssh server@192.168.56.101
```

![Adresse IP de la VM](captures/07a_ip_a.png)

### 7.3 Authentification par clé (depuis le PC)

```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh server@192.168.56.101 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh server@192.168.56.101
```

![Connexion SSH par clé](captures/07_connexion_ssh_cle.png)

### 7.4 Durcissement

Fichier `/etc/ssh/sshd_config.d/01-hardening.conf` :

```text
PermitRootLogin no
PasswordAuthentication no
MaxAuthTries 3
```

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|maxauthtries"
```

![Configuration de durcissement SSH](captures/08_ssh_hardening.png)

Test depuis le PC :

```powershell
ssh -o PubkeyAuthentication=no server@192.168.56.101
```

![Mot de passe refusé](captures/08b_mot_de_passe_refuse.png)

**Explication :** le fichier `01-hardening.conf` est placé dans le dossier de configuration complémentaire de SSH. Le préfixe `01-` permet de le lire avant les fichiers dont le nom vient ensuite dans l'ordre alphabétique. `PermitRootLogin no` interdit la connexion directe avec le compte root. `PasswordAuthentication no` désactive l'authentification par mot de passe : la connexion doit donc utiliser une clé autorisée. Enfin, `MaxAuthTries 3` limite le nombre de tentatives d'authentification par connexion. La commande `sshd -t` vérifie la syntaxe avant le redémarrage du service.

---

## 8. Pare-feu UFW

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw enable
sudo ufw status verbose
```

![Statut UFW](captures/09_ufw_status.png)

**Explication :** UFW est un outil de gestion simplifiée du pare-feu. La politique `deny incoming` bloque par défaut les connexions entrantes, tandis que `allow outgoing` autorise les connexions initiées depuis le serveur. Seules les connexions SSH sur le port 22 et HTTP sur le port 80 sont explicitement autorisées. La règle SSH est ajoutée avant l'activation du pare-feu afin de conserver l'accès distant à la VM.

---

## 9. Serveur web Apache

```bash
sudo apt install apache2 -y
systemctl status apache2 --no-pager
curl http://localhost
echo "<h1>Serveur Lab - Smaoui Eya</h1>" | sudo tee /var/www/html/index.html
```

![Statut d'Apache](captures/10_apache_status.png)

![Page web dans le navigateur du PC](captures/11_page_web_navigateur.png)

**Explication :** Apache est le serveur web utilisé pour publier une page accessible avec un navigateur. `systemctl status` permet de vérifier que le service fonctionne et `curl http://localhost` teste la réponse web directement depuis la VM. La dernière commande remplace la page d'accueil par défaut. L'accès à `http://192.168.56.101` depuis le PC hôte vérifie que la communication Host-only fonctionne, qu'Apache répond et que le pare-feu autorise le port 80.

---

## 10. Fail2ban

Fichier `/etc/fail2ban/jail.local` :

```ini
[DEFAULT]
backend = systemd
banaction = ufw

[sshd]
enabled = true
port = ssh
maxretry = 3
findtime = 10m
bantime = 10m
```

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

![Statut du jail sshd](captures/12_fail2ban_status.png)

Test de bannissement (depuis le PC) :

```powershell
1..5 | ForEach-Object { ssh -o BatchMode=yes baduser@192.168.56.101 }
```

![IP bannie par Fail2ban](captures/12b_fail2ban_ban.png)

**Explication :** Fail2ban surveille les journaux de connexion et bloque temporairement les adresses IP qui échouent trop souvent à s'authentifier. Avec cette configuration, trois échecs détectés pendant une période de dix minutes entraînent un bannissement de dix minutes. `banaction = ufw` demande à Fail2ban d'appliquer le blocage avec le pare-feu UFW. `backend = systemd` permet de lire les événements SSH enregistrés dans le journal systemd (`journald`).

---

## 11. Supervision

```bash
sudo apt install htop -y
htop
systemctl list-units --type=service --state=running --no-pager
systemctl is-enabled ssh apache2 fail2ban ufw
```

![htop](captures/13_htop.png)

![Services actifs](captures/13b_services_actifs.png)

**Explication :** cette partie réalise une supervision de base directement sur le serveur. `htop` affiche en temps réel l'utilisation du processeur, de la mémoire et les processus en cours. `systemctl list-units` liste les services actuellement actifs, tandis que `systemctl is-enabled` indique si les services concernés sont configurés pour démarrer automatiquement avec le système. Ces vérifications aident à repérer rapidement un service arrêté ou mal configuré.

---

## 12. Sauvegarde automatique avec cron

Script `/usr/local/bin/backup.sh` :

```bash
#!/bin/bash
DEST=/var/backups/lab
mkdir -p "$DEST"
tar -czf "$DEST/projet_$(date +%F_%H%M).tar.gz" /srv/projet /var/www/html
find "$DEST" -type f -name "projet_*.tar.gz" -mtime +7 -delete
echo "$(date '+%F %T') sauvegarde OK" >> /var/log/backup_lab.log
```

![Script de sauvegarde](captures/15_script_backup.png)

Planification (`sudo crontab -e`) :

```text
0 2 * * * /usr/local/bin/backup.sh
```

![Cron, archives et log](captures/16_cron_et_backup_ok.png)

Test de restauration :

```bash
sudo tar -tzf /var/backups/lab/projet_2026-11-09_1448.tar.gz | head
mkdir -p ~/restore_test
sudo tar -xzf /var/backups/lab/projet_2026-11-09_1448.tar.gz -C ~/restore_test
ls ~/restore_test/var/www/html
```

![Restauration d'une sauvegarde](captures/16b_restauration.png)

**Explication :** le script crée une archive compressée contenant le dossier du projet et les fichiers du site web. Le nom de l'archive inclut la date et l'heure pour distinguer les sauvegardes. La commande `find` supprime les archives de plus de sept jours afin de limiter l'espace occupé, puis une ligne est ajoutée au journal pour garder une trace de l'exécution. La tâche cron lance le script chaque jour à 2 h du matin. Enfin, le test de restauration vérifie que l'archive peut être lue et que ses fichiers peuvent être extraits. Une sauvegarde doit toujours être testée : sa présence seule ne garantit pas qu'elle pourra être restaurée.

---

## 13. Mises à jour automatiques de sécurité

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure -plow unattended-upgrades
systemctl status unattended-upgrades --no-pager
cat /etc/apt/apt.conf.d/20auto-upgrades
```

![Unattended upgrades](captures/17_unattended_upgrades.png)

**Explication :** `unattended-upgrades` automatise l'installation des mises à jour configurées, notamment les mises à jour de sécurité. `dpkg-reconfigure` permet d'activer cette fonctionnalité, `systemctl status` vérifie l'état du service et le fichier `20auto-upgrades` permet de contrôler les paramètres d'actualisation et d'installation automatique. Cela réduit le risque de conserver trop longtemps des failles déjà corrigées.

---

## 14. Vérification finale

```bash
systemctl is-active ssh apache2 fail2ban ufw cron
sudo ufw status verbose
sudo fail2ban-client status sshd
```

![Vérification finale](captures/18_verification_finale.png)

![Liste des snapshots VirtualBox](captures/19_snapshots_virtualbox.png)

Snapshot final : `99_etat_final`.

---

## 15. Problèmes rencontrés et solutions

### 15.1 Le service SSH refuse de démarrer

- **Symptôme :** `Job for ssh.service failed`, avec `ExecStartPre=/usr/sbin/sshd -t (status=1/FAILURE)`.
- **Diagnostic :** systemd teste la configuration avant de lancer SSH. La commande `sudo sshd -t` affiche la cause exacte de l'échec.
- **Solution :** corriger la cause relevée, puis relancer le service :

```bash
sudo systemctl reset-failed ssh
sudo systemctl restart ssh
```

Causes courantes selon le message affiché par `sudo sshd -t` :

| Message | Cause | Correction |
|---|---|---|
| `no hostkeys available` | Clés d'hôte absentes | `sudo ssh-keygen -A` |
| `Missing privilege separation directory: /run/sshd` | Dossier manquant | `sudo mkdir -p /run/sshd` |
| `Bad configuration option` | Ligne invalide dans `sshd_config` ou `sshd_config.d/` | Corriger ou supprimer la ligne |
| `Permissions ... are too open` | Droits trop larges sur une clé d'hôte | `sudo chmod 600 /etc/ssh/ssh_host_*_key` |

<!-- ![Erreur SSH](captures/06a_erreur_ssh.png) -->

### 15.2 Commandes tapées dans le mauvais terminal

- **Symptôme :** `-bash: type: ... not found`, et `ssh-copy-id` qui échoue avec `Host key verification failed`.
- **Cause :** la session SSH vers la VM était restée ouverte dans PowerShell. Les commandes s'exécutaient donc dans la VM et non sur le PC.
- **Solution :** `exit` pour revenir à l'invite `PS C:\...>`, puis relancer les commandes sur le PC.

<!-- ![Erreur mauvais terminal](captures/07c_erreur_mauvais_terminal.png) -->

### 15.3 Espace manquant dans le script de sauvegarde

- **Symptôme :** `date+%F_%H%M: command not found` et fichier `projet_.tar.gz` au nom incomplet.
- **Cause :** `date+%F_%H%M` au lieu de `date +%F_%H%M` (espace manquant).
- **Solution :** correction du script et suppression de l'archive mal nommée.

<!-- ![Erreur date](captures/16a_erreur_date.png) -->

### 15.4 Joker `*` avec tar

- **Symptôme :** `tar: ... Not found in archive`.
- **Cause :** le `*` a été développé en deux noms d'archives. Tar a interprété le second comme un fichier à chercher dans le premier.
- **Solution :** indiquer le nom complet d'une seule archive.

<!-- ![Erreur tar](captures/16c_erreur_tar_wildcard.png) -->

---

## 16. Conclusion

Ce projet m'a permis de mettre en pratique plusieurs tâches courantes de l'administration système sous Linux : créer et gérer des utilisateurs, organiser les permissions d'accès, configurer une connexion distante sécurisée, contrôler les connexions avec un pare-feu et protéger le service SSH contre les tentatives répétées. J'ai également découvert l'intérêt de l'automatisation avec cron et des mises à jour de sécurité, ainsi que l'importance de vérifier les services et de tester la restauration des sauvegardes.

Les difficultés rencontrées m'ont appris à lire les messages d'erreur, à vérifier la configuration avant de redémarrer un service et à distinguer les commandes à exécuter sur le PC hôte de celles à exécuter dans la VM. Ce projet constitue une première maquette d'administration : il ne remplace pas une infrastructure de production, mais permet de comprendre et de tester ses principaux mécanismes.

### Pour aller plus loin

- Attribuer une adresse IP fixe au serveur avec Netplan.
- Proposer un accès web sécurisé en HTTPS avec un certificat.
- Superviser le serveur depuis une autre machine.
- Envoyer les sauvegardes vers un disque ou un serveur distant.
- Réaliser un audit avec Lynis pour identifier d'autres points de durcissement.

---

## Structure du dépôt

```text
projet-linux-serveur/
├── README.md
└── captures/
    ├── 01_vbox_config_reseau.png
    ├── 02_maj_systeme.png
    └── ...
```
