# PLAN D'IMPLÉMENTATION — Projet **AEGIS**

> **Document destiné à une IA de codage.** Chaque phase est autonome, ordonnée, et se termine par des **critères d'acceptation** vérifiables. Ne pas passer à la phase suivante tant que les critères de la phase courante ne sont pas remplis.
> **Référence design** : [`GDD.md`](GDD.md) (v1.0). En cas de conflit, le GDD fait foi pour le *quoi*, ce document fait foi pour le *comment* et l'*ordre*.
> **Stack** : Godot **4.x** (dernière version stable), GDScript pour gameplay/UI, C# autorisé pour les systèmes lourds (procgen, pathfinding) uniquement si un besoin de performance est démontré.

---

## 0. Conventions générales (à respecter dans TOUTES les phases)

### 0.1 Structure du projet Godot
```
res://
├── project.godot
├── addons/                    # plugins éventuels
├── assets/                    # art, audio (placeholders au début)
│   ├── sprites/
│   ├── tilesets/
│   └── audio/
├── data/                      # TOUTES les données de gameplay (data-driven)
│   ├── spells/                # *.tres (SpellData)
│   ├── classes/               # *.tres (ClassData)
│   ├── weapons/               # *.tres (WeaponArchetypeData)
│   ├── items/                 # bases d'objets (*.tres ItemBaseData)
│   ├── affixes/               # *.tres (AffixData)
│   ├── surfaces/              # *.tres (SurfaceData) + règles de combinaison
│   ├── monsters/              # *.tres (MonsterData)
│   ├── weather/               # *.tres (WeatherData)
│   └── biomes/                # *.tres (BiomeData)
├── scripts/
│   ├── core/                  # autoloads, helpers, RNG, événements
│   ├── resources/             # définitions des schémas Resource (SpellData, ItemBaseData...)
│   ├── grid/                  # grille, pathfinding, ligne de vue
│   ├── combat/                # machine à états, tours, actions (Command)
│   ├── entities/              # personnage, monstre, objet de terrain
│   ├── items/                 # génération d'items, inventaire, équipement
│   ├── world/                 # génération procédurale, chunks, météo
│   ├── ai/                    # IA des monstres
│   └── ui/                    # HUD, inventaire, menus
├── scenes/                    # scènes .tscn (miroir de scripts/)
└── tests/                     # tests GUT (voir §0.4)
```

### 0.2 Règles de code
- **Langue** : code et identifiants en **anglais** ; textes affichés en français via un fichier de localisation (`res://data/locale/fr.csv`) dès le début.
- **Data-driven obligatoire** : aucun sort, item, affixe, monstre ou surface codé en dur. Tout est un `Resource` (`.tres`) chargé depuis `res://data/`. Le code implémente des *comportements génériques* paramétrés par ces données.
- **Signaux/EventBus** : un autoload `EventBus` centralise les événements de gameplay (ex. `turn_started`, `entity_died`, `surface_created`, `item_dropped`). Les systèmes s'abonnent, jamais de couplage direct UI↔gameplay.
- **RNG déterministe** : un autoload `Rng` encapsulant `RandomNumberGenerator` avec seed exposée (indispensable pour la procgen, les tests et le futur réseau).
- **Pattern Command pour toute action de combat** (déplacement, sort, interaction décor) : chaque action est un objet avec `validate()`, `get_preview()`, `execute()`, et est **sérialisable en Dictionary** (préparation réseau/replay).
- Typage GDScript strict (`--warnings-as-errors` sur les types), noms de classes en `PascalCase` via `class_name`.

### 0.3 Autoloads (créés en Phase 0)
| Autoload | Rôle |
|---|---|
| `EventBus` | signaux globaux de gameplay |
| `Rng` | RNG seedé central |
| `Db` | chargement/registre de toutes les Resources de `res://data/` au démarrage |
| `GameState` | état de la partie (personnage courant, monde, seed, mode) |
| `SaveManager` | sauvegarde/chargement (Phase 10) |

