# Phase 1 — Reconnaissance

## C'est quoi l'objectif ?

Avant de toucher quoi que ce soit, un bon attaquant **observe**. L'idée c'est de répondre à ces questions :

- Est-ce qu'il y a un domaine AD sur ce réseau ?
- Où est le contrôleur de domaine ?
- Quelles machines sont présentes ?
- Quels services tournent ?

!!! tip "Règle d'or"
    **Plus tu es discret à cette étape, moins tu te fais détecter.**

Il y a deux niveaux de reconnaissance.

---

## 1.1 — Reconnaissance Passive

Tu **n'envoies rien** sur le réseau cible. Tu collectes des infos depuis l'extérieur.

Dans un contexte AD interne (tu es déjà dans le réseau, comme avec l'accès SSH de l'exam), la reconnaissance passive c'est :

- Écouter le trafic réseau sans rien envoyer
- Regarder les broadcasts qui circulent naturellement

```bash
# Responder en mode analyse (écoute passive — ne répond pas)
sudo responder -I $IFACE -A
```

Si tu vois des requêtes LLMNR/NBT-NS apparaître → vecteur Responder actif disponible en Phase 2.

---

## 1.2 — Reconnaissance Active avec Nmap

Tu **envoies des paquets** pour sonder le réseau.

### Ce que Nmap fait concrètement

Il envoie des paquets sur chaque IP/port et analyse les réponses pour savoir :

- Quelles machines sont allumées
- Quels ports sont ouverts
- Quels services tournent derrière ces ports
- La version exacte du service

### Les ports qui trahissent un AD

Quand tu scannes un réseau d'entreprise, certains ports te disent immédiatement que tu as affaire à un AD :

| Port | Service | Ce que ça signifie |
|------|---------|-------------------|
| **88** | Kerberos | **C'est un DC** |
| **389** | LDAP | Annuaire AD |
| **636** | LDAPS | LDAP chiffré |
| **445** | SMB | Partages fichiers |
| **3268** | Global Catalog | DC principal |
| **5985** | WinRM | Admin à distance |
| **3389** | RDP | Bureau à distance |

!!! important "Port 88 = DC"
    Si tu vois le port 88 sur une machine → c'est ton contrôleur de domaine. **C'est ta cible prioritaire.**

---

## 1.3 — Commandes essentielles

### Découverte des hôtes vivants

```bash
# Scan ping rapide
nmap -sn $RANGE -oA ~/certif/scans/hosts_alive

# Découverte SMB simultanée (plus rapide pour les réseaux Windows)
nxc smb $RANGE 2>/dev/null | tee ~/certif/scans/smb_hosts.txt
```

### Trouver le DC

```bash
# Chercher le port 88 (Kerberos = signature d'un DC)
nmap -p 88 --open $RANGE

# Confirmer via DNS reverse lookup
nslookup $DC_IP

# Confirmer via DNS SRV record
nslookup -type=SRV _ldap._tcp.dc._msdcs.$DOMAIN_FQDN $DC_IP
```

### Scan approfondi du DC

```bash
# Scan ciblé sur les ports AD
nmap -p 88,389,445,3268,5985 $RANGE

# Scan complet avec détection de services sur le DC
nmap -Pn -sV -sC \
    -p 53,88,135,139,389,445,464,636,3268,3269,3389,5985 \
    $DC_IP -oA ~/certif/scans/dc_full

# Options :
# -sV  → détecte les versions des services
# -sC  → lance les scripts de base (tests automatiques)
# -p-  → scanne les 65535 ports
# -sn  → ping scan, pas de scan de ports
```

### Recherche de vulnérabilités SMB

```bash
nmap -Pn --script smb-vuln* -p 139,445 $RANGE | tee ~/certif/scans/vulns.txt
```

### Informations réseau

```bash
# DHCP broadcast
sudo nmap --script broadcast-dhcp-discover -e $IFACE

# Zone transfer DNS (si fonctionnel)
dig axfr $DOMAIN_FQDN @$DC_IP
```

---

## 1.4 — Interpréter les résultats

Une fois le scan terminé, tu as une carte du réseau :

```
192.168.X.10  →  ports 88, 389, 445, 3268  →  ★ Contrôleur de domaine
192.168.X.20  →  ports 445, 3389           →  Serveur Windows (RDP)
192.168.X.30  →  ports 445, 5985           →  Serveur Windows (WinRM)
192.168.X.40  →  ports 80, 443             →  Serveur web
192.168.X.50  →  port 445                  →  Poste de travail
```

**Ce que tu notes dans `notes.txt` :**

```bash
echo "DC_IP=192.168.X.10" >> ~/certif/notes.txt
echo "DOMAIN_FQDN=entreprise.local" >> ~/certif/notes.txt
echo "SERVEURS=192.168.X.20,192.168.X.30" >> ~/certif/notes.txt
echo "POSTES=192.168.X.50" >> ~/certif/notes.txt

# Mettre à jour les variables
export DC_IP="192.168.X.10"
export DC_FQDN="dc01.entreprise.local"
export DOMAIN_FQDN="entreprise.local"
echo "$DC_IP  $DC_FQDN  $DOMAIN_FQDN" | sudo tee -a /etc/hosts
echo "nameserver $DC_IP" | sudo tee /etc/resolv.conf
```

Tu sais maintenant **où taper**. La prochaine étape : obtenir un premier accès.

---

## Checklist Phase 1

- [ ] Hôtes vivants listés dans `~/certif/scans/hosts_alive`
- [ ] DC identifié (port 88 confirmé)
- [ ] `DC_IP` et `DOMAIN_FQDN` notés dans `notes.txt`
- [ ] `/etc/hosts` et `/etc/resolv.conf` configurés
- [ ] Scan SMB `--gen-relay-list` lancé → `~/certif/targets_relay.txt`
- [ ] Responder passif lancé — trafic LLMNR observé ?
- [ ] Variables `DC_IP`, `DC_FQDN`, `DOMAIN`, `RANGE` mises à jour
- [ ] Suite → [Phase 2 — Accès Initial](02-acces-initial.md)
