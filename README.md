# LoraTap - 3 boutons (MQTT direct)

Blueprint Home Assistant pour piloter les 3 boutons d'une télécommande Zigbee LoraTap (via Zigbee2MQTT) à partir du topic MQTT brut du device : appui simple, double et long pour chaque bouton, soit 9 actions au total.

Testé et validé avec :

- Télécommande **LoraTap SS6400ZB 3 boutons** (Tuya)
- Home Assistant + add-on **Zigbee2MQTT** + broker **Mosquitto**
- Intégration **MQTT** installée dans Home Assistant

## Prérequis

1. **Home Assistant** (syntaxe YAML moderne `triggers:`/`actions:`).
2. Un broker MQTT — l'add-on **Mosquitto broker** par exemple.
3. **Zigbee2MQTT** (add-on ou installation standalone) avec la télécommande LoraTap appairée et opérationnelle.
4. L'**intégration MQTT** installée dans Home Assistant (**Paramètres → Appareils et services → Ajouter une intégration → MQTT**). C'est le pont entre le broker et HA : sans elle, aucun trigger MQTT ne fonctionne, aucun onglet MQTT dans les Outils de développement, aucune entité MQTT.
5. Connaître le **topic MQTT exact** de la télécommande : dans Zigbee2MQTT, ouvrir le device et lire le champ *MQTT base topic* (ou écouter dans HA via **Outils de développement → MQTT → Écouter** et appuyer sur les boutons).

## Pourquoi ce pattern ?

Ce blueprint utilise directement le trigger MQTT sur le topic du device (`zigbee2mqtt/Loratap Remote` par exemple) plutôt que l'entité `event.*` ou le capteur d'action de Zigbee2MQTT. Sur certaines télécommandes, l'exposition de ces entités est capricieuse ; le trigger MQTT brut fonctionne dans tous les cas.

L'automatisation suit le pattern éprouvé : **trigger large + condition de garde + choose**.

- Le trigger s'abonne au topic complet du device (préfixe `zigbee2mqtt/` inclus).
- La condition `{{ trigger.payload_json.action is defined }}` ignore les messages sans action (état, batterie, linkquality…).
- Le `choose` dispatche chaque valeur d'action (`1_single`, `1_double`, `1_hold`, `2_*`, `3_*`) vers la case correspondante. Un appui non mappé ne déclenche rien.

## Installation

**Option A — Import direct (recommandé)**

1. **Paramètres → Automatisations et scènes → Blueprints → Importer un blueprint**.
2. Coller l'adresse du fichier :
   ```
   https://github.com/thierrydolmiere/loratap-mqtt-blueprint/blob/main/loratap_bouton.yaml
   ```
3. Cliquer sur **Aperçu**, puis importer.

Attention : l'adresse doit pointer vers le **fichier** du blueprint, pas vers la page racine du dépôt GitHub — sinon l'import échoue avec une erreur YAML (HA essaie de parser la page HTML).

**Option B — Copie manuelle**

1. Copier `loratap_bouton.yaml` dans le dossier des blueprints de Home Assistant :
   - Via File Editor / Samba / SSH : `/config/blueprints/automation/custom/loratap_bouton.yaml`
2. Recharger les blueprints : **Outils de développement → YAML → Recharger les automatisations** (ou redémarrer HA).

Dans les deux cas, créer ensuite une automatisation : **Paramètres → Automatisations → Créer → Utiliser un blueprint**, puis choisir « LoraTap - 3 boutons (MQTT direct) ».

## Configuration

| Champ | Description |
|---|---|
| Topic MQTT du device | Le topic Zigbee2MQTT **exact** du device, copié depuis l'onglet d'écoute. Ex. `zigbee2mqtt/Loratap Remote` |
| Bouton 1/2/3 — appui simple | Actions exécutées sur `1_single` / `2_single` / `3_single` |
| Bouton 1/2/3 — appui double | Actions exécutées sur `1_double` / `2_double` / `3_double` |
| Bouton 1/2/3 — appui long | Actions exécutées sur `1_hold` / `2_hold` / `3_hold` |

Toutes les cases sont optionnelles (vides par défaut) : un appui correspondant à une case vide est silencieusement ignoré.

## Limitations connues (firmware LoraTap)

Bug connu sur la famille SS6400ZB : certaines versions émettent un `hold` **générique sans numéro de bouton** (voir [zigbee2mqtt #31862](https://github.com/Koenkk/zigbee2mqtt/issues/31862)). Ce blueprint testant l'égalité exacte (`1_hold`, `2_hold`…), un tel `hold` serait ignoré. À adapter si le cas apparaît.

## Dépannage

Vérifier dans l'ordre :

1. **Outils de développement → MQTT → Écouter** le topic du device et appuyer sur les boutons : confirmer le topic exact et les valeurs d'action. Le payload attendu ressemble à `{"action":"1_single","battery":80,"linkquality":42,"voltage":2900}`.
2. **Traces de l'automatisation** (Outils de développement → Suivi) :
   - Aucune trace → le topic est faux (casse, espaces, préfixe `zigbee2mqtt/` manquant).
   - Trace bloquée sur la condition → le message n'a pas de champ `action`.
   - « Aucune branche » → la valeur d'action n'est pas mappée ; la lire dans la trace.
3. Pièges fréquents :
   - Le topic MQTT est **sensible à la casse et aux espaces** : le copier caractère pour caractère.
   - L'intégration **MQTT de Home Assistant** doit être installée, sinon aucun trigger MQTT ne fonctionne.
   - Un `linkquality` faible (~40) peut faire perdre des appuis : ajouter un routeur Zigbee secteur à mi-chemin.
4. Après chaque modification de l'automatisation, les **traces sont effacées** : enregistrer, tester, puis lire la trace.

## Crédits

Blueprint réalisé avec l'inspiration de [HomeAssistant-zigbee2mqtt-remote](https://github.com/tommi00/HomeAssistant-zigbee2mqtt-remote/tree/main) de tommi00.
