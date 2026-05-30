# Reconnaissance Wi-Fi

## Prérequis : le mode moniteur

Par défaut, une carte Wi-Fi ne voit que le réseau auquel elle est connectée. Le **mode moniteur** enlève cette restriction : tu vois toutes les trames dans l'air, sur tous les canaux, peu importe qu'elles te soient destinées ou non.

C'est la condition obligatoire pour toute reconnaissance ou attaque Wi-Fi.

### Activer le mode moniteur

```bash
# Lister les interfaces Wi-Fi disponibles et noter le nom exact
sudo airmon-ng

# Tuer les processus qui pourraient interférer
sudo airmon-ng check kill

# Activer le mode moniteur sur l'interface trouvée ci-dessus
# Exemple : wlan0, wlp3s0, wlx... → adapter selon la sortie de airmon-ng
sudo airmon-ng start <IFACE>
# Crée <IFACE>mon (ex: wlan0 → wlan0mon)

# Vérifier
sudo iwconfig
```

!!! warning "NetworkManager"
    `airmon-ng check kill` stoppe NetworkManager et wpa_supplicant. Ta connexion réseau normale sera coupée pendant la session de test. C'est voulu.

---

## Scan passif avec airodump-ng

`airodump-ng` écoute passivement et affiche en temps réel tous les réseaux et clients détectés.

```bash
# Scan global toutes bandes
sudo airodump-ng <IFACE_MON>

# Scan toutes bandes (2.4 GHz + 5 GHz) avec WPS et fabricant
sudo airodump-ng <IFACE_MON> --manufacturer --wps --band abg

# Enregistrer la capture dans des fichiers
sudo airodump-ng <IFACE_MON> -w ~/wifi/scan --band abg
```

### Tableau du haut : les points d'accès

| Colonne | Signification | Remarques |
|---|---|---|
| **BSSID** | Adresse MAC de l'AP | Unique par borne |
| **PWR** | Puissance signal reçu (dBm) | Proche de 0 = fort (-30 excellent, -80 faible) |
| **Beacons** | Trames "beacon" capturées | L'AP envoie "je suis là" en permanence |
| **#Data** | Paquets de données capturés | Si ça monte vite : réseau actif |
| **#/s** | Paquets par seconde | Utile pour mesurer l'effet d'une attaque |
| **CH** | Canal utilisé | Indispensable pour capturer handshake ou IV |
| **MB** | Débit max annoncé | Donne le standard (b/g/n/ac) |
| **ENC** | Type de chiffrement | OPN, WEP, WPA2, WPA3 |
| **CIPHER** | Algorithme de chiffrement | TKIP (vieux), CCMP (WPA2), GCMP (WPA3) |
| **AUTH** | Méthode d'auth | PSK (perso), MGT/802.1X (enterprise) |
| **ESSID** | Nom du réseau | Si caché : `<length:7>` |

### Tableau du bas : les clients

| Colonne | Signification | Remarques |
|---|---|---|
| **STATION** | Adresse MAC du client | Unique par appareil |
| **PWR** | Puissance signal du client | Même logique que AP |
| **Rate** | Débit TX/RX | Ex : `1e-54` = 1 Mbps TX / 54 Mbps RX |
| **Lost** | Paquets perdus | Si élevé : signal instable |
| **Frames** | Trames capturées du client | Activité du device |
| **Probed ESSIDs** | Réseaux cherchés par le client | Très utile : révèle ses réseaux mémorisés |

---

## Cibler un AP précis

Une fois l'AP identifié, filtrer sur son BSSID et son canal pour ne capturer que son trafic.
Le BSSID et le canal (`<BSSID>`, `<CH>`) sont lus dans le scan global ci-dessus.

```bash
# Filtrer sur l'AP cible
sudo airodump-ng --bssid <BSSID> -c <CH> <IFACE_MON>

# Avec enregistrement (utile pour capturer un handshake ou analyser plus tard)
sudo airodump-ng --bssid <BSSID> -c <CH> -w ~/wifi/capture <IFACE_MON>
```

!!! tip "Fixer le canal"
    En mode moniteur, la carte saute automatiquement de canal (channel hopping), pratique
    pour un scan global. Pour attaquer un AP précis, il faut se fixer sur son canal avec
    `-c` dans airodump-ng, ou manuellement :
    ```bash
    iwconfig <IFACE_MON> channel <CH>
    ```

---

## Identifier les clients d'un réseau

```bash
# Même commande que "cibler un AP" : les clients associés apparaissent dans le tableau du bas
sudo airodump-ng --bssid <BSSID> -c <CH> <IFACE_MON>
```

Les clients associés apparaissent dans le tableau du bas avec leur adresse MAC (**STATION**).

---

## Lire les probes (Probe Requests)

Les appareils mémorisent les réseaux Wi-Fi déjà connus et envoient régulièrement des **Probe Requests** pour les retrouver. Ces trames sont visibles dans la colonne **Probed ESSIDs** du tableau des clients.

```
# Exemple de ligne dans airodump-ng (tableau du bas) :
(not associated)   XX:XX:XX:XX:XX:XX  -49    0-1   18   492   wifi-corp,wifi-guest
#                                                              ^^^^^^^^^^^^^^^^^^^^^
#                                              réseaux mémorisés par ce client (Probed ESSIDs)
```

Informations exploitables :
- Noms de réseaux mémorisés par la cible
- SSID des réseaux d'entreprise ou personnels
- Point de départ pour un Evil Twin (le client se connectera automatiquement si tu crées un AP avec ce nom)

---

## Révéler un SSID caché

Un AP avec SSID caché n'inclut pas son nom dans ses beacons. Airodump-ng l'affiche comme `<length:7>` (7 = nombre de caractères du SSID).

Deux techniques pour le révéler :

### Attente passive

Si un client légitime se connecte, la poignée de main inclut le SSID en clair. Airodump-ng l'affiche automatiquement.

### Bruteforce actif avec mdk4

L'AP caché répond aux Probe Requests quand il reconnaît son vrai nom.
`mdk4` en mode `p` envoie ces probes en masse depuis une wordlist.

Si les SSIDs visibles suivent un format prévisible (ex: `wifi-IT`, `wifi-corp`, `wifi-guest`),
construire la wordlist avec ce préfixe :

```bash
# Générer une wordlist avec le préfixe observé
cat <WORDLIST> | awk '{print "<PREFIXE>-" $1}' > ~/wifi/ssid-wordlist.txt

# Envoyer les probes vers l'AP caché
# <BSSID> : adresse MAC de l'AP caché (visible dans airodump-ng même sans SSID)
sudo mdk4 <IFACE_MON> p -t <BSSID> -f ~/wifi/ssid-wordlist.txt
```

Quand l'AP reconnaît son nom, il répond. Le SSID s'affiche alors dans airodump-ng.

!!! tip "Forcer la reconnexion d'un client"
    Si un client est déjà associé, tu peux aussi le déauthentifier (voir section Accès Initial) pour forcer sa reconnexion et capturer le SSID lors du réassociation.

---

## Checklist Reconnaissance

- [ ] Mode moniteur activé (`airmon-ng check kill` + `airmon-ng start <IFACE>`)
- [ ] Scan global lancé (`airodump-ng <IFACE_MON> --band abg`)
- [ ] Pour chaque AP identifié : BSSID, canal, chiffrement (ENC), auth (AUTH) notés
- [ ] Clients listés et leurs probes lus
- [ ] SSID cachés identifiés (colonne ESSID = `<length:N>`)
- [ ] Si SSID caché : bruteforce mdk4 ou attente d'une reconnexion client
- [ ] Capture enregistrée (`-w`) pour analyse ultérieure
