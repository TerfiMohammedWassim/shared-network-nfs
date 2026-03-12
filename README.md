# 🌐 Infrastructure Réseau Segmentée — NFS, VLAN, LACP, iptables

> **Master 1 Computer & Network Systems — Université d'Évry Paris-Saclay**  
> Automatisation avec Ansible | Cumulus Linux | Debian | NFS | VLAN | Bonding 802.3ad

---

## 👥 Auteurs

| Nom | Numéro étudiant |
|-----|----------------|
| Terfi Mohammed Wassim | 20235148 |
| Andrii Kostiuk | 20253848 |

---

## 📋 Objectifs du Projet

- 🔒 **Isoler** complètement le trafic entre Machine1 (Service A) et Machine2 (Service B) via des VLANs distincts
- 📁 **Partager** un serveur de fichiers NFS accessible en lecture/écriture depuis les deux machines
- ⚡ **Assurer la haute disponibilité** du serveur NFS via une agrégation de liens LACP (bonding 802.3ad)
- 🔀 **Confiner** le routage et le firewall de niveau 3 exclusivement au Switch3
- 🤖 **Automatiser** l'ensemble de la configuration via des playbooks Ansible

---

## 🏗️ Architecture Réseau

```
                        ┌─────────────────┐
                        │    Switch3 (L3) │
                        │  VLAN10: .10.254│
                        │  VLAN20: .20.254│
                        │    + iptables   │
                        └────────┬────────┘
                    trunk        │        trunk
                 VLAN10+20       │     VLAN10+20
          ┌──────────────────────┴──────────────────┐
          │                                         │
   ┌──────┴──────┐                         ┌────────┴────┐
   │  Switch1    │                         │  Switch2    │
   │    (L2)     │                         │    (L2)     │
   └──┬──────┬───┘                         └──────┬──────┘
      │      │                                    │
   bond0   swp4                                 swp1
  (LACP)  VLAN10                               VLAN20
      │      │                                    │
┌─────┴──┐ ┌─┴────────┐                  ┌────────┴───┐
│Serveur │ │ Machine1 │                  │  Machine2  │
│  NFS   │ │VLAN 10   │                  │  VLAN 20   │
│.10.10  │ │.10.3     │                  │  .20.3     │
└────────┘ └──────────┘                  └────────────┘
```

---

## 📡 Plan d'adressage IP

| Équipement | Interface | Adresse IP | Masque | VLAN |
|------------|-----------|------------|--------|------|
| Machine1 (Service A) | ens33 | 192.168.10.3 | /24 | VLAN 10 |
| Machine2 (Service B) | ens33 | 192.168.20.3 | /24 | VLAN 20 |
| Serveur NFS | bond0 (ens33+ens34) | 192.168.10.10 | /24 | VLAN 10 |
| Switch3 GW VLAN10 | vlan10 (SVI) | 192.168.10.254 | /24 | VLAN 10 |
| Switch3 GW VLAN20 | vlan20 (SVI) | 192.168.20.254 | /24 | VLAN 20 |

---

## 📁 Structure du Projet

```
.
├── README.md
├── machine1.yml          # Playbook Machine1 — VLAN 10
├── machine2.yml          # Playbook Machine2 — VLAN 20
├── machineServer.yml     # Playbook Serveur NFS (bonding LACP + exports)
├── switch1.yml           # Playbook Switch1 — L2, bond LACP + VLAN 10
├── switch2.yml           # Playbook Switch2 — L2, VLAN 20
└── switch3.yml           # Playbook Switch3 — L3, routage + iptables
```

---

## ⚙️ Technologies Utilisées

| Technologie | Usage |
|-------------|-------|
| **Ansible** | Automatisation de la configuration |
| **Cumulus Linux (NCLU)** | Configuration des switches L2/L3 |
| **Debian** | OS des machines et du serveur NFS |
| **NFS (nfs-kernel-server)** | Partage de fichiers réseau |
| **Bonding 802.3ad (LACP)** | Haute disponibilité du serveur NFS |
| **iptables** | Isolation inter-VLAN sur Switch3 |
| **VLAN (802.1Q)** | Segmentation réseau |

---

## 🚀 Déploiement

### Prérequis

- Ansible installé sur la machine de contrôle
- Accès SSH aux machines et switches
- Fichier `inventory` configuré avec les hôtes : `machine1`, `machine2`, `nfs_server`, `Switch1`, `Switch2`, `Switch3`

