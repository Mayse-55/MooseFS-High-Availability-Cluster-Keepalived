# MooseFS High Availability Cluster with Keepalived

[![MooseFS](https://img.shields.io/badge/MooseFS-4.0+-blue.svg)](https://moosefs.com/)
[![Keepalived](https://img.shields.io/badge/Keepalived-2.3+-green.svg)](https://www.keepalived.org/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## 🎯 Description

MooseFS est un système de fichiers distribué tolérant aux pannes qui permet de créer un pool de stockage unifié à partir de plusieurs serveurs. Cette configuration implémente une architecture à 3 nœuds maîtres avec :

- **Haute disponibilité automatique** : Basculement transparent entre les nœuds maîtres sans interruption de service
- **Redondance complète** : 3 nœuds maîtres garantissent la continuité même en cas de panne de 2 serveurs
- **IP virtuelle (VIP)** : Point d'accès unique simplifiant la configuration des clients
- **Synchronisation temps réel** : Réplication automatique des métadonnées via rsync et inotify
- **Stockage distribué** : Les données (chunks) sont réparties sur les 3 nœuds pour performance et résilience

### Cas d'usage

- Stockage partagé pour clusters de machines virtuelles (Proxmox, VMware, KVM)
- Infrastructure de sauvegarde distribuée
- Stockage pour applications conteneurisées (Docker, Kubernetes)
- Systèmes de fichiers partagés pour environnements de développement/production
- Archives de données avec redondance

## 📋 Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Prérequis](#prérequis)
- [Installation de base](#installation-de-base)
- [Configuration par nœud](#configuration-par-nœud)
- [Configuration Keepalived](#configuration-keepalived)
- [Synchronisation des métadonnées](#synchronisation-des-métadonnées)
- [Démarrage du cluster](#démarrage-du-cluster)
- [Tests et validation](#tests-et-validation)
- [Commandes utiles](#commandes-utiles)
- [Troubleshooting](#troubleshooting)

---

## 🎯 Vue d'ensemble

Ce guide décrit la mise en place d'un cluster MooseFS hautement disponible avec 3 nœuds maîtres utilisant Keepalived pour la gestion automatique du basculement (failover).

### Fonctionnalités

- ✅ **Haute disponibilité** : Basculement automatique en cas de panne
- ✅ **3 nœuds master** : Redondance complète
- ✅ **IP virtuelle (VIP)** : Point d'accès unique pour les clients
- ✅ **Synchronisation automatique** : Réplication des métadonnées en temps réel
- ✅ **Stockage distribué** : Chunks répartis sur les 3 nœuds

---

## 📦 Prérequis

### Configuration réseau

- 3 serveurs avec Debian 12 (Trixie) ou Ubuntu 22.04/24.04
- Connexion réseau stable entre les nœuds
- IP statiques configurées :
  - NODE1 : 192.168.25.200
  - NODE2 : 192.168.25.210
  - NODE3 : 192.168.25.220
  - VIP : 192.168.25.230

### Ressources recommandées

- **CPU** : 2+ cores par nœud
- **RAM** : 4GB+ par nœud
- **Stockage** : 200GB+ pour les chunks (par nœud)
- **Réseau** : 1Gbps minimum

---

## 🚀 Installation de base

### Les étapes suivantes sont à effectuer sur **NODE1, NODE2 ET NODE3**

---

### 1. Extension de la partition root (optionnel)

Si vous utilisez Proxmox et nécessitez de l'espace supplémentaire :

```bash
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```

---

### 2. Configuration des dépôts MooseFS

```bash
sudo mkdir -p /etc/apt/keyrings
curl https://repository.moosefs.com/moosefs.key | \
  gpg -o /etc/apt/keyrings/moosefs.gpg --dearmor

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/moosefs.gpg] http://repository.moosefs.com/moosefs-4/apt/debian/trixie trixie main" > /etc/apt/sources.list.d/moosefs.list
```

---

### 3. Mise à jour du système

```bash
sudo apt update
sudo apt dist-upgrade -y
sudo apt autoremove -y
```

---

### 4. Installation des paquets

#### 4.1. Dépendances système

```bash
sudo apt install -y build-essential libpcap-dev zlib1g-dev libfuse3-dev pkg-config fuse3 inotify-tools rsync
```

#### 4.2. Paquets MooseFS

```bash
sudo apt install -y moosefs-master moosefs-chunkserver moosefs-client moosefs-cgi moosefs-cgiserv moosefs-cli
```

#### 4.3. Installation de Keepalived

```bash
sudo apt install -y keepalived
```

---

### 5. Configuration du Master Server

#### 5.1. Préparation des répertoires

```bash
sudo mkdir -p /var/lib/mfs
sudo chown -R mfs:mfs /var/lib/mfs
```

#### 5.2. Configuration initiale

```bash
cd /etc/mfs
sudo cp mfsmaster.cfg.sample mfsmaster.cfg
sudo cp mfsexports.cfg.sample mfsexports.cfg
```

#### 5.3. Personnalisation de la configuration

```bash
sudo nano /etc/mfs/mfsmaster.cfg
```

Paramètres principaux à vérifier :
```bash
WORKING_USER = mfs
WORKING_GROUP = mfs
DATA_PATH = /var/lib/mfs
```

#### 5.4. Configuration des permissions d'accès

```bash
sudo nano /etc/mfs/mfsexports.cfg
```

Configuration recommandée :

```bash
# Autorisation globale (adaptez selon vos besoins de sécurité)
*       /       rw,alldirs,maproot=0
```

---

### 6. Configuration du Chunkserver

#### 6.1. Préparation des répertoires de stockage

```bash
sudo mkdir -p /mnt/local-mpx/moosefs_chunks
sudo mkdir -p /mnt/local-mpx/moosefs_data
sudo chown -R mfs:mfs /mnt/local-mpx/moosefs_chunks
sudo chmod 755 /mnt/local-mpx/moosefs_chunks
```

#### 6.2. Configuration initiale

```bash
cd /etc/mfs
sudo cp mfschunkserver.cfg.sample mfschunkserver.cfg
sudo cp mfshdd.cfg.sample mfshdd.cfg
```

#### 6.3. Configuration du chunkserver

```bash
sudo nano /etc/mfs/mfschunkserver.cfg
```

Ajouter/modifier :

```bash
MASTER_HOST = 192.168.25.230
MASTER_PORT = 9420
DATA_PATH = /var/lib/mfs
```

#### 6.4. Définition du stockage des chunks

```bash
sudo nano /etc/mfs/mfshdd.cfg
```

Ajouter :

```bash
/mnt/local-mpx/moosefs_chunks
```

---

### 7. Configuration de la résolution DNS

Sur **les 3 nœuds**, éditer `/etc/hosts` :

```bash
sudo nano /etc/hosts
```

Ajouter :

```bash
# MooseFS Cluster
192.168.25.200    mpx-1
192.168.25.210    mpx-2
192.168.25.220    mpx-3
192.168.25.230    mfsmaster
```

---

### 8. Configuration du client MooseFS

```bash
cd /etc/mfs
sudo cp mfsmount.cfg.sample mfsmount.cfg
sudo nano /etc/mfs/mfsmount.cfg
```

Configurer :

```bash
MASTER_HOST = 192.168.25.230
MASTER_PORT = 9421
```

---

### 9. Configuration du montage automatique

```bash
sudo nano /etc/fstab
```

Ajouter :

```bash
# MooseFS - Montage automatique
mfsmaster:/  /mnt/local-mpx/moosefs_data  moosefs  defaults,mfsdelayedinit,_netdev,nonempty  0 0
```

---

## ⚙️ Configuration par nœud

### 📌 Configuration spécifique à NODE1

#### Initialisation des métadonnées (**UNIQUEMENT sur NODE1**)

```bash
cd /var/lib/mfs
sudo cp metadata.mfs.empty metadata.mfs
sudo chown mfs:mfs metadata.mfs
sudo rm metadata.mfs.empty
```

#### Configuration SSH sans mot de passe

```bash
# Générer la clé SSH
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

# Copier vers les autres nœuds
ssh-copy-id root@192.168.25.210
ssh-copy-id root@192.168.25.220
```

#### Script de synchronisation rsync

```bash
sudo mkdir -p /etc/script

cat > /etc/script/mtd-rsync.sh << 'EOF'
#!/bin/bash
# Script de synchronisation des métadonnées MooseFS - NODE1

SRC="/var/lib/mfs"
DEST1="root@192.168.25.210:/var/lib/mfs"
DEST2="root@192.168.25.220:/var/lib/mfs"

# Boucle infinie : inotifywait détecte les modifications
while true; do
    inotifywait -r -e close_write,moved_to,create,delete "$SRC"

    # Synchronisation parallèle vers NODE2 et NODE3
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST1" &
    
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST2"
done
EOF

chmod +x /etc/script/mtd-rsync.sh
```

---

### 📌 Configuration spécifique à NODE2

#### Configuration SSH sans mot de passe

```bash
# Générer la clé SSH
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

# Copier vers les autres nœuds
ssh-copy-id root@192.168.25.200
ssh-copy-id root@192.168.25.220
```

#### Copie initiale des métadonnées depuis NODE1

```bash
rsync -avz --delete \
    --exclude='mfsmaster.lock' \
    --exclude='sessions.mfs*' \
    root@192.168.25.200:/var/lib/mfs/ /var/lib/mfs/

sudo chown -R mfs:mfs /var/lib/mfs/
```

#### Script de synchronisation rsync

```bash
sudo mkdir -p /etc/script

cat > /etc/script/mtd-rsync.sh << 'EOF'
#!/bin/bash
# Script de synchronisation des métadonnées MooseFS - NODE2

SRC="/var/lib/mfs"
DEST1="root@192.168.25.200:/var/lib/mfs"
DEST2="root@192.168.25.220:/var/lib/mfs"

# Boucle infinie : inotifywait détecte les modifications
while true; do
    inotifywait -r -e close_write,moved_to,create,delete "$SRC"

    # Synchronisation parallèle vers NODE1 et NODE3
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST1" &
    
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST2"
done
EOF

chmod +x /etc/script/mtd-rsync.sh
```

---

### 📌 Configuration spécifique à NODE3

#### Configuration SSH sans mot de passe

```bash
# Générer la clé SSH
ssh-keygen -t rsa -N "" -f /root/.ssh/id_rsa

# Copier vers les autres nœuds
ssh-copy-id root@192.168.25.200
ssh-copy-id root@192.168.25.210
```

#### Copie initiale des métadonnées depuis NODE1

```bash
rsync -avz --delete \
    --exclude='mfsmaster.lock' \
    --exclude='sessions.mfs*' \
    root@192.168.25.200:/var/lib/mfs/ /var/lib/mfs/

sudo chown -R mfs:mfs /var/lib/mfs/
```

#### Script de synchronisation rsync

```bash
sudo mkdir -p /etc/script

cat > /etc/script/mtd-rsync.sh << 'EOF'
#!/bin/bash
# Script de synchronisation des métadonnées MooseFS - NODE3

SRC="/var/lib/mfs"
DEST1="root@192.168.25.200:/var/lib/mfs"
DEST2="root@192.168.25.210:/var/lib/mfs"

# Boucle infinie : inotifywait détecte les modifications
while true; do
    inotifywait -r -e close_write,moved_to,create,delete "$SRC"

    # Synchronisation parallèle vers NODE1 et NODE2
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST1" &
    
    rsync -az --delete \
        --exclude='mfsmaster.lock' \
        --exclude='sessions.mfs*' \
        "$SRC/" "$DEST2"
done
EOF

chmod +x /etc/script/mtd-rsync.sh
```

---

## 🔄 Service de synchronisation automatique

### À effectuer sur NODE1, NODE2 ET NODE3

```bash
cat > /etc/systemd/system/mtd-rsync.service << 'EOF'
[Unit]
Description=Rsync Watch Daemon for MooseFS Metadata
After=network.target moosefs-master.service

[Service]
Type=simple
ExecStart=/etc/script/mtd-rsync.sh
Restart=always
User=root
Environment=HOME=/root

[Install]
WantedBy=multi-user.target
EOF

# Recharger systemd
sudo systemctl daemon-reload

# Activer le service (ne pas le démarrer maintenant)
sudo systemctl enable mtd-rsync.service
```

---

## 🛡️ Configuration Keepalived

### Scripts de transition (sur les 3 nœuds)

#### Script promote_master.sh

```bash
cat > /etc/keepalived/promote_master.sh << 'EOF'
#!/bin/bash
# Script appelé quand ce nœud devient MASTER

logger "MOOSEFS-HA: ===== DEVENIR MASTER ====="

# Attendre que la VIP soit bien configurée
sleep 10

# Démarrer le Master
logger "MOOSEFS-HA: Démarrage du Master..."
systemctl start moosefs-master

if [ $? -eq 0 ]; then
    logger "MOOSEFS-HA: ✓ Master démarré avec succès"
else
    logger "MOOSEFS-HA: ✗ ERREUR - Échec du démarrage du Master"
fi
EOF

chmod +x /etc/keepalived/promote_master.sh
```

#### Script demote_master.sh

```bash
cat > /etc/keepalived/demote_master.sh << 'EOF'
#!/bin/bash
# Script appelé quand ce nœud devient BACKUP

logger "MOOSEFS-HA: ===== DEVENIR BACKUP ====="

# Arrêter le Master
systemctl stop moosefs-master
logger "MOOSEFS-HA: Master arrêté"
EOF

chmod +x /etc/keepalived/demote_master.sh
```

---

### Configuration Keepalived par nœud

#### 🔴 NODE1 (Master - Priority 103)

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id MOOSEFS_NODE1
    enable_script_security
    script_user root
}

vrrp_script check_mfsmaster {
    script "/usr/bin/pgrep mfsmaster"
    interval 2
    weight 2
    fall 2
    rise 2
}

vrrp_instance VI_MOOSEFS {
    state MASTER
    interface vmbr0
    virtual_router_id 51
    priority 103
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass MooseSec
    }
    
    virtual_ipaddress {
        192.168.25.230/24
    }
    
    track_script {
        check_mfsmaster
    }
    
    notify_master "/etc/keepalived/promote_master.sh"
    notify_backup "/etc/keepalived/demote_master.sh"
    notify_fault "/etc/keepalived/demote_master.sh"
}
EOF
```

> **⚠️ Important** : Remplacez `vmbr0` par votre interface réseau réelle (vérifiez avec `ip a`)

---

#### 🟡 NODE2 (Backup - Priority 102)

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id MOOSEFS_NODE2
    enable_script_security
    script_user root
}

vrrp_script check_mfsmaster {
    script "/usr/bin/pgrep mfsmaster"
    interval 2
    weight 2
    fall 2
    rise 2
}

vrrp_instance VI_MOOSEFS {
    state BACKUP
    interface vmbr0
    virtual_router_id 51
    priority 102
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass MooseSec
    }
    
    virtual_ipaddress {
        192.168.25.230/24
    }
    
    track_script {
        check_mfsmaster
    }
    
    notify_master "/etc/keepalived/promote_master.sh"
    notify_backup "/etc/keepalived/demote_master.sh"
    notify_fault "/etc/keepalived/demote_master.sh"
}
EOF
```

> **⚠️ Important** : Remplacez `vmbr0` par votre interface réseau réelle

---

#### 🟢 NODE3 (Backup - Priority 101)

```bash
cat > /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    router_id MOOSEFS_NODE3
    enable_script_security
    script_user root
}

vrrp_script check_mfsmaster {
    script "/usr/bin/pgrep mfsmaster"
    interval 2
    weight 2
    fall 2
    rise 2
}

vrrp_instance VI_MOOSEFS {
    state BACKUP
    interface vmbr0
    virtual_router_id 51
    priority 101
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass MooseSec
    }
    
    virtual_ipaddress {
        192.168.25.230/24
    }
    
    track_script {
        check_mfsmaster
    }
    
    notify_master "/etc/keepalived/promote_master.sh"
    notify_backup "/etc/keepalived/demote_master.sh"
    notify_fault "/etc/keepalived/demote_master.sh"
}
EOF
```

> **⚠️ Important** : Remplacez `vmbr0` par votre interface réseau réelle

---

## 🎬 Démarrage du cluster

### 🔴 Sur NODE1 (Master principal)

```bash
# 1. Démarrer MooseFS Master
sudo systemctl start moosefs-master
sudo systemctl enable moosefs-master

# 2. Démarrer Chunkserver
sudo systemctl start moosefs-chunkserver
sudo systemctl enable moosefs-chunkserver

# 3. Démarrer CGI
sudo systemctl start moosefs-cgiserv
sudo systemctl enable moosefs-cgiserv

# 4. Démarrer le service de synchronisation
sudo systemctl start mtd-rsync.service

# 5. Démarrer Keepalived
sudo systemctl start keepalived
sudo systemctl enable keepalived

# 6. Vérifier la VIP
ip a | grep 192.168.25.230
# ✅ La VIP doit être présente sur NODE1

# 7. Vérifier les services
sudo systemctl status moosefs-master
sudo systemctl status moosefs-chunkserver
sudo systemctl status keepalived
```

---

### 🟡 Sur NODE2 (Backup)

```bash
# 1. S'assurer que le Master est ARRÊTÉ
sudo systemctl stop moosefs-master

# 2. Démarrer Chunkserver
sudo systemctl start moosefs-chunkserver
sudo systemctl enable moosefs-chunkserver

# 3. Démarrer CGI
sudo systemctl start moosefs-cgiserv
sudo systemctl enable moosefs-cgiserv

# 4. Démarrer le service de synchronisation
sudo systemctl start mtd-rsync.service

# 5. Démarrer Keepalived
sudo systemctl start keepalived
sudo systemctl enable keepalived

# 6. Vérifier qu'il N'Y A PAS la VIP
ip a | grep 192.168.25.230
# ❌ Ne doit RIEN afficher

# 7. Vérifier les services
sudo systemctl status moosefs-chunkserver
sudo systemctl status keepalived
```

---

### 🟢 Sur NODE3 (Backup)

```bash
# 1. S'assurer que le Master est ARRÊTÉ
sudo systemctl stop moosefs-master

# 2. Démarrer Chunkserver
sudo systemctl start moosefs-chunkserver
sudo systemctl enable moosefs-chunkserver

# 3. Démarrer CGI
sudo systemctl start moosefs-cgiserv
sudo systemctl enable moosefs-cgiserv

# 4. Démarrer le service de synchronisation
sudo systemctl start mtd-rsync.service

# 5. Démarrer Keepalived
sudo systemctl start keepalived
sudo systemctl enable keepalived

# 6. Vérifier qu'il N'Y A PAS la VIP
ip a | grep 192.168.25.230
# ❌ Ne doit RIEN afficher

# 7. Vérifier les services
sudo systemctl status moosefs-chunkserver
sudo systemctl status keepalived
```

---

## ✅ Tests et validation

### Vérification de l'état du cluster

```bash
# Vérifier quel nœud a la VIP (master actif)
# Sur chaque nœud :
ip a | grep 192.168.25.230

# Vérifier les chunkservers connectés (depuis le master actif)
mfscli -SCS

# Vérifier l'interface web
# http://192.168.25.230:9425
```

### Test de failover

#### Test 1 : Arrêt du master actif

```bash
# Sur NODE1
sudo systemctl stop moosefs-master

# Attendre 10-15 secondes

# Sur NODE2, vérifier
ip a | grep 192.168.25.230  # ✅ La VIP doit apparaître
sudo systemctl status moosefs-master  # ✅ Doit être actif

# Vérifier les logs
sudo tail -20 /var/log/syslog | grep MOOSEFS-HA
```

#### Test 2 : Rétablissement

```bash
# Sur NODE1
sudo systemctl start moosefs-master

# Attendre 10-15 secondes

# Sur NODE1, vérifier
ip a | grep 192.168.25.230  # ✅ La VIP doit revenir

# Sur NODE2, vérifier
ip a | grep 192.168.25.230  # ❌ La VIP doit disparaître
sudo systemctl status moosefs-master  # ❌ Doit être inactif
```

### Test de montage client

```bash
# Créer le point de montage
sudo mkdir -p /mnt/moosefs

# Monter le système de fichiers
sudo mfsmount /mnt/moosefs -H 192.168.25.230

# Vérifier
df -h | grep moosefs
ls -lah /mnt/moosefs

# Tester l'écriture
echo "Test MooseFS HA" > /mnt/moosefs/test.txt
cat /mnt/moosefs/test.txt
```

---

## 📊 Commandes utiles

### Monitoring

```bash
# État Keepalived
sudo systemctl status keepalived

# Logs Keepalived en temps réel
sudo tail -f /var/log/syslog | grep -E "Keepalived|MOOSEFS-HA"

# État du master
sudo systemctl status moosefs-master

# État du chunkserver
sudo systemctl status moosefs-chunkserver

# État de la synchronisation
sudo systemctl status mtd-rsync

# Vérifier la VIP
ip a | grep 192.168.25.230

# Liste des chunkservers connectés
mfscli -SCS

# Espace disque utilisé
mfscli -SHD

# Informations générales
mfscli -SIN
```

### Gestion des services

```bash
# Redémarrer tous les services MooseFS
sudo systemctl restart moosefs-*

# Redémarrer Keepalived
sudo systemctl restart keepalived

# Redémarrer la synchronisation
sudo systemctl restart mtd-rsync

# Voir les logs du master
sudo journalctl -u moosefs-master -f

# Voir les logs de synchronisation
sudo journalctl -u mtd-rsync -f
```

---

## 🐛 Troubleshooting

### La VIP ne bascule pas

**Vérifier le firewall :**

```bash
# VRRP utilise le multicast 224.0.0.18
sudo iptables -A INPUT -p vrrp -j ACCEPT
sudo iptables -A OUTPUT -p vrrp -j ACCEPT
```

**Vérifier la connectivité VRRP :**

```bash
sudo tcpdump -i vmbr0 vrrp
```

**Vérifier l'interface réseau :**

```bash
# Vérifier le nom de l'interface
ip a

# Adapter keepalived.conf si nécessaire
sudo nano /etc/keepalived/keepalived.conf
# Remplacer "vmbr0" par votre interface réelle
```

### Le master ne démarre pas

**Vérifier les métadonnées :**

```bash
ls -lah /var/lib/mfs/metadata.mfs
sudo chown -R mfs:mfs /var/lib/mfs/
```

**Consulter les logs :**

```bash
sudo journalctl -u moosefs-master -n 50
sudo tail -50 /var/log/syslog | grep mfsmaster
```

**Copier les métadonnées depuis NODE1 :**

```bash
# Sur NODE2 ou NODE3
rsync -avz --delete \
    --exclude='mfsmaster.lock' \
    --exclude='sessions.mfs*' \
    root@192.168.25.200:/var/lib/mfs/ /var/lib/mfs/

sudo chown -R mfs:mfs /var/lib/mfs/
```

### Problèmes de synchronisation rsync

**Vérifier SSH :**

```bash
# Tester la connexion
ssh root@192.168.25.200 "hostname"
ssh root@192.168.25.210 "hostname"
ssh root@192.168.25.220 "hostname"

# Vérifier les clés SSH
ls -la /root/.ssh/
chmod 700 /root/.ssh/
chmod 600 /root/.ssh/authorized_keys
```

**Vérifier /etc/ssh/sshd_config :**

```bash
sudo nano /etc/ssh/sshd_config
```

S'assurer que :

```bash
PermitRootLogin yes
PubkeyAuthentication yes
```

Redémarrer SSH :

```bash
sudo systemctl restart sshd
```

**Vérifier le service mtd-rsync :**

```bash
sudo systemctl status mtd-rsync
sudo journalctl -u mtd-rsync -n 50

# Si le service spam les connexions SSH, le redémarrer
sudo systemctl restart mtd-rsync
```

### Rate limiting SSH

Si vous voyez des erreurs `connections without attempting authentication` :

```bash
# Arrêter temporairement mtd-rsync
sudo systemctl stop mtd-rsync

# Attendre 60 secondes
sleep 60

# Redémarrer proprement
sudo systemctl start mtd-rsync
```

### Split-brain (2 masters actifs)

```bash
# 1. Arrêter Keepalived sur tous les nœuds
sudo systemctl stop keepalived

# 2. Arrêter tous les masters sauf NODE1
sudo systemctl stop moosefs-master

#
