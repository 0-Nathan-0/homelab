# Homelab OptiPlex : décisions d'architecture (ADR)

*Dernière mise à jour : 2026-11-05.*

Une fiche ADR (Architecture Decision Record) explique **pourquoi** un choix a été fait. Les fiches sont numérotées et ne sont jamais effacées : en cas de changement d'avis, une nouvelle fiche remplace l'ancienne et l'ancienne passe au statut « remplacée ». L'architecture décrite est dans [homelab-architecture.md](homelab-architecture.md).

## Index

| N° | Décision | Statut |
|---|---|---|
| ADR-001 | Hyperviseur Proxmox VE 9.2, version x86-64 | Validée |
| ADR-002 | Stack média dans une VM avec Docker | Validée |
| ADR-003 | Secure Boot conservé | Validée |
| ADR-004 | Démarrer avec 8 Go de RAM | Validée |
| ADR-005 | Système de fichiers ext4 avec LVM-thin | Validée |
| ADR-006 | Nom d'hôte pve1.home.arpa | Proposée |
| ADR-007 | SSD pour le système, HDD pour les médias | Validée |
| ADR-008 | Adressage réseau de l'hôte | Proposée |
| ADR-009 | Contrôleur SATA en mode AHCI | Validée |

## Modèle

```
### ADR-NNN : titre
- Statut : proposée / validée / remplacée par ADR-XXX
- Contexte : le problème ou la contrainte
- Décision : ce qui est choisi
- Alternatives écartées : ce qui a été envisagé, et pourquoi pas
- Conséquences : ce que ça apporte, ce que ça coûte
- Réversibilité : facile / difficile
```

---

### ADR-001 : Hyperviseur Proxmox VE 9.2, version x86-64

- **Statut** : validée.
- **Contexte** : l'objectif est de séparer des environnements (prod, test), d'avoir des snapshots et d'apprendre la virtualisation comme en entreprise, sur une seule machine à processeur Intel x86-64.
- **Décision** : installer Proxmox VE 9.2 à la place de Windows, avec l'ISO officielle x86-64 (version 9.2-1).
- **Alternatives écartées** :
  - Debian ou Ubuntu Server avec Docker directement : plus simple et plus léger, et le iGPU fonctionne sans passthrough, mais ne répond pas à l'objectif de séparer prod et test ni à celui d'apprendre la virtualisation.
  - Debian avec KVM/libvirt, XCP-ng, TrueNAS, Unraid, Incus : non retenus, Proxmox étant la référence open source accessible pour un homelab, avec interface web, VM, conteneurs, snapshots et sauvegardes.
  - ISO arm64 de Proxmox VE : publiée officiellement en août 2026, mais destinée à des processeurs ARM, elle ne démarre pas sur un i5-9500.
- **Conséquences** : une couche de complexité supplémentaire (passthrough du iGPU) et une consommation de RAM de l'hôte sur une machine à 8 Go. Sans abonnement, les dépôts par défaut sont ceux de l'offre payante : à reconfigurer après l'installation.
- **Réversibilité** : difficile une fois les VM créées, mais le déploiement par `docker-compose` rend la stack portable vers un autre support.

---

### ADR-002 : Stack média dans une VM avec Docker

- **Statut** : validée.
- **Contexte** : Jellyfin, Seerr et la stack \*arr sont déployés avec Docker. Le transcodage matériel par le iGPU (Quick Sync) est une exigence. Un LXC partagerait plus facilement le iGPU, car il partage le noyau de l'hôte, mais l'isolation y est plus faible.
- **Décision** : une VM Debian avec Docker, déployée avec un `docker-compose` versionné dans Git.
- **Alternatives écartées** :
  - Docker dans un LXC : accès au iGPU plus simple et ressources plus légères, mais isolation plus faible et dépendance au noyau de l'hôte. À notre connaissance, la documentation Proxmox déconseille cette combinaison : *à vérifier et citer avant de s'appuyer dessus.*
  - Docker directement sur l'hôte Proxmox : mélange l'hyperviseur et les applications.
  - Jellyfin seul en LXC et le reste en VM : option intermédiaire, plus complexe pour le partage des fichiers entre les deux.
- **Conséquences** : le iGPU doit être passé en exclusivité à la VM (passthrough PCI). Cela exige VT-d (activé dans le BIOS) et peut être capricieux avec un iGPU Intel. Le iGPU n'est alors plus disponible pour l'hôte.
- **Plan B** : si le passthrough échoue après une durée limite fixée à l'avance, basculer la stack vers un LXC avec le même `docker-compose`, ou vers Debian avec Docker. *Durée limite à fixer.*
- **Réversibilité** : facile, grâce au `docker-compose`.

---

### ADR-003 : Secure Boot conservé

