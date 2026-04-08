# 📚 HeroBook — Livre dont vous êtes le Héros

> Application Android pour jouer des aventures interactives au format "Livre dont vous êtes le Héros" — avec inventaire, sauvegardes, conditions, effets.

---

## ✨ Fonctionnalités

- 📖 **Lecture d'histoires interactives** au format JSON (nœuds, choix, fins multiples)
- 🎒 **Inventaire dynamique** — ajout, retrait et consultation des objets en temps réel
- ⚔️ **Conditions sur les choix** — un choix peut être verrouillé si le joueur ne remplit pas les prérequis (objet manquant, stat insuffisante, drapeau absent…)
- ✨ **Effets appliqués** à la sélection d'un choix ou à l'entrée dans un nœud (items, stats, flags)
- 🎲 **Lancer de dé** intégré dans le moteur de jeu
- 💾 **Sauvegarde automatique** par histoire (nœud courant, inventaire, stats, drapeaux, historique)
- 📥 **Import de fichiers JSON** depuis le stockage du téléphone
- 🔐 **Chiffrement AES-256-GCM** des histoires (format `.hbe`) via un script Python fourni
- 🎨 **Material Design 3** avec thème sombre/clair

---

## 📸 Aperçu

| Bibliothèque | Écran de jeu | Inventaire |
|:---:|:---:|:---:|
| *(liste des histoires importées)* | *(texte narratif + choix)* | *(bottom sheet)* |


### Dépendances principales

| Bibliothèque | Usage |
|---|---|
| Room 2.6 | Persistance des sauvegardes et histoires |
| Navigation Component 2.8 | Navigation entre fragments |
| ViewModel / LiveData 2.8 | Architecture MVVM |
| Gson 2.11 | Parsing des fichiers JSON |
| Coroutines 1.8 | Appels asynchrones |
| Material Design 3 | Interface utilisateur |

---

## 🏗️ Architecture

Le projet suit une architecture **MVVM** avec séparation stricte des couches :

```
app/src/main/java/com/herobook/app/
├── model/
│   └── Models.kt           — Data classes (StoryJson, StoryNode, Choice,
│                             Condition, ItemEffect, SaveState…)
├── data/
│   ├── Database.kt         — Base Room (SaveEntity, StoryEntity, DAOs)
│   └── Repository.kt       — Couche d'accès aux données (import, save/load)
├── engine/
│   └── GameEngine.kt       — Moteur de jeu : navigation, évaluation des
│                             conditions, application des effets, dé
├── ui/
│   ├── LibraryFragment.kt  — Écran bibliothèque (liste + import)
│   ├── LibraryViewModel.kt
│   ├── GameFragment.kt     — Écran de lecture/jeu
│   ├── GameViewModel.kt
│   ├── Adapters.kt         — RecyclerView adapters (histoires, choix, inventaire)
│   └── GameFragmentArgs.kt
├── util/
│   └── StoryEncryption.kt  — Chiffrement/déchiffrement AES-256-GCM
└── MainActivity.kt
```

Le `GameEngine` est **entièrement découplé de l'UI**, ce qui facilite les tests unitaires et l'extension du moteur.

---

## 📖 Format des histoires (JSON)

Les histoires sont des fichiers `.json` importés dans l'application. Deux histoires d'exemple sont incluses dans `sample_stories/`.

### Structure racine

```json
{
  "id": "mon_histoire_v1",
  "title": "Titre de l'histoire",
  "author": "Nom de l'auteur",
  "version": "1.0",
  "cover_description": "Description affichée dans la bibliothèque",
  "starting_node": "debut",
  "initial_inventory": [],
  "nodes": [ ... ]
}
```

### Structure d'un nœud

```json
{
  "id": "debut",
  "title": "Titre du passage",
  "text": "Le texte narratif du passage...",
  "is_ending": false,
  "actions_on_enter": [],
  "choices": [ ... ]
}
```

> `is_ending: true` → fin de partie — les boutons **Recommencer** et **Bibliothèque** s'affichent à la place des choix.

### Structure d'un choix

```json
{
  "id": "c1",
  "text": "Texte affiché au joueur",
  "target_node": "id_du_noeud_cible",
  "conditions": [ ... ],
  "effects": [ ... ]
}
```

---

## ⚙️ Conditions

Les conditions **verrouillent** un choix si non satisfaites. Le choix reste visible mais grisé avec le message d'erreur.

