# YMMO : Infrastructure réseau sécurisée (lab)

Projet de B2 cybersécurité (Ynov Campus) : conception et déploiement, en environnement VMware, de l'infrastructure d'une agence immobilière fictive, **YMMO** (siège à Aix-en-Provence, 12 agences).

> Projet réalisé en lab. Ma contribution : la réalisation technique de l'ensemble de l'infrastructure décrite ci-dessous.

## Objectif

Mettre en place un réseau **sécurisé et segmenté** reliant un siège (~30 postes, 2 serveurs) à 12 agences (~5 postes chacune), avec gestion centralisée des identités, filtrage inter-services et accès distant chiffré.

## Architecture

![Schéma réseau](docs/schema-reseau.png)

| VLAN | Pôle | Réseau | Passerelle (VyOS) |
|------|------|--------|-------------------|
| 10 | Direction | 192.168.10.0/24 | 192.168.10.1 |
| 20 | Commercial (+ serveur) | 192.168.20.0/24 | 192.168.20.1 |
| 30 | RH / Marketing | 192.168.30.0/24 | 192.168.30.1 |
| 40 | IT / Support | 192.168.40.0/24 | 192.168.40.1 |
| 50 | Agences | 192.168.50.0/24 | 192.168.50.1 |

VPN distant : pool `10.10.10.0/24`.

## Ce qui a été déployé

| Service | Rôle | Détails |
|---------|------|---------|
| **Active Directory** | Identités centralisées | Domaine `ymmo.local`, 5 OU (Direction, Commercial, Comm-Mktg, Admin-RH, IT-Support), comptes au format `initiale.nom` |
| **GPO** | Politique de sécurité | Politique de mots de passe (8 caractères min., complexité, durée max. 90 jours), fond d'écran imposé, blocage du Panneau de configuration. Vérification avec `rsop` |
| **DHCP** | Adressage automatique | Une étendue par VLAN, bail de 8 jours, **DHCP relay** sur VyOS vers le serveur Windows |
| **DNS** | Résolution interne | Hébergé sur le contrôleur de domaine, distribué via DHCP |
| **Routeur VyOS** | Routage et filtrage | Sous-interfaces VLAN (routage inter-VLAN), route par défaut vers le WAN |
| **Firewall / ACL** | Isolation des pôles | Règles `forward filter` protégeant la Direction et isolant RH et Commercial |
| **VPN** | Accès distant | PPTP (voir *Limites*) |
| **IIS** | Site web interne | Site YMMO servi en HTTP sur le réseau interne |
| **Sauvegarde** | Continuité | Windows Server Backup, sauvegarde complète quotidienne sur disque dédié |
| **Snapshots VMware** | Reprise en lab | Un snapshot à chaque étape validée (serveur, client, routeur) |

### Règles de filtrage inter-VLAN

| Règle | Source | Destination | Action |
|-------|--------|-------------|--------|
| 10 | Commercial | Direction | drop |
| 20 | RH | Commercial | drop |
| 30 | Commercial | RH | drop |
| 40 | IT | Direction | drop |
| 50 | Agences | Direction | drop |

Les configurations VyOS sont dans [`configs/`](configs/) (mots de passe remplacés par des placeholders).

## Tests effectués

- Routage inter-VLAN : poste VLAN 10 vers serveur VLAN 20 (succès)
- ACL : VLAN 20 (Commercial) vers VLAN 10 (Direction) : bloqué
- GPO appliquées à l'utilisateur de test, vérifiées avec `rsop`
- Jonction d'un poste client au domaine et obtention d'une IP via DHCP relay

## Limites connues et pistes d'amélioration

Ce que j'ai identifié comme insuffisant pour un environnement de production :

- **VPN PPTP** : le tunnel IPsec site-à-site prévu n'a pas pu monter, VMware bloquant ESP dans mon lab. PPTP est obsolète et cryptographiquement faible. À remplacer par IPsec (IKEv2) ou WireGuard hors VMware.
- **Firewall en `default accept`** : à passer en *default deny* avec des autorisations explicites, et à aligner complètement sur la matrice des droits.
- **IIS en HTTP** : ajouter HTTPS (certificat TLS) et l'authentification Windows intégrée.
- **Absence de supervision, WSUS, IDS/IPS (Suricata) et MFA** : évolutions prévues dans la spécification.
- **Extension cloud AWS** (VPC, EC2, RDS, S3, VPN site-à-site) : étudiée et chiffrée en **conception uniquement**, non déployée.

## Compétences mises en œuvre

Active Directory · GPO · DHCP / DHCP relay · DNS · VLAN et routage inter-VLAN · VyOS · Filtrage et ACL · VPN · Windows Server · IIS · Sauvegarde et restauration · Segmentation réseau · Moindre privilège · Documentation technique

## Documentation

La spécification technique complète (captures, configurations, politique de sécurité, budgétisation) est dans [`docs/ymmo_spec_infra.pdf`](docs/ymmo_spec_infra.pdf).

## Structure du dépôt

```
ymmo-infra-securisee/
├── README.md
├── docs/
│   ├── ymmo_spec_infra.pdf
│   └── schema-reseau.png
└── configs/
    ├── vyos-siege.txt
    ├── vyos-agence.txt
    └── vyos-firewall.txt
```
