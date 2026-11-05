# Homelab OptiPlex 7070 SFF : architecture

*Dernière mise à jour : 2026-11-05. Statut : avant installation de Proxmox.*

Les décisions et leurs justifications sont dans [homelab-adr.md](homelab-adr.md). Ce document décrit **l'état cible** et **l'état actuel**.

> Périmètre : l'OptiPlex uniquement. Le serveur DNS/DHCP du réseau est une dépendance externe, hors périmètre.

Légende : ✅ en place ou décidé · 🟡 proposé, à confirmer · ⬜ à faire

---

## 1. Objectifs

- Héberger Jellyfin, Seerr et la stack \*arr, en local uniquement (rien d'exposé sur Internet).
- Travailler comme en environnement professionnel : accès SSH par clé, séparation prod/test, documentation tenue à jour.
- Produire un projet présentable sur un CV.

---

## 2. Vue d'ensemble (état cible)

```
Réseau local 192.168.1.0/24
        │
        │ Ethernet
┌───────┴──────────────────────────────────────────┐
│ Dell OptiPlex 7070 SFF                           │
│  Proxmox VE 9.2 (x86-64), hôte pve1.home.arpa    │
│  IP 192.168.1.10                                 │
│                                                  │
│   ┌────────────────────────┐   ┌──────────────┐  │
│   │ VM de prod (Debian)    │   │ VM de test   │  │
│   │  Docker                │   │ (plus tard)  │  │
│   │   Jellyfin, Seerr, *arr│   │              │  │
│   │  iGPU en passthrough   │   │              │  │
│   └────────────────────────┘   └──────────────┘  │
│                                                  │
│  SSD : système, VM, configuration                │
│  HDD : médias (à acheter)                        │
└──────────────────────────────────────────────────┘
```

---

## 3. Matériel

| Élément | Valeur |
|---|---|
| Machine | Dell OptiPlex 7070 SFF (achat Leboncoin) |
| Processeur | Intel Core i5-9500, 3,00 GHz |
| Graphique intégré | Intel UHD Graphics 630 (Quick Sync) |
| RAM | 8 Go DDR4 DIMM 2666 MT/s, 1 barrette (DIMM1) |
| Emplacements RAM | 4 au total, 3 libres, maximum supporté 64 Go |
| Stockage actuel | 1 SSD de 512 Go (affiché 476 Gio par l'installeur) |
| Stockage prévu | 1 HDD pour les médias, pas encore acheté |
| BIOS | Dell 1.35.0, à jour au 2026-11-05 |
| Virtualization (BIOS) | Activée |
| VT for Direct I/O (BIOS) | Activé (nécessaire au passthrough du iGPU) |
| Secure Boot | Activé, mode « Deployed » |
| SATA Operation (BIOS) | AHCI (était « RAID On » : le SSD était invisible pour l'installeur) |
| Réseau | Ethernet, IPv4 uniquement |
| OS actuel | Windows, sera effacé par l'installation |

---

## 4. Hyperviseur

| Paramètre | Valeur | Statut |
|---|---|---|
| Produit | Proxmox VE 9.2 | ✅ |
| Architecture | x86-64 (ISO « 9.2-1 », pas la variante arm64) | ✅ |
| Démarrage | UEFI, Secure Boot actif | ✅ |
| Système de fichiers | ext4 avec LVM-thin | ✅ |
| Nom d'hôte | pve1.home.arpa | 🟡 |
| Accès d'administration | Interface web (port 8006) et SSH par clé | ⬜ |
| Compte root | Phrase de passe stockée dans le gestionnaire de mots de passe (jamais dans la doc) | ✅ |
| E-mail de notification | Adresse personnelle réellement consultée | ✅ |

---

## 5. Réseau

| Paramètre | Valeur | Statut |
|---|---|---|
| Plage DHCP dynamique du réseau | 192.168.1.100 à 192.168.1.200 | ✅ |
| Passerelle (box) | 192.168.1.254 | ✅ |
| Adresse de l'hôte | 192.168.1.10/24, écrite sur l'hôte | 🟡 |
| DNS primaire | serveur DNS du réseau | 🟡 |
| DNS secondaire | 192.168.1.254 (box), indépendant du serveur DNS du réseau | 🟡 |
| Interface de gestion | nic0, carte Intel (pilote e1000e) | ✅ |
| Nommage des interfaces | Épinglage activé (option par défaut de l'installeur), nom lié à l'adresse MAC | ✅ |
| Adresse MAC de la carte réseau | e4:54:e8:6b:0c:01 | ✅ |
| Bail statique côté DHCP | *à créer après l'installation* | ⬜ |

### Plan d'adressage

| Adresse | Nom | Rôle |
|---|---|---|
| 192.168.1.10 | pve1 | Hôte Proxmox |
| *à définir* | *à définir* | VM de prod |
| *à définir* | *à définir* | VM de test |

Règle : les adresses de l'infrastructure sont hors de la plage dynamique et réservées côté DHCP.

### Dépendance au serveur DNS/DHCP du réseau

- Le DHCP tombant en panne n'affecte pas l'hôte : son adresse est écrite sur lui.
- Le DNS tombant en panne n'affecte l'hôte qu'à moitié : la box prend le relais en secondaire.
- Les VM qui obtiennent leur adresse par DHCP dépendent du serveur DHCP : prévoir des adresses fixes ou des réservations.

---

## 6. Stockage

| Volume | Support | Contenu | Statut |
|---|---|---|---|
| Système et VM | SSD | Proxmox, disques des VM, configuration des services | ✅ |
| Médias | HDD | Bibliothèque Jellyfin et téléchargements | ⬜ (disque non acheté) |

### Découpage du SSD (installeur Proxmox)

Tous les champs avancés laissés vides : l'installeur applique ses valeurs par défaut.

| Paramètre | Rôle | Valeur |
|---|---|---|
| hdsize | Espace du SSD utilisé | 476 Gio (tout le disque) |
| swapsize | Swap, débordement de la RAM sur disque | Défaut (taille de la RAM, entre 4 et 8 Gio) |
| maxroot | Volume système (Proxmox, ISO, templates) | Défaut (environ un quart du disque, plafonné) |
| minfree | Espace libre réservé dans le groupe LVM | Défaut (plafonné à 16 Gio) |
| maxvz | Volume `data` en LVM-thin, disques des VM | Défaut (le reste) |

Valeurs réelles à relever après installation avec `lsblk` et `lvs` : *à compléter*.

Contrainte à respecter pour la stack \*arr : les téléchargements et la bibliothèque doivent être sur le **même système de fichiers** pour permettre les hardlinks. Organisation des dossiers : *à définir* (voir questions ouvertes).

---

## 7. Services (état cible)

| Service | Rôle | Où | Statut |
|---|---|---|---|
| Jellyfin | Lecture des médias, transcodage matériel Quick Sync | VM de prod, Docker | ⬜ |
| Seerr | Demandes de médias | VM de prod, Docker | ⬜ |
| Stack \*arr | Gestion de la bibliothèque | VM de prod, Docker | ⬜ |

Déploiement par un `docker-compose` versionné dans Git. L'accès est local uniquement.

---

## 8. Budget RAM ⬜

| Composant | RAM estimée | Source |
|---|---|---|
| Hôte Proxmox | *à chiffrer* | |
| VM de prod (Debian, Docker) | *à chiffrer* | |
| Jellyfin | *à chiffrer* | |
| Stack \*arr et client de téléchargement | *à chiffrer* | |
| VM de test | *à chiffrer* | |
| iGPU (mémoire partagée) | *à chiffrer* | |
| **Total** | | |

Extension possible : une barrette identique (DDR4 2666) dans un des 3 emplacements libres.

---

## 9. Checklist avant installation

- [x] Relevé du matériel (CPU, RAM, BIOS)
- [x] BIOS à jour
- [x] Virtualization et VT for Direct I/O activés
- [x] Décision sur le Secure Boot
- [x] Rien à récupérer sur le Windows actuel
- [x] ISO Proxmox VE x86-64 officielle téléchargée (version 9.2-1)
- [ ] Empreinte SHA256 de l'ISO vérifiée avec celle de la page officielle
- [x] Clé USB prête et démarrage réussi en UEFI
- [x] SATA Operation passé en AHCI
- [ ] 192.168.1.10 libre (pas de réponse au ping)
- [ ] Réseau bien en /24

---

## 10. Après l'installation

- [ ] Accéder à l'interface web (`https://192.168.1.10:8006`)
- [ ] Créer le bail statique (MAC e4:54:e8:6b:0c:01 → 192.168.1.10)
- [ ] Configurer un relais SMTP pour que les alertes e-mail de Proxmox arrivent vraiment
- [ ] Accès SSH par clé, puis désactiver l'authentification par mot de passe
- [ ] Configurer les dépôts de mises à jour sans abonnement, mettre à jour l'hôte
- [ ] Vérifier l'état du Secure Boot et de l'IOMMU depuis Linux
- [ ] Tester le passthrough du iGPU
- [ ] Créer la VM de prod, installer Docker
- [ ] Déployer Jellyfin, Seerr et \*arr via `docker-compose` versionné
- [ ] Créer la VM de test
- [ ] Mettre en place des sauvegardes et tester une restauration

---

## 11. Critères de réussite

- Jellyfin transcode un film 4K HEVC en temps réel, avec le transcodage matériel affiché dans ses statistiques.
- Accès SSH par clé uniquement.
- L'environnement de test se recrée à partir des fichiers versionnés, sans copie manuelle de la prod.
- Documentation à jour, avec la raison de chaque choix.

---

## 12. Questions ouvertes

- Codecs gérés en matériel par le iGPU UHD 630 (décodage et encodage) : à vérifier sur la fiche Intel ARK et dans la documentation Jellyfin.
- Organisation des dossiers sur le HDD pour les hardlinks.
- Ce qui doit rester identique entre prod et test, et ce qui doit différer (données, secrets, ports).
- Stratégie de sauvegarde : quoi, où, à quelle fréquence.
- Plan B si le passthrough du iGPU échoue, avec une durée limite fixée à l'avance.

---

## 13. Journal

- **2026-11-05** : relevé du matériel, vérification du BIOS, choix de Proxmox, de la VM avec Docker pour la stack média, du Secure Boot conservé et de la version x86-64 de l'ISO. Préparation de l'installation.
- **2026-11-05, 14 h** : premier lancement de l'installeur, erreur « No Hard Disk Found ». Cause : contrôleur SATA en mode « RAID On » (Intel RST), non lu par le noyau Linux de l'installeur. Correction : SATA Operation passé en AHCI dans le BIOS (Windows ne démarre plus, sans conséquence). Installation relancée avec ext4 et le découpage par défaut du SSD (476 Gio).
