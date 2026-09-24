# RC MultiSwitch-E — portage ESP32-S3 Super Mini

**Documentation française — v0.2h2 (24 septembre 2026)**  
**Projet d’origine : WMuCpp / RC MultiSwitch-E de Wilhelm Meier.**  
**Statut : portage partiel, fonctionnel pour les sorties, Intervall, PWM et Morse ; il ne s’agit pas du firmware STM32 complet.**

## 1. Projet original, sources et licence

Ce firmware adapte à l’**ESP32-S3 Super Mini** une partie de RC MultiSwitch-E, développé par **Wilhelm Meier** dans le projet **WMuCpp**. L’objectif est de conserver l’adressage et les commandes MultiSwitch CRSF ainsi que la configuration par paramètres CRSF affichés sur la radio, tout en remplaçant les périphériques STM32 par ceux de l’ESP32-S3.

Sources du projet d’origine :

- Dépôt WMuCpp : <https://github.com/wimalopaan/wmucpp>
- Application STM32 MultiSwitch : <https://github.com/wimalopaan/wmucpp/tree/master/boards/rcmultiswitchG030>
- Point d’entrée d’origine : <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/msw30.cc>
- Menu des paramètres CRSF : <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/crsf_cb.h>
- Décodage des commandes : <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/switch_cb.h>
- Paramètres persistants : <https://github.com/wimalopaan/wmucpp/blob/master/boards/rcmultiswitchG030/eeprom.h>
- Décodeur et protocole CRSF : <https://github.com/wimalopaan/wmucpp/blob/master/include_stm32/rc/crsf_2.h>
- Moteur de sorties, clignotement et Morse : <https://github.com/wimalopaan/wmucpp/blob/master/include_stm32/blinker.h>

**Licence du code distribué : GNU GPL v3 ou ultérieure (`GPL-3.0-or-later`) ; voir `LICENSE` dans le projet.** Les mentions de copyright et l’attribution à Wilhelm Meier doivent être conservées dans les fichiers dérivés. Les adaptations ESP32-S3 sont distribuées dans le même cadre. Le script `elrsV3.lua` et le widget sont des éléments séparés : ce document ne leur attribue pas de licence qui n’a pas été vérifiée.

### Crédits / Credits

- **Wilhelm Meier** — auteur du projet WMuCpp / RC MultiSwitch-E d’origine, de la logique et des paramètres auxquels ce portage se réfère.
- **Pierrot** — initiative du portage ESP32-S3, câblage et essais réels avec RadioMaster ER8/EdgeTX, validation des sorties, Intervall, PWM et Morse, retours et choix fonctionnels.
- **ChatGPT (OpenAI)** — assistance à l’adaptation Arduino/ESP32-S3, aux diagnostics CRSF et à la rédaction de cette documentation, avec validation matérielle par Pierrot.

Ce portage n’est **pas une version officielle publiée par Wilhelm Meier**.

## 2. Matériel et câblage

Le montage de référence utilise un **RadioMaster ER8 sous ELRS/CRSF**, une **ESP32-S3 Super Mini** et des sorties pour LED ou étages de puissance adaptés.

| Signal | ESP32-S3 | À relier à |
|---|---|---|
| CRSF RX | **GPIO12** | TX du récepteur ER8 |
| CRSF TX | **GPIO13** | RX du récepteur ER8 |
| Masse | **GND** | GND du récepteur ER8 |
| OUT0 à OUT7 | **GPIO4 à GPIO11** | Huit sorties logiques ; OUT0 = LED 1, OUT7 = LED 8 |
| LED RGB de statut | **GPIO48** | WS2812 embarquée, selon la carte Super Mini utilisée |
| Console | USB / `Serial` | **115200 bauds** |

Le lien CRSF utilise **420000 bauds, 8N1**, sur **deux fils de données distincts** plus une **masse commune**. **GPIO12 est une entrée RX et absolument pas un GND.** La configuration actuelle n’implémente pas le CRSF sur un seul fil (half-duplex), ni la recherche automatique de débit. Respecter les niveaux logiques 3,3 V.

Les GPIO ne doivent pas alimenter directement une charge de puissance : LED avec résistance adaptée, MOSFET ou étage de commande pour les projecteurs, moteurs et relais.

### Polarité électrique des sorties

Dans `devices_3.h` :

```cpp
#define MSW_OUTPUT_ACTIVE_LOW 1
```