- **Statut** : validée.
- **Contexte** : le Secure Boot est actif en mode « Deployed ». Il faut que Proxmox démarre et que les modules du noyau nécessaires au passthrough soient acceptés.
- **Décision** : conserver le Secure Boot actif et installer avec l'ISO officielle.
- **Raisons (d'après la documentation de Proxmox)** :
  - depuis Proxmox VE 8.1, le chargeur de démarrage est signé avec le certificat Secure Boot de Microsoft, donc l'ISO officielle démarre telle quelle ;
  - depuis le noyau 6.2.16-8, les modules livrés avec le noyau sont signés et acceptés par celui-ci.
- **Alternatives écartées** : désactiver le Secure Boot dans le BIOS (plus simple, mais perd la protection du démarrage, sans nécessité).
- **Conséquences** : les modules tiers non signés seraient refusés. Aucun n'est prévu.
- **Réversibilité** : facile, le Secure Boot se désactive dans le BIOS. Si cela arrive, noter la raison dans le journal d'architecture.

---

### ADR-004 : Démarrer avec 8 Go de RAM

- **Statut** : validée.
- **Contexte** : une barrette DDR4 de 8 Go, 3 emplacements libres, maximum supporté de 64 Go.
- **Décision** : commencer l'installation avec la RAM actuelle, sans achat préalable.
- **Alternatives écartées** : acheter une barrette avant de commencer (dépense avant d'avoir mesuré le besoin).
- **Conséquences** : la VM de test permanente et ZFS (cache en RAM) sont déconseillés dans cette configuration. Le budget RAM chiffré reste à produire.
- **Réversibilité** : facile. Extension possible avec une barrette identique (DDR4 2666). Penser au double canal, qui profite aussi au iGPU, et vérifier l'emplacement recommandé dans le manuel Dell.

---

### ADR-005 : Système de fichiers ext4 avec LVM-thin

- **Statut** : validée (appliquée à l'installation du 2026-11-05).
- **Contexte** : un seul SSD de 476 Gio, 8 Go de RAM.
- **Décision** : ext4, option par défaut de l'installeur, qui crée un volume LVM-thin pour les disques des VM. Découpage du disque (hdsize, swapsize, maxroot, minfree, maxvz) laissé aux valeurs par défaut de l'installeur.
- **Alternatives écartées** :
  - ZFS : sur un seul disque, il détecte la corruption de données mais ne peut pas la réparer sans redondance, et son cache consomme de la RAM par défaut.
  - Un découpage manuel du disque : aucun besoin identifié qui justifie de s'écarter des valeurs par défaut.
- **Conséquences** : pas d'auto-réparation des données. Les snapshots de VM restent disponibles via LVM-thin. Swap d'environ la taille de la RAM, utile avec 8 Go pour éviter qu'une VM soit tuée en cas de débordement, au prix d'une usure du SSD si elle est très sollicitée.
- **Réversibilité** : difficile, un changement demande de réinstaller.

---

### ADR-006 : Nom d'hôte pve1.home.arpa

- **Statut** : proposée.
- **Contexte** : le nom d'hôte apparaît dans l'interface web, les invites SSH, les baux DHCP et les fichiers de configuration. Le renommer après installation est délicat.
- **Décision proposée** : nom descriptif `pve1`, domaine `home.arpa`, suffixe réservé aux réseaux domestiques (RFC 8375).
- **Alternatives écartées** : un nom thématique (moins parlant avec plusieurs machines), le prénom (le serveur n'est pas une personne, et ça vieillit mal dans une documentation), `.local` (réservé à mDNS, source de bugs).
- **Réversibilité** : difficile.

---

### ADR-007 : SSD pour le système, HDD pour les médias

- **Statut** : validée.
- **Contexte** : le système, les services et leurs bases de données sont petits et sollicités en accès aléatoire. Les médias sont volumineux et lus de façon séquentielle.
- **Décision** : système, VM et configuration sur le SSD ; médias sur le HDD, à acheter.
- **Conséquences** :
  - les téléchargements et la bibliothèque de médias doivent être sur le même système de fichiers pour permettre les hardlinks dans la stack \*arr ;
  - le chemin des médias doit rester un paramètre modifiable, puisque le HDD n'est pas encore là ;
  - la séparation des supports répond à des enjeux de performance et de domaine de panne, pas à des « conflits d'écriture » entre VM.
- **Réversibilité** : moyenne.

---

### ADR-008 : Adressage réseau de l'hôte

- **Statut** : proposée.
- **Contexte** : le DHCP du réseau distribue 192.168.1.100 à 192.168.1.200. L'installeur Proxmox attend une configuration statique. Le serveur DNS/DHCP du réseau est un point de défaillance unique.
- **Décision proposée** : adresse 192.168.1.10/24 écrite sur l'hôte, hors de la plage dynamique, plus un bail statique côté DHCP ; passerelle 192.168.1.254 ; DNS primaire le serveur DNS du réseau, DNS secondaire la box (192.168.1.254).
- **Raisons** : l'adresse écrite sur l'hôte lui permet de rester joignable si le DHCP tombe ; le bail statique évite qu'un autre appareil prenne l'adresse ; un DNS secondaire indépendant du serveur DNS du réseau garde la résolution de noms active en cas de panne.
- **Conséquences** : lorsque la box répond en secondaire, la protection du DNS filtrant est contournée pour cette requête. Acceptable pour un serveur.
- **Réversibilité** : facile.

---

### ADR-009 : Contrôleur SATA en mode AHCI

- **Statut** : validée.
- **Contexte** : au premier lancement, l'installeur Proxmox affichait « No Hard Disk Found ». Le BIOS réglait SATA Operation sur « RAID On » (Intel RST), un mode que Windows lit avec son pilote mais pas le noyau Linux de l'installeur.
- **Décision** : passer SATA Operation en AHCI.
- **Alternatives écartées** : garder « RAID On » (le SSD reste invisible pour Linux, installation impossible).
- **Conséquences** : le SSD est détecté par l'installeur. Windows ne démarre plus avec ce réglage, sans importance puisque le disque est effacé. Aucun RAID matériel n'est utilisé, donc rien n'est perdu.
- **Réversibilité** : facile côté BIOS, mais à ne pas changer une fois Proxmox installé.
