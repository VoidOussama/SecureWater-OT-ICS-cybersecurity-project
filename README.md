# SecureWater OT

**Architecture Zero Trust segmentée, inspirée du modèle Purdue, pour la sécurisation d'une infrastructure ICS/OT simulée de traitement de l'eau.**




> Auteur : **Oussama ERREMICH**

---

## Sommaire

- [Le problème](#le-problème)
- [L'approche](#lapproche)
- [Architecture](#architecture)
- [Politique de filtrage Zero Trust](#politique-de-filtrage-zero-trust)
- [Composants déployés](#composants-déployés)
- [Résultats de validation](#résultats-de-validation)
- [Constats d'audit](#constats-daudit)
- [Reconstruire le laboratoire](#reconstruire-le-laboratoire)
- [Structure du dépôt](#structure-du-dépôt)
- [Référentiels](#référentiels)
- [Avertissement](#avertissement)

---

## Le problème

Modbus TCP, le protocole qui relie l'automate à la supervision dans la plupart des installations industrielles, a été publié en 1979 pour des réseaux fermés. Il n'intègre **aucune authentification, aucun chiffrement, aucun contrôle d'intégrité et aucune protection contre le rejeu**.

Ce n'est pas un défaut d'implémentation : c'est une propriété de sa conception. Aucun correctif ne la supprimera. Toute machine capable d'ouvrir une session TCP sur le port 502 est acceptée comme client légitime — et peut donc **lire l'état du procédé comme écrire dans ses registres**.

Il en découle une conséquence qui structure tout ce projet :

> Puisque le protocole ne sait pas distinguer un client autorisé d'un attaquant, **la seule frontière de sécurité disponible est le réseau**.

## L'approche

Quatre principes, issus du NIST SP 800-207 et du NIST SP 800-82 Rev. 3 :

| Principe | Mise en œuvre concrète |
|---|---|
| **Point de contrôle unique** | Toutes les zones sont raccordées à un seul pare-feu OPNsense. Aucune adjacence directe entre deux zones. |
| **Deny by Default** | Aucune communication n'est autorisée tant qu'une règle explicite ne l'a pas permise. Politique en liste blanche. |
| **Moindre privilège réseau** | Une autorisation = un couple source/destination + un service précis. Jamais de plage d'adresses ni de plage de ports. |
| **Présomption de compromission** | L'architecture est conçue en supposant qu'une machine est déjà aux mains d'un attaquant. La question n'est pas d'empêcher l'intrusion, mais de limiter ce qu'elle permet d'atteindre. |

## Architecture

![Architecture globale](docs/images/01-architecture-globale.png)

Quatre zones de confiance, chacune sur un réseau virtuel isolé, reliées par un pare-feu à cinq interfaces.

| Zone | Réseau | Passerelle | Contenu | Criticité |
|---|---|---|---|---|
| **IT / audit** | `192.168.10.0/24` — VMnet2 | `192.168.10.1` (LAN) | Windows 11, Kali Linux | Faible — zone la moins fiable |
| **ICS/OT** | `192.168.20.0/24` — VMnet3 | `192.168.20.1` (OPT1) | OpenPLC, FUXA, Sysmon, agent Wazuh | **Élevée** — procédé physique |
| **DMZ** | `192.168.40.0/24` — VMnet4 | `192.168.40.1` (OPT2) | Suricata, Fail2Ban, agent Wazuh | Moyenne — zone tampon |
| **SOC** | `192.168.50.0/24` — VMnet5 | `192.168.50.1` (OPT3) | Wazuh Manager, Indexer, Dashboard | Élevée — intégrité de la preuve |
| **WAN** | `192.168.95.0/24` — VMnet8 (NAT) | `192.168.95.139` (DHCP) | Accès sortant temporaire d'installation | — |

Les zones IT, DMZ et SOC sont en mode **host-only** : elles n'ont aucune connectivité vers le réseau physique. Un balayage lancé depuis Kali ne peut structurellement pas atteindre un équipement réel.

**Transposition Purdue** — la maquette compresse la pile de référence sur quatre zones :

| Niveau Purdue | Dans SecureWater OT |
|---|---|
| Niveau 0 — procédé | Réservoir et capteur de niveau simulés |
| Niveau 1 — contrôle | OpenPLC |
| Niveau 2 — supervision | FUXA |
| Niveau 3 — opérations de site | *non implémenté* |
| DMZ industrielle | Zone DMZ (Suricata, Fail2Ban) |
| Niveaux 4/5 — entreprise | Zone IT |
| Transverse | Zone SOC (Wazuh) |

Les niveaux 1 et 2 partagent le même segment : c'est un écart assumé par rapport au modèle, documenté dans [`docs/01-architecture.md`](docs/01-architecture.md).

## Politique de filtrage Zero Trust

![Politique Zero Trust](docs/images/02-politique-zero-trust.png)

| Source | Destination | Service | Action | Pourquoi |
|---|---|---|---|---|
| IT | ICS/OT | Modbus TCP `502` | **PASS** | Unique flux industriel. L'administration (8080) et la supervision (1881) restent fermées. |
| IT | DMZ | HTTPS `443` | **PASS** | Services exposés, session chiffrée. |
| ICS/OT | SOC | Wazuh `1514` | **PASS** | Remontée des journaux. Flux ascendant uniquement. |
| DMZ | SOC | Wazuh `1514` | **PASS** | Remontée des alertes. Flux ascendant uniquement. |
| Attacker | ICS/OT | tout | **BLOCK** | Fermeture du pivot direct vers l'automate. |
| Attacker | DMZ | tout | **BLOCK** | Fermeture du rebond par la zone tampon. |
| DMZ | ICS/OT | tout | **BLOCK** | Principe de la DMZ industrielle : jamais de relais vers la conduite. |
| SOC | toutes zones | tout | **BLOCK** | Le SOC reçoit, il n'initie pas. Protège l'intégrité de la preuve. |
| ICS/OT | WAN | tout | **BLOCK** | Ferme le canal de commande et de contrôle sortant. |
| ICS/OT | IT | tout | **BLOCK** | La zone critique n'initie pas vers la moins fiable. |
| `any` | `any` | tout | **BLOCK** | Règle terminale — *Deny by Default*. |

Les règles telles qu'enregistrées dans OPNsense :

![Règles ZeroTrust_Principles](docs/images/06-opnsense-regles-zerotrust.png)

Et la vérification que l'interface exposée n'accepte rien en entrée — *« No WAN rules have been defined. All incoming connections on this interface will be blocked. »* :

![WAN deny by default](docs/images/05-opnsense-wan-deny.png)

Détail complet et analyse chemin par chemin : [`docs/02-zero-trust-policy.md`](docs/02-zero-trust-policy.md).

## Composants déployés

Chaque brique observe un niveau différent de la pile — c'est ce qui en fait une défense en profondeur et non un empilement.

| Composant | Niveau observé | Répond à la question | Zone |
|---|---|---|---|
| **OPNsense** 26.7 | Réseau — L3/L4 | *Cette communication a-t-elle le droit d'exister ?* | Transverse |
| **Suricata** | Réseau — L3 à L7 | *Le contenu de ce trafic autorisé est-il malveillant ?* | DMZ |
| **Sysmon for Linux** | Système — hôte | *Que s'est-il passé sur cette machine ?* | ICS/OT |
| **Fail2Ban** | Applicatif — SSH | *Ce comportement d'authentification est-il anormal ?* | DMZ |
| **Wazuh** 4.13.1 | Corrélation | *Ces événements isolés forment-ils un incident ?* | SOC |

Retirer une couche laisse un angle mort. C'est la définition même de la défense en profondeur.

**Environnement industriel** — OpenPLC v3 comme automate programmable virtuel (interface web `:8080`, serveur Modbus TCP `:502`) et FUXA comme interface SCADA/HMI (`:1881`). Le procédé simulé est un réservoir d'eau équipé d'un capteur de niveau.

```
Capteur de niveau → OpenPLC (logique ST) → Modbus TCP :502 → FUXA (IHM) → opérateur
```

![Synoptique FUXA](docs/images/12-fuxa-synoptique-editeur.png)

## Résultats de validation

Tests conduits depuis **Kali Linux (`192.168.10.50`)**, placée dans la zone IT — la position exacte d'un poste bureautique compromis.

### Le service industriel n'est pas joignable

```bash
sudo nmap -Pn -p 502 --script modbus-discover 192.168.20.50
```

![nmap Modbus](docs/images/18-nmap-modbus-filtered.png)

Résultat : `502/tcp filtered mbap`, et le script `modbus-discover` **ne produit aucune sortie**.

### Les interfaces de supervision et d'administration non plus

```bash
sudo nmap -Pn -sV -p 1881,8080,8443 192.168.20.60
```

![nmap services](docs/images/19-nmap-services-filtered.png)

Les trois ports sont `filtered`, et la colonne `VERSION` reste vide — aucune bannière n'a pu être collectée.

### Pourquoi `filtered` est le résultat qui compte

| État nmap | Ce que ça signifie | Ce que ça dit de l'architecture |
|---|---|---|
| `open` | Réponse `SYN/ACK` | Le service est joignable → segmentation en échec |
| `closed` | Réponse `RST` | Le paquet **atteint la machine** → segmentation en échec malgré tout |
| `filtered` | Aucune réponse exploitable | **Le paquet n'atteint jamais la cible** — OPNsense l'a supprimé en chemin |

La suppression silencieuse est le comportement recherché : elle ne renvoie aucune information à l'émetteur et rend la cartographie du réseau coûteuse pour l'adversaire.

### Correspondance MITRE ATT&CK for ICS

| Technique | Objectif adverse | Traitée par |
|---|---|---|
| [`T0846`](https://attack.mitre.org/techniques/T0846/) — Remote System Discovery | Identifier les hôtes et services joignables | Segmentation + filtrage |
| [`T0861`](https://attack.mitre.org/techniques/T0861/) — Point & Tag Identification | Énumérer les registres pour cibler l'action | Port 502 non joignable depuis l'IT |
| [`T0855`](https://attack.mitre.org/techniques/T0855/) — Unauthorized Command Message | Émettre une commande vers un actionneur | Réduction des sources capables d'atteindre 502 |
| `T0832` — Manipulation of View | Falsifier l'affichage opérateur | IHM confinée en zone OT |
| `T0814` — Denial of Service | Saturer le service ou la pile réseau | **Non traité** — aucune limitation de débit |

## Constats d'audit

Un projet de sécurité qui ne publie que ses succès n'est pas un projet de sécurité. Trois écarts subsistent dans la version livrée du laboratoire, et ils sont documentés parce qu'ils sont instructifs.

### 1. La VM DMZ contourne le pare-feu

![Adressage DMZ](docs/images/09-adressage-dmz-dualhomed.png)

La machine de la DMZ porte **deux interfaces actives** :

- `ens33` → `192.168.20.100/24` — réseau **ICS/OT**
- `ens38` → `192.168.40.10/24` — réseau **DMZ**

Elle est donc adjacente à la zone industrielle au niveau 2. Le trafic qu'elle émet vers `192.168.20.0/24` emprunte directement le commutateur virtuel et **ne traverse jamais OPNsense**. La règle `DMZ → OT : BLOCK` existe dans la configuration et reste inopérante pour cette machine.

> **L'enseignement.** Cette faille ne réside dans aucune règle de pare-feu — elle réside dans la topologie. Un audit limité à la relecture de la configuration de filtrage aurait conclu à tort que le chemin était fermé.
>
> **Une politique de segmentation ne se vérifie pas sur le pare-feu, elle se vérifie sur les machines.**

*Correction :* supprimer `ens33`, ou la reconfigurer sans adresse IP en mode promiscuité si l'objectif est l'observation du trafic OT par Suricata.

### 2. Un agent Wazuh sur deux est déconnecté

![Agents Wazuh](docs/images/14-wazuh-agents.png)

L'agent `001 — dmz` (`192.168.40.10`) est en état `disconnected`. Le SOC ne reçoit donc plus les événements de la zone qui héberge Suricata et Fail2Ban — précisément celle qui devrait produire les alertes.

*Piste :* la règle `DMZ → SOC` n'autorise que le port `1514`. Le port `1515`, nécessaire à la ré-inscription d'un agent, n'est pas ouvert.

### 3. La détection est déployée mais non éprouvée

![Suricata](docs/images/16-suricata-status.png)

Suricata est actif, mais la ligne de commande observée est `suricata --af-packet -c /etc/suricata/suricata.yaml` : c'est une **capture passive, donc un fonctionnement en mode IDS**. Le mode IPS exigerait `copy-mode: ips` avec une paire d'interfaces appairées, ou une redirection vers `NFQUEUE`.

Aucune alerte n'a été collectée à ce jour. Un service actif n'est pas une détection démontrée — la distinction est faite explicitement dans [`docs/05-findings.md`](docs/05-findings.md).

Une cause structurelle mérite d'être notée : la sonde est en DMZ, or le balayage IT → OT est supprimé par le pare-feu **avant** d'atteindre la DMZ. Plus la prévention est efficace, moins la détection a de matière. La source la plus pertinente pour ce scénario n'est donc pas Suricata mais **le journal de blocage d'OPNsense remonté dans Wazuh**.

## Reconstruire le laboratoire

### Prérequis

- VMware Workstation (ou un hyperviseur équivalent)
- ~16 Go de RAM et ~200 Go de disque sur l'hôte
- Images : OPNsense 26.7 amd64, Ubuntu Server 24.04 LTS, Kali Linux, Windows 11

### 1. Réseaux virtuels

Dans *Virtual Network Editor*, créer :

| VMnet | Mode | Sous-réseau | DHCP |
|---|---|---|---|
| VMnet2 | Host-only | `192.168.10.0/24` | désactivé |
| VMnet3 | Custom | `192.168.20.0/24` | désactivé |
| VMnet4 | Host-only | `192.168.40.0/24` | désactivé |
| VMnet5 | Host-only | `192.168.50.0/24` | désactivé |
| VMnet8 | NAT | `192.168.95.0/24` | activé |

### 2. Pare-feu

Déployer OPNsense avec **cinq cartes réseau** (VMnet8, 2, 3, 4, 5 dans cet ordre), puis depuis la console :

```
1) Assign interfaces         → WAN=le0, LAN=le1, OPT1=le2, OPT2=le3, OPT3=le4
2) Set interface IP address  → 192.168.10.1 / .20.1 / .40.1 / .50.1  (statique /24)
                             → WAN : DHCP
```

![Interfaces OPNsense](docs/images/04-opnsense-interfaces.png)

### 3. Machines et composants

Les scripts d'installation sont dans [`install/`](install/) et suivent les procédures officielles des éditeurs.

```bash
git clone https://github.com/<votre-compte>/securewater-ot.git
cd securewater-ot/install
chmod +x *.sh

# Sur chaque machine Ubuntu
sudo ./00-prepare.sh

# Zone ICS/OT
sudo ./01-openplc.sh
sudo ./02-fuxa.sh
sudo ./07-sysmon-linux.sh
sudo ./04-wazuh-agent.sh 192.168.50.10

# Zone DMZ
sudo ./05-suricata.sh
sudo ./06-fail2ban.sh
sudo ./04-wazuh-agent.sh 192.168.50.10

# Zone SOC
sudo ./03-wazuh-server.sh

# Vérification sur n'importe quelle machine
sudo ./08-verify.sh
```

### 4. Règles de filtrage

Appliquer la matrice ci-dessus dans OPNsense. La procédure détaillée est dans [`firewall/README.md`](firewall/README.md).

### 5. Tests

```bash
cd tests
./run-scans.sh 192.168.20.50 192.168.20.60
```

## Structure du dépôt

```
securewater-ot/
├── README.md                      Ce document
├── docs/
│   ├── 01-architecture.md         Zones, adressage, transposition Purdue
│   ├── 02-zero-trust-policy.md    Matrice de flux et analyse chemin par chemin
│   ├── 03-risk-analysis.md        Registre de 14 risques évalués
│   ├── 04-security-tests.md       Protocole de test et résultats
│   ├── 05-findings.md             Constats d'audit et plan de correction
│   └── images/                    19 captures du laboratoire
├── install/                       Scripts d'installation par composant
├── plc/                           Programme automate (ST) — à déposer
├── scada/                         Projet FUXA et table des registres — à déposer
├── firewall/                      Configuration OPNsense — à déposer
├── soc/                           Configurations Wazuh, Suricata, Sysmon — à déposer
└── tests/                         Scripts de test de segmentation
```

Les dossiers marqués *à déposer* contiennent un `README.md` qui précise exactement quels fichiers y placer. Ils sont volontairement vides plutôt que remplis de configurations approximatives.

## Référentiels

Ces documents ont guidé la conception. Ils sont utilisés comme **cadres méthodologiques**.

- [NIST SP 800-207](https://doi.org/10.6028/NIST.SP.800-207) — *Zero Trust Architecture*
- [NIST SP 800-82 Rev. 3](https://doi.org/10.6028/NIST.SP.800-82r3) — *Guide to Operational Technology (OT) Security*
- [NIST CSF 2.0](https://doi.org/10.6028/NIST.CSWP.29) — *Cybersecurity Framework*
- [MITRE ATT&CK for ICS](https://attack.mitre.org/matrices/ics/)
- [IEC 62443](https://www.iec.ch/cyber-security) — *Security for Industrial Automation and Control Systems*
- [Modbus Application Protocol V1.1b3](https://www.modbus.org/specs.php)

## Avertissement

**Ce laboratoire n'est ni conforme, ni certifié, ni audité** au regard des normes citées. Aucune démarche de conformité n'a été engagée. Les référentiels servent de guides de conception, pas de labels.

Tous les tests de sécurité ont été menés **exclusivement** dans cet environnement virtualisé isolé, sur des machines créées pour le projet. Les réseaux des zones testées sont en mode *host-only*, sans connectivité vers un réseau physique ou vers Internet. Aucune exploitation de vulnérabilité, aucune élévation de privilèges et aucune écriture dans un registre Modbus n'ont été effectuées.

Ce dépôt est publié à des fins pédagogiques. **N'appliquez aucune de ces configurations à une installation industrielle en production sans une étude de risque préalable.** Un procédé industriel réel a des contraintes de sûreté que ce laboratoire ne reproduit pas.

## Licence

MIT — voir [LICENSE](LICENSE).