| Valeur | Sortie ON | Sortie OFF |
|---|---|---|
| `1` (défaut du portage) | LOW, environ 0 V | HIGH, environ 3,3 V |
| `0` | HIGH, environ 3,3 V | LOW, environ 0 V |

La polarité concerne les huit GPIO et le PWM. **Dans la v0.2h2, elle reste un choix global à la compilation : elle n’est pas réglable depuis `elrsV3.lua`, ni individuellement par sortie.** La polarité électrique n’inverse pas les commandes logiques ON/OFF et ne change pas `outs=XX`.

Attention avec un **ULN2803** : une entrée HIGH active son transistor de sortie, qui tire sa charge vers GND. Choisir la polarité en fonction du câblage réel, et non uniquement du type de LED.

## 3. Comment fonctionne l’ensemble

1. L’ER8 transmet les trames CRSF à l’ESP32-S3 sur GPIO12.
2. Le décodeur valide les trames (CRC8 CRSF, polynôme `0xD5`), reçoit les voies et les commandes MultiSwitch et vérifie l’**adresse logique de commutation**.
3. Les commandes MultiSwitch **`Set`, `Set4`, `Set4M`** pilotent les états ON/OFF ; **`Prop`** fournit une valeur proportionnelle pour le PWM.
4. L’ESP32-S3 répond aux requêtes de découverte et de paramètres CRSF de la radio sur GPIO13. Le script **`elrsV3.lua` affiche les paramètres annoncés par le firmware** : il n’est pas nécessaire de le modifier pour voir nos nouvelles rubriques.
5. Le widget **`lvglMultiSw`** pilote les sorties via les commandes CRSF ; les paramètres du script configurent la manière dont chaque sortie réagit. Une sortie peut par exemple clignoter *et* être atténuée par PWM.
6. Les réglages persistants sont stockés dans la **NVS** ESP32, espace `msw-s3`, après environ **3 secondes sans nouvelle modification**. Attendre quelques secondes après le dernier changement avant de couper l’alimentation.

L’**adresse de commutation** (`Switch Addr`, réglée à `2` sur le montage d’essai) est distincte de l’**adresse de périphérique CRSF** interne (`0xC8` dans cette version). Le portage prend en charge une seule adresse logique de commutation.

### Évolution du portage

| Version | Principales étapes |
|---|---|
| `v0.2` à `v0.2c` | Dialogue CRSF avec la radio, menu de base, adresse, failsafe, sorties ON/OFF et polarité à la compilation. |
| `v0.2d` | Réception `HardwareSerial` accélérée, table CRC, traitement par lots, tampon RX accru. |
| `v0.2d1` | Observateur facultatif `C=1 / C? / C=0` pour diagnostiquer la réception sans modifier le décodeur fonctionnel. |
| `v0.2f` | **Operate** et clignotement **Intervall** par sortie. |
| `v0.2g` | **PWM LEDC** matériel sur huit sorties et traitement de **Prop**. |
| `v0.2h` | **Morse** : texte commun et cinq durées. |
| `v0.2h1` | **Repeat** et **Repeat pause**, extension spécifique ESP32-S3. |
| **`v0.2h2`** | Dossier d’aide Morse en lecture seule dans le menu de la radio. |

Le code conserve la structure Arduino (`.ino` → `msw30.cpp`) et les noms de plusieurs éléments d’origine, mais les pilotes GPIO, UART, LED RGB, NVS et PWM ont été adaptés à l’ESP32-S3. Les anciennes versions d’essai, en particulier la branche UART natif `v0.2e`, **ne sont pas la base de la version actuelle**.

## 4. Guide complet des options de `elrsV3.lua`

Le menu affiché dépend des paramètres publiés par le firmware chargé. Les descriptions ci-dessous correspondent à **v0.2h2** : elles ne décrivent pas toutes les options du STM32 original. Les libellés anglais sont conservés pour pouvoir les retrouver sur la radio.

### Informations et `Global`

| Champ | Explication |
|---|---|
| `Version(HW/SW)` | Information sur la variante matérielle et la version interne du portage. Ce n’est pas la version matérielle d’une carte STM32 Wilhelm. |
| `Global → Switch Addr` | Adresse logique à laquelle le MultiSwitch répond. À faire correspondre à l’adresse utilisée dans le widget ; enregistrée en NVS. |

**Pas encore disponible depuis le menu actuel :** inversion de polarité, choix du GPIO, adresse CRSF réglable, fréquence PWM réglable, réinitialisation générale et options de télémétrie avancée.