### Lancer les playbooks

```bash
# Configurer les machines Debian
ansible-playbook machine1.yml
ansible-playbook machine2.yml

# Configurer le serveur NFS
ansible-playbook machineServer.yml

# Configurer les switches (ordre important)
ansible-playbook switch1.yml
ansible-playbook switch2.yml
ansible-playbook switch3.yml
```

### Tout déployer en une commande

```bash
ansible-playbook machine1.yml machine2.yml machineServer.yml switch1.yml switch2.yml switch3.yml
```

---

## 🔒 Isolation Inter-VLAN (iptables sur Switch3)

Switch3 est le seul équipement L3. Les règles iptables suivantes sont appliquées sur la chaîne `FORWARD` :

```bash
# Autoriser les connexions établies (réponses retour NFS → VLAN20)
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Autoriser VLAN10 → Serveur NFS
iptables -A FORWARD -i vlan10 -d 192.168.10.10 -j ACCEPT

# Autoriser VLAN20 → Serveur NFS
iptables -A FORWARD -i vlan20 -d 192.168.10.10 -j ACCEPT

# Bloquer VLAN10 → VLAN20 (isolation Machine1 / Machine2)
iptables -A FORWARD -i vlan10 -d 192.168.20.0/24 -j DROP

# Bloquer VLAN20 → VLAN10
iptables -A FORWARD -i vlan20 -d 192.168.10.0/24 -j DROP
```

### Tableau des flux

| Source | Destination | Résultat |
|--------|-------------|----------|
| Machine1 (VLAN10) | Serveur NFS (192.168.10.10) | ✅ AUTORISÉ |
| Machine2 (VLAN20) | Serveur NFS (192.168.10.10) | ✅ AUTORISÉ |
| Serveur NFS | Machine2 (réponse) | ✅ AUTORISÉ (ESTABLISHED) |
| Machine1 (VLAN10) | Machine2 (VLAN20) | ❌ BLOQUÉ |
| Machine2 (VLAN20) | Machine1 (VLAN10) | ❌ BLOQUÉ |

---

## 📂 Montage NFS

Sur Machine1 et Machine2 :

```bash
# Installation du client
apt install nfs-common -y

# Montage
mkdir -p /mnt/nfs
mount -t nfs 192.168.10.10:/srv/partage /mnt/nfs

# Persistance au reboot (/etc/fstab)
echo "192.168.10.10:/srv/partage  /mnt/nfs  nfs  defaults,_netdev  0  0" >> /etc/fstab
```

### Test lecture/écriture

```bash
# Depuis Machine1
echo "Fichier créé par Machine1" > /mnt/nfs/machine1.txt

# Depuis Machine2
echo "Fichier créé par Machine2" > /mnt/nfs/machine2.txt

# Vérification croisée (Machine1 lit le fichier de Machine2)
cat /mnt/nfs/machine2.txt
```

---

## 🐛 Problèmes Rencontrés et Solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| Ping gateway impossible | Mauvais câblage VLAN dans VMware | Correction du segment réseau VMware |
| Machine2 ne ping pas le NFS | Gateway absente sur le serveur NFS | Ajout `gateway 192.168.10.254` dans `/etc/network/interfaces` |
| ACL Cumulus refusées | Cumulus VX ne supporte pas les ACL hardware | Utilisation d'`iptables` directement |
| Machine2 perd l'accès NFS après iptables | La règle DROP bloquait aussi les réponses NFS | Ajout de la règle `ESTABLISHED,RELATED` en premier |
| Machine1 et Machine2 se pinguent | Switch3 route par défaut entre tous les VLANs | Règles iptables DROP sur les flux inter-VLAN |
| IP Machine2 incorrecte (`192.168.1.1`) | Erreur dans la config initiale | Correction à `192.168.20.3` |

---

## 📝 Notes

> **Cumulus VX** : Les ACL hardware (`cl-acltool`) ne sont pas disponibles sur la version virtualisée. Les règles iptables sont appliquées directement sur le noyau Linux et doivent être rendues persistantes via `netfilter-persistent save` ou un script dans `/etc/network/if-up.d/`.

> **Bonding LACP** : Le module `bonding` doit être chargé au boot via `/etc/modules`. En cas de panne d'une interface (ens33 ou ens34), le trafic bascule automatiquement sur l'interface restante.

---

*Université d'Évry Paris-Saclay — Master 1 Computer & Network Systems — Mars 2026*
