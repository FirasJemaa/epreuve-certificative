# Phase 3 : Énumération

## Pourquoi c'est crucial

Avec un compte lambda, tu peux **interroger l'AD** pour obtenir une quantité massive d'informations :

- La liste de tous les utilisateurs
- La liste de tous les groupes et leurs membres
- Les relations de confiance entre domaines
- Les machines du domaine
- Les ACL (qui a le droit de faire quoi sur quoi)

> Un attaquant sans énumération, c'est comme un cambrioleur dans le noir. L'énumération allume les lumières.

!!! tip "Règle absolue"
    Lancer **BloodHound immédiatement** dès le premier compte obtenu. C'est ta carte du territoire. Sans lui, tu travailles à l'aveugle.

---

## 3.1 : BloodHound (priorité absolue)

C'est **l'outil le plus important** pour l'énumération AD. Il cartographie toutes les relations de l'AD et les affiche sous forme de **graphe**.

### Comment ça marche

Un collecteur (**bloodhound-python**) interroge l'AD et collecte toutes les données. BloodHound les affiche et te permet de chercher :

> *"Quel est le chemin le plus court entre le compte Alice et Domain Admin ?"*

### Collecte

```bash
# Collecte complète depuis Linux
bloodhound-python \
    -u $USER \
    -p $PWD \
    -d $DOMAIN_FQDN \
    -dc $DC_FQDN \
    -ns $DC_IP \
    -c All \
    --zip \
    -o ~/certif/bloodhound/

# Avec hash NT
bloodhound-python \
    -u $USER \
    --hashes :$NT_HASH \
    -d $DOMAIN_FQDN \
    -ns $DC_IP \
    -c All --zip

# Via nxc
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --bloodhound --collection All --dns-server $DC_IP
```

### Démarrer BloodHound

```bash
sudo neo4j start
bloodhound &
# Ouvrir http://localhost:7474
# Importer le .zip généré (bouton Upload Data)
```

### Exemple de chemin BloodHound

BloodHound va te montrer des chemins comme :

```
Alice → membre de → HelpDesk
HelpDesk → a le droit GenericWrite sur → Bob
Bob → membre de → Domain Admins
```

Ce chemin te dit exactement quoi faire pour devenir Domain Admin.

### Requêtes prioritaires (menu Analysis)

```
① Find Shortest Paths to Domain Admins
   → C'est ton chemin principal vers DA

② List all Kerberoastable Accounts
   → Comptes de service à cracker (TGS)

③ Find AS-REP Roastable Users
   → Comptes sans pré-authentification

④ Find Principals with DCSync Rights
   → Qui peut répliquer les hashes ?

⑤ Shortest Paths to Unconstrained Delegation
   → Machines exploitables pour vol de TGT

⑥ Find Computers where Domain Users are Local Admin
   → Machines accessibles avec n'importe quel compte

⑦ Shortest Paths from Kerberoastable Users
   → Si on crack ce compte, où ça mène ?
```

### Analyser son propre compte

```
1. Node Search → taper son username
2. Clic droit → "Shortest Paths to Domain Admins"
3. Onglet "Outbound Object Control" → droits ACL qu'on possède
4. Onglet "Reachable High Value Targets"
```

---

## 3.2 : NetExec (nxc) : Couteau suisse

```bash
# Vérifier ses accès sur tout le réseau (chercher (Pwn3d!))
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN

# Lister les partages SMB
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --shares | tee ~/certif/loot/shares.txt

# Lister les utilisateurs du domaine
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --users | tee ~/certif/loot/users.txt

# Lister les groupes
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --groups | tee ~/certif/loot/groups.txt

# Sessions actives
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --sessions

# Utilisateurs connectés en ce moment
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --loggedon-users

# Politique de mot de passe
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --pass-pol
```

---

## 3.3 : Énumération LDAP

L'AD est une base de données LDAP. On peut l'interroger directement.

```bash
# Avec ldapsearch
ldapsearch -x -H ldap://$DC_IP \
    -D "$USER@$DOMAIN_FQDN" -w $PWD \
    -b "DC=${DOMAIN_FQDN/./,DC=}" \
    "(objectClass=user)" sAMAccountName description memberOf

# Via NetExec
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN --users
```

### ldapdomaindump : Dump LDAP complet

```bash
ldapdomaindump \
    -u "$DOMAIN\\$USER" \
    -p $PWD \
    $DC_IP \
    -o ~/certif/loot/ldap/

# Lire les résultats dans le navigateur
firefox ~/certif/loot/ldap/domain_users.html &

# Chercher des credentials dans les descriptions
grep -i "pass\|pwd\|cred\|secret\|flag" ~/certif/loot/ldap/domain_users.json
```

---

## 3.4 : Exploration des partages SMB

### Lister et explorer les partages

```bash
# Lister tous les partages
smbclient -L //$DC_IP -U $DOMAIN/$USER%$PWD

# Se connecter à un partage
smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD

# Dans smbclient :
# smb: \> ls          → lister
# smb: \> recurse ON  → mode récursif
# smb: \> mget *      → tout télécharger
```

### SYSVOL : Mine à flags et credentials

```bash
# Télécharger SYSVOL en entier
smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD \
    -c "recurse ON; mget *" 2>/dev/null

# Chercher des cpassword GPP dans les fichiers XML
find ~/certif/loot/ -name "*.xml" \
    | xargs grep -l "cpassword" 2>/dev/null

# Déchiffrer les cpassword trouvés
gpp-decrypt 'VALEUR_CPASSWORD'
```

**Fichiers à cibler dans SYSVOL/NETLOGON :**

| Fichier | Contenu possible |
|---------|-----------------|
| `Groups.xml` | cpassword GPP : mot de passe admin local |
| `*.ps1` | Scripts PowerShell avec credentials hardcodés |
| `*.bat` | Scripts batch |
| `*.ini` | Fichiers de configuration |
| `*.txt` | Notes, flags |

### Spider automatique

```bash
# Spider complet (cherche tous les fichiers)
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN -M spider_plus

# manspider : chercher des mots-clés dans les fichiers
manspider $RANGE -c passw password cred flag \
    -e txt xml ini config ps1 bat \
    -d $DOMAIN -u $USER -p $PWD
```

---

## 3.5 : Énumération complémentaire

```bash
# ADCS : Chercher les templates vulnérables
certipy find -u $USER@$DOMAIN_FQDN -p $PWD \
    -dc-ip $DC_IP -vulnerable -stdout | tee ~/certif/loot/adcs.txt

# LAPS : Lire les mots de passe admin locaux (si droits)
nxc ldap $DC_IP -d $DOMAIN -u $USER -p $PWD --module laps

# gMSA : Mots de passe de comptes de service gérés
nxc ldap $DC_IP -u $USER -p $PWD --gmsa

# enum4linux-ng : Énumération complète
enum4linux-ng -a -u $USER -p $PWD $DC_IP | tee ~/certif/loot/enum4linux.txt
```

---

## Checklist Phase 3

- [ ] BloodHound collecté et importé
- [ ] Requêtes BloodHound lancées → **chemins vers DA identifiés**
- [ ] `nxc smb` sur tout le réseau → machines avec `(Pwn3d!)` notées
- [ ] Partages listés → SYSVOL/NETLOGON explorés
- [ ] `ldapdomaindump` effectué + descriptions cherchées
- [ ] Fichiers `Groups.xml` trouvés → `gpp-decrypt` utilisé
- [ ] Spider lancé sur les partages inhabituels
- [ ] Chemins d'escalade BloodHound lus → technique choisie
- [ ] Suite → [Phase 4 : Escalade](04-escalade.md)
