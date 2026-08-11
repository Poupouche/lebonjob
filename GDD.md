# Game Design Document — Projet « Sans Titre » (nom de code : **AEGIS**)

> **Version** : 0.1 (premier jet — à affiner avec les réponses aux questions du chapitre 15)
> **Plateforme** : PC (Windows / Linux / macOS)
> **Moteur** : Godot 4.x
> **Genre** : RPG tactique tour par tour, monde persistant, factions, hardcore/full-loot, loot à affixes procéduraux, environnements de combat interactifs

---

## 1. Vision & concept

### 1.1 Pitch
**AEGIS** est un RPG tour par tour sur grille (inspiré de **Dofus**) dans un monde ouvert et dangereux où trois factions se disputent territoires et ressources (inspiré d'**Albion Online**). La mort y a un vrai poids : dans les zones à haut risque, le joueur perd son équipement (**full loot**) et peut activer un mode **hardcore** (mort permanente). Le loot est généré procéduralement avec des **affixes** aléatoires (inspiré de **Diablo**), et chaque combat se joue dans un **environnement interactif** : surfaces élémentaires, objets déplaçables/destructibles, hauteur, pièges (inspiré de **D&D** / **Divinity: Original Sin 2**).

### 1.2 Piliers de design
1. **Le combat est un puzzle** — chaque tour offre des choix tactiques riches : placement, PA/PM, combos élémentaires, exploitation du décor.
2. **Le risque donne de la valeur** — plus une zone est dangereuse, meilleures sont les récompenses ; la peur de perdre son stuff crée la tension et l'économie.
3. **Le loot raconte une histoire** — chaque objet est unique grâce aux affixes procéduraux ; trouver, crafter et perdre des objets alimente une économie vivante.
4. **Le monde appartient aux factions** — le territoire, les ressources et la politique sont façonnés par les joueurs, pas par des scripts.

### 1.3 Fantasme joueur
« Je pars en expédition en zone rouge avec mon groupe, je risque tout ce que je porte, je gagne des combats tactiques en retournant le décor contre mes ennemis, et je reviens (peut-être) avec un objet légendaire aux affixes parfaits. »

### 1.4 Public cible
- Joueurs de RPG tactiques (Dofus, Wakfu, Divinity, Baldur's Gate 3).
- Joueurs de MMO sandbox à économie joueur (Albion, EVE).
- Joueurs de hack'n'slash amateurs de theorycraft d'objets (Diablo, Path of Exile).

---

## 2. Boucle de gameplay

### 2.1 Boucle courte (minute)
Explorer → engager un combat tour par tour → exploiter le terrain → looter → gérer inventaire/poids.

### 2.2 Boucle moyenne (session)
Préparer un « loadout » (équipement qu'on accepte de perdre) → partir en expédition (PvE/PvM ou PvP) → rapporter butin et ressources en zone sûre → crafter / améliorer / vendre.

### 2.3 Boucle longue (semaines)
Progression du personnage (niveaux, sorts, maîtrises) → progression de la faction (territoires, guerres, avant-postes) → progression économique (marchés, artisanat de haut niveau) → chasse aux affixes parfaits (endgame).

---

## 3. Système de combat tour par tour (type Dofus)

### 3.1 Structure
- **Combat instancié sur grille** : quand un combat démarre, la zone de la carte devient une **arène tactique en grille** (cases carrées par défaut — voir question Q3.1 pour l'option isométrique/hexagonale).
- **Phase de placement** : avant le tour 1, chaque camp choisit ses cases de départ parmi des cases de placement.
- **Tour par tour séquentiel** : l'ordre d'initiative est déterminé par la statistique *Initiative* ; timer de tour (30 s par défaut, configurable).
- **Taille des combats** : 1v1 jusqu'à 5v5 (groupes) ; monstres en groupes de 1 à 8.

### 3.2 Ressources d'action
| Ressource | Rôle | Base |
|---|---|---|
| **PA** (Points d'Action) | Lancer des sorts / utiliser des objets | 6 |
| **PM** (Points de Mouvement) | Se déplacer d'une case | 3 |
| **PW** (Points de Volonté) | Ressource rare pour capacités ultimes / interactions majeures avec le décor | 1 (regagné tous les 2 tours) |

- PA/PM sont **buffables/débuffables** (retrait PA/PM avec jets d'esquive, comme Dofus).
- Les interactions avec l'environnement coûtent des PA (pousser un rocher : 3 PA) ou des PW (effondrer un pilier : 1 PW).

### 3.3 Sorts et écoles
- Chaque classe possède ~20 sorts, débloqués par niveau.
- Sorts définis par : coût PA, portée (min/max, modifiable, ligne de vue oui/non), zone d'effet (croix, ligne, cercle...), relance, effets.
- **Écoles élémentaires** : Feu, Eau, Air, Terre + Lumière/Ombre (soutien/entrave). Les éléments interagissent avec les **surfaces** (voir §5).

### 3.4 Statistiques principales
- **Vitalité** (PV), **Force** (dégâts Terre + poids portable), **Intelligence** (dégâts Feu + soins), **Chance** (dégâts Eau + prospection/loot), **Agilité** (dégâts Air + tacle/fuite + initiative), **Sagesse** (XP + résistance retrait PA/PM).
- Résistances élémentaires (%) et fixes, dommages critiques, tacle/fuite, portée, invocations.

### 3.5 Tacle, ligne de vue, prévisualisation
- **Tacle/Fuite** : quitter une case adjacente à un ennemi coûte PA/PM selon un rapport tacle-fuite.
- **Ligne de vue** bloquée par obstacles hauts (et par certaines invocations/objets du décor).
- **Prévisualisation systématique** : dégâts estimés, cases atteignables, trajectoires de poussée, propagation de surfaces — le joueur ne doit jamais être puni par un manque d'information.

---

## 4. Classes & progression

### 4.1 Classes de lancement (5, extensibles)
| Classe | Fantasme | Rôle principal |
|---|---|---|
| **Bastion** | Chevalier tacticien | Tank, contrôle de zone, attirance/poussée |
| **Pyromant** | Mage des flammes et surfaces | Dégâts de zone, ignition du décor |
| **Sylve** | Archer druidique | Dégâts à distance, pièges, ronces (créées sur le terrain) |
| **Ombrelame** | Assassin | Burst mono-cible, mobilité, invisibilité |
| **Oracle** | Soutien | Soins, buffs PA/PM, manipulation de l'initiative |

### 4.2 Progression
- **Niveaux 1 → 100** ; points de caractéristiques à répartir + variantes de sorts (chaque sort a 2 variantes exclusives, à la Dofus 2.0/Wakfu).
- **Maîtrises horizontales** : arbres secondaires (artisanat, survie, faction) progressant à l'usage (à l'Albion) — pas de niveau requis pour s'équiper, mais des **paliers de maîtrise d'arme/armure**.
- En **hardcore**, la progression du *compte* (recettes connues, maîtrises partielles, réputation) survit partiellement à la mort du personnage (voir §7.4).

---

## 5. Environnement interactif en combat (type D&D / DOS2)

### 5.1 Surfaces élémentaires
| Surface | Création | Effets | Interactions |
|---|---|---|---|
| **Feu** | sorts de feu, braseros renversés | dégâts/tour, brûlure | + Eau → Vapeur ; + Huile → explosion |
| **Eau** | sorts d'eau, tonneaux, pluie | mouillé (vulnérable foudre, éteint le feu) | + Froid → Glace ; + Feu → Vapeur |
| **Glace** | eau gelée | glissade (trajectoire forcée), -fuite | + Feu → Eau |
| **Huile** | tonneaux, sols de donjon | -PM, inflammable | + Feu → nappe de feu |
| **Poison** | sorts, créatures | dégâts/tour, empoisonné | + Feu → nuage toxique explosif |
| **Vapeur/Fumée** | combinaisons | bloque la ligne de vue | dissipée par le vent (sorts d'air) |

### 5.2 Objets de terrain
- **Poussable/attirable** : rochers, caisses, tonneaux (peuvent écraser : dégâts de collision, comme les poussées Dofus contre un mur).
- **Destructible** : murets, piliers, portes — détruire ouvre des lignes de vue, effondrer inflige des dégâts de zone.
- **Déclencheurs** : leviers, plaques de pression, pièges (désamorçables), braseros, lustres à faire tomber.
- **Hauteur** : cases surélevées = +portée et +10 % de dégâts vers le bas ; escalade coûte des PM supplémentaires.

### 5.3 Règles de conception des arènes
- Chaque arène doit contenir **au moins 3 éléments interactifs exploitables par les deux camps**.
- Les monstres **utilisent aussi l'environnement** (IA : évaluation des surfaces et poussées dans l'arbre de décision).
- Les cartes du monde ouvert génèrent leurs arènes à partir du décor réel à l'endroit du combat (biome → set d'obstacles et de surfaces).

---

## 6. Factions & territoire (type Albion)

### 6.1 Les trois factions
Trois factions jouables et irréconciliables (noms de travail) :
- **Le Concordat** — ordre, loi, cité-forteresse.
- **La Marée** — marchands, pirates, économie libre.
- **Les Racines** — druides, tribus, symbiose avec le monde sauvage.

Le choix de faction se fait vers le niveau 10 ; en changer est coûteux (perte de réputation, quarantaine).

### 6.2 Territoires
- La carte du monde est découpée en **régions revendicables** contenant des ressources rares, des donjons et des **avant-postes**.
- **Capture** : événements de siège planifiés (fenêtres horaires) où les combats se résolvent en **batailles tour par tour en escouades** (série de combats 5v5 sur des points de contrôle).
- Une région contrôlée donne à sa faction : bonus de récolte, accès à des artisans exclusifs, taxes de marché.

### 6.3 Réputation & guerre
- Réputation individuelle par faction : gagnée en missions, escortes, PvP de faction ; perdue en tuant des membres de sa propre faction (statut **hors-la-loi**).
- Guerres de faction : objectifs saisonniers (saisons de ~3 mois) avec classements et récompenses cosmétiques/titres.

---

## 7. Zones de danger, full loot & hardcore

### 7.1 Zonage (règles à la Albion)
| Zone | PvP | Perte à la mort | Récompenses |
|---|---|---|---|
| **Bleue** (sûre) | impossible | aucune (réparation d'équipement) | faibles |
| **Jaune** | duel/consenti | durabilité + partie des ressources transportées | moyennes |
| **Rouge** | ouvert entre factions | **full loot** (tout l'équipement + inventaire lootables) | élevées |
| **Noire** | ouvert à tous (même faction) | full loot + pas de karma | maximales, ressources endgame |

### 7.2 Full loot & économie de la casse
- À la mort en zone rouge/noire, l'équipement tombe au sol : ~30 % des objets sont **détruits** (« trash rate »), le reste est lootable → **pompe économique** qui entretient la demande d'artisanat.
- Un objet équipé est « lié au risque », jamais lié au compte : **tout se vend, tout se perd**.

### 7.3 Sanctuaires & logistique
- Banques uniquement en zones bleues/jaunes ; transporter des marchandises entre zones = gameplay de convoi (risque/récompense).
- **Poids d'inventaire** : influence les PM hors combat et la fuite.

### 7.4 Mode Hardcore (opt-in)
- À la création : personnage **Normal** ou **Hardcore**.
- Hardcore : la mort (en toute zone hors bleue) est **permanente**. Le personnage devient un « Écho » consultable (mémorial, tableau des morts).
- **Héritage** : 10 % des maîtrises, les recettes apprises et 25 % de la réputation de faction sont transmises au personnage suivant du compte.
- Serveurs/ladders hardcore saisonniers avec classement « distance parcourue avant la mort ».
- Récompenses exclusivement **cosmétiques** (pas d'avantage de puissance) pour éviter de forcer la main aux joueurs normaux.

---

## 8. Équipement & affixes procéduraux (type Diablo)

### 8.1 Slots d'équipement
Arme, second main (bouclier/focus), casque, plastron, bottes, ceinture, 2 anneaux, amulette, cape, trophée (10–11 slots).

### 8.2 Raretés
| Rareté | Couleur | Affixes |
|---|---|---|
| Commun | blanc | 0 |
| Magique | bleu | 1 préfixe et/ou 1 suffixe |
| Rare | jaune | 2–4 affixes |
| Épique | violet | 4–5 affixes |
| **Légendaire** | orange | 3–4 affixes + **1 pouvoir unique** (change une règle : « vos poussées créent de la glace », « +1 PA si vous commencez le tour sur une surface de feu »...) |
| **Set** | vert | affixes + bonus de panoplie (à la Dofus) |

### 8.3 Génération procédurale
- Chaque **base d'objet** (ex. « Épée longue T4 ») a un budget d'affixes et des **pools** de préfixes/suffixes pondérés par : niveau de zone, tier de l'objet, tags (arme/armure/bijou).
- **Affixes** = { stat, plage de valeurs par tier (T1–T8), poids de tirage, tags d'exclusion }.
  - Préfixes (offensifs) : +dégâts élémentaires, +dommages critiques, +PA (très rare), +portée, « les dégâts de feu enflamment les surfaces d'huile à coût réduit »...
  - Suffixes (défensifs/utilitaires) : +vitalité, +résistances, +tacle/fuite, +vitesse de récolte, -coût PW des interactions décor, +prospection...
- **Affixes environnementaux** (signature du jeu) : une famille d'affixes dédiée aux interactions de terrain (ex. « immunisé aux surfaces de glace », « +2 cases de poussée »).
- Score d'objet visible (« puissance d'objet ») + comparaison automatique dans l'UI.

### 8.4 Artisanat & fine-tuning
- **Craft** : ressources récoltées (zonées par danger) + recette → objet avec affixes tirés aléatoirement (le crafteur choisit la base et le tier, pas les affixes).
- **Réforge** : re-tirer un affixe (coût croissant) ; **Extraction** : détruire un légendaire pour capturer son pouvoir unique et l'imprimer sur un autre objet (1 fois).
- **Signature du crafteur** sur l'objet (réputation d'artisan, à l'Albion).

### 8.5 Économie
- **Marchés régionaux** (pas de marché global) : les prix varient par région → gameplay de transport et de spéculation.
- Taxes de marché reversées à la faction contrôlant la région.
- Or comme monnaie ; **gemmes premium uniquement cosmétiques** (à confirmer, Q10.2).

---

## 9. Monde, PvE & contenu

- **Monde semi-ouvert** : cartes interconnectées écran par écran (à la Dofus) ; biomes : plaines, forêt, marais, montagne, ruines, profondeurs.
- **Donjons** : instances de 4–6 salles avec combats scénarisés (arènes conçues main, riches en interactions) + boss à mécaniques uniques ; clés de donjon craftables.
- **Événements dynamiques** : caravanes de faction, invasions de monstres, boss mondiaux en zone noire (déclenchent des combats multi-groupes séquentiels).
- **Quêtes** : trame principale légère (découverte du monde et des factions) + contrats répétables régionaux ; la narration profonde passe par l'environnement et les saisons de faction.

---

## 10. Multijoueur & architecture réseau

- **Modèle** : serveur autoritaire, clients Godot ; monde partagé (méga-serveur avec canaux par région) — *ambition à valider, voir Q1.2 : le MVP peut être coop en ligne à petite échelle (serveurs de ~100 joueurs) avant le massivement multijoueur.*
- Combats instanciés côté serveur : le tour par tour est peu sensible à la latence (avantage majeur du genre).
- Anti-triche : toute résolution (RNG de loot, jets, dégâts) est serveur ; le client n'affiche que des prévisualisations.

---

## 11. Interface & expérience utilisateur

- **HUD combat** : barre de sorts, PA/PM/PW, timeline d'initiative, prévisualisation de dégâts/poussées/surfaces au survol.
- **Inventaire** : grille avec poids, comparateur d'objets, filtre par affixes, loadouts sauvegardés (« kit zone rouge »).
- **Carte du monde** : contrôle territorial en temps réel, niveaux de danger, événements actifs.
- Accessibilité : mode daltonien pour les surfaces (motifs en plus des couleurs), vitesse d'animation des tours réglable, timer de tour adaptable en PvE.

---

## 12. Direction artistique & audio (première intention)

- **DA** : stylisée semi-réaliste, lisibilité tactique avant tout (silhouettes claires, surfaces très identifiables). 2D isométrique ou 3D à caméra fixe — à trancher (Q3.2).
- **Audio** : ambiances par biome, « stingers » d'initiative, sons distinctifs par type de surface (feedback tactique aveugle possible).

---

## 13. Implémentation Godot (cadrage technique)

- **Godot 4.x**, GDScript en priorité (C# pour les systèmes chauds si nécessaire : résolution de combat, pathfinding).
- **Grille & pathfinding** : `TileMapLayer` + `AStarGrid2D` (ou AStar custom si hexagones) ; ligne de vue par lancer de rayon sur grille (algorithme de Bresenham).
- **Combat** : machine à états (placement → boucle de tours → résolution) ; actions = **pattern Command** (rejouables, annulables en prévisualisation, sérialisables pour le réseau et les replays).
- **Surfaces** : couche de grille dédiée avec règles de propagation/combinaison data-driven (`Resource` Godot).
- **Objets & affixes** : définitions en `Resource`/JSON (bases, pools d'affixes, courbes par tier) → génération 100 % data-driven, moddable et équilibrable sans code.
- **Réseau** : API multiplayer haut niveau de Godot pour le prototype ; serveur dédié headless Godot (`--headless`) ; base de données côté serveur (PostgreSQL) pour personnages/économie.
- **Sauvegarde** : aucune donnée d'autorité côté client (full loot + hardcore l'exigent).

---

## 14. Périmètre & jalons (proposition)

1. **Prototype tactique** (le « fun » d'abord) : 1 classe, 10 sorts, grille, PA/PM, 3 surfaces, poussées, 1 arène, PvE local.
2. **Vertical slice** : 3 classes, loot à affixes (3 raretés), 1 biome, 1 donjon, inventaire/équipement complet.
3. **Alpha en ligne** : serveur autoritaire, 5v5, zones bleue/rouge, full loot, marché basique.
4. **Bêta factions** : 3 factions, territoires, sièges, hardcore opt-in, saisons.

---

## 15. Questions pour affiner le GDD

### A. Vision & périmètre
- **Q1.1** — Quelle est l'ambition réseau réelle : MMO persistant (très coûteux), multijoueur en ligne à petite échelle (~50–200 joueurs par serveur), coop 2–8 joueurs, ou d'abord un jeu solo/coop avec du PvP en arène ? C'est LA décision structurante du projet.
    On va finalement opter pour du dolo/coop a 4 en PvM, pour l'instant on oublie le mode faction MMO trop ambitieux.
- **Q1.2** — Taille de l'équipe et compétences disponibles (code, art 2D/3D, réseau, serveur) ? Budget/temps visé pour un premier jouable ?
    Equipe de 4 joueurs max. Les compétences seront moitié "play what you wear", moitié sort dépendant de la classe. un kit fix de sorts par classe et pour quelques sorts, ils dépendront des équipements du joueur: toutes les classes peuvent jouer toutes les armes mais auront des sort qui diffèrent un peut selon les classes (par exemple un pyromant avec une épée mais aura le sort "épée magique" tandis que le sylve avec la meme épée aura le sort "estoc") donc la variété des sort dépendra du nombre de classes et du nombre d'archétype d'item differents.
- **Q1.3** — Modèle économique : premium (achat unique), free-to-play + cosmétiques, abonnement ? (impacte le design de l'économie et du hardcore)
    Free to play pour l'instant.
- **Q1.4** — Y a-t-il des jeux de référence supplémentaires dont tu veux copier un système précis (ex. Wakfu, Path of Exile, Baldur's Gate 3) ?
    wakfu pour les varietes de sort et d'interraction entre les sorts differents, les classes tournent autour de mechaniques de classes tres importantes dans le game play.
### B. Combat
- **Q2.1** — Grille carrée (DOS2), losange/isométrique (Dofus) ou hexagonale ? As-tu une préférence de lisibilité/feeling ?
    losange/isométrique (Dofus).
- **Q2.2** — Le trio PA/PM te convient-il, ou préfères-tu un pool d'action unifié (à la DOS2 : tout coûte des points d'action) ? Faut-il garder les PW (3e ressource) ?
    vie/mana/energie. mana pour les actions type spell etc, energie pour deplacement et interraction avec le décor (pousser un tonneau etc).
- **Q2.3** — Timer de tour strict (PvP nerveux) ou tours longs/illimités en PvE ?
    Tour long: on part sur une base de 2 minutes par tour de chaques joueurs.
- **Q2.4** — Tour par tour strictement séquentiel par personnage, ou par équipe (toute l'équipe joue en même temps, à la DOS2 en mode round) ?
    Tour par tour strictement séquentiel par personnage
- **Q2.5** — Combien de joueurs max dans un même combat (5v5 ? 8v8 ?) et faut-il des combats « rejoignables » (aggro d'un combat en cours, comme Albion, ou combats verrouillés comme Dofus) ?
    4 joueurs, et jusque 10 unités enemis.

### C. Environnement interactif
- **Q3.1** — Jusqu'où pousser la simulation : surfaces + poussées + destructibles suffisent-ils, ou veux-tu aussi hauteur/escalade, météo dynamique, téléportation d'objets ?
    météo oui, téleportation d'objets: certaines classes auront des sorts comme ca.
- **Q3.2** — 2D isométrique (moins cher, très lisible) ou 3D caméra tactique (plus cher, permet la vraie verticalité) ?
    2D isométrique
- **Q3.3** — L'IA des monstres doit-elle exploiter agressivement le décor dès le début, ou est-ce un raffinement post-MVP ?
    l'IA des monstres doit exploiter l'environement si elle le peut, dépendant des actions et sorts que peuvent faire chaques monstres.

### D. Classes & progression
- **Q4.1** — Classes fixes (à la Dofus) ou système sans classe basé sur l'équipement porté (à l'Albion : « you are what you wear ») ? Les deux se marient différemment avec le full loot.
  les deux comme précisé dans q1.2
- **Q4.2** — Niveau max et durée de progression visée (heures pour atteindre le « endgame ») ?
  pour l'instant 100 et durée de 1 mois pour etre MAX
- **Q4.3** — Les 5 archétypes proposés (§4.1) te parlent-ils ? Lesquels garder/modifier ?
  c bien pour l'instant

### E. Factions
- **Q5.1** — Trois factions prédéfinies, ou des guildes de joueurs qui revendiquent elles-mêmes les territoires (modèle Albion pur) ? Ou un hybride (factions + guildes internes) ?
  on enleve les factions
- **Q5.2** — Les sièges de territoire en « batailles tour par tour 5v5 sur points de contrôle » te semblent-ils la bonne résolution, ou imagines-tu de grandes batailles uniques ?
   on enleve les factions
- **Q5.3** — Le PvP même-faction doit-il être possible (hors-la-loi) ou strictement interdit ?
   on enleve les factions

### F. Hardcore & full loot
- **Q6.1** — Le hardcore (mort permanente) est-il un mode opt-in par personnage, un serveur dédié, ou la règle pour tout le monde ?
  le hardcore est présent partout, des le début du jeu. tu meurs: tu perds ton loot a jamais.
- **Q6.2** — En full loot, quel « trash rate » (part d'objets détruits à la mort) te semble juste ? 0 %, 30 %, 50 % ?
- **Q6.3** — Faut-il un filet de sécurité pour débutants (assurance d'objet, première mort pardonnée, zones bleues étendues) ?
  non hardcore des le début
- **Q6.4** — L'héritage hardcore proposé (§7.4 : 10 % maîtrises, recettes, 25 % réputation) est-il trop généreux, pas assez ?

### G. Items & affixes
- **Q7.1** — Combien de raretés veux-tu vraiment ? Le schéma Diablo complet (6 raretés dont sets) ou plus resserré (3–4) ?
  6 c'est bien
- **Q7.2** — Les légendaires « qui changent les règles » doivent-ils être trouvés uniquement (drop) ou aussi craftables/extractibles ?
  je sais pas encore, mais ils doivent etre rare
- **Q7.3** — Full loot + affixes procéduraux = perdre un objet unique fait très mal. Assumes-tu cette brutalité, ou veux-tu des mécanismes d'atténuation (empreinte de recette, re-craft à l'identique coûteux) ?
  le craft permettra de toujours pouvoir avoir un équipement "potable", mais full harcore c'est ce que je veux.
- **Q7.4** — Le craft doit-il être la source *principale* d'objets (Albion) ou le drop domine-t-il (Diablo) ?
    drop doit dominer

### H. Monde & contenu
- **Q8.1** — Monde en cartes interconnectées écran par écran (Dofus) ou zones continues avec chargements ?
    zone continue type wakfu
- **Q8.2** — Génération procédurale du monde/donjons, ou tout est conçu à la main ?
    génération procedurale
- **Q8.3** — Quelle importance pour la narration/quêtes : minimale (sandbox pur) ou trame scénarisée notable ?
    minimal

### I. Direction artistique
- **Q9.1** — As-tu une référence visuelle (Dofus cartoon ? DOS2 réaliste ? pixel art ?) et des ressources art disponibles ?
  pixel art pour l'instant
- **Q9.2** — Univers : médiéval-fantasy classique, ou envie d'un twist (post-apo, steampunk, mythologie précise) ?
  médiéval-fantasy, avec un twist steampunk et mythologie skandinave et asiatique.

### J. Production & technique
- **Q10.1** — GDScript seul, ou es-tu à l'aise pour mixer avec C# ?
  c# aussi
- **Q10.2** — Confirmation : monétisation cosmétique uniquement ? (impacte l'économie full loot)
 F2P
- **Q10.3** — Quel est le premier livrable que tu veux construire : le prototype de combat local (recommandé), ou directement une base réseau ?
  proto local en solo pour l'instant

---

*Document vivant : chaque réponse aux questions ci-dessus déclenchera une mise à jour de la section correspondante et une montée de version (0.2, 0.3, ...).*
