# BloodHound — Référence complète

> BloodHound cartographie toutes les relations dans un AD et trouve les **chemins d'escalade vers Domain Admin**. C'est l'outil le plus important de l'énumération AD.

---

## Démarrage rapide

```bash
# 1. Démarrer le backend Neo4j
sudo neo4j start

# 2. Lancer BloodHound
bloodhound &

# 3. Ouvrir dans le navigateur
# → http://localhost:7474
# → Identifiants par défaut : neo4j / neo4j (changer au premier login)
```

---

## Collecte — bloodhound-python

```bash
# Collecte complète (recommandé)
bloodhound-python \
    -u $USER \
    -p $PWD \
    -d $DOMAIN_FQDN \
    -dc $DC_FQDN \
    -ns $DC_IP \
    -c All \
    --zip \
    -o ~/certif/bloodhound/

# Avec hash NT au lieu du mot de passe
bloodhound-python \
    -u $USER \
    --hashes :$NT_HASH \
    -d $DOMAIN_FQDN \
    -ns $DC_IP \
    -c All \
    --zip

# Collecte via nxc (alternative)
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --bloodhound --collection All --dns-server $DC_IP

# Collections disponibles
# All       → Tout collecter (recommandé en exam)
# DCOnly    → Seulement depuis le DC (plus rapide)
# Group     → Membres des groupes
# LocalAdmin → Admins locaux
# Session   → Sessions actives
# Trusts    → Relations de confiance
# ACL       → Droits ACL
# Container → Objets container
# ObjectProps → Propriétés des objets
```

---

## Import des données

```bash
# Depuis l'interface BloodHound :
# Bouton "Upload Data" (icône flèche montante) → sélectionner le .zip
# OU
# Glisser-déposer le .zip dans l'interface
```

---

## Requêtes prioritaires (menu Analysis)

### Chemins vers Domain Admin

```
① Find Shortest Paths to Domain Admins
   → Le chemin le plus court depuis n'importe quel objet vers DA
   → Ta priorité #1

② Find Shortest Paths to Domain Admins from Owned Principals
   → Chemin depuis tes comptes marqués "owned"

③ Shortest Paths from Kerberoastable Users
   → Si on crack ce compte de service, où ça mène ?

④ Shortest Paths to Unconstrained Delegation Systems
   → Machines exploitables pour vol de TGT DC
```

### Comptes vulnérables

```
⑤ List all Kerberoastable Accounts
   → Comptes avec SPN (hash TGS crackable)

⑥ Find AS-REP Roastable Users (DontReqPreAuth)
   → Comptes sans pré-authentification

⑦ Find Computers where Domain Users are Local Admin
   → Machines accessibles avec n'importe quel compte de domaine
```

### Droits et ACL

```
⑧ Find Principals with DCSync Rights
   → Qui peut déclencher un DCSync ?

⑨ Find All Paths from Domain Users to High Value Targets
   → Vue d'ensemble de tous les chemins possibles

⑩ Shortest Paths to High Value Targets
   → Chemins vers DA, EA, DC, serveurs critiques
```

---

## Analyser son propre compte

```
1. Node Search (barre de recherche) → taper son username
2. Clic sur son nœud → panneau latéral
3. Onglet "Outbound Object Control" :
   → Droits ACL qu'on possède sur d'autres objets
   → C'est là que se cachent les vecteurs d'escalade !
4. Onglet "Group Membership" :
   → Groupes dont on est membre (direct et indirect)
5. Bouton "Shortest Paths to Domain Admins" (depuis son nœud)
```

---

## Marquer les objets comme "Owned"

```
Clic droit sur un nœud → "Mark User as Owned"
→ Permet d'utiliser les requêtes "from Owned Principals"
→ À faire pour chaque compte compromis
```

---

## Lire les arêtes (edges) — Actions possibles

| Arête BloodHound | Action possible |
|-----------------|----------------|
| `GenericAll` | Reset mdp, modifier SPN, ajouter au groupe, shadow creds |
| `GenericWrite` | Modifier attributs (SPN → Kerberoasting, KeyCredential → shadow creds) |
| `WriteDACL` | Modifier les ACL → se donner GenericAll |
| `WriteOwner` | Devenir propriétaire → WriteDACL → GenericAll |
| `ForceChangePassword` | Reset mdp sans connaître l'actuel |
| `AddMember` | S'ajouter dans un groupe |
| `AddSelf` | S'ajouter soi-même dans un groupe |
| `AllExtendedRights` | Inclut ForceChangePassword + autres |
| `ReadLAPSPassword` | Lire le mdp admin local via LAPS |
| `DCSync` | Répliquer les hashes du DC |
| `Owns` | Propriétaire → WriteDACL implicite |
| `AllowedToDelegate` | Délégation Kerberos vers ce service |
| `AllowedToAct` | RBCD — peut se faire déléguer vers cette machine |
| `HasSession` | Utilisateur a une session active sur cette machine |
| `AdminTo` | Admin local sur cette machine |
| `MemberOf` | Membre de ce groupe |
| `Contains` | Objet contenu dans cette OU |
| `GpLink` | GPO liée à cette OU |
| `TrustedBy` | Relation de confiance avec ce domaine |

---

## Requêtes Cypher personnalisées

Utiliser la console Cypher (icône terminal dans BloodHound) :

```cypher
// Trouver tous les comptes avec SPN (Kerberoastable)
MATCH (u:User {hasspn:true}) RETURN u.name, u.serviceprincipalnames

// Trouver les comptes sans pré-auth (AS-REP Roastable)
MATCH (u:User {dontreqpreauth:true}) RETURN u.name

// Trouver les ACL dangereuses depuis un compte
MATCH (u:User {name:"ALICE@DOMAIN.LOCAL"})-[r:GenericAll|GenericWrite|WriteDACL|ForceChangePassword]->(t) 
RETURN u.name, type(r), t.name

// Trouver les machines avec Unconstrained Delegation
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c.name

// Trouver les DA avec session active
MATCH (u:User)-[:MemberOf*1..]->(g:Group {name:"DOMAIN ADMINS@DOMAIN.LOCAL"})
MATCH (c:Computer)-[:HasSession]->(u)
RETURN u.name, c.name

// Comptes locaux admin sur le DC
MATCH (u)-[:AdminTo]->(c:Computer {name:"DC01.DOMAIN.LOCAL"}) 
RETURN u.name
```

---

## Workflow exam

```
1. Lancer la collecte bloodhound-python (dès qu'on a 1 compte)
2. Importer le zip dans BloodHound
3. Marquer ses comptes comme "owned"
4. Lancer "Shortest Paths to Domain Admins from Owned Principals"
5. Analyser le chemin → identifier la technique à utiliser
6. Vérifier dans "Outbound Object Control" les droits ACL qu'on possède
7. Exécuter la technique → marquer les nouveaux comptes comme owned
8. Relancer la recherche depuis les nouveaux owned
9. Répéter jusqu'à Domain Admin
```

---

## Erreurs fréquentes

```bash
# Erreur "Failed to connect to LDAP"
# → Vérifier que le DNS pointe vers le DC
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf

# Erreur "Could not connect to Neo4j"
sudo neo4j start
sudo neo4j status

# Erreur "Clock skew"
sudo ntpdate $DC_IP

# bloodhound-python introuvable
pip3 install bloodhound
# OU
sudo apt install bloodhound python3-bloodhound
```