### `Failsafe`

Le failsafe s’applique lors du **passage de la liaison CRSF à l’état déconnecté** (absence de nouvelles trames de voies valides pendant environ 500 ms). Il ne remplace pas le fonctionnement normal des commandes du widget tant que la liaison est active.

| Champ | Explication |
|---|---|
| `Mode → Hold` | Conserver la dernière demande/valeur des sorties pendant la perte de liaison. |
| `Mode → All-Off` | Forcer toutes les sorties à OFF ; pour le PWM, extinction à 0 %. |
| `Mode → Set` | Appliquer les états individuels indiqués dans `Set Output 0…7`. |
| `Set Output 0…7 → Off/On` | État de chaque sortie **uniquement quand `Mode=Set` et que le failsafe est déclenché**. Ce ne sont pas des commandes manuelles permanentes. |

Avec **`Set` + huit valeurs `Off`**, le résultat à la perte de liaison est le même que `All-Off`. Avec `Hold`, une sortie qui clignote ou répète du Morse peut continuer à fonctionner puisqu’elle conserve sa demande ON. **Sur une sortie `PWM Mode=Remote`**, `Set=On` correspond à 100 % et `Set=Off` à 0 % ; la consigne failsafe reste en place jusqu’à un nouveau `Prop` reçu. Tester toute charge motorisée sans hélice et en sécurité.

### `Operate`

| Champ | Explication |
|---|---|
| `Output 0…7 → Off/On` | Commande immédiate de la sortie choisie **depuis le script**, sans passer par le widget. L’ordre n’est pas un réglage NVS : il ne sera pas restauré comme commande active au prochain démarrage. |

`Operate` et le widget envoient tous deux des ordres logiques. Les effets `Intervall`, `PWM` et `Morse` configurés pour la sortie concernée s’appliquent à ces ordres. En `PWM Mode=Remote`, la luminosité est pilotée par `Prop` plutôt que par le bouton ON/OFF.

### `Output 0` … `Output 7` — paramètres individuels

Chaque dossier correspond à un GPIO : `Output 0` = GPIO4 ; `Output 7` = GPIO11. Les huit sorties partagent le **même fonctionnement**, mais leurs réglages Intervall et PWM sont indépendants et sauvegardés en NVS.

| Champ | Valeurs | Effet |
|---|---|---|
| `Intervall Mode` | `Off / On / Morse` | Sortie fixe / clignotement en groupes / émission du texte Morse. |
| `Intervall(on)` | 1–255, **× 50 ms** | Durée d’un éclat dans le mode `On`. |
| `Intervall(off)` | 1–255, **× 50 ms** | Pause entre groupes d’éclats en mode `On`. |
| `Intervall(count)` | 1–4 | Nombre d’éclats par groupe. L’espace entre deux éclats d’un groupe utilise également `Intervall(on)` dans ce portage. |
| `PWM Mode` | `Off / On / Remote / Global/Indiv` | Voir les modes PWM ci-dessous. |
| `PWM Duty` | 1–99 % | Intensité configurée ; 50 % par défaut. |
| `PWM Expo` | 0–100 | Paramètre mémorisé **sans effet sur le signal à ce stade** ; la fonction `expo()` du code d’origine est également vide. |

**`Intervall(on/off/count)` n’agit pas sur le rythme Morse.** Pour le Morse, utiliser les durées du dossier `Morse`.

#### Choix de `PWM Mode`

| Mode | Comportement réel dans v0.2h2 |
|---|---|
| `Off` | Sortie numérique ON/OFF ordinaire, pleine puissance quand allumée. |
| `On` | PWM à l’intensité choisie, **activé/désactivé par le widget ou Operate** ; la luminosité reste compatible avec Intervall et Morse. |
| `Remote` | La valeur **`Prop` CRSF (0–100 %)** commande directement le duty ; le bouton ON/OFF n’est pas la commande de luminosité. Après démarrage : 0 % jusqu’au premier `Prop`. |
| `Global/Indiv` | Comme `On`, avec possibilité interne de multiplier le duty individuel par une valeur globale. **La commande Lua/virtuelle du Global Dimming n’est pas encore portée ; facteur global = 100 % par défaut.** |

