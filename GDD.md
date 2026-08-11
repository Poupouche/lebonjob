# Game Design Document — Projet **AEGIS** (nom de code)

> **Version** : 1.0 (décisions actées — voir §14)
> **Plateforme** : PC (Windows / Linux / macOS)
> **Moteur** : Godot 4.x (GDScript + C#)
> **Genre** : RPG tactique tour par tour, solo / coop 4 joueurs (PvM), hardcore full-loot, loot à affixes procéduraux, monde continu procédural, environnements de combat interactifs
> **Direction artistique** : pixel art — médiéval-fantasy avec twist steampunk et mythologies scandinave & asiatique
> **Document compagnon** : [`PLAN_IMPLEMENTATION.md`](PLAN_IMPLEMENTATION.md) — étapes de codage précises, à donner à une IA de développement.

---

## 1. Vision & concept

### 1.1 Pitch
**AEGIS** est un RPG tactique tour par tour sur grille isométrique (inspiré de **Dofus/Wakfu**), jouable en **solo ou en coop jusqu'à 4 joueurs**, dans un **monde continu généré procéduralement**. Le jeu est **hardcore dès la première minute** : à la mort, tout l'équipement porté et l'inventaire transporté sont **perdus à jamais**. Le loot est généré procéduralement avec des **affixes** aléatoires (inspiré de **Diablo**), et chaque combat exploite un **environnement interactif** : surfaces élémentaires, météo, objets déplaçables/destructibles, pièges (inspiré de **D&D** / **Divinity: Original Sin 2**).

### 1.2 Piliers de design
1. **Le combat est un puzzle** — chaque tour offre des choix tactiques riches : placement, gestion Mana/Énergie, combos élémentaires, exploitation du décor et de la météo.
2. **Le risque donne de la valeur** — hardcore full-loot permanent : la peur de perdre son stuff crée la tension ; le craft garantit toujours un équipement « potable » pour repartir.
3. **Le loot raconte une histoire** — chaque objet est unique grâce aux affixes procéduraux ; le **drop domine** et la chasse aux affixes parfaits est le moteur de l'endgame.
4. **La classe ET l'arme définissent le gameplay** — kit de sorts fixe par classe + sorts dépendants de l'arme équipée, différents selon la classe (matrice classe × arme, inspirée de Wakfu et d'Albion).

### 1.3 Fantasme joueur
« Je pars en expédition avec mes 3 amis, je risque tout ce que je porte, je gagne des combats tactiques en retournant le décor contre les monstres, et je reviens (peut-être) avec un objet légendaire aux affixes parfaits. Si je meurs, je perds tout — mais je re-crafte un kit correct et j'y retourne. »

### 1.4 Public cible
- Joueurs de RPG tactiques (Dofus, Wakfu, Divinity, Baldur's Gate 3).
- Joueurs de roguelikes/hardcore qui aiment le risque permanent.
- Joueurs de hack'n'slash amateurs de theorycraft d'objets (Diablo, Path of Exile).

### 1.5 Modèle & périmètre
- **Free-to-play** (monétisation à définir plus tard, hors périmètre du premier jouable).
- **Pas de MMO ni de factions** : le mode faction/territoires à la Albion est abandonné (trop ambitieux). Le jeu est **PvM** (joueurs contre monstres).
- **Premier livrable : prototype local solo** (voir `PLAN_IMPLEMENTATION.md`), la coop en ligne à 4 vient ensuite.

---

## 2. Boucle de gameplay

### 2.1 Boucle courte (minute)
Explorer le monde continu → engager un combat tour par tour → exploiter le terrain et la météo → looter → gérer inventaire/poids.

### 2.2 Boucle moyenne (session)
Préparer un « loadout » au camp de base (équipement qu'on accepte de perdre) → partir en expédition (solo ou coop 4) → rapporter butin et ressources au camp → crafter / stocker / améliorer.

### 2.3 Boucle longue (semaines)
Progression du personnage (niveaux 1→100 en ~1 mois, sorts, maîtrises) → progression du camp de base (ateliers de craft, coffre) → chasse aux affixes parfaits et aux légendaires (endgame) → biomes et donjons procéduraux de difficulté croissante.

---

## 3. Système de combat tour par tour

### 3.1 Structure
- **Grille isométrique en losange** (type Dofus). En exploration le monde est continu ; quand un combat démarre, la zone locale devient une **arène tactique en grille** générée à partir du décor réel (obstacles, objets interactifs, surfaces, météo du moment).
- **Phase de placement** : avant le tour 1, chaque joueur choisit sa case de départ parmi des cases de placement.
- **Tour par tour strictement séquentiel par personnage** : ordre déterminé par l'*Initiative*.
- **Timer de tour long : 2 minutes par tour de chaque joueur** (le jeu est réfléchi, pas nerveux). Les monstres jouent sans timer perceptible (IA).
- **Taille des combats : jusqu'à 4 joueurs contre jusqu'à 10 unités ennemies.**
- Combats **verrouillés** (pas de « join in progress ») pour le premier périmètre.

### 3.2 Ressources du personnage
| Ressource | Rôle | Base |
|---|---|---|
| **Vie (PV)** | Points de vie ; à 0 → mort (hardcore, voir §6) | selon classe/niveau |
| **Mana** | Lancer des sorts et capacités | 6 / tour |
| **Énergie** | Se déplacer (1 case = 1 Énergie) et **interagir avec le décor** (pousser un tonneau, actionner un levier, escalader) | 3 / tour |

- Mana/Énergie se **régénèrent intégralement en début de tour** ; buffables/débuffables (retrait avec jet de résistance, à la Dofus).
- Les interactions décor coûtent de l'Énergie (ex. pousser un tonneau : 2 Énergie ; actionner un levier : 1 Énergie) — le joueur arbitre en permanence entre **bouger** et **manipuler le terrain**.

### 3.3 Sorts
- Sorts définis par : coût en Mana (parfois en Énergie), portée (min/max, modifiable, ligne de vue oui/non), zone d'effet (croix, ligne, cercle, cône), relance (cooldown), limite par tour/cible, effets.
- **Écoles élémentaires** : Feu, Eau, Air, Terre + Lumière/Ombre (soutien/entrave). Les éléments interagissent entre eux, avec les **surfaces** et avec la **météo** (voir §5) — les combos inter-sorts et inter-joueurs sont un pilier (référence Wakfu).
- Certains sorts **manipulent le décor à distance** : téléportation d'objets (caisses, tonneaux), attirance/poussée, création de surfaces.

### 3.4 Statistiques principales
- **Vitalité** (PV), **Force** (dégâts Terre + poids portable), **Intelligence** (dégâts Feu + soins), **Chance** (dégâts Eau + prospection/loot), **Agilité** (dégâts Air + tacle/fuite + initiative), **Sagesse** (XP + résistance aux retraits Mana/Énergie).
- Résistances élémentaires (%) et fixes, dégâts critiques, tacle/fuite, portée, invocations.

### 3.5 Tacle, ligne de vue, prévisualisation
- **Tacle/Fuite** : quitter une case adjacente à un ennemi coûte Mana/Énergie selon un rapport tacle-fuite.
- **Ligne de vue** bloquée par obstacles hauts, certaines invocations, fumées.
- **Prévisualisation systématique** : dégâts estimés, cases atteignables, trajectoires de poussée, propagation de surfaces — le joueur ne doit jamais être puni par un manque d'information.

---

## 4. Classes, armes & progression

### 4.1 Système hybride « classe × arme »
Le cœur de l'identité du jeu (décision Q1.2/Q4.1) :
- Chaque classe possède un **kit fixe de sorts de classe** (~12 sorts, débloqués par niveau), construits autour d'une **mécanique de classe forte** (référence Wakfu : chaque classe tourne autour de sa mécanique).
- **Toutes les classes peuvent équiper toutes les armes**, mais chaque **archétype d'arme** (épée, arc, bâton, dague, marteau, focus…) accorde **des sorts d'arme différents selon la classe**.
  - Exemple : avec une **épée**, le **Pyromant** obtient « Épée magique » (frappe enflammée créant une surface de feu), tandis que le **Sylve** obtient « Estoc » (perce-armure à ronces).
- La variété totale de sorts = (nb de classes) × (nb d'archétypes d'armes) + kits fixes → la profondeur de build vient du **choix d'arme** autant que de la classe, ce qui se marie avec le full-loot (perdre son arme change réellement le gameplay).

### 4.2 Classes de lancement (5, extensibles)
| Classe | Fantasme | Rôle principal | Mécanique de classe |
|---|---|---|---|
| **Bastion** | Chevalier tacticien | Tank, contrôle de zone | **Rempart** : génère et consume de l'armure en poussant/attirant |
| **Pyromant** | Mage des flammes | Dégâts de zone, ignition du décor | **Surchauffe** : jauge qui monte à chaque sort de feu, débloque des versions améliorées mais risque l'auto-brûlure |
| **Sylve** | Archer druidique | Dégâts à distance, zoning | **Ronces** : plante des ronces sur le terrain qui alimentent ses sorts |
| **Ombrelame** | Assassin | Burst mono-cible, mobilité | **Ombres** : pose des marques/clones téléporteurs |
| **Oracle** | Soutien | Soins, buffs Mana/Énergie | **Prescience** : manipule l'ordre d'initiative et « voit » un tour à l'avance |

### 4.3 Archétypes d'armes de lancement (6)
Épée (mêlée polyvalente), Dague (mêlée burst), Marteau (mêlée zone/poussée), Arc (distance physique), Bâton (distance élémentaire), Focus (soutien/utilitaire). Chaque archétype donne **2 sorts d'arme** par classe → 5 classes × 6 armes × 2 sorts = 60 sorts d'arme + 5 × 12 sorts de classe = **120 sorts** à terme (le prototype en implémente un sous-ensemble, voir plan).

### 4.4 Progression
- **Niveaux 1 → 100**, endgame atteignable en **~1 mois** de jeu régulier.
- Points de caractéristiques à répartir + **variantes de sorts** (chaque sort de classe a 2 variantes exclusives, à la Wakfu).
- **Maîtrises à l'usage** : maîtrises d'archétype d'arme et d'artisanat progressant en les utilisant (à l'Albion).
- La progression du personnage (niveaux, maîtrises, recettes) **survit à la mort** — seul l'équipement/inventaire est perdu (voir §6).

---

## 5. Environnement interactif en combat

### 5.1 Surfaces élémentaires
| Surface | Création | Effets | Interactions |
|---|---|---|---|
| **Feu** | sorts de feu, braseros renversés | dégâts/tour, brûlure | + Eau → Vapeur ; + Huile → explosion |
| **Eau** | sorts d'eau, tonneaux, pluie | mouillé (vulnérable foudre, éteint le feu) | + Froid → Glace ; + Feu → Vapeur |
| **Glace** | eau gelée | glissade (trajectoire forcée), -fuite | + Feu → Eau |
| **Huile** | tonneaux, sols de donjon | -Énergie, inflammable | + Feu → nappe de feu |
| **Poison** | sorts, créatures | dégâts/tour, empoisonné | + Feu → nuage toxique explosif |
| **Vapeur/Fumée** | combinaisons | bloque la ligne de vue | dissipée par le vent (sorts d'air) |

### 5.2 Météo dynamique (décision Q3.1)
La météo du monde s'applique aux combats qui s'y déclenchent :
| Météo | Effet en combat |
|---|---|
| **Pluie** | crée/étend des surfaces d'eau, affaiblit le feu, renforce la foudre |
| **Neige/Gel** | l'eau gèle en fin de tour, -fuite global |
| **Canicule** | surfaces d'eau s'évaporent, feu se propage plus loin |
| **Brouillard** | portée de ligne de vue réduite |
| **Tempête** | vent : les projectiles/poussées dévient d'une case, fumées dissipées |

### 5.3 Objets de terrain
- **Poussable/attirable/téléportable** : rochers, caisses, tonneaux (dégâts de collision contre mur/entité ; certains sorts de classe téléportent des objets).
- **Destructible** : murets, piliers, portes — détruire ouvre des lignes de vue, effondrer inflige des dégâts de zone.
- **Déclencheurs** : leviers, plaques de pression, pièges (désamorçables), braseros, lustres à faire tomber.
- **Hauteur (légère)** : cases surélevées = +1 portée et +10 % de dégâts vers le bas ; escalade coûte +1 Énergie. (2D isométrique : hauteur limitée à 1–2 niveaux pour rester lisible.)

### 5.4 Règles de conception des arènes
- Chaque arène (générée procéduralement à partir du décor local) doit contenir **au moins 3 éléments interactifs exploitables par les deux camps**.
- **Les monstres exploitent l'environnement dès le MVP** (décision Q3.3) : l'IA évalue surfaces, poussées et objets dans son arbre de décision, selon les actions dont chaque monstre dispose.

---

## 6. Hardcore & mort (full-loot permanent)

### 6.1 Règle centrale (décision Q6.1/Q6.3)
**Le hardcore s'applique partout, dès le début du jeu, sans filet de sécurité** :
- À la mort (PV à 0 en combat sans résurrection alliée avant la fin du combat), **tout l'équipement porté et tout l'inventaire transporté sont perdus à jamais** (détruits).
- Pas de première mort pardonnée, pas d'assurance, pas de zone « sûre » qui annule la règle.
- Le **personnage survit** : il conserve niveaux, caractéristiques, maîtrises, recettes et son camp de base. Il se réveille au camp, nu.

### 6.2 En coop
- Un allié à 0 PV est **agonisant** pendant 2 tours : il peut être ranimé par un sort/consommable. Si le combat se termine (victoire) avec un agonisant, il survit à 1 PV. S'il n'est pas ranimé après 2 tours ou si l'équipe est éliminée → mort réelle, full-loot.

### 6.3 Le craft comme filet (décision Q7.3)
- Le **camp de base** contient un **coffre** (ce qui y est stocké est en sécurité) et des **ateliers de craft**.
- Le craft garantit de toujours pouvoir refabriquer un **équipement « potable »** (bases communes/magiques) à partir de ressources récoltables en zone facile → on ne reste jamais bloqué, mais les meilleurs objets viennent du **drop** en zones dangereuses.

### 6.4 Conséquences de design
- L'économie d'objets est une **pompe** : tout finit par être détruit → le loot garde de la valeur indéfiniment.
- L'UI doit rendre le risque lisible en permanence : valeur estimée de ce qu'on porte, distance au camp, difficulté de la zone.

---

## 7. Équipement & affixes procéduraux (type Diablo)

### 7.1 Slots d'équipement
Arme, second main (bouclier/focus), casque, plastron, bottes, ceinture, 2 anneaux, amulette, cape, trophée (11 slots).

### 7.2 Raretés (6, décision Q7.1)
| Rareté | Couleur | Affixes |
|---|---|---|
| Commun | blanc | 0 |
| Magique | bleu | 1 préfixe et/ou 1 suffixe |
| Rare | jaune | 2–4 affixes |
| Épique | violet | 4–5 affixes |
| **Légendaire** | orange | 3–4 affixes + **1 pouvoir unique** (change une règle : « vos poussées créent de la glace », « +1 Mana si vous commencez le tour sur une surface de feu »...) — **très rare**, source (drop seul ou aussi craft) à trancher (Q-ouverte §14.2) |
| **Set** | vert | affixes + bonus de panoplie (à la Dofus) |

### 7.3 Génération procédurale
- Chaque **base d'objet** (ex. « Épée longue T4 ») a un budget d'affixes et des **pools** de préfixes/suffixes pondérés par : niveau de zone, tier de l'objet, tags (arme/armure/bijou).
- **Affixe** = { stat, plage de valeurs par tier (T1–T8), poids de tirage, tags d'exclusion }.
  - Préfixes (offensifs) : +dégâts élémentaires, +critiques, +Mana (très rare), +portée...
  - Suffixes (défensifs/utilitaires) : +vitalité, +résistances, +tacle/fuite, -coût Énergie des interactions décor, +prospection...
- **Affixes environnementaux** (signature du jeu) : famille dédiée aux interactions de terrain (« immunisé aux surfaces de glace », « +2 cases de poussée », « vos sorts d'eau gèlent sous la pluie »...).
- Score de puissance visible + comparaison automatique dans l'UI.

### 7.4 Sources d'objets (décision Q7.4)
- **Le drop domine** : monstres, coffres, boss de donjons procéduraux. La qualité/tier des drops suit la difficulté de la zone.
- **Craft = filet de sécurité + bases** : recettes → objets communs/magiques fiables ; **Réforge** (re-tirer un affixe, coût croissant) pour le fine-tuning des drops.

---

## 8. Monde, exploration & PvE

### 8.1 Monde continu procédural (décisions Q8.1/Q8.2)
- **Monde continu** (type Wakfu — pas d'écrans séparés) : régions traversées sans rupture, chargement en streaming par chunks.
- **Génération procédurale** du monde et des donjons : assemblage de « briques » conçues à la main (patterns de biomes, salles, points d'intérêt) par graine (seed), pour un monde rejouable et surprenant.
- **Biomes** : plaines, forêt, marais, montagne (twist steampunk : ruines mécaniques), terres gelées (mythologie scandinave), vallées brumeuses (mythologie asiatique).
- **Difficulté par distance** : plus on s'éloigne du camp de base, plus les monstres, les tiers de loot — et le risque hardcore — augmentent.

### 8.2 Donjons
- Instances procédurales de 4–6 salles (combats + énigmes environnementales + salle au trésor) + **boss à mécaniques uniques** exploitant le décor.

### 8.3 Narration (décision Q8.3)
- **Minimale** : pas de trame scénarisée lourde. Le lore (steampunk + mythes nordiques/asiatiques) passe par l'environnement, les objets, les descriptions et les boss.

### 8.4 Récolte & ressources
- Nœuds de récolte (bois, minerai, plantes, essences) zonés par difficulté ; alimentent le craft du camp de base.

---

## 9. Solo & coop 4 joueurs

- **Solo complet** : tout le contenu est jouable seul (équilibrage dynamique : nombre/PV des monstres selon la taille du groupe).
- **Coop jusqu'à 4** : un hôte, jusqu'à 3 invités ; monde de l'hôte ; le loot est instancié par joueur (pas de vol de loot entre alliés).
- **Ordre de développement** : 1) prototype **local solo** (décision Q10.3) → 2) coop en ligne (serveur = hôte autoritaire, l'architecture combat en pattern Command est conçue dès le départ pour être sérialisable réseau).

---

## 10. Interface & expérience utilisateur

- **HUD combat** : barre de sorts (kit de classe + sorts d'arme, visuellement distingués), Vie/Mana/Énergie, timeline d'initiative, timer de tour (2 min), prévisualisation dégâts/poussées/surfaces au survol.
- **Inventaire** : grille avec poids, comparateur d'objets, filtre par affixes, **indicateur de valeur à risque** (ce que je perds si je meurs).
- **Carte du monde** : brouillard de découverte, difficulté des régions, météo, position du camp.
- Accessibilité : mode daltonien pour les surfaces (motifs en plus des couleurs), vitesse d'animation des tours réglable.

---

## 11. Direction artistique & audio

- **Pixel art** isométrique 2D (décisions Q3.2/Q9.1) : lisibilité tactique avant tout — silhouettes claires, surfaces identifiables par texture ET motif.
- **Univers** : médiéval-fantasy avec **twist steampunk** (machines, vapeur, engrenages) et influences **mythologie scandinave** (jotuns, runes, biome gelé) et **asiatique** (esprits, sanctuaires, biome brumeux) (décision Q9.2).
- **Audio** : ambiances par biome et météo, « stingers » d'initiative, sons distinctifs par type de surface.

---

## 12. Cadrage technique Godot

- **Godot 4.x**, **GDScript + C#** (décision Q10.1) : GDScript pour le gameplay/UI (itération rapide), C# pour les systèmes chauds (génération procédurale du monde, résolution de combat, pathfinding) si profilage le justifie.
- **Grille & pathfinding** : `TileMapLayer` isométrique (losange) + `AStarGrid2D` ; ligne de vue par Bresenham sur grille.
- **Combat** : machine à états (exploration → placement → boucle de tours → résolution) ; actions = **pattern Command** (prévisualisables, annulables, sérialisables pour le futur réseau et les replays).
- **Surfaces & météo** : couches de grille dédiées, règles de propagation/combinaison **data-driven** (`Resource` Godot / JSON).
- **Objets & affixes** : définitions data-driven (bases, pools, courbes par tier) → équilibrage sans code.
- **Monde procédural** : génération par chunks avec seed ; briques assemblées (rooms/patterns) + bruit (biomes).
- **Sauvegarde** : fichier local chiffré/checksummé (hardcore : limiter la triche naïve du save-scumming — sauvegarde à la volée, une seule slot par personnage).

Le détail complet (architecture, schémas de données, ordre des tâches, critères d'acceptation) est dans **`PLAN_IMPLEMENTATION.md`**.

---

## 13. Périmètre & jalons

| Jalon | Contenu | Référence plan |
|---|---|---|
| **M1 — Prototype tactique local** | 1 classe (Pyromant), 8 sorts, grille iso, Vie/Mana/Énergie, 3 surfaces, poussées, 1 arène fixe, IA basique, PvE solo | Phases 0–7 |
| **M2 — Boucle hardcore** | loot à affixes (4 raretés), inventaire/équipement, mort full-loot, camp + coffre + craft basique | Phases 8–10 |
| **M3 — Vertical slice** | 2e classe, sorts d'arme (matrice classe×arme), monde procédural 2 biomes, météo, 1 donjon, boss | Phases 11–13 |
| **M4 — Coop** | coop en ligne 4 joueurs, équilibrage groupe, loot instancié | Phase 14 |

---

## 14. Journal des décisions & questions ouvertes

### 14.1 Décisions actées (v1.0 — réponses aux questions de la v0.1)
| Sujet | Décision |
|---|---|
| Ambition réseau | **Solo/coop 4 en PvM** ; factions & MMO abandonnés |
| Modèle éco | Free-to-play (détail hors périmètre) |
| Références | + **Wakfu** : variété de sorts, interactions inter-sorts, mécaniques de classe centrales |
| Grille | **Losange/isométrique** (Dofus) |
| Ressources | **Vie / Mana (sorts) / Énergie (déplacement + interactions décor)** |
| Timer | Tours longs : **2 min/joueur** |
| Séquencement | Strictement **séquentiel par personnage** |
| Taille combat | **4 joueurs vs jusqu'à 10 ennemis** |
| Simulation | Surfaces + poussées + destructibles + **météo** + **téléportation d'objets** (sorts de classe) |
| Rendu | **2D isométrique pixel art** |
| IA | Les monstres **exploitent le décor dès le début**, selon leurs capacités |
| Classes | **Hybride** : kit fixe par classe + sorts d'arme variant par classe ; 5 archétypes validés |
| Progression | Niveau max 100, ~**1 mois** pour le max |
| Factions | **Supprimées** |
| Hardcore | **Partout, dès le début** : mort = perte définitive du loot porté ; pas de filet |
| Raretés | **6** (schéma Diablo complet) |
| Craft vs drop | **Drop domine** ; craft = filet « équipement potable » |
| Monde | **Continu type Wakfu**, **génération procédurale** |
| Narration | **Minimale** |
| DA | Pixel art ; médiéval-fantasy + steampunk + mythologies scandinave/asiatique |
| Langages | GDScript **+ C#** |
| Premier livrable | **Prototype local solo** |

### 14.2 Questions encore ouvertes (à trancher en cours de production)
1. **Légendaires** : uniquement drop, ou aussi craftables/extractibles ? (décidé : « très rares » ; source exacte à trancher — recommandation : drop only au début, extraction de pouvoir ajoutée en M3+).
2. **Permadeath total** : la v1.0 acte « mort = perte du loot, le personnage survit ». Un mode optionnel « permadeath du personnage » (ladder) pourra être ajouté plus tard.
3. **Trash rate** : sans PvP, les objets du mort sont simplement **détruits** (100 %) — à revalider si un mode récupération de corps est souhaité.
4. **Monétisation F2P** : cosmétiques ? à définir bien après le prototype.

---

*Document vivant — version 1.0. Toute nouvelle décision met à jour la section concernée et le journal §14.*