### 0.4 Tests & validation
- Installer **GUT** (Godot Unit Test) via l'AssetLib dès la Phase 0.
- Chaque phase ajoute des tests unitaires sur sa logique pure (règles de combat, génération d'affixes, propagation de surfaces...). La logique de règles doit être **testable sans scène** (classes `RefCounted` séparées des nœuds).
- Critère permanent : `godot --headless -s addons/gut/gut_cmdln.gd` passe sans échec, le projet se lance sans erreur ni warning de parse.

---

## PHASE 0 — Initialisation du projet

**Objectif** : projet Godot vide, propre, exécutable, avec l'ossature ci-dessus.

Tâches :
1. Créer le projet Godot 4.x « aegis », renderer **Compatibility** (pixel art 2D), résolution de base 1280×720, `stretch mode = canvas_items`, filtre de texture **Nearest** (pixel art).
2. Créer l'arborescence §0.1 (dossiers + `.gitkeep`), le `.gitignore` Godot standard (`.godot/`, exports).
3. Créer les autoloads `EventBus`, `Rng`, `Db`, `GameState` (squelettes documentés).
4. Installer GUT, créer `tests/test_smoke.gd` (1 test trivial qui passe).
5. Scène `scenes/main.tscn` : un `Node2D` racine + label « AEGIS prototype » ; définie comme scène principale.

**Critères d'acceptation** :
- [ ] Le projet s'ouvre et se lance sans erreur (`godot --headless --quit` OK).
- [ ] GUT s'exécute et le test smoke passe.
- [ ] Les 4 autoloads existent et sont enregistrés dans `project.godot`.

---

## PHASE 1 — Grille isométrique, curseur, pathfinding

**Objectif** : une arène isométrique en losange affichée, un curseur de case, du pathfinding.

Tâches :
1. `scripts/grid/grid_map.gd` (`class_name CombatGrid`, `RefCounted`) : modèle **logique** de grille (indépendant du rendu) — dimensions, cellules `{walkable, blocks_los, height, occupant, surface}` ; conversions coordonnées grille ↔ monde isométrique (losange, type Dofus : cases 64×32 px).
2. Rendu : `TileMapLayer` en mode isométrique + tileset placeholder (2 tuiles : sol, obstacle). La grille logique est la source de vérité ; le TileMap n'est que la vue.
3. Curseur de case : surbrillance de la cellule sous la souris (conversion écran→grille).
4. Pathfinding : wrapper autour d'`AStarGrid2D` (mode diagonal désactivé — déplacements orthogonaux sur le losange, comme Dofus) exposant `find_path(from, to) -> Array[Vector2i]` et `cells_within_range(from, energy) -> Array[Vector2i]` (Dijkstra borné).
5. Ligne de vue : `has_los(from, to) -> bool` par Bresenham sur la grille logique, bloquée par `blocks_los`.
6. Scène de test `scenes/combat/arena_test.tscn` : arène 15×15 avec obstacles placés à la main, affichage du chemin au clic.

**Critères d'acceptation** :
- [ ] Cliquer une case affiche le chemin le plus court en surbrillance ; les obstacles sont contournés.
- [ ] `cells_within_range` surligne exactement les cases atteignables pour N points.
- [ ] Tests unitaires : conversions de coordonnées, pathfinding (cas simple, obstacle, cible inaccessible), LOS (dégagée, bloquée, diagonale).

---

## PHASE 2 — Entités & machine à états de combat

**Objectif** : des unités sur la grille, un déroulé de combat complet (placement → tours → victoire/défaite).

Tâches :
1. `scripts/entities/combat_entity.gd` : entité de combat (joueur ou monstre) = données (`stats`, `team`, position grille) + nœud visuel (sprite placeholder). Stats de base : `max_hp, hp, mana_per_turn, energy_per_turn, initiative, tackle, dodge, resistances{}`.
2. `scripts/combat/combat_state_machine.gd` : états `SETUP → PLACEMENT → TURN_LOOP (start_turn → acting → end_turn) → RESULT`. Implémentation en machine à états explicite (enum + `_enter/_exit`).
3. Ordre d'initiative : tri décroissant, recalculé si l'initiative change ; **timeline** exposée pour l'UI.
4. Tour : au `start_turn`, recharge Mana/Énergie ; timer de tour **120 s** (config dans `GameState`) ; bouton/entrée « Fin de tour » ; à expiration → fin de tour automatique.
5. Phase de placement : cases de départ par équipe (2 zones), clic pour placer, bouton « Prêt ».
6. Conditions de fin : une équipe éliminée → état `RESULT` (écran victoire/défaite placeholder).
7. `EventBus` : émettre `combat_started, placement_done, turn_started(entity), turn_ended(entity), combat_ended(winner)`.

**Critères d'acceptation** :
- [ ] Un combat 1 joueur vs 2 mannequins (immobiles) se déroule : placement, tours dans l'ordre d'initiative, timer visible, fin de combat détectée.
- [ ] Tests unitaires : ordre d'initiative (égalités incluses), recharge des ressources, transitions d'états.

---

## PHASE 3 — Actions en pattern Command : déplacement & tacle

**Objectif** : le socle de TOUTES les actions de combat, avec le déplacement comme première implémentation.

Tâches :
1. `scripts/combat/actions/combat_action.gd` (`class_name CombatAction`, abstraite) : interface `validate(state) -> bool`, `get_preview(state) -> PreviewData`, `execute(state) -> void` (émet les événements), `to_dict()/from_dict()` (sérialisation).
2. `MoveAction` : coût = 1 Énergie/case, suit le chemin A*, animation de déplacement case par case (tween), met à jour la grille logique.
3. **Tacle/fuite** : quitter une case adjacente à un ennemi coûte Mana+Énergie selon la formule Dofus-like `perte = clamp(...)` basée sur `tackle` vs `dodge` — implémentée dans une classe de règles pure `scripts/combat/rules/tackle_rules.gd` avec tests.
4. Prévisualisation : au survol, affichage du chemin, du coût en Énergie, et de la perte de tacle éventuelle **avant** de cliquer.
5. Historique : `CombatLog` (autoload ou nœud du combat) enregistre chaque action sérialisée (base des replays/réseau).

**Critères d'acceptation** :
- [ ] Le joueur se déplace avec coût d'Énergie correct, ne peut pas dépasser son Énergie, subit le tacle en quittant un ennemi adjacent.
- [ ] Toute action exécutée apparaît sérialisée dans le log.
- [ ] Tests unitaires : `MoveAction.validate` (énergie insuffisante, case occupée), formule de tacle, aller-retour `to_dict/from_dict`.

---

## PHASE 4 — Système de sorts data-driven

**Objectif** : lancer des sorts définis entièrement en données, avec ciblage, zones d'effet et prévisualisation.

Tâches :
1. `scripts/resources/spell_data.gd` (`class_name SpellData`, `Resource`) — les définitions de schémas `Resource` vivent dans `scripts/resources/`, les fichiers de données `.tres` dans `res://data/` :
   ```
   id, display_name_key, icon, element (enum FIRE/WATER/AIR/EARTH/LIGHT/SHADOW),
   mana_cost, energy_cost, range_min, range_max, needs_los, range_modifiable,
   aoe_shape (enum SINGLE/CROSS/LINE/CIRCLE/CONE), aoe_size,
   cooldown, uses_per_turn, uses_per_target,
   effects: Array[EffectData]
   ```
2. `EffectData` (`Resource`) : effets composables — `DAMAGE {element, base_min, base_max}`, `HEAL`, `PUSH {cells}`, `PULL {cells}`, `TELEPORT_ENTITY`, `TELEPORT_OBJECT`, `CREATE_SURFACE {surface_id, pattern}`, `BUFF/DEBUFF {stat, amount, duration}`, `RESOURCE_DRAIN {mana|energy}`, `SUMMON {monster_id}`. Chaque type d'effet = une petite classe d'exécution enregistrée dans un registre `EffectExecutor`.
3. `CastSpellAction` (Command) : validation (coût, portée avec LOS, cooldown, limites), calcul des cases d'AoE selon la forme, exécution des effets sur chaque cible.
4. Formule de dégâts (classe pure `damage_rules.gd`) : `final = (base + stat_scaling) * (1 - res_%) - res_fixe`, critiques inclus. Testée unitairement.
5. **Poussée** : trajectoire case par case ; collision avec mur/entité = dégâts de collision aux deux (formule dédiée testée) ; s'arrête sur obstacle.
6. UI : barre de sorts (chargée depuis les `SpellData` du personnage), tooltip complet, surbrillance portée (bleu) et AoE (rouge) au survol, **prévisualisation des dégâts estimés** sur les cibles.
7. Créer **8 sorts du Pyromant** en `.tres` dans `data/spells/pyromant/` (4 feu dont 2 créant des surfaces, 1 poussée, 1 téléport d'objet, 1 buff Surchauffe placeholder, 1 drain d'Énergie).

**Critères d'acceptation** :
- [ ] Les 8 sorts se lancent avec coûts, portées, cooldowns et AoE corrects ; ajouter un 9e sort ne demande **que** la création d'un `.tres`.
- [ ] La poussée déplace la cible et inflige des dégâts de collision contre un mur.
- [ ] Tests unitaires : formule de dégâts (résistances, critique), validation de portée/LOS, formes d'AoE, trajectoires de poussée.

---

## PHASE 5 — Surfaces élémentaires & météo

**Objectif** : la couche « environnement réactif » : surfaces qui se créent, se combinent, se propagent, et météo qui module tout.

Tâches :
1. `SurfaceData` (`Resource`) : `id, display_name_key, tile, on_enter_effects, on_turn_start_effects, blocks_los, energy_penalty, duration_turns`.
2. **Table de combinaison data-driven** : `data/surfaces/combinations.tres` — liste de règles `{surface_a, incoming_element_or_surface, result}` (ex. `WATER + FIRE_ELEMENT → STEAM`, `OIL + FIRE → FIRE_SURFACE(explosion)`). Le moteur applique la table, jamais de `if` codés en dur par combinaison.
3. Intégration grille : couche `surface` par cellule ; rendu par `TileMapLayer` dédié + motif distinctif par surface (accessibilité daltonien : texture ≠ couleur seule).
4. Déclencheurs : entrée sur case (glissade de la glace = déplacement forcé dans la direction du mouvement), début de tour (dégâts du feu/poison), sort touchant la case (combinaisons), expiration (durée).
5. Implémenter les 6 surfaces du GDD §5.1 : feu, eau, glace, huile, poison, vapeur/fumée.
6. **Météo** : `WeatherData` (`Resource`) : `id, surface_modifiers, element_modifiers, los_range_cap, wind_direction`. Le combat lit la météo courante de `GameState`. Implémenter pluie (eau + feu affaibli) et canicule (évaporation + feu étendu) ; les 3 autres météos en M3.
7. Prévisualisation : le survol d'un sort affiche les surfaces qui seront créées/transformées.

**Critères d'acceptation** :
- [ ] Feu sur huile explose et enflamme la nappe ; eau + froid gèle ; feu + eau fait de la vapeur qui bloque la LOS ; la glace fait glisser.
- [ ] Sous la pluie, les surfaces de feu s'éteignent en fin de tour et les dégâts de feu sont réduits du % défini en donnée.
- [ ] Ajouter une combinaison = 1 ligne dans la table, zéro code.
- [ ] Tests unitaires : moteur de combinaison (table complète), durées/expirations, glissade, modificateurs météo.

---

## PHASE 6 — Objets de terrain interactifs

**Objectif** : le décor devient un acteur du combat.

Tâches :
1. `scripts/entities/terrain_object.gd` : objet de grille avec tags `{pushable, destructible, trigger}` + PV éventuels + contenu éventuel (tonneau d'huile → surface d'huile à la destruction).
2. `InteractAction` (Command) : interactions au contact coûtant de l'**Énergie** (pousser : 2, levier : 1) — réutilise la logique de poussée de la Phase 4 pour les objets.
3. Destructibles : PV, détruits par dégâts ; à la destruction → événement (`surface spawn`, ouverture de LOS, dégâts de zone pour un pilier).
4. Déclencheurs : levier (ouvre porte), plaque de pression (déclenche piège), brasero (enflamme la case), lustre (chute = dégâts de zone) — chacun paramétré en données (`TerrainObjectData`).
5. Hauteur légère : cases `height=1` → +1 portée, +10 % dégâts vers le bas, +1 Énergie pour monter (règles dans `height_rules.gd`, testées).
6. Peupler l'arène de test : 2 tonneaux d'huile, 1 rocher poussable, 1 muret destructible, 1 brasero, 1 zone surélevée (≥ 3 interactifs, règle GDD §5.4).

**Critères d'acceptation** :
- [ ] Pousser un tonneau d'huile contre un mur le détruit et crée une nappe d'huile, qu'un sort de feu fait exploser.
- [ ] Détruire le muret ouvre une ligne de vue auparavant bloquée.
- [ ] La hauteur applique portée/dégâts/coût conformes.
- [ ] Tests unitaires : règles de hauteur, destruction et effets à la mort d'objet, coûts d'interaction.

---

## PHASE 7 — IA des monstres (exploite le décor) — fin du jalon M1

**Objectif** : des monstres crédibles qui utilisent sorts ET environnement.

Tâches :
1. `MonsterData` (`Resource`) : stats, sorts (réutilise `SpellData`), comportement (`role : MELEE/RANGED/CASTER/SUPPORT`), pondérations d'IA.
2. IA **utility-based** (`scripts/ai/`) : à son tour, le monstre génère toutes les actions possibles (déplacements ciblés, chaque sort sur chaque cible valide, interactions décor accessibles), les **score** (dégâts espérés, position — distance/LOS/hauteur, valeur environnementale : pousser vers le feu, enflammer l'huile sous un joueur, éviter les surfaces nocives), et exécute la meilleure séquence jusqu'à épuisement des ressources.
   - Le score environnemental réutilise les **previews** des Commands (aucune duplication de règles).
   - Budget : profondeur de simulation 1 tour, cap de temps 2 s/monstre (au-delà → meilleure action trouvée).
3. Créer 3 monstres : « Gobelin sabreur » (mêlée, pousse les joueurs vers les surfaces), « Chaman des braises » (caster feu, enflamme l'huile), « Crapaud des marais » (poison + attire avec la langue).
4. Combat de test : 1 Pyromant vs 4–6 monstres mixtes dans l'arène de la Phase 6.

**Critères d'acceptation** :
- [ ] Un monstre pousse un joueur dans le feu ou enflamme une nappe d'huile sous un joueur quand l'occasion se présente.
- [ ] Les monstres ne marchent pas volontairement dans les surfaces qui les blessent (sauf si le gain scoré est supérieur).
- [ ] Le tour complet des 6 monstres se résout en < 15 s.
- [ ] Tests unitaires : fonction de scoring sur des situations construites (poussée mortelle disponible → choisie).
- [ ] **Jalon M1 atteint : le prototype tactique est jouable et « fun-testable ».**

---

## PHASE 8 — Items, affixes procéduraux, inventaire & équipement

**Objectif** : la génération de loot type Diablo, entièrement data-driven.

Tâches :
1. Resources :
   - `ItemBaseData` : `id, slot (enum des 11 slots GDD §7.1), tier (1–8), implicit_stats, tags [weapon/armor/jewelry/...], weapon_archetype?, base_weight`.
   - `AffixData` : `id, type (PREFIX/SUFFIX), stat, value_range_per_tier (Array[Vector2]), weight, allowed_tags, excluded_tags, family (mutuellement exclusifs par famille)`.
   - `RarityData` : `id, color, affix_count_min/max, drop_weight, unique_power (légendaire)`.
2. `ItemGenerator` (classe pure, seedée via `Rng`) : `generate(base_id, item_level, forced_rarity?) -> ItemInstance`.
   - Tirage de rareté pondéré → tirage d'affixes dans les pools filtrés par tags/tier, sans doublon de famille → tirage des valeurs dans les plages.
   - `ItemInstance` = `{uuid, base_id, rarity, affixes:[{affix_id, value}], item_level, power_score}` — **sérialisable en Dictionary/JSON** (sauvegarde).
3. Contenu initial : 12 bases (2 armes par archétype manquant plus tard — au minimum épée, bâton, plastron, bottes, amulette, anneau...), 25 affixes (dont 5 **environnementaux**, GDD §7.3), 4 raretés (commun→épique ; légendaire/set en M3).
4. **Inventaire & équipement** : `Inventory` (liste + poids max porté par la Force), `Equipment` (11 slots) ; équiper/déséquiper recalcule les stats du personnage (agrégation implicits + affixes) via `StatSheet` (classe pure testée).
5. **Loot en combat** : table de loot par monstre (`loot_table` dans `MonsterData`) ; à la victoire, écran de butin ; la **Chance** (prospection) module la quantité/rareté.
6. UI inventaire : grille, tooltip d'objet complet (base, affixes colorés par rareté, score de puissance), comparaison avec l'objet équipé, poids.

**Critères d'acceptation** :
- [ ] 1000 générations d'items ne produisent aucune violation (doublon de famille, affixe hors tags, valeur hors plage) — test automatisé.
- [ ] Même seed → même item (déterminisme testé).
- [ ] Équiper une arme/armure modifie les stats en combat (dégâts, résistances) de façon vérifiable.
- [ ] Tests unitaires : générateur (distributions, contraintes), `StatSheet`, poids d'inventaire.

---

## PHASE 9 — Matrice classe × arme & 2e classe

**Objectif** : le système de sorts hybride, signature du jeu (GDD §4).

Tâches :
1. `ClassData` (`Resource`) : `id, base_stats, class_spells (Array[SpellData] avec niveau de déblocage), class_mechanic_id`.
2. `WeaponArchetypeData` : `id (SWORD/DAGGER/HAMMER/BOW/STAFF/FOCUS), granted_spells: Dictionary[class_id -> Array[SpellData]]` — **la matrice classe×arme est une donnée**, pas du code.
3. Recalcul de la barre de sorts à l'équipement : kit de classe (fixe) + sorts d'arme (dépendent de l'arme équipée ET de la classe). Distinction visuelle dans l'UI (cadre différent).
4. Mécanique de classe v1 : implémenter **Surchauffe** du Pyromant (jauge 0–100, +X par sort de feu, paliers modifiant les sorts, auto-brûlure au-delà d'un seuil) comme un composant générique `ClassMechanic` piloté par données autant que possible.
5. Ajouter la 2e classe : **Sylve** (kit de 8 sorts + mécanique **Ronces** : les sorts posent/consomment des ronces sur le terrain — réutilise le système de surfaces avec une surface `BRAMBLE` alliée).
6. Contenu : sorts d'arme pour épée, arc et bâton pour les 2 classes (2 sorts × 3 armes × 2 classes = 12 `.tres`).

**Critères d'acceptation** :
- [ ] Changer d'arme en combat (hors combat pour v1 si plus simple) change les sorts d'arme affichés, différents entre Pyromant et Sylve pour la même épée.
- [ ] La Surchauffe monte, modifie les sorts au palier, inflige l'auto-brûlure au seuil.
- [ ] Les Ronces du Sylve se posent et sont consommées par ses sorts.
- [ ] Tests unitaires : résolution de la matrice (classe, arme) → sorts, logique de jauge Surchauffe.

---

## PHASE 10 — Hardcore, mort, camp de base, craft & sauvegarde — fin du jalon M2

**Objectif** : la boucle de risque complète : mourir = tout perdre, revenir au camp, se re-stuffer par le craft.

Tâches :
1. **Mort hardcore** : à la défaite du combat (ou mort non ranimée — l'agonie coop est en Phase 14, en solo : défaite = mort), destruction définitive de l'équipement porté + inventaire transporté. Le personnage conserve niveaux/maîtrises/recettes. Écran de mort récapitulant ce qui a été perdu.
2. **Camp de base** : scène dédiée (hub) avec **coffre** (stockage sûr, capacité limitée extensible), **atelier de craft**, point de réapparition.
3. **Craft v1** : `RecipeData` (`Resource`) : ingrédients (ressources), base d'objet produite, tier ; produit des objets **communs/magiques** (affixes tirés aléatoirement pour magique). Ressources lootées sur les monstres/nœuds pour l'instant.
4. **Réforge v1** : re-tirer un affixe choisi contre ressources, coût croissant par re-tirage (compteur sur l'`ItemInstance`).
5. **Sauvegarde** (`SaveManager`) : sérialisation JSON de `GameState` (personnage, stats, inventaire/coffre/équipement en `ItemInstance` dicts, seed du monde, position) ; **une seule slot par personnage, écriture immédiate à chaque événement clé** (anti save-scumming : sauvegarde à la mort AVANT l'écran de mort) ; checksum simple du fichier.
6. **XP & niveaux** : gain d'XP en fin de combat (courbe data-driven `data/progression/xp_curve.tres`), points de caractéristiques à répartir (UI simple), déblocage des sorts de classe par niveau.

**Critères d'acceptation** :
- [ ] Mourir détruit définitivement le stuff porté (vérifié après rechargement de la sauvegarde) ; le coffre du camp est intact.
- [ ] Tuer le jeu (force quit) pendant l'écran de mort ne permet PAS de récupérer le stuff (la sauvegarde a eu lieu avant).
- [ ] On peut crafter un kit commun complet au camp et repartir combattre.
- [ ] Tests unitaires : sérialisation aller-retour complète de `GameState`, destruction à la mort, coûts de réforge croissants.
- [ ] **Jalon M2 atteint : la boucle hardcore complète est jouable.**

---

## PHASE 11 — Monde continu procédural (exploration)

**Objectif** : sortir de l'arène de test : un monde continu type Wakfu, généré par seed, exploré en temps réel.

Tâches :
1. **Exploration temps réel** : déplacement libre du personnage (8 directions, `CharacterBody2D`) sur le monde isométrique ; la grille de combat n'existe qu'en combat.
2. **Génération par chunks** : monde découpé en chunks (ex. 32×32 tuiles) générés à la demande depuis la **seed** ; pipeline : bruit (FBM) → biome par région (2 biomes v1 : plaines, forêt) → placement de patterns conçus main (bosquets, ruines, camps de monstres, nœuds de récolte) via WFC simple ou stamping pondéré. **Écrire cette couche en C# si le profilage GDScript montre > 16 ms/chunk.**
3. **Difficulté par distance** : `danger_level = f(distance au camp)` → tier des monstres et du loot ; affiché sur l'UI.
4. **Spawns & déclenchement de combat** : groupes de monstres visibles sur la carte (2–10 unités) ; contact → transition vers le combat ; **l'arène est générée à partir du décor local** (obstacles/objets/surfaces/météo du lieu, en garantissant ≥ 3 interactifs et des zones de placement valides).
5. **Météo du monde** : cycle météo par région (horloge de jeu) ; la météo courante est passée au combat (Phase 5).
6. **Nœuds de récolte** : interaction en exploration → ressources de craft (alimente la Phase 10).
7. Carte du monde (UI) : brouillard de découverte, position du camp, niveau de danger, météo.

**Critères d'acceptation** :
- [ ] Même seed → même monde (test automatisé sur les données de 10 chunks).
- [ ] On marche du camp vers l'extérieur sans écran de chargement ; les monstres et le loot deviennent plus forts avec la distance.
- [ ] Un combat déclenché dans la forêt sous la pluie produit une arène avec des arbres (obstacles), des interactifs et la météo pluie active.
- [ ] Génération d'un chunk < 16 ms en moyenne (pas de freeze perceptible).

---

## PHASE 12 — Donjons procéduraux & boss

**Objectif** : le contenu endgame : donjons générés, boss à mécaniques.

Tâches :
1. Générateur de donjons : graphe de 4–6 salles (combat / énigme environnementale / trésor / boss) ; salles = templates conçus main instanciés avec variations (seed).
2. Entrées de donjon placées dans le monde (Phase 11), tier lié au danger local.
3. **1 boss** : « Forgeron de Surtr » (thème nordique/steampunk) — mécaniques scriptées par données autant que possible : phases par seuil de PV, invocations, manipulation massive de surfaces (le sol s'enflamme par motifs télégraphiés un tour à l'avance).
4. Table de loot de donjon : garantie de rare+, chance d'épique, ressources de craft rares.

**Critères d'acceptation** :
- [ ] Un donjon complet (entrée → salles → boss → trésor) se joue de bout en bout ; deux seeds donnent deux agencements différents.
- [ ] Le boss télégraphie ses zones un tour à l'avance et change de comportement par phase.

---

## PHASE 13 — Légendaires, sets, 5 classes, polissage — fin du jalon M3

**Objectif** : compléter le vertical slice.

Tâches :
1. Raretés **Légendaire** (pouvoirs uniques : implémentés comme des « règles modificatrices » enregistrées, déclenchées par événements de l'EventBus — ex. « vos poussées créent de la glace » s'abonne à `entity_pushed`) et **Set** (bonus par nombre de pièces). 5 légendaires + 1 set.
2. Classes restantes : **Bastion**, **Ombrelame**, **Oracle** (kits + mécaniques + sorts d'arme pour les 6 archétypes).
3. Météos restantes : neige/gel, brouillard, tempête (vent déviant les poussées).
4. Biomes 3 et 4 : terres gelées (scandinave), vallées brumeuses (asiatique).
5. Variantes de sorts (2 par sort de classe, choix exclusif dans l'UI de progression).
6. Équilibrage : passe sur les courbes (dégâts, XP, drops) — toutes en données, exportables en CSV pour itération.
7. Accessibilité : motifs daltoniens vérifiés, vitesse d'animation réglable.

**Critères d'acceptation** :
- [ ] Un légendaire change concrètement une règle de jeu, sans code spécifique dans les systèmes de combat (uniquement via le registre de pouvoirs).
- [ ] 5 classes jouables avec leurs mécaniques ; 4 biomes ; 5 météos.
- [ ] **Jalon M3 (vertical slice) atteint.**

---

## PHASE 14 — Coop en ligne 4 joueurs — jalon M4

**Objectif** : la coop, en capitalisant sur l'architecture Command.

Tâches :
1. API multiplayer **Godot 4** : `ENetMultiplayerPeer` + `MultiplayerAPI` (RPCs `@rpc`), avec `MultiplayerSpawner`/`MultiplayerSynchronizer` pour la réplication de scène (ne PAS utiliser les patterns Godot 3). **Hôte autoritaire** : les invités envoient leurs `CombatAction` sérialisées (déjà prêtes depuis la Phase 3), l'hôte valide/exécute/rediffuse.
2. Lobby simple : héberger / rejoindre par IP ou code (pas de matchmaking).
3. Exploration synchronisée (positions, spawns, météo, seed partagée) ; combat : jusqu'à 4 joueurs dans la timeline, chacun jouant son tour (les autres voient les previews de l'actif).
4. **Agonie coop** (GDD §6.2) : 2 tours pour ranimer, sorts/consommables de résurrection.
5. Équilibrage dynamique : PV/nombre de monstres selon la taille du groupe (courbe en donnée).
6. **Loot instancié par joueur** ; le hardcore s'applique individuellement.
7. Résilience : déconnexion d'un invité → son personnage devient IA passive (skip de tour) jusqu'à reconnexion.

**Critères d'acceptation** :
- [ ] 2 instances du jeu (même machine) : l'invité rejoint, explore, combat en tour par tour partagé, loot son propre butin.
- [ ] Toute action d'un invité invalide côté hôte est rejetée proprement (pas de désync).
- [ ] La mort d'un joueur détruit SON stuff seulement.
- [ ] **Jalon M4 atteint.**

---

## Annexe A — Schémas de données récapitulatifs (contrats pour l'IA de codage)

```gdscript
# ItemInstance (Dictionary sérialisable)
{
  "uuid": String, "base_id": String, "rarity": String,
  "item_level": int, "power_score": int, "reforge_count": int,
  "affixes": [ { "affix_id": String, "value": float } ],
  "unique_power_id": String # légendaires uniquement, sinon absent
}

# CombatAction sérialisée (Dictionary)
{ "type": "move|cast|interact|end_turn", "actor_uuid": String,
  "payload": { ... } , "seq": int }

# Sauvegarde (JSON, 1 fichier par personnage)
# NB : partout ci-dessous, "ItemInstance" désigne la forme Dictionary sérialisée
# définie plus haut (jamais une référence d'objet GDScript).
{ "version": int, "checksum": String, "world_seed": int,
  "character": { "class_id", "level", "xp", "stats", "masteries", "recipes",
                  "position", "equipment": { "<slot>": <ItemInstance dict> },
                  "inventory": [ <ItemInstance dict> ] },
  "stash": [ <ItemInstance dict> ], "camp": {...}, "clock": {...} }
```

## Annexe B — Ordre de bataille résumé

| # | Phase | Jalon |
|---|---|---|
| 0 | Setup projet | — |
| 1 | Grille iso + pathfinding + LOS | — |
| 2 | Entités + machine à états de combat | — |
| 3 | Pattern Command + déplacement + tacle | — |
| 4 | Sorts data-driven + poussées + 8 sorts Pyromant | — |
| 5 | Surfaces + combinaisons + météo (2) | — |
| 6 | Objets de terrain interactifs + hauteur | — |
| 7 | IA utility exploitant le décor | **M1 — proto tactique** |
| 8 | Items + affixes + inventaire + loot | — |
| 9 | Matrice classe×arme + Sylve + mécaniques | — |
| 10 | Hardcore + camp + craft + sauvegarde + XP | **M2 — boucle hardcore** |
| 11 | Monde continu procédural + météo monde | — |
| 12 | Donjons procéduraux + boss | — |
| 13 | Légendaires/sets + 5 classes + 4 biomes | **M3 — vertical slice** |
| 14 | Coop en ligne 4 joueurs | **M4 — coop** |

**Règles d'or pour l'IA de codage** : 1) une phase à la fois, dans l'ordre ; 2) tout contenu de gameplay en `Resource`/données, jamais en dur ; 3) toute règle de jeu dans une classe pure testée ; 4) les critères d'acceptation d'une phase doivent tous être verts (tests GUT inclus) avant d'entamer la suivante.