Le PWM est produit par **LEDC matériel, 1 kHz, résolution 8 bits** sur les huit GPIO, et non par un `delay()` ou par le timer du port CRSF. Avec la polarité active LOW : 0 % = sortie HIGH/OFF ; 100 % = LOW/ON. Les valeurs exactement 0 % et 100 % sont des états électriques fixes. Les commandes `Prop` modifient la valeur en RAM sans remplacer le `PWM Duty` sauvegardé.

#### Exemples combinés

| PWM Mode | Intervall Mode | Effet lorsque la sortie est demandée ON |
|---|---|---|
| `Off` | `Off` | Allumage continu à pleine puissance. |
| `Off` | `On` | Clignotement à pleine puissance. |
| `On` | `Off` | Éclairage fixe atténué. |
| `On` | `On` | Clignotement atténué. |
| `On` | `Morse` | Morse lumineux atténué. |

Ces combinaisons décrivent le PWM **géré par ON/OFF** ; `Remote` suit `Prop` indépendamment de la demande de clignotement/Morse.

### `Morse` — paramètres communs aux huit sorties

Une sortie joue le message `Text1` si **`Intervall Mode=Morse`** et qu’elle reçoit ON depuis le widget ou `Operate`. `Text1`, les durées et `Repeat` sont **communs aux huit sorties** ; chaque sortie possède son propre état d’exécution.

| Champ | Valeur et signification |
|---|---|
| `Text1` | Message Morse commun, **15 caractères maximum**, `SOS` par défaut. Lettres, chiffres, espaces et `. , : ; ? ! - = +` pris en charge ; autres symboles refusés ou non codés. |
| `Dit duration` | Durée d’allumage d’un **point** (`.`), en pas de 100 ms. |
| `Dah duration` | Durée d’allumage d’un **trait** (`-`), en pas de 100 ms. |
| `Intra S. Gap dur.` | Pause **entre signes d’une même lettre**. |
| `Inter S. Gap dur.` | **Complément à Intra** pour la pause entre deux lettres. |
| `Inter W. Gap dur.` | **Complément à Intra** pour la pause entre deux mots, lorsqu’il y a un espace dans `Text1`. |
| `Repeat → Off/On` | **Off** : un message par commande OFF→ON (comme l’original) ; **On** : message répété tant que la demande de sortie reste ON. Extension de ce portage, absente du code Wilhelm d’origine. |
| `Repeat pause` | Pause **entre messages complets**, 1–100 × 100 ms (0,1–10 s) ; défaut `9` = 0,9 s. Applicable si `Repeat=On`. Extension ESP32-S3. |
| `Aide durees` | Sous-dossier **en lecture seule** expliquant les durées ; ne change aucun paramètre. Extension ESP32-S3 de v0.2h2. |

Les cinq durées Morse sont réglables de **1 à 10**, en unités de **100 ms**. Dans **ce portage**, la différence entre Intra et Inter est particulièrement importante :

| Espace | Calcul effectif |
|---|---|
| Entre deux points/traits d’une même lettre | `Intra × 100 ms` |
| Entre lettres | `(Intra + Inter S.) × 100 ms` |
| Entre mots | `(Intra + Inter W.) × 100 ms` |
| Entre deux messages répétés | `Repeat pause × 100 ms` |

**Exemple de rapports Morse usuels :** `Dit=1`, `Dah=3`, `Intra=1`, `Inter S.=2`, `Inter W.=6` produisent 100 ms/300 ms de lumière, 100 ms entre signes, 300 ms entre lettres et 700 ms entre mots. `Repeat pause=9` ajoute 900 ms avant le nouveau message complet. Ces valeurs sont un **exemple**, pas une modification automatique des valeurs enregistrées.

Avec `Repeat=Off`, le message s’éteint à la fin ; faire OFF puis ON pour le rejouer. Avec `Repeat=On`, il se répète jusqu’au OFF. Une commande OFF coupe immédiatement, même au milieu d’un point ou pendant la pause. Le PWM règle la **luminosité** des points et traits sans modifier leurs durées.

## 5. LED RGB de statut sur GPIO48

| Affichage | Sens dans ce firmware |
|---|---|
| **Vert fixe** | Des trames de voies CRSF valides arrivent récemment. |
| **Bref bleu** | Une commande MultiSwitch vient d’être traitée (environ 120 ms), puis retour au vert. C’est pourquoi le widget peut faire brièvement clignoter la LED même lorsque la réception reste excellente. |
| **Rouge** | Pas de trame CRSF récente / aucune liaison utilisable depuis au moins environ 1 s. |
| **Orange / éteint alternés** | Des trames CRSF récentes existent, mais aucune trame de voies valide depuis au moins 500 ms. |

