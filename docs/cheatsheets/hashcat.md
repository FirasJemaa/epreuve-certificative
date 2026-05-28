# Cheatsheet — Hashcat

---

## Modes les plus importants en pentest AD

| Mode | Type de hash | Obtenu via |
|------|-------------|-----------|
| `-m 5600` | **NetNTLMv2** | Responder (trafic réseau) |
| `-m 5500` | NetNTLMv1 | Responder (rare) |
| `-m 1000` | **NTLM (NT hash)** | secretsdump, SAM, NTDS |
| `-m 18200` | **Kerberos AS-REP** | AS-REP Roasting |
| `-m 13100` | **Kerberos TGS RC4** | Kerberoasting |
| `-m 19600` | Kerberos TGS AES128 | Kerberoasting (AES) |
| `-m 19700` | Kerberos TGS AES256 | Kerberoasting (AES) |
| `-m 31300` | **TimeRoast** | Comptes machine |

---

## Commandes types

### NetNTLMv2 (Responder)

```bash
hashcat -m 5600 ~/certif/hashes/ntlmv2.hash /usr/share/wordlists/rockyou.txt

# Avec règles
hashcat -m 5600 ~/certif/hashes/ntlmv2.hash /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule

# Attaque hybride (dictionnaire + masque)
hashcat -m 5600 ~/certif/hashes/ntlmv2.hash /usr/share/wordlists/rockyou.txt -a 6 ?d?d
```

### NT Hash (NTLM brut)

```bash
hashcat -m 1000 ~/certif/hashes/ntlm.hash /usr/share/wordlists/rockyou.txt
```

### AS-REP Roasting

```bash
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt

# Avec règles (améliore les chances)
hashcat -m 18200 ~/certif/hashes/asrep.hash /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule \
    -r /usr/share/hashcat/rules/d3ad0ne.rule
```

### Kerberoasting (RC4 — le plus courant)

```bash
hashcat -m 13100 ~/certif/hashes/kerberoast.hash /usr/share/wordlists/rockyou.txt

# Avec règles (important — les mots de passe service sont souvent complexes)
hashcat -m 13100 ~/certif/hashes/kerberoast.hash /usr/share/wordlists/rockyou.txt \
    -r /usr/share/hashcat/rules/best64.rule
```

---

## Modes d'attaque (`-a`)

| Mode | Description | Commande |
|------|-------------|---------|
| `-a 0` | Dictionnaire (défaut) | `hashcat -a 0 -m MODE hash.txt dict.txt` |
| `-a 1` | Combinatoire (2 dicos) | `hashcat -a 1 -m MODE hash.txt dict1.txt dict2.txt` |
| `-a 3` | Brute force / masque | `hashcat -a 3 -m MODE hash.txt ?a?a?a?a?a?a?a?a` |
| `-a 6` | Hybride dico + masque | `hashcat -a 6 -m MODE hash.txt dict.txt ?d?d?d?d` |
| `-a 7` | Hybride masque + dico | `hashcat -a 7 -m MODE hash.txt ?d?d?d?d dict.txt` |

---

## Masques utiles (`-a 3`)

| Masque | Description | Exemple |
|--------|-------------|---------|
| `?l` | Minuscule | `a-z` |
| `?u` | Majuscule | `A-Z` |
| `?d` | Chiffre | `0-9` |
| `?s` | Symbole | `!@#$%` |
| `?a` | Tous | `?l?u?d?s` |

```bash
# Exemple : mot de passe de 8 chars commençant par maj + 6 minuscules + 1 chiffre
hashcat -a 3 -m 1000 hash.txt ?u?l?l?l?l?l?l?d

# Mots de passe saisonniers courants
hashcat -a 3 -m 5600 hash.txt 'Password?d?d?d?d'
hashcat -a 3 -m 5600 hash.txt 'Welcome?d?d?d?d'
hashcat -a 6 -m 5600 hash.txt /usr/share/wordlists/rockyou.txt '?d?d?d?d'
```

---

## Règles utiles

```bash
# best64.rule — Les 64 meilleures règles
-r /usr/share/hashcat/rules/best64.rule

# d3ad0ne.rule — Plus agressif
-r /usr/share/hashcat/rules/d3ad0ne.rule

# Combiner plusieurs règles
-r /usr/share/hashcat/rules/best64.rule -r /usr/share/hashcat/rules/toggles1.rule

# Génération de règles personnalisées (ex: ajouter un chiffre à la fin)
echo '$1' > ~/certif/custom.rule
echo '$2' >> ~/certif/custom.rule
hashcat -m 5600 hash.txt dict.txt -r ~/certif/custom.rule
```

---

## Wordlists disponibles sur Kali

```bash
ls /usr/share/wordlists/
# rockyou.txt       ← La principale
# rockyou.txt.gz    ← Compressée (décompresser avec gunzip)

ls /usr/share/seclists/Passwords/
# Common-Credentials/10-million-password-list-top-100.txt
# Common-Credentials/best1050.txt
# Leaked-Databases/rockyou-75.txt

# Décompresser rockyou
gunzip /usr/share/wordlists/rockyou.txt.gz
```

---

## Options utiles

```bash
# Afficher la progression
hashcat -m 5600 hash.txt dict.txt --status

# Utiliser le GPU (si disponible)
hashcat -m 5600 hash.txt dict.txt -d 1  # GPU device 1

# Voir les résultats cracqués
hashcat -m 5600 hash.txt dict.txt --show

# Sauvegarder les résultats
hashcat -m 5600 hash.txt dict.txt -o ~/certif/cracked.txt

# Reprendre une session interrompue
hashcat -m 5600 hash.txt dict.txt --restore

# Limiter le temps
hashcat -m 5600 hash.txt dict.txt --runtime 300  # 5 minutes max
```

---

## Alternative — John the Ripper

```bash
# NetNTLMv2
john --format=netntlmv2 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# AS-REP Roasting
john --format=krb5asrep hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Kerberoasting
john --format=krb5tgs hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# NTLM
john --format=NT hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Voir les résultats
john --show hash.txt
```

---

## Identifier un hash inconnu

```bash
# Avec hashid
hashid 'HASH_VALUE'

# Avec hash-identifier
hash-identifier

# Avec haiti
haiti 'HASH_VALUE'
```
