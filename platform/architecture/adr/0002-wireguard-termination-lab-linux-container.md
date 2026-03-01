# ADR 0002 : Terminaison du tunnel WireGuard côté lab (conteneur/VM Linux dédiée)

## État
**Accepté**

## Contexte
Le laboratoire privé est situé derrière une connexion **5G** (routeur opérateur), généralement fournie via **CGNAT** (Carrier-Grade NAT), rendant l’exposition directe de services depuis le lab peu fiable/impossible.

L’architecture cible repose sur une passerelle publique (VPS en Suisse) et un tunnel **WireGuard** persistant vers le lab.  
Le lab utilise une **UniFi Dream Machine Pro (UDM Pro)** comme routeur central (routage inter-VLAN, firewall).

Un choix d’implémentation est requis : **où terminer WireGuard côté lab**, tout en maintenant une segmentation stricte (DMZ vs MGMT/INFRA) et en maximisant la reproductibilité.

## Décision
Terminer le tunnel **WireGuard site-to-site** côté lab sur un **hôte Linux dédié** exécuté sur l’infrastructure du lab, sous la forme d’un **conteneur** (ou, à défaut, d’une VM/CT).

Le VPS en Suisse demeure l’unique point d’entrée public. Le trafic public est terminé au niveau du VPS (reverse proxy/TLS), puis routé vers le lab via WireGuard.  
La **UDM Pro** conserve la responsabilité du routage inter-VLAN et du firewall ; des règles explicites sont ajoutées afin de permettre la circulation strictement nécessaire entre le concentrateur WireGuard et la **DMZ**.

## Critères de décision
- **Reproductibilité** : configuration WireGuard sous forme de fichiers standard Linux, versionnables (sans secrets), et automatisables (Ansible).
- **Portabilité** : capacité à migrer la terminaison WireGuard vers tout hôte Linux (changement de routeur/fournisseur sans refonte).
- **Auditabilité** : visibilité sur la configuration effective, logs, et durcissement (approche “security-by-design”).
- **Segmentation** : contrôle fin des flux (accès initial limité à la **DMZ**).
- **Coût** : réutilisation du compute existant (Proxmox), sans dépendance à une feature spécifique UniFi.

## Options considérées
### Option A — Terminaison WireGuard sur l’UDM Pro (non retenue)
**Motifs :**
- Automatisation “as code” moins directe qu’une config Linux versionnée.
- Dépendance accrue à l’écosystème UniFi pour une brique critique (vendor-specific).

### Option B — Terminaison WireGuard sur un conteneur/VM Linux (retenue)
**Motifs :**
- Configuration standard, portable et auditable.
- Intégration naturelle à l’outillage de montée en compétence (Linux, Ansible, observabilité, hardening).

## Justification
1. **Approche vendor-neutral** : la terminaison du tunnel est réalisée sur Linux via WireGuard, sans dépendre de capacités spécifiques d’un équipement réseau.
2. **Infra-as-code et audit** : la configuration est documentable, testable et versionnable ; les changements deviennent traçables.
3. **Granularité sécurité** : possibilité d’appliquer un durcissement système, une politique firewall locale, et une observabilité fine (logs/metrics) sur le nœud tunnel.
4. **Alignement “montée en compétences”** : renforce les compétences Linux/réseau (routing, iptables/nftables, systemd, troubleshooting).

## Conséquences

### Positives
- Configuration WireGuard portable et reproductible.
- Meilleure intégration à l’automatisation (Ansible) et à la supervision (Prometheus/Wazuh).
- Possibilité de documenter précisément les flux et les contrôles de sécurité.

### Négatives / risques
- Dépendance au compute du lab : si l’hôte (Proxmox) ou le conteneur est indisponible, le tunnel est indisponible.
- Complexité réseau accrue : ajout de règles sur l’UDM Pro (routage, firewall) pour atteindre la DMZ via le concentrateur WireGuard.
- Risque de dérive de configuration si les règles UDM Pro ne sont pas documentées et versionnées.

### Mesures de mitigation (préconisées)
- Exécuter WireGuard sur un hôte stable et minimal, avec redémarrage automatique.
- Mettre en place un healthcheck et un alerting “tunnel down”.
- Limiter strictement les réseaux accessibles via le tunnel (objectif initial : **DMZ uniquement**).
- Documenter les règles UniFi (firewall + routes) et conserver une procédure de repli (terminaison sur UDM Pro si nécessaire).

## Definition of Done (validation)
- Tunnel WireGuard **UP** entre VPS et lab avec auto-reconnexion.
- Routage fonctionnel vers la **DMZ** uniquement (pas d’accès MGMT/INFRA par défaut).
- Règles UDM Pro documentées (deny-by-default inter-VLAN, exceptions explicites).
- Supervision minimale en place (check tunnel + alerting).
- Procédure de redémarrage/récupération documentée (runbook).