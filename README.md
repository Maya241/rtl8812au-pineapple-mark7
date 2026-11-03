# RTL8812AU pour WiFi Pineapple Mark VII

**Correctifs expérimentaux du pilote Realtek RTL8812AU pour l'adaptateur Alfa AWUS036ACH sur WiFi Pineapple Mark VII.**

Ce dépôt documente et conserve les modifications apportées au pilote `rtl8812au` afin de rétablir le changement de canal en mode moniteur sous Linux 5.4, d'accepter les configurations HT40/VHT80 et de corriger la détection des deux chaînes RF (2T2R).

> [!IMPORTANT]
> Il s'agit d'un **projet communautaire non officiel**. Les tests décrits ci-dessous ont été réalisés sur **un** WiFi Pineapple Mark VII et **une** Alfa AWUS036ACH. La configuration VHT80 est acceptée par le pilote, mais le **décodage effectif de trames transmises sur 80 MHz n'a pas encore été démontré**. Respectez la réglementation radio applicable et utilisez les fonctions de capture uniquement sur les réseaux pour lesquels vous êtes autorisé.

## Environnement validé

| Composant | Configuration testée |
|---|---|
| Appareil | Hak5 WiFi Pineapple Mark VII |
| Adaptateur externe | Alfa AWUS036ACH, chipset Realtek RTL8812AU |
| Système embarqué | OpenWrt 21.02.1 (environnement Hak5) |
| Noyau | Linux `5.4.154` |
| Architecture | `mipsel_24kc` — MIPS32 Release 2 |
| Paquet OpenWrt | `kmod-rtl8812au-ct` |
| Source amont | [`greearb/rtl8812AU_8821AU_linux`](https://github.com/greearb/rtl8812AU_8821AU_linux) |
| Révision amont | `e0d586aa93cb8687dd7dc0e593b6a820df2d6e1d` |
| Version du pilote Realtek | `v4.3.14_13455.20150212_BTCOEX20150128-51` |
| SDK de compilation | OpenWrt `21.02.1`, cible `ramips/mt76x8`, GCC 8.4.0, musl |
| Version du paquet corrigé | `5.4.154+2020-01-12-e0d586aa-7` (V6) |

Le Mark VII observé utilisait l'adaptateur Alfa sur **`wlan3`** ; le numéro `phy` peut changer après un rechargement du module ou un redémarrage.

## Problèmes corrigés

### 1. Changement de canal en mode moniteur

Le pilote amont n'exposait pas le callback `cfg80211_ops.set_monitor_channel` requis pour le changement de canal en mode moniteur avec le noyau 5.4. La commande `iw dev wlan3 set channel ...` échouait ou ne modifiait pas réellement le canal de réception.

La correction introduit `cfg80211_rtw_set_monitor_channel()` et configure le matériel au moyen de `set_channel_bwmode()`.

### 2. Largeurs HT40 et VHT80

Le callback prend en charge :

- **HT20** (20 MHz) ;
- **HT40+ / HT40−** (40 MHz), avec calcul de l'offset du canal primaire ;
- **VHT80 contigu** (80 MHz), avec vérification du canal central.

Le pilote vérifie la définition du canal via `cfg80211` et rejette notamment les configurations désactivées, celles nécessitant DFS dans cette implémentation et les largeurs non prises en charge. **Il ne contourne pas les restrictions réglementaires.**

### 3. Déclaration des capacités 802.11ac

Une initialisation des capacités `vht_cap` sur la bande 5 GHz permet à `cfg80211` d'accepter une demande de canal VHT80. La déclaration reste **conservatrice : deux flux RX/TX, VHT MCS 0–7**, sans annonce de 160 ou 80+80 MHz.

### 4. Détection RF 2T2R

Un défaut d'ordre d'initialisation faisait lire `registrypriv.rf_config` avant le chargement effectif du paramètre du module. Le RTL8812AU pouvait alors être initialisé en **1T1R** malgré sa configuration automatique.

La correction dans `os_dep/linux/usb_intf.c` initialise `rf_config` et `special_rf_path` **avant** `rtw_hal_read_chip_version()`. Les essais ont ensuite montré `rf_type=2`, soit **2T2R**, et l'annonce de HT MCS 0–15 et de deux flux VHT.

## Patches OpenWrt

Les éléments de reconstruction sont rangés dans [`packaging/openwrt/`](packaging/openwrt/) :

| Fichier | Rôle |
|---|---|
| `patches/100-monitor-channel.patch` | Callback de changement de canal HT20 |
| `patches/101-monitor-ht40-vht80.patch` | Prise en charge HT40 et VHT80 |
| `patches/102-monitor-vht-capabilities.patch` | Déclaration des capacités VHT |
| `patches/104-early-rf-init.patch` | Initialisation anticipée des paramètres RF |
| `Makefile` | Recette de paquet OpenWrt, révision `7` |

Le patch temporaire de diagnostic `103-rf-type-diagnostics.patch` **ne fait pas partie de la V6**.

**Important :** les patches s'appliquent au **code source amont `e0d586aa`**, et non à la branche Git de ce dépôt dont les modifications C sont déjà intégrées. Ne les appliquez pas deux fois.

La compilation utilisée pendant le développement nécessitait également les correctifs de compatibilité fournis par OpenWrt, notamment `003-wireless-5.8.patch`, ainsi que les en-têtes `mac80211-backport` correspondant au SDK. Les quatre patches ci-dessus **ne remplacent pas** ces prérequis.

## Compilation avec le SDK OpenWrt

La procédure suivante résume l'environnement de compilation validé. Elle suppose que le SDK OpenWrt 21.02.1 pour `ramips/mt76x8` est déjà configuré, que la recette `rtl8812au-ct` et ses patches de compatibilité OpenWrt sont présents, et que `mac80211-backport/backport/autoconf.h` est disponible dans le *staging directory*.

```bash
# Sur le poste Linux de compilation
SDK="$HOME/openwrt-build/openwrt-sdk-21.02.1-ramips-mt76x8_gcc-8.4.0_musl.Linux-x86_64"
REPO="$HOME/src/rtl8812AU_8821AU_linux"

# Copier la recette et les patches personnalisés SANS effacer
# les patches de compatibilité OpenWrt déjà présents.
mkdir -p "$SDK/package/kernel/rtl8812au-ct/patches"
cp "$REPO/packaging/openwrt/Makefile" \
   "$SDK/package/kernel/rtl8812au-ct/Makefile"
cp "$REPO"/packaging/openwrt/patches/*.patch \
   "$SDK/package/kernel/rtl8812au-ct/patches/"

cd "$SDK"
# Vérifier que CONFIG_PACKAGE_kmod-rtl8812au-ct=m est sélectionné.
grep '^CONFIG_PACKAGE_kmod-rtl8812au-ct=' .config

make -j1 package/kernel/rtl8812au-ct/clean V=s
make -j1 package/kernel/rtl8812au-ct/prepare V=s
make -j1 package/kernel/rtl8812au-ct/compile V=s

find bin -name 'kmod-rtl8812au-ct*-7_*.ipk'
```

> [!NOTE]
> Le SDK OpenWrt standard peut ne pas contenir directement la recette `rtl8812au-ct` ni les en-têtes `mac80211-backport` nécessaires : lors du développement, ils ont été préparés à partir de l'arbre OpenWrt **v21.02.1**. L'utilisation d'un autre noyau ou SDK peut nécessiter des adaptations supplémentaires. Ne supposez pas qu'un IPK compilé pour une autre image du Mark VII sera compatible.

Le module attendu est `rtl8812au.ko`. Le `vermagic` validé était :

```text
5.4.154 mod_unload MIPS32_R2 32BIT
```

## Installation sur le Mark VII

**Attention :** ces manipulations touchent un module noyau. Conserver un accès SSH par l'interface **USB de gestion** et une copie du module actuellement installé. Vérifier les dépendances exactes du paquet, dont la version de noyau et son identifiant ABI.

Sur Debian, transférer l'IPK :

```bash
scp -O chemin/vers/kmod-rtl8812au-ct_5.4.154+2020-01-12-e0d586aa-7_mipsel_24kc.ipk \
  root@172.16.42.1:/tmp/rtl8812au-v6.ipk
```

Avant le transfert, sur Debian, consulter la dépendance noyau déclarée dans le paquet :

```bash
IPK="chemin/vers/kmod-rtl8812au-ct_5.4.154+2020-01-12-e0d586aa-7_mipsel_24kc.ipk"
tar -xOf "$IPK" ./control.tar.gz | tar -xzOf - ./control \
  | grep -E '^(Package|Version|Depends):'
```

Sur le Pineapple :

```sh
# Sauvegarder l'ancien module, idéalement aussi sur le PC de développement.
mkdir -p /root/rtl8812au-backup
cp -p /lib/modules/5.4.154/rtl8812au.ko \
  /root/rtl8812au-backup/rtl8812au-before-v6.ko

# Vérifications AVANT installation.
df -h /overlay
opkg status kernel | grep Version
opkg status kmod-rtl8812au-ct | grep Version

# Installer uniquement si la dépendance noyau/ABI du fichier IPK
# a été vérifiée et correspond au noyau du Pineapple.
opkg install /tmp/rtl8812au-v6.ipk
reboot
```

Après redémarrage :

```sh
opkg status kmod-rtl8812au-ct | grep Version
lsmod | grep rtl8812au
sha256sum /lib/modules/5.4.154/rtl8812au.ko
```

**Empreinte SHA-256 de la V6 effectivement testée** :

```text
04378f6f0b941e1c0ab683ea2101d63201df150c4497dfe0cad36eb485d926f4
```

Cette empreinte concerne le fichier `rtl8812au.ko` validé, **pas** l'archive `.ipk`. Une reconstruction peut produire un binaire différent ; vérifier alors sa provenance et ses métadonnées.

## Mode moniteur et tests

Après le redémarrage, l'interface Alfa `wlan3` revient habituellement en mode **managed**. Activer le mode moniteur :

```sh
ip link set wlan3 down
iw dev wlan3 set type monitor
ip link set wlan3 up
```

Exemples de sélection de canal sur des fréquences autorisées dans votre environnement :

```sh
# HT20 — 2,4 GHz, canal 11
iw dev wlan3 set channel 11 HT20

# HT40+ — primaire 36, centre 38
iw dev wlan3 set channel 36 HT40+

# HT40- — primaire 40, centre 38
iw dev wlan3 set channel 40 HT40-

# VHT80 — primaire 36 (5180 MHz), centre 42 (5210 MHz)
iw dev wlan3 set freq 5180 80 5210

# Lire l'état radio Realtek
cat /proc/net/rtl8812au/wlan3/rf_info
```

Valeurs `oper_*` observées lors des essais :

| Configuration | `oper_ch` | `oper_bw` | `oper_ch_offet` |
|---|---:|---:|---:|
| HT20 canal 11 | 11 | 0 | 0 |
| HT40+ canal 36 | 36 | 1 | 1 |
| HT40− canal 40 | 40 | 1 | 2 |
| VHT80 primaire 36 | 36 | 2 | 1 |

Pour identifier automatiquement le `phy` sans présumer de son numéro :

```sh
PHY=$(basename "$(readlink -f /sys/class/net/wlan3/phy80211)")
iw phy "$PHY" info | grep 'HT TX/RX MCS'
iw phy "$PHY" info | sed -n '/VHT Capabilities/,+22p'
```

Capture passive de contrôle :

```sh
iw dev wlan3 set channel 11 HT20
tcpdump -i wlan3 -e -n -vv -c 20
```

## Résultats de validation V6

| Test | Résultat |
|---|---|
| Installation du paquet `-7` et rechargement après redémarrage | **Validé** |
| Reconnaissance de l'Alfa AWUS036ACH et interface `wlan3` | **Validé** |
| Détection / annonce de deux chaînes RF | **Validé** (HT MCS 0–15, deux flux VHT) |
| HT20 et réception effective sur canal 11 à 2462 MHz | **Validé** |
| Réception de données 802.11n MCS 7 à 20 MHz | **Validé** |
| HT40+ et HT40− : configuration du pilote | **Validé** |
| VHT80 : commande acceptée, `oper_bw=2` | **Validé au niveau de l'état déclaré par le pilote** |
| 5 cycles HT20 → HT40+ → HT40− → VHT80 | **20 changements réussis**, sans erreur critique visible |
| Réception après les changements de largeur | **Validé en HT20** (20 trames, 0 perte noyau signalée) |
| Décodage de trames réellement transmises en HT40 à 40 MHz | **Non démontré** |
| Décodage de trames réellement transmises en VHT80 | **Non démontré** |
| VHT MCS 8–9 / débit maximal théorique 867 Mbit/s | **Non activé / non validé** |

**Interprétation :** `oper_bw=2` et un retour `iw` égal à zéro démontrent que la configuration VHT80 est acceptée. Ils **ne prouvent pas**, à eux seuls, la démodulation de données VHT transmises sur toute une largeur de 80 MHz. Les trames observées à 5180 MHz étaient principalement au format 802.11a ; la réception VHT80 requiert un essai avec un émetteur 80 MHz et un moyen de vérifier les métadonnées ou les états matériels.

## Limitations connues

- La sélection VHT80 reste **expérimentale** tant qu'un test de réception VHT80 de bout en bout n'a pas été réalisé.
- Les capacités VHT annoncées sont volontairement limitées à **MCS 0–7 pour deux flux** ; MCS 8–9 reste à qualifier.
- Le pilote peut afficher `cur_ch=1, cur_bw=0` tandis que `oper_ch` et `oper_bw` suivent les commandes de sélection. Ces compteurs internes ne sont pas parfaitement synchronisés ; les essais de réception restent indispensables.
- Après redémarrage, le passage en mode moniteur est **manuel** dans la configuration testée.
- Le matériel observé communiquait via **USB 2.0 High-Speed / EHCI**, ce qui peut limiter les performances pratiques.
- Les canaux DFS et les largeurs 160 / 80+80 MHz ne sont pas pris en charge par ce correctif.
- La compatibilité est documentée pour **l'image et le noyau testés**, pas pour toutes les versions de firmware Hak5.

## Organisation du dépôt Git

- Branche de développement publiée : [`fix/rtl8812au-early-rf-init`](https://github.com/Maya241/rtl8812au-pineapple-mark7/tree/fix/rtl8812au-early-rf-init).
- Tag de la version du pilote : [`rtl8812au-pineapple-mark7-v6`](https://github.com/Maya241/rtl8812au-pineapple-mark7/tree/rtl8812au-pineapple-mark7-v6).
- À la publication initiale, la branche `main` contenait un README minimal ; **vérifiez la branche sélectionnée sur GitHub** pour afficher le code du pilote.

Le tag V6 désigne le commit du pilote au moment où il a été créé ; les ajouts ultérieurs à la documentation ne changent pas rétroactivement ce tag.

## Historique succinct

- **V1** : callback du mode moniteur et changement HT20.
- **V2** : prise en charge des largeurs HT40 et VHT80.
- **V3** : déclaration expérimentale des capacités 802.11ac.
- **V4** : instrumentation de diagnostic pour suivre la détection RF.
- **V5** : correction de l'ordre d'initialisation du paramètre RF ; détection 2T2R confirmée.
- **V6** : retrait des traces de diagnostic, compilation OpenWrt, essais de non-régression et installation persistante vérifiée après redémarrage.

## Provenance et licence

Ce travail est basé sur le projet amont de [Ben Greear / greearb](https://github.com/greearb/rtl8812AU_8821AU_linux), ainsi que sur la recette et les patches de compatibilité OpenWrt correspondants. **Conservez les avis de copyright et les conditions de licence du code amont**, notamment la GPL version 2 indiquée par les fichiers sources et la recette OpenWrt.

Ce projet n'est ni édité, ni certifié, ni pris en charge officiellement par **Hak5**, **Alfa Network** ou **Realtek**.
