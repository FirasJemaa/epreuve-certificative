# Active Directory — Fondamentaux

## L'analogie de l'immeuble de bureaux

Imagine une grande entreprise avec 500 employés. Sans organisation :

- Chaque employé a son propre badge pour chaque porte
- Le service IT doit gérer 500 × N badges
- Quelqu'un quitte l'entreprise → il faut retrouver tous ses badges

C'est le chaos. Alors on crée **un système centralisé** : une **réception unique** qui gère qui a le droit d'aller où.

**C'est exactement ça, l'Active Directory.**

> L'AD est un service Microsoft qui centralise la **gestion des identités et des accès** dans un réseau d'entreprise.

---

## Les composants fondamentaux

### Le Domaine
C'est le périmètre de gestion. Tous les objets (users, machines, imprimantes...) appartiennent à un domaine. Il a un nom DNS : `entreprise.local`

### Le Contrôleur de Domaine (DC)
C'est **le serveur le plus important** du réseau. Il fait tourner l'AD. C'est lui qui :

- Authentifie les utilisateurs
- Stocke tous les comptes (dans `NTDS.dit`)
- Applique les politiques de sécurité (GPO)
- Héberge Kerberos (port 88) et LDAP (port 389)

!!! danger "Pour un attaquant"
    **Compromettre le DC = game over.** Il contrôle tout le domaine.

### Les objets AD
L'AD contient des objets de plusieurs types :

| Objet | Description | Intérêt attaquant |
|-------|-------------|-------------------|
| **User** | Un compte utilisateur (Alice, Bob...) | Cible principale |
| **Computer** | Une machine membre du domaine | Pivot potentiel |
| **Group** | Un groupe d'utilisateurs (ex: "Domain Admins") | Élévation de privilèges |
| **GPO** | Une politique appliquée à des machines/users | Vecteur de persistance |
| **OU** | Un dossier pour organiser les objets | Structure de l'AD |
| **SPN** | Service Principal Name — lie un service à un compte | Cible Kerberoasting |

---

## Pourquoi l'AD est une cible prioritaire en pentest

Parce que dans 90% des entreprises :

- Tout passe par l'AD (authentification, accès aux fichiers, emails...)
- Un compte **Domain Admin** donne accès à **tout**
- Les AD sont souvent **mal configurés** car complexes à maintenir
- Des vulnérabilités existent **par design** dans les protocoles NTLM et Kerberos

---

## La méthodologie Red Team sur un AD

Voici **l'ordre logique d'une attaque AD** :

```
[1] Reconnaissance
        ↓
[2] Accès Initial (obtenir un premier compte)
        ↓
[3] Énumération (comprendre l'environnement)
        ↓
[4] Escalade de privilèges
        ↓
[5] Mouvement latéral
        ↓
[6] Compromission totale (Domain Admin)
```

---

## Fichiers critiques sur le DC

| Fichier | Chemin | Contenu |
|---------|--------|---------|
| `NTDS.dit` | `C:\Windows\NTDS\NTDS.dit` | **Tous les comptes AD + hashes NTLM** |
| `SAM` | `C:\Windows\System32\config\SAM` | Comptes locaux + hashes locaux |
| `SYSTEM` | `C:\Windows\System32\config\SYSTEM` | Clé pour déchiffrer SAM/NTDS |

---

## Ports qui trahissent un AD

| Port | Service | Ce que ça signifie |
|------|---------|-------------------|
| **88** | Kerberos | **C'est un DC** (port signature) |
| **389** | LDAP | Annuaire AD interrogeable |
| **636** | LDAPS | LDAP chiffré |
| **445** | SMB | Partages fichiers, mouvement latéral |
| **3268** | Global Catalog | DC principal du domaine |
| **3269** | Global Catalog SSL | DC principal chiffré |
| **53** | DNS | Résolution de noms AD |
| **5985** | WinRM | Admin à distance (PowerShell remoting) |
| **3389** | RDP | Bureau à distance |
| **464** | Kpasswd | Changement de mot de passe Kerberos |

!!! important "Règle d'or"
    Si tu vois le **port 88 ouvert** sur une machine → c'est ton contrôleur de domaine. **C'est ta cible prioritaire.**

---

## Groupes privilégiés à cibler

| Groupe | Niveau de privilège | Pourquoi c'est une cible |
|--------|--------------------|-----------------------|
| **Domain Admins** | Maximum | Admin sur tout le domaine |
| **Enterprise Admins** | Maximum | Admin sur toute la forêt |
| **Backup Operators** | Élevé | Peut lire NTDS.dit |
| **Account Operators** | Élevé | Peut créer/modifier des comptes |
| **Server Operators** | Élevé | Peut se connecter aux DC |
| **Print Operators** | Élevé | Peut charger des drivers |
| **DNSAdmins** | Élevé | Peut injecter une DLL dans le service DNS |

---

## Schéma de l'architecture AD

```
                    [Forêt]
                       |
              [Domaine racine]
              entreprise.local
                       |
           ┌───────────┴───────────┐
    [DC Principal]          [DC Secondaire]
    192.168.1.10            192.168.1.11
           |
    ┌──────┼──────┐
  [OU IT] [OU RH] [OU Admin]
     |        |         |
  [Users] [Users]  [Groups DA]
  [Computers]
```

---

## Réplication et NTDS.dit

Le DC maintient une base de données `NTDS.dit` avec :
- Tous les objets AD
- Les **hashes NTLM** de tous les utilisateurs
- Les **clés Kerberos** (incluant `KRBTGT`)

La réplication entre DC utilise le protocole **DRSUAPI** — c'est ce que l'attaque **DCSync** exploite.
