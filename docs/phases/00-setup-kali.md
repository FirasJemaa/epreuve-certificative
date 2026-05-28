# Phase 0 — Setup Kali

> Tu viens de recevoir l'accès SSH. **Ne lance aucune commande d'attaque avant d'avoir fait cette phase.**

---

## 0.1 — Lire le brief et noter les informations fournies

Le professeur te donnera probablement :
- Une IP réseau ou l'IP du DC
- Peut-être un nom de domaine
- Peut-être des credentials de départ

**Note TOUT immédiatement** dans ton fichier de travail.

---

## 0.2 — Créer le dossier de travail

```bash
mkdir -p ~/certif/{scans,hashes,tickets,loot,logs,bloodhound}
cd ~/certif
touch notes.txt

# Remplir les infos initiales
cat > notes.txt << 'EOF'
=== CERTIF AD ===
DATE: $(date)

# Informations fournies
DC_IP=
DOMAIN=
DC_FQDN=
RANGE=

# Credentials obtenus
USER=
PWD=
HASH=

# Flags trouvés
FLAG1=
FLAG2=
FLAG3=
EOF
```

---

## 0.3 — Vérifier les interfaces réseau

```bash
# Voir les interfaces et leurs IPs
ip a

# Voir les routes (pour identifier le réseau cible)
ip route

# Identifier son IP d'attaquant (IFACE = l'interface sur le réseau cible)
ip a show eth0   # ou eth1, ens33, etc.
```

**Ce qu'on cherche :**
- Quelle interface est sur le réseau cible (ex: `192.168.X.X/24`)
- Son IP d'attaquant à noter dans les variables

---

## 0.4 — Définir les variables d'environnement

Copier-coller ce bloc et remplir les valeurs :

```bash
# ═══ VARIABLES DE TRAVAIL ═══
export IFACE="eth0"                          # Interface réseau sur le réseau cible
export RANGE="192.168.X.0/24"               # Plage réseau cible
export DC_IP="192.168.X.10"                 # IP du contrôleur de domaine
export DC_FQDN="dc01.entreprise.local"      # FQDN du DC
export DOMAIN="ENTREPRISE"                  # Nom NetBIOS du domaine
export DOMAIN_FQDN="entreprise.local"       # Nom DNS du domaine
export DOMAIN_SID="S-1-5-21-XXX-YYY-ZZZ"   # SID (à remplir après secretsdump)
export ATTACKER_IP="192.168.X.99"           # Mon IP d'attaquant

# Credentials (à remplir dès qu'on en a)
export USER="alice"
export PWD="Password2024"
export HASH="aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF0011"
export NT_HASH="${HASH##*:}"                # Juste la partie NT

# Raccourci pratique
alias nxcsmb="nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN"
```

> **Astuce :** Ajouter ces lignes à `~/.bashrc` ou les recharger avec `source ~/.bashrc` pour qu'elles persistent entre les sessions.

---

## 0.5 — Vérifier les outils disponibles

```bash
# Vérification rapide des outils essentiels
which nxc netexec crackmapexec 2>/dev/null
which responder 2>/dev/null
which mitm6 2>/dev/null
which bloodhound-python 2>/dev/null
which impacket-secretsdump impacket-psexec 2>/dev/null
which hashcat john 2>/dev/null
which evil-winrm 2>/dev/null
which kerbrute 2>/dev/null

# Localiser les wordlists
ls /usr/share/wordlists/rockyou.txt 2>/dev/null || locate rockyou.txt
ls /usr/share/seclists/Usernames/ 2>/dev/null

# Vérifier Impacket (scripts python)
find / -name "GetUserSPNs.py" 2>/dev/null | head -3
find / -name "ntlmrelayx.py" 2>/dev/null | head -3
```

---

## 0.6 — Configurer /etc/hosts (si FQDN fourni)

```bash
# Ajouter le DC dans /etc/hosts pour la résolution Kerberos
echo "$DC_IP  $DC_FQDN  $DOMAIN_FQDN" | sudo tee -a /etc/hosts

# Vérifier la résolution
ping -c 1 $DC_FQDN
nslookup $DC_FQDN
```

---

## 0.7 — Configurer le DNS Kali (si nécessaire)

```bash
# Pointer le DNS vers le DC pour résoudre les noms AD
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf

# Tester
nslookup $DOMAIN_FQDN
nslookup $DC_FQDN
```

---

## 0.8 — Structure du dossier de travail

```
~/certif/
├── notes.txt          ← Toutes tes infos, flags, progression
├── scans/             ← Outputs nmap, nxc découverte
│   ├── nmap_hosts.txt
│   ├── nmap_dc.txt
│   └── smb_hosts.txt
├── hashes/            ← Hashes capturés, à cracker
│   ├── ntlmv2.hash    ← Responder
│   ├── asrep.hash     ← AS-REP Roasting
│   └── kerb.hash      ← Kerberoasting
├── tickets/           ← .ccache Kerberos
├── loot/              ← Flags, fichiers intéressants, credentials
│   ├── shares/        ← Contenus des partages SMB
│   └── ldap/          ← ldapdomaindump output
├── bloodhound/        ← JSON BloodHound
└── logs/              ← Logs des attaques
```

---

## Checklist Phase 0

- [ ] Dossier `~/certif/` créé avec sous-dossiers
- [ ] `notes.txt` rempli avec les infos initiales
- [ ] Interface réseau identifiée (`ip a`)
- [ ] Variables d'environnement définies
- [ ] `/etc/hosts` configuré avec le DC
- [ ] Outils vérifiés (nxc, impacket, bloodhound, responder)
- [ ] Wordlists localisées (rockyou.txt)
- [ ] Prêt pour la reconnaissance → [Phase 1](01-reconnaissance.md)
