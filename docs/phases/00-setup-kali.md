# Phase 0 : Setup Kali

> Tu viens de recevoir l'accès SSH. **Ne lance aucune commande d'attaque avant d'avoir fait cette phase.**

---

## 0.1 : Lire le brief et noter les informations fournies

Le professeur te donnera probablement :

- **DC_IP** : l'IP du contrôleur de domaine (ex: `192.168.1.10`)
- **DOMAIN** : le nom NetBIOS du domaine (ex: `ENTREPRISE`)
- **DOMAIN_FQDN** : le nom DNS du domaine (ex: `entreprise.local`) (parfois à déduire du nom NetBIOS)
- Des credentials de départ (user/password) (pas toujours)

**Si le prof ne te donne pas le FQDN ni la plage réseau**, pas de panique : tu les récupères en Phase 1 via `nmap` et `nslookup`. Remplis ce que tu as maintenant, le reste viendra.

**Note TOUT immédiatement** dans ton fichier de travail.

---

## 0.2 : Créer le dossier de travail

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

## 0.3 : Vérifier les interfaces réseau

```bash
# Voir les interfaces et leurs IPs
ip a

# Voir les routes (pour identifier le réseau cible)
ip route

# Identifier son IP d'attaquant (IFACE = l'interface sur le réseau cible)
ip a show eth0   # ou eth1, ens33, etc.
```

**Ce qu'on lit dans les résultats :**

```
# ip a show eth0 → exemple de sortie
2: eth0: ...
    inet 192.168.1.99/24  ← ton ATTACKER_IP = 192.168.1.99
                    /24   ← le /24 donne RANGE = 192.168.1.0/24
```

```
# ip route → exemple de sortie
192.168.1.0/24 dev eth0  ← confirme RANGE et IFACE
```

- **IFACE** = le nom de l'interface (`eth0`, `eth1`, `ens33`...)
- **RANGE** = la plage réseau (`192.168.1.0/24`)
- **ATTACKER_IP** = ton IP sur cette interface (`192.168.1.99`)

---

## 0.4 : Définir les variables d'environnement

Les variables sont remplies en **trois temps** selon ce que tu as ou découvres.

```bash
# ═══ ÉTAPE 1 : ce que le prof t'a donné (remplir maintenant) ═══
export DC_IP="192.168.X.10"                 # IP du DC (fourni par le prof)
export DOMAIN="ENTREPRISE"                  # Nom NetBIOS du domaine
export DOMAIN_FQDN="entreprise.local"       # Nom DNS (parfois fourni, sinon Phase 1)
export USER=""                              # Compte de départ si fourni
export PWD=""                               # Mot de passe si fourni

# ═══ ÉTAPE 2 : après la section 0.3 (ip a + ip route) ═══
export IFACE="eth0"                          # Interface vers le réseau cible (ip a)
export RANGE="192.168.X.0/24"               # Plage réseau (ip route)
export ATTACKER_IP="192.168.X.99"           # Ton IP sur ce réseau (ip a show $IFACE)

# ═══ ÉTAPE 3 : après Phase 1 (reconnaissance nmap/nslookup) ═══
export DC_FQDN="dc01.entreprise.local"      # nslookup $DC_IP → te donne le FQDN

# ═══ ÉTAPE 4 : après Phase 2 (premiers credentials obtenus) ═══
export HASH="aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF0011"
export NT_HASH="${HASH##*:}"

# ═══ ÉTAPE 5 : après Phase 6 (DCSync) ═══
export DOMAIN_SID="S-1-5-21-XXX-YYY-ZZZ"   # Apparaît dans l'output de secretsdump

# Raccourci pratique (réexécuter dès que USER/PWD sont remplis)
alias nxcsmb="nxc smb $DC_IP -u $USER -p $PWD -d $DOMAIN"
```

> **Astuce :** Ajouter ces lignes à `~/.bashrc` ou les recharger avec `source ~/.bashrc` pour qu'elles persistent entre les sessions.

---

## 0.5 : Vérifier les outils disponibles

```bash
# Vérification rapide des outils essentiels
which nxc netexec crackmapexec 2>/dev/null
which responder 2>/dev/null
which mitm6 2>/dev/null
which bloodhound-python 2>/dev/null
which hashcat john 2>/dev/null
which evil-winrm 2>/dev/null
which kerbrute 2>/dev/null

# Localiser les wordlists
ls /usr/share/wordlists/rockyou.txt 2>/dev/null || locate rockyou.txt
ls /usr/share/seclists/Usernames/ 2>/dev/null
```

### Vérifier Impacket (outil central)

Impacket est **la suite d'outils la plus importante** du pentest AD. Elle couvre tout : exécution à distance, dump de credentials, Kerberoasting, Golden Ticket, relay NTLM, RBCD.

```bash
# Vérifier l'installation
pip3 show impacket

# Localiser les scripts (chemin où sont les .py)
pip3 show impacket | grep Location
# → ex: Location: /usr/lib/python3/dist-packages
# → les scripts sont dans : /usr/share/doc/python3-impacket/examples/
#   ou directement dans PATH si installé via pip --user ou apt

# Vérifier que les scripts clés sont accessibles
which secretsdump.py psexec.py wmiexec.py ntlmrelayx.py 2>/dev/null \
    || find / -name "secretsdump.py" 2>/dev/null | head -3

# Si les scripts ne sont pas dans PATH, les ajouter (selon le chemin trouvé)
export PATH="$PATH:/usr/share/doc/python3-impacket/examples"
```

**Les scripts Impacket utilisés dans ce guide :**

| Script | Phase | Usage |
|--------|-------|-------|
| `ntlmrelayx.py` | Accès initial | Relay NTLM |
| `GetNPUsers.py` | Accès initial | AS-REP Roasting sans credentials |
| `GetUserSPNs.py` | Escalade | Kerberoasting |
| `secretsdump.py` | Compromission | DCSync, dump SAM/LSA |
| `psexec.py` | Mouvement latéral | Shell SYSTEM via SMB |
| `wmiexec.py` | Mouvement latéral | Shell via WMI (discret) |
| `getTGT.py` | Kerberos | Obtenir un TGT |
| `getST.py` | RBCD | Délégation contrainte |
| `ticketer.py` | Compromission | Golden / Silver Ticket |
| `lookupsid.py` | Pré-compromission | Récupérer le SID du domaine |

> Référence complète : [Impacket](../outils/impacket.md)

---

## 0.6 : Configurer /etc/hosts (si FQDN fourni)

```bash
# Ajouter le DC dans /etc/hosts pour la résolution Kerberos
echo "$DC_IP  $DC_FQDN  $DOMAIN_FQDN" | sudo tee -a /etc/hosts

# Vérifier la résolution
ping -c 1 $DC_FQDN
nslookup $DC_FQDN
```

---

## 0.7 : Configurer le DNS Kali (si nécessaire)

```bash
# Pointer le DNS vers le DC pour résoudre les noms AD
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf

# Tester
nslookup $DOMAIN_FQDN
nslookup $DC_FQDN
```

---

## 0.8 : Structure du dossier de travail

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
