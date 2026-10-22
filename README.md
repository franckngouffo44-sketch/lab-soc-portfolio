# Lab SOC maison — Active Directory (Samba) + Wazuh + simulation d'attaques

Projet personnel de portfolio réalisé dans le cadre de ma recherche d'alternance en administration réseau et sécurité informatique.

L'objectif : reproduire, à l'échelle d'un petit lab, l'infrastructure d'une entreprise (annuaire Active Directory, poste client, serveur de supervision de sécurité) puis **simuler des attaques réelles** pour vérifier que ces attaques sont bien détectées — la chaîne complète d'un mini-SOC, de l'attaque à l'alerte.

## Sommaire

- [Architecture du lab](#architecture-du-lab)
- [Contraintes techniques](#contraintes-techniques)
- [1. Virtualisation sous VMware Fusion (Apple Silicon)](#1-virtualisation-sous-vmware-fusion-apple-silicon)
- [2. Active Directory avec Samba AD](#2-active-directory-avec-samba-ad)
- [3. Jonction du poste Windows au domaine](#3-jonction-du-poste-windows-au-domaine)
- [4. Déploiement de Wazuh (SIEM)](#4-déploiement-de-wazuh-siem)
- [5. Déploiement des agents](#5-déploiement-des-agents)
- [6. Simulation d'attaques et détection](#6-simulation-dattaques-et-détection)
- [Difficultés rencontrées et résolutions](#difficultés-rencontrées-et-résolutions)
- [Ce que ce projet démontre](#ce-que-ce-projet-démontre)
- [Pistes d'amélioration](#pistes-daméliration)

## Architecture du lab

| Machine | Rôle | OS | IP |
|---|---|---|---|
| **dc1** | Contrôleur de domaine Active Directory | Ubuntu Server 24.04 ARM64 + Samba AD | 192.168.134.20 |
| **wazuh1** | Serveur de supervision (SIEM) | Ubuntu Server 24.04 ARM64 + Wazuh 4.14 | 192.168.134.30 |
| **DESKTOP-6UK9RME** | Poste client joint au domaine | Windows 11 Pro ARM64 | 192.168.134.130 |
| **Kali** | Machine d'attaque | Kali Linux ARM64 | 192.168.134.x |

Domaine Active Directory : `lab.lan` (realm Kerberos `LAB.LAN`, NetBIOS `LAB`)

```
                         ┌──────────────────┐
                         │   Kali (attaque)  │
                         └─────────┬────────┘
                                   │ nmap, Hydra (SSH / RDP)
                   ┌───────────────┼────────────────┐
                   ▼                                 ▼
        ┌─────────────────────┐          ┌─────────────────────┐
        │   dc1 — Samba AD     │◄────────►│ DESKTOP-6UK9RME      │
        │   DNS + Kerberos     │  domaine │ Windows 11 (jdupont) │
        └──────────┬───────────┘          └──────────┬───────────┘
                    │ agent Wazuh                     │ agent Wazuh
                    └────────────────┬─────────────────┘
                                     ▼
                         ┌───────────────────────┐
                         │  wazuh1 — SIEM Wazuh   │
                         │  manager + indexer +   │
                         │  dashboard             │
                         └───────────────────────┘
```

Toutes les VM tournent sous **VMware Fusion** sur un **MacBook Pro M3 (Apple Silicon)**, en réseau NAT.

## Contraintes techniques

Le choix le plus structurant du projet vient d'une contrainte matérielle : un Mac Apple Silicon ne peut virtualiser que des systèmes **ARM64**, pas x86/x64.

- **Windows Server** n'existe pas en version ARM64 stable (seulement des builds Insider Preview, retirés régulièrement des serveurs Microsoft). Après plusieurs tentatives via UUP dump, j'ai choisi une alternative pleinement fonctionnelle : **Samba Active Directory sur Ubuntu Server ARM64**, qui reproduit fidèlement un contrôleur de domaine (LDAP, Kerberos, DNS, sysvol/netlogon) et joint sans problème un client Windows.
- **Kali, Ubuntu Server, Windows 11** existent tous en version ARM64 officielle.

Cette contrainte, documentée ci-dessous, a aussi été l'occasion de montrer une capacité d'adaptation face à un blocage technique plutôt que d'abandonner le projet.

## 1. Virtualisation sous VMware Fusion (Apple Silicon)

Installation de quatre VM (Kali, Ubuntu Server ×2, Windows 11 Pro) en ARM64, réseau NAT, IP fixes via Netplan (Linux) et `Set-DnsClientServerAddress` (Windows).

![Écran clé de produit Windows 11, installation sans clé](images/01-cle-produit-windows11.png)
*Installation de Windows 11 Pro ARM64 sans clé de licence (suffisant pour un lab).*

![Écran de configuration réseau lors de l'OOBE Windows](images/02-ecran-reseau-oobe.png)
*Contournement de l'écran réseau obligatoire de Windows (`OOBE\BYPASSNRO`) pour créer un compte local sans connexion internet, le temps d'installer les pilotes VMware Tools.*

## 2. Active Directory avec Samba AD

Provisionnement du domaine `lab.lan` avec `samba-tool domain provision`, création d'une unité d'organisation, de groupes (`Comptabilite`, `IT`, `Direction`) et de quatre comptes utilisateurs de test, en respectant le principe du moindre privilège (droits attribués aux groupes, pas aux individus).

![Résultat du provisionnement du domaine Samba AD](images/03-samba-domain-provision.png)
*`samba-tool domain provision` : le domaine `lab.lan` est créé, le contrôleur devient `active directory domain controller`.*

![Création des utilisateurs et groupes AD](images/04-ad-users-groups-creation.png)
*Création des groupes et des comptes utilisateurs (`jdupont`, `mmartin`, `ldurand`, `pbernard`) via `samba-tool user create` / `group addmembers`.*

![Test Kerberos kinit et klist réussi](images/05-kerberos-kinit-klist.png)
*Vérification de l'authentification Kerberos (`kinit` + `klist`) et des partages `sysvol` / `netlogon` indispensables au fonctionnement du domaine.*

## 3. Jonction du poste Windows au domaine

Configuration du DNS du client Windows vers le contrôleur de domaine, puis jonction via `Add-Computer`.

![Test DNS et ping du contrôleur de domaine depuis Windows](images/06-windows-dns-ping-dc1.png)
*Le client Windows résout et joint `dc1.lab.lan` via le DNS du domaine avant la jonction.*

![Échec de jonction au domaine pour mot de passe incorrect](images/07-add-computer-echec-mdp.png)
*Premier échec de jonction : mot de passe administrateur oublié, résolu par réinitialisation (`samba-tool user setpassword Administrator`).*

![Jonction au domaine réussie, session jdupont ouverte](images/08-domaine-rejoint-succes.png)
*Jonction réussie et connexion d'un compte du domaine (`jdupont@lab.lan`) sur le poste Windows.*

## 4. Déploiement de Wazuh (SIEM)

Installation de Wazuh 4.14 (manager, indexer, dashboard) en mode tout-en-un sur une VM Ubuntu dédiée, avec son DNS pointé vers le contrôleur de domaine.

![Installation Wazuh terminée avec identifiants](images/09-wazuh-install-finished.png)
*Fin de l'installation Wazuh : les trois services (manager, indexer, dashboard) sont actifs.*

## 5. Déploiement des agents

Déploiement des agents Wazuh sur le poste Windows et sur le contrôleur de domaine `dc1`, pour couvrir à la fois le client et l'infrastructure d'identité.

![Agent Wazuh actif sur le poste Windows](images/10-wazuh-agent-windows-actif.png)
*L'agent Wazuh est actif sur `DESKTOP-6UK9RME` (Windows 11 Pro), visible dans le dashboard Endpoints.*

![Session utilisateur du domaine ouverte sur Windows](images/11-session-jdupont-domaine.png)
*Le poste Windows, joint au domaine, avec une session ouverte par un compte Active Directory.*

## 6. Simulation d'attaques et détection

### 6.1 Reconnaissance réseau (Nmap)

Scan du réseau du lab depuis Kali pour cartographier les machines et services exposés (étape classique de reconnaissance avant une attaque).

![Scan Nmap du réseau du lab](images/12-nmap-scan-reconnaissance.png)
*`nmap -sV -O 192.168.134.0/24` : identification des machines du lab et de leurs services.*

### 6.2 Brute-force SSH sur le contrôleur de domaine

Attaque par dictionnaire (Hydra) sur le service SSH de `dc1`, avec une liste de mots de passe de test incluant le bon mot de passe du compte `jdupont`.

![Attaque Hydra SSH réussie](images/13-hydra-ssh-bruteforce-succes.png)
*Hydra trouve le mot de passe valide de `jdupont` après plusieurs tentatives (`hydra -l jdupont -P mots.txt ssh://...`).*

![Alerte Wazuh de connexions SSH échouées](images/14-wazuh-alerte-rule60122.png)
*Détection côté Wazuh : 10 alertes `rule.id:60122` (« Logon Failure – Unknown user or bad password ») correspondant aux tentatives Hydra.*

![Statistiques Wazuh sur dc1 : échecs et réussites d'authentification](images/15-wazuh-dc1-stats-bruteforce.png)
*Vue d'ensemble de l'agent `dc1` sur 24h : échecs et réussites d'authentification, répartis entre bruit normal de l'infrastructure et tentatives de l'attaque simulée.*

### 6.3 Brute-force RDP sur le poste Windows

Même scénario sur le protocole RDP (Bureau à distance) du poste Windows, avec un test de la restriction d'accès par groupe local et de l'authentification au niveau réseau (NLA).

![Bureau à distance activé sur le poste Windows](images/16-rdp-active-test.png)
*Activation du Bureau à distance sur `DESKTOP-6UK9RME` pour le test.*

![Port RDP 3389 ouvert détecté par Nmap](images/17-nmap-port-3389-ouvert.png)
*Confirmation du port RDP (3389) ouvert avant l'attaque.*

![NLA désactivée temporairement pour le test](images/18-nla-desactive-test.png)
*Désactivation temporaire de l'authentification au niveau réseau (NLA) — un réglage de lab à ne jamais désactiver en production, qu'il convient de réactiver après le test.*

![Alerte Wazuh de connexion RDP détectée sur Windows](images/19-wazuh-rdp-bruteforce-detection.png)
*Côté Wazuh (journaux d'évènements Windows), détection des tentatives échouées (`rule.id:60122`, Event ID 4625) et d'une tentative de connexion NTLM suspecte (`rule.id:92657`).*

### 6.4 Vue d'ensemble du SIEM

![Dashboard global Wazuh avec mapping MITRE ATT&CK](images/20-wazuh-dashboard-global-mitre.png)
*Vue d'ensemble sur les trois agents (dc1, wazuh1, Windows) : 1 298 évènements sur 24h, et une répartition MITRE ATT&CK qui fait apparaître clairement les tactiques simulées (`Brute Force`, `Valid Accounts`, `Password Guessing`, `SSH`, `Account Access Removal`).*

## Difficultés rencontrées et résolutions

Documenter les problèmes rencontrés fait partie du projet : un lab qui fonctionne du premier coup n'apprend pas grand-chose.

| Problème | Cause | Résolution |
|---|---|---|
| Windows Server indisponible en ARM64 stable | Apple Silicon ne virtualise pas le x86 ; builds Insider retirés par Microsoft | Bascule sur Samba Active Directory (fonctionnellement équivalent pour un lab) |
| ISO généré en amd64 au lieu d'arm64 | Mauvaise sélection du build sur UUP dump | Vérification systématique de l'architecture dans le nom du fichier avant utilisation |
| `dpkg: erreur... disque plein` pendant l'installation Wazuh | Disque de la VM sous-dimensionné (20 Go) pour manager + indexer + dashboard | Agrandissement du disque à 50 Go et extension de la partition (`growpart` + `resize2fs`) |
| Dérive régulière de l'horloge des VM | VM suspendues plutôt qu'éteintes, horloge non synchronisée | `timedatectl set-ntp`, resynchronisation manuelle ; point de vigilance car Kerberos refuse plus de 5 min d'écart |
| Échecs répétés de `samba-tool` (permissions, fichiers introuvables) | Fautes de frappe dans des commandes longues, fichiers mal nommés, actions lancées sans `sudo` | Copier-coller systématique des commandes, vérification du nom exact des fichiers (`ls`, complétion Tab) |
| Mot de passe administrateur AD oublié | — | Réinitialisation via `samba-tool user setpassword Administrator` depuis le contrôleur de domaine |
| Hydra ne testait qu'un seul mot de passe | Confusion entre l'option `-p` (un seul mot de passe) et `-P` (fichier de mots de passe) | Relecture de la documentation de l'outil avant de conclure à un échec de l'attaque |
| Mot de passe RDP valide mais connexion refusée | Compte utilisateur standard, non membre du groupe local « Utilisateurs du Bureau à distance » | Mis en évidence comme un exemple de défense en profondeur (mot de passe correct ≠ accès accordé) |

## Ce que ce projet démontre

- **Administration système et réseau** : installation et configuration de serveurs Linux (Ubuntu Server), configuration réseau statique (Netplan), gestion DNS.
- **Active Directory** : provisionnement d'un domaine, gestion des utilisateurs/groupes/OU, Kerberos, jonction de postes clients.
- **Sécurité défensive (Blue Team)** : déploiement d'un SIEM (Wazuh), configuration d'agents, lecture et interprétation d'alertes, règles de corrélation.
- **Sécurité offensive (Red Team, dans un cadre contrôlé)** : reconnaissance réseau (Nmap), attaques par force brute (Hydra) sur SSH et RDP.
- **Méthodologie** : documentation systématique des problèmes rencontrés et de leur résolution, adaptation à une contrainte matérielle imprévue (ARM64), hygiène de base (snapshots avant chaque étape risquée, mots de passe de test séparés des mots de passe personnels).

## Pistes d'amélioration

- Règle de corrélation personnalisée pour détecter un scan de ports (actuellement un simple Nmap ne génère pas d'alerte dédiée).
- Second contrôleur de domaine pour tester la haute disponibilité de l'AD.
- Intégration d'un pare-feu (pfSense) en amont du lab pour segmenter les réseaux.
- Automatisation des tests d'attaque et de la remontée d'IOC via les scripts Python développés en parallèle de ce projet.

---

*Lab réalisé en octobre 2026 dans le cadre de ma recherche d'alternance en administration réseau et cybersécurité. N'hésitez pas à me contacter pour toute question sur la méthodologie ou les choix techniques.*