Le bleu constitue volontairement un témoin visuel de l’activité du widget ; **il n’indique pas à lui seul une erreur de réception**.

## 6. Console de diagnostic USB

À **115200 bauds** ; commandes avec Entrée (`CR` ou `LF`) :

| Commande | Fonction |
|---|---|
| `D=0` | Diagnostics périodiques silencieux (défaut au démarrage). |
| `D=1` | Afficher une fois par seconde `[MSW]`, `[LINK]` et le bilan d’observation éventuel. |
| `D=2` | Diagnostics plus détaillés sur l’UART, le menu, la sauvegarde et les CRC. À réserver aux essais. |
| `D?` | Afficher le niveau courant ; `D` bascule entre 0 et 1. |
| `C=1` | Démarrer/remettre à zéro l’observateur brut CRSF. |
| `C?` | Lire ses compteurs et son échantillon `BEFORE / BAD / AFTER`, si disponible. |
| `C=0` | Arrêter l’observateur. |
| `?` ou `help` | Aide sur les commandes. |

`[MSW] outs=XX` représente les **sorties logiquement visibles/actives**, pas les fronts du PWM. `crcErr` et `[RX-CHECK]` servent au diagnostic : une trame candidate mal décodée ne prouve pas à elle seule un défaut du fil. `D=0` et `C=0` conviennent à l’exploitation normale.

## 7. Installation et premier test

1. Conserver une sauvegarde de la version précédente qui fonctionne. Ouvrir `RCMultiSwitch_ESP32S3.ino` dans **Arduino IDE**, avec la carte ESP32-S3 et un core Arduino-ESP32 compatible avec ce projet (travaux réalisés autour de la branche **3.0.7**).
2. Vérifier le **vrai GND commun**, GPIO12/13 croisés avec TX/RX de l’ER8, et l’absence de charge trop importante sur les GPIO.
3. Compiler et téléverser, puis ouvrir la console USB à 115200 bauds. L’adresse `Switch Addr` enregistrée en NVS peut différer de sa valeur par défaut.
4. Sur la radio, ouvrir `elrsV3.lua`, sélectionner le MultiSwitch et vérifier `Global`, `Failsafe`, `Operate`, les huit dossiers `Output` et `Morse`.
5. Sur OUT0 (GPIO4), essayer successivement : ON/OFF ; `Intervall Mode=On` ; `PWM Mode=On`, duty 10/50/90 % ; `Intervall Mode=Morse`, `Text1=SOS`, puis `Repeat=On`. Tester OFF à chaque étape.
6. Attendre **au moins 3–4 secondes après les derniers réglages** avant de redémarrer, puis vérifier qu’ils ont été mémorisés.

Ne pas confondre **PWM de gradation 1 kHz** avec les impulsions d’un **servo RC 1–2 ms** : les sorties de cette version ne sont pas des sorties servo.

## 8. Ce qui reste à porter ou à développer

Cette liste distingue volontairement le **code déjà présent** de la **feuille de route** :

- **Virtuals** : adresse et sorties virtuelles, groupes de sorties.
- **Global Dimming réellement commandable** depuis une adresse/commande virtuelle ; seul le calcul de multiplication est préparé dans `Global/Indiv`.
- **Patterns** : séquences entre plusieurs sorties et enchaînements.
- Autres options conditionnelles de l’original : commandes maître/esclave, menu Reset, télémétrie/capteurs et fonctions dépendantes de la carte STM32.
- **Réglage de polarité depuis `elrsV3.lua`** (global ou sortie par sortie) : **proposé, pas encore implémenté**.
- **Sortie UART série configurable** sur OUT0…7 : **pas implémentée**, et à distinguer du port CRSF de GPIO12/13.

Ne pas annoncer ces fonctions comme disponibles dans v0.2h2. Le principe de développement adopté est de préserver la réception CRSF fonctionnelle, puis d’ajouter et de valider les fonctions une par une sur la carte réelle.

---

**Crédits et attribution :** WMuCpp / RC MultiSwitch-E © Wilhelm Meier ; portage et essais ESP32-S3 avec Pierrot et assistance ChatGPT. **Licence du code : GPL-3.0-or-later.** Le projet d’origine et le présent portage doivent rester clairement distingués.