| Type | Paramètres | Description |
|---|---|---|
| `has_item` | `item_id`, `quantity`, `message` | Le joueur doit posséder l'objet en quantité suffisante |
| `not_has_item` | `item_id`, `quantity`, `message` | Le joueur ne doit PAS posséder l'objet |
| `stat_gte` | `stat`, `value`, `message` | Statistique ≥ valeur |
| `stat_lte` | `stat`, `value`, `message` | Statistique ≤ valeur |
| `stat_eq` | `stat`, `value`, `message` | Statistique == valeur |
| `flag_set` | `flag`, `message` | Le drapeau doit être activé |
| `flag_not_set` | `flag`, `message` | Le drapeau ne doit PAS être activé |

```json
{
  "type": "has_item",
  "item_id": "golden_key",
  "quantity": 1,
  "message": "Vous n'avez pas la clé dorée !"
}
```

---

## 🎒 Effets

Les effets sont appliqués à la sélection d'un choix ou à l'entrée dans un nœud (`actions_on_enter`).

| Type | Paramètres | Description |
|---|---|---|
| `add_item` | `item_id`, `item_name`, `quantity` | Ajouter un objet à l'inventaire |
| `remove_item` | `item_id`, `quantity` | Retirer un objet de l'inventaire |
| `set_stat` | `stat`, `value` | Définir une statistique |
| `add_stat` | `stat`, `value` | Ajouter à une statistique (peut être négatif) |
| `set_flag` | `flag` | Activer un drapeau booléen |
| `unset_flag` | `flag` | Désactiver un drapeau booléen |

```json
{ "type": "add_item", "item_id": "gold_coins", "item_name": "Pièces d'or", "quantity": 10 }
{ "type": "add_stat", "stat": "points_vie", "value": -5 }
{ "type": "set_flag", "flag": "a_rencontre_le_roi" }
```

---

## 💡 Exemple complet d'un passage

```json
{
  "id": "taverne",
  "title": "La Taverne du Dragon Ivre",
  "text": "Vous entrez dans une taverne bruyante. Le tavernier vous propose une bière...",
  "actions_on_enter": [
    { "type": "set_flag", "flag": "visite_taverne" }
  ],
  "choices": [
    {
      "id": "c1",
      "text": "Acheter une bière (2 pièces d'or)",
      "target_node": "apres_biere",
      "conditions": [
        { "type": "has_item", "item_id": "gold_coins", "quantity": 2, "message": "Vous n'avez pas assez d'or !" }
      ],
      "effects": [
        { "type": "remove_item", "item_id": "gold_coins", "quantity": 2 },
        { "type": "add_stat", "stat": "moral", "value": 5 }
      ]
    },
    {
      "id": "c2",
      "text": "Demander des informations",
      "target_node": "infos_quete",
      "conditions": [],
      "effects": [
        { "type": "set_flag", "flag": "connait_quete" }
      ]
    }
  ]
}
```

---

## 🔐 Chiffrement des histoires

HeroBook supporte le chiffrement des fichiers d'histoires au format **`.hbe`** (HeroBook Encrypted) via **AES-256-GCM**.


---

## 📥 Importer une histoire

1. Depuis la bibliothèque, appuyez sur le bouton flottant **"Importer une histoire"**
2. Sélectionnez un fichier `.json` ou `.hbe` dans le gestionnaire de fichiers
3. L'histoire apparaît dans la bibliothèque

Ou appuyez sur **"Charger l'histoire d'exemple"** pour tester directement avec une aventure incluse.

---

## 🗂️ Histoires incluses

| Fichier | Titre | Nœuds | Description |
|---|---|:---:|---|
| `dungeon_adventure.json` | Les Profondeurs du Donjon | 11 | Exploration d'un donjon avec clés, combats, pièces d'or et coffre au trésor |
| `village_mystery.json` | Le Mystère du Village | 8 | Enquête dans un village avec suspects et indices à réunir |

---

## 💾 Sauvegarde

- Sauvegarde via le bouton **💾** dans la barre d'outils de l'écran de jeu
- **Une sauvegarde par histoire** (écrasement)
- Contenu sauvegardé : nœud courant, inventaire complet, statistiques, drapeaux actifs, historique des nœuds visités
- Au retour sur une histoire sauvegardée : choix entre **Reprendre** ou **Nouvelle partie**

---

## 🔧 Étendre le moteur

Pour ajouter de nouveaux types de conditions ou d'effets :

- `GameEngine.kt` → méthodes `evaluateSingle()` et `applyEffect()`
- `Models.kt` → ajoutez les champs nécessaires à `Condition` et `ItemEffect`

---

## 📋 Feuille de route

- [ ] Éditeur d'histoires intégré
- [ ] Support des images par nœud
- [ ] Musique et effets sonores
- [ ] Export / partage d'histoires depuis l'app
- [ ] Sauvegarde multiple par histoire
- [ ] Localisation (EN / ES / DE)


