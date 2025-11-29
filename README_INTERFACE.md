# Guide d'installation de l'interface ESP32 Electricity Cost

## 📋 Prérequis
- ESP32-C6 avec écran Waveshare 1.47in
- Home Assistant configuré
- Sensors existants: `sensor.prix_journalier_total` et `sensor.prix_mois_total`

## 🚀 Installation

### 1. Configuration ESPHome
Le fichier `electricityCostEspHome.yml` est déjà configuré avec:
- Interface graphique style "ELECTRIC COST"
- Récupération des données depuis Home Assistant
- Graphique des 4 derniers jours
- Contrôle de luminosité

### 2. Configuration Home Assistant

#### Option A: Via l'interface (Recommandé)
1. Allez dans **Paramètres** → **Appareils et services** → **Helpers**
2. Cliquez sur **+ Créer un helper** → **Nombre**
3. Créez 3 helpers:
   - Nom: `Prix Jour -3`, ID: `prix_jour_moins_3`
     - Min: 0, Max: 1000, Step: 0.01, Unité: €
   - Nom: `Prix Jour -2`, ID: `prix_jour_moins_2`
     - Min: 0, Max: 1000, Step: 0.01, Unité: €
   - Nom: `Prix Jour -1`, ID: `prix_jour_moins_1`
     - Min: 0, Max: 1000, Step: 0.01, Unité: €

#### Option B: Via configuration.yaml
Copiez le contenu du fichier `home_assistant_config.yaml` dans votre `configuration.yaml`

### 3. Créer l'automation de décalage

1. Allez dans **Paramètres** → **Automations et scènes**
2. Cliquez sur **+ Créer une automation** → **Créer une nouvelle automation**
3. Mode YAML et collez:

```yaml
alias: Décaler historique coût électricité
description: Décale les valeurs des 3 derniers jours chaque jour à minuit
trigger:
  - platform: time
    at: "00:00:01"
action:
  - service: input_number.set_value
    target:
      entity_id: input_number.prix_jour_moins_3
    data:
      value: "{{ states('input_number.prix_jour_moins_2') | float }}"
  - service: input_number.set_value
    target:
      entity_id: input_number.prix_jour_moins_2
    data:
      value: "{{ states('input_number.prix_jour_moins_1') | float }}"
  - service: input_number.set_value
    target:
      entity_id: input_number.prix_jour_moins_1
    data:
      value: "{{ states('sensor.prix_journalier_total') | float }}"
mode: single
```

### 4. Initialiser les valeurs historiques

Pour remplir l'historique initial (les 3 jours précédents):

1. Allez dans **Outils de développement** → **Services**
2. Pour chaque jour, appelez le service `input_number.set_value`:
   - Entity: `input_number.prix_jour_moins_3`
   - Value: `1.20` (votre valeur réelle)
3. Répétez pour `prix_jour_moins_2` et `prix_jour_moins_1`

Ou utilisez le script fourni dans `home_assistant_config.yaml`

### 5. Flash l'ESP32

```bash
esphome run electricityCostEspHome.yml
```

## 🎨 Personnalisation

### Modifier les polices
Les polices sont définies dans la section `font:` du fichier YAML.
Ajustez les tailles selon vos préférences.

### Modifier les couleurs
Dans le lambda du display:
- `Color::WHITE` → Blanc (255, 255, 255)
- `Color(200, 160, 0)` → Doré (couleur du graphique)
- `Color(180, 180, 180)` → Gris pour "This Month"

### Ajuster le graphique
```cpp
int graph_x = 30;        // Position X
int graph_y = 250;       // Position Y de la base
int graph_width = 112;   // Largeur
int graph_height = 50;   // Hauteur max
```

## 📊 Fonctionnement du graphique

Le système conserve les 4 dernières valeurs:
- **J-3**: Prix d'il y a 3 jours
- **J-2**: Prix d'il y a 2 jours  
- **J-1**: Prix d'hier
- **J actuel**: Prix du jour (sensor.prix_journalier_total)

Chaque jour à minuit, l'automation décale automatiquement les valeurs:
```
J-3 ← J-2 ← J-1 ← J actuel
```

## 🔧 Dépannage

### L'écran affiche "--,--€"
- Vérifiez que Home Assistant est connecté
- Vérifiez les entity_id dans le fichier YAML
- Vérifiez les logs ESPHome: `esphome logs electricityCostEspHome.yml`

### Le graphique ne s'affiche pas
- Vérifiez que les 4 sensors ont des valeurs
- Vérifiez les logs pour voir si les sensors sont bien récupérés
- Initialisez manuellement les valeurs historiques

### Les valeurs ne se décalent pas
- Vérifiez que l'automation est activée
- Vérifiez les logs Home Assistant
- Testez manuellement l'automation

## 📝 Structure des sensors requis

Dans Home Assistant, vous devez avoir:
```
sensor.prix_journalier_total    → Prix du jour actuel
sensor.prix_mois_total          → Prix du mois actuel
input_number.prix_jour_moins_3  → Prix J-3
input_number.prix_jour_moins_2  → Prix J-2
input_number.prix_jour_moins_1  → Prix J-1
```

## 🌟 Améliorations possibles

1. **Ajouter les heures creuses/pleines** avec des couleurs différentes
2. **Animation de transition** entre les valeurs
3. **Afficher la date** sous chaque point du graphique
4. **Ajouter un indicateur de tendance** (↑↓)
5. **Mode nuit** avec luminosité réduite automatiquement
