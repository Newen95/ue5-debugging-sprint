# Sprint 6 - Journal de debug

Notes prises au fil de l'eau pendant que je répare le projet. Pour chaque bug : ce que j'ai observé, ce que j'ai compris, ce que j'ai changé.

## Contexte

Projet Unreal reçu volontairement cassé. Plusieurs systèmes à remettre d'aplomb :

- le projet n'ouvre pas
- rien ne spawne au lancement
- contour blanc indésirable sur les cristaux
- sons des cristaux muets
- interaction non fonctionnelle
- puzzle qui ne réagit pas aux bonnes entrées
- perfs catastrophiques (~10-20 FPS sur scène vide)

Une fois stable, ajout d'une feature personnelle avec feedback visuel + sonore.

## Setup

- **2026-05-07** - Récupération du projet de base, création du repo `Sprint6_Berlin`, import du contenu initial.

## Bugs corrigés

### Bug 1 - Le projet ne s'ouvre pas

**Symptôme :**
Au double-clic sur `Berlin2025_Sprint6.uproject`, l'éditeur n'arrive pas à
ouvrir le projet. Il me demande de re-choisir la version d'Unreal (5.4),
puis affiche une modale rouge :
`Couldn't set association for project. Check the file is writeable.`
À ce stade aucun moyen d'entrer dans le projet, donc tous les autres
bugs sont bloqués derrière celui-là.

**Diagnostic :**
J'ai commencé par ouvrir le `.uproject` à la main dans un éditeur de
texte (c'est un simple JSON), et j'ai regardé le contenu du dossier
`Plugins/`. Deux choses anormales m'ont sauté aux yeux : une virgule
traînante après `"Enabled": true` sur l'entrée `DeadPlugin`, et un
plugin nommé `DeadPlugin` qui ne contenait que son `.uplugin`  - 
aucun dossier `Source/`, aucun `Binaries/`. Sa propre description dit
d'ailleurs : *« Un plugin cassé volontairement »*. Donc deux pièges
empilés, qu'il fallait isoler.

**Cause :**
1. **Le `.uproject` n'est pas un JSON valide.** La virgule traînante
   après le dernier champ d'objet est interdite par JSON strict
   (contrairement à JavaScript qui la tolère). Comme le parser de
   Unreal rejette le fichier dès l'erreur de syntaxe, l'éditeur n'arrive
   pas à lire `EngineAssociation`. C'est ce qui explique qu'il me
   redemande la version d'engine. Quand je la fournis, l'éditeur
   essaie d'écrire l'association de retour dans le fichier
   (load → modifier → save), et l'étape de load échoue → message
   générique "couldn't set association / check writeable". Le fichier
   est techniquement writeable ; le message est trompeur.

2. **Le plugin `DeadPlugin` est un piège séparé.** Le `.uplugin`
   déclare un module de type `Runtime` avec
   `LoadingPhase: Default` :

   ```json
   "Modules": [
     { "Name": "DeadPlugin", "Type": "Runtime", "LoadingPhase": "Default" }
   ]
   ```

   Pour qu'un module se charge, Unreal a besoin soit de sources C++
   dans `Source/<module>/`, soit de binaires précompilés dans
   `Binaries/Win64/UnrealEditor-<module>.dll`. Aucun des deux n'est
   présent. Donc même si je n'avais corrigé que la virgule, le projet
   aurait planté un cran plus loin avec une erreur du type
   *"The following modules are missing or built with a different engine
   version: DeadPlugin"*. Il faut traiter les deux causes ; corriger
   la première ne fait que démasquer la seconde.

**Correction :**
- Supprimé le dossier `Plugins/DeadPlugin/` en entier. J'ai préféré la
  suppression à un simple `"Enabled": false` : le plugin n'a ni
  sources ni binaires, le garder dans le repo n'apporte rien et ne
  ferait qu'ajouter du bruit lors d'un éventuel re-scan des plugins
  par Unreal.
- Supprimé l'entrée `DeadPlugin` dans le tableau `Plugins` du
  `.uproject`. La virgule fautive disparaît avec le bloc, et la
  virgule entre les deux plugins disparaît aussi puisqu'il ne reste
  plus que `ModelingToolsEditorMode`.

**Vérification :**
Double-clic sur le `.uproject` → l'éditeur s'ouvre, la map se charge,
le bouton Play lance le jeu (le bug 2 sur le spawn se manifeste alors,
mais c'est bien Bug 1 qui est levé).

**Test croisé fait pendant le diagnostic :**
J'ai d'abord supprimé seulement le dossier `Plugins/DeadPlugin/` sans
toucher au `.uproject` pour confirmer mon hypothèse. L'erreur n'a pas
changé d'un poil - exactement le même message sur "couldn't set
association". Ça m'a confirmé que la virgule était le bloqueur initial
(Unreal n'arrive même pas à dépasser le parsing du manifeste pour
constater l'absence du dossier). C'est seulement après le second fix
que l'éditeur s'est ouvert.

---

### Bug 3 - Contour blanc sur les cristaux

**Symptôme :**
Au lancement, **tous les cristaux sont entièrement blancs** dans la
scène, alors que selon la vidéo de référence ils devraient avoir leur
couleur propre selon leur émotion (rouge pour Anger, bleu pour
Sadness, vert pour Hope, jaune pour Joy, rose/magenta pour Fear,
orange pour Remission). Dans la vidéo, ils ne deviennent blancs qu'au
pickup.

À noter : le sujet appelle ce bug *"contour blanc indésirable"*. En
pratique ce n'est pas un fin contour mais un voile blanc qui couvre
toute la forme du cristal - la formulation du sujet est trompeuse.
Même origine, symptôme amplifié.

**Diagnostic :**
J'ai d'abord vérifié la chaîne de matériaux côté logique du cristal,
qui s'est révélée saine :

- L'enum `E_CrystalsType` a bien 6 valeurs (Anger, Sadness, Hope, Joy,
  Fear, Remission)
- `BP_Item_Crystals.Event BeginPlay` exécute un `Switch on
  E_CrystalsType` qui pose les bons matériaux/SFX/VFX/Light Color
  selon le type, puis applique tout ça aux composants
- Chaque instance dans la map a un `CrystalType` correctement défini
- Les material instances (`M_Crystal_Inst3`, `_Inst4`, etc.) ont bien
  des couleurs distinctes - vérifié dans le content browser

J'ai donc cherché ailleurs et trouvé dans `BP_MasterItem` (la classe
parent de `BP_Item_Crystals`) un composant **Decal_Outline** :

- `Decal Material` non assigné (None), `Decal Color = white`
- `Hidden in Game = false` par défaut (visible au lancement)
- L'`Event BeginPlay` de `BP_MasterItem` appelle `DisableOutline` pour
  cacher le decal au démarrage
- Des events `StartLookAtInteractables` / `StopLookAtInteractables`
  (via l'interface `BPI_Interactables`) sont câblés pour show/hide le
  decal quand le joueur regarde l'objet

Ce decal est donc le *"contour"* mentionné dans le sujet : il sert de
feedback visuel au survol. Mais sa couleur de fallback est blanche et
sa box englobe le cristal → quand il est visible, il fait passer tout
le cristal en blanc.

**Cause :**
`BP_Item_Crystals` override l'`Event BeginPlay` (pour son Switch sur
le type de cristal) **mais ne fait pas de `Parent: Begin Play`**. Sans
cet appel explicite, le BeginPlay du parent (`BP_MasterItem`) n'est
jamais exécuté → `DisableOutline` ne s'exécute pas → le
`Decal_Outline` reste visible dès le démarrage → tous les cristaux
apparaissent blancs.

C'est un piège classique d'Unreal : quand on override une méthode
virtuelle (BeginPlay, Tick, etc.) sur une classe enfant, il faut
appeler explicitement le parent si on veut que sa logique d'init
continue de tourner. Pas d'appel automatique.

**Test de confirmation avant fix :**
Décoché temporairement `Visible` sur `Decal_Outline` dans les defaults
de `BP_MasterItem`. Lancement → cristaux colorés correctement →
diagnostic confirmé. Réverti pour appliquer le vrai fix proprement.

**Correction :**
Dans `BP_Item_Crystals.Event BeginPlay`, ajout d'un node `Parent:
Begin Play` inséré entre `Event BeginPlay` et `Switch on E_CrystalsType`
(clic droit sur Event BeginPlay → *Add Call to Parent Function*).

Désormais :

- `Event BeginPlay` du cristal s'exécute
- `Parent: Begin Play` appelle le BeginPlay de `BP_MasterItem` →
  `DisableOutline` → decal caché
- Puis `Switch on E_CrystalsType` configure matériaux/SFX/VFX/Light
  Color selon le type

**Vérification :**
Les cristaux apparaissent désormais avec leurs couleurs propres au
lancement. Le decal reste caché tant que le joueur ne regarde pas
l'objet. ✅

**Notes pour plus tard :**
- Le `Decal Material` du `Decal_Outline` est `None`. Conséquence :
  même quand `EnableOutline` est appelé (regard sur le cristal), aucun
  outline visible ne se dessine. La vidéo d'exemple ne montre pas ce
  highlight, donc je laisse comme ça pour l'instant. Si à terme je
  veux un vrai feedback visuel *on look*, il faudra créer un decal
  material approprié.
- À vérifier : `BP_Item_CrystalPedestre` hérite probablement aussi de
  `BP_MasterItem`. S'il override BeginPlay sans `Parent: Begin Play`,
  même bug à corriger là-bas. À tester quand on touchera aux
  piédestaux.

---

### Bug 4 - Sons des cristaux muets

**Symptôme :**
Silence total en jeu. Aucun son sur aucun cristal, aucune ambiance.
Dans la vidéo de référence, chaque cristal émet un SFX d'ambiance lié
à son émotion (percussion sourde et inquiétante pour Anger, nappe
mélancolique pour Sadness, etc.) qu'on entend en s'approchant. Chez
moi, rien - quel que soit le cristal, quelle que soit la distance.

**Diagnostic :**
J'ai d'abord cartographié les assets audio du projet pour comprendre
ce qui était censé jouer :

- `Content/02-Musics_SFX/Crystals/` contient 6 Sound Cues
  (`SC_Anger`, `SC_Sadness`, `SC_Hope`, `SC_Joy`, `SC_Fear`,
  `SC_Remission`), un par valeur de `E_CrystalsType`
- `Content/02-Musics_SFX/SC_Master.uasset` (Sound Class racine)
- `Content/02-Musics_SFX/SC_Crystals.uasset` (Sound Class enfant)
- `Content/02-Musics_SFX/SA_Crystals.uasset` (Sound Attenuation)

Tous les ingrédients sont là. J'ai ensuite remonté la chaîne du son
en partant du BP cristal et en allant vers la sortie speaker, et j'ai
trouvé **quatre couches de sabotage empilées**, chacune suffisante à
elle seule pour rendre tout silencieux.

**Cause :**

1. **`SET CrystalSFX` vides dans le Switch.** Dans
   `BP_Item_Crystals.Event BeginPlay → Switch on E_CrystalsType`,
   chacune des 6 branches commence par un `SET CrystalSFX` mais la
   pin de valeur affichait `Select Asset` (None). Donc à la fin du
   Switch, la variable `CrystalSFX` valait None quel que soit le
   type, et le `SpawnSoundAttached` plus loin dans le graphe lançait
   un son nul.

2. **Sound Cues débranchées en interne.** Même après avoir branché
   les 6 cues sur les `SET`, toujours pas de son. En ouvrant
   `SC_Anger` dans l'éditeur de Sound Cue, j'ai vu un node
   `Looping Wave Player : Percussion_Loop_13` et un node `Output`,
   mais **aucun fil entre les deux**. La cue chargeait bien la wave
   mais sa sortie flottait dans le vide. Identique sur les 6 cues.

3. **Sound Class `SC_Master` à `Volume = 0`.** Une fois les cues
   recâblées, le bouton "Play Cue" de l'éditeur restait silencieux
   alors que la wave jouait correctement en preview directe. Les
   cues passent par `Class = SC_Master` (vérifié dans les Details de
   chaque cue), et `SC_Master.Volume` était à 0. Toute Sound Class
   à 0 dans la chaîne mute en cascade.

4. **`SA_Crystals` avec `Inner Radius = 0` et `Falloff Distance = 0`.**
   Avec le mute global levé, "Play Cue" produit du son dans
   l'éditeur, mais en jeu toujours rien. Le `SpawnSoundAttached` du
   BP applique l'atténuation `SA_Crystals` ; or les deux distances
   étaient à 0. Conséquence : le son est full volume sur un point
   géométrique unique (le pivot du cristal) puis tombe instantanément
   à zéro. Inaudible quelle que soit la position du joueur.

**Correction :**

1. Dans `BP_Item_Crystals` Event Graph, chaque branche du Switch reçoit
   le cue correspondant sur `SET CrystalSFX` :
   - Anger → `SC_Anger`
   - Sadness → `SC_Sadness`
   - Hope → `SC_Hope`
   - Joy → `SC_Joy`
   - Fear → `SC_Fear`
   - Remission → `SC_Remission`

2. Dans chaque Sound Cue, ajout d'un fil entre la pin de sortie du
   `Looping Wave Player` et la pin d'input du node `Output`.

3. `SC_Master.Volume` remis à `1.0`.

4. `SA_Crystals` reconfigurée :
   - `Inner Radius = 150` (full volume dans 1.5 m autour du cristal)
   - `Falloff Distance = 450` (fondu linéaire jusqu'au silence sur
     4.5 m supplémentaires, donc inaudibilité au-delà de ~6 m)
   Valeurs choisies par comparaison avec la vidéo de référence :
   les cristaux y sonnent comme des sources très locales, pas comme
   une nappe de cave globale. À 150/450 le rendu correspond à la
   vidéo (on entend le SFX en s'approchant, il s'éteint vite quand
   on s'éloigne, et les sons des différents cristaux ne se
   chevauchent pas dans le puzzle).

**Vérification :**
PIE → mouvement libre dans la cave. Chaque cristal émet bien son
ambiance émotionnelle propre quand on s'en approche, et le mix global
est cohérent avec la vidéo de référence. ✅

**Notes pour l'oral :**
- Les 4 couches sont indépendantes : si je n'en avais corrigé que 3,
  le son serait resté coupé. Le pattern *"sabotages empilés"* est le
  même que pour le bug 1 (virgule + plugin sans source). Stratégie
  pédagogique du sujet : forcer à diagnostiquer chaque maillon de la
  chaîne plutôt que de "chercher LE bug".
- Volet ambiance globale : la vidéo n'a pas de musique de fond
  diégétique au-delà des cristaux. Asset `OuterWild.uasset` (32 MB,
  un long wav) est présent dans `02-Musics_SFX/` mais référencé nulle
  part dans le projet. Je ne l'ai pas câblé puisque ce n'est pas
  attendu par le sujet/la vidéo. À garder en tête au cas où le
  correcteur le mentionne.

---

### Bug 5 - Interaction non fonctionnelle

**Symptôme :**
En jeu, je peux me déplacer librement et voir les cristaux, mais
impossible d'interagir avec eux : aucun feedback visuel quand je les
vise (la vidéo de référence montre un filtre blanc qui apparaît sur
le cristal visé), aucun prompt à l'écran, le clic gauche ne fait
rien. Le système d'interaction est entièrement muet.

**Diagnostic :**
J'ai cartographié l'architecture d'interaction dans `BP_FPSPlayer` :

- Une fonction `LineTrace` lance un raycast forward depuis la caméra
  chaque tick (longueur = `LineTraceLength`, channel = `Visibility`,
  `Draw Debug Type = For One Frame`)
- Si le hit a le tag `Item`, on stocke `OutHitActor` et on appelle
  `Start Look at Interactables` via l'interface `BPI_Interactables`
  (qui déclenche `EnableOutline` côté `BP_MasterItem` → le decal
  blanc s'affiche)
- Sur `IA_Player_Interact (Started)` : si on tient déjà quelque chose
  on `Detach`, sinon on `Attach` l'`OutHitActor` au socket `hand_r`
- `Should Raycast?` gate le tick (false pendant qu'on tient un objet
  pour éviter les traces parasites)

L'architecture est saine, donc le bug est forcément dans des valeurs
sabotées ou de la logique manquante. J'ai démasqué **trois couches**
empilées :

**Cause :**

1. **`Start with Tick Enabled` décoché sur la Class Defaults de
   `BP_FPSPlayer`.** Conséquence : `Event Tick` n'est jamais appelé,
   donc la fonction `LineTrace` n'exécute jamais
   `LineTraceByChannel`. Aucune trace, aucun debug visible, aucun
   hit possible. Pour le repérer j'ai utilisé le `Draw Debug Type =
   For One Frame` déjà présent dans le BP : si la trace tournait
   j'aurais vu une ligne rouge au centre de l'écran. Pas de ligne →
   le tick ne tourne pas. C'est ce qui m'a orienté vers les Class
   Defaults.

2. **Tags array vide sur `BP_Item_Crystals`.** Une fois le tick
   restauré la trace fonctionne (ligne rouge visible) mais aucun
   feedback. Le `Actor Has Tag (Item)` retournait toujours false.
   Côté `BP_MasterItem` (parent) le tag `Item` est bien défini, mais
   l'enfant `BP_Item_Crystals` avait son array Tags **explicitement
   réécrit à vide** dans ses Class Defaults - ce qui écrase
   l'héritage du parent. Reset de l'override → l'enfant ré-hérite
   du tag.

3. **`HoldingObject` jamais remis à `None` au relâchement.** Dernière
   couche : le premier grab/ungrab fonctionnait, mais ensuite on ne
   pouvait plus rien grab. Trace de la state machine :
   - Grab → `SET HoldingObject = OutHitActor` (crystal1).
   - Ungrab → `Detach From Actor` + `SET Should Raycast = true`,
     **mais aucun `SET HoldingObject = None`**.
   - Donc après ungrab, `HoldingObject` pointe toujours vers
     crystal1 alors qu'on ne le tient plus.
   - Re-vise crystal1 : `Branch (OutHitActor == HoldingObject)` →
     true → la branche `Start Look at Interactables` est skippée →
     pas de feedback visuel.
   - Vise crystal2 : `Start Look` se déclenche (objets différents),
     mais clic gauche → `IsValid (HoldingObject)` → true (variable
     périmée) → on tombe dans la branche ungrab qui appelle Detach
     sur un objet déjà détaché (no-op). Impossible de grab quoi que
     ce soit.

   Le pattern de variable d'état non clear après usage est un bug de
   logique classique sur les state machines manuelles en Blueprint.

**Correction :**

1. Class Defaults de `BP_FPSPlayer` → `Actor Tick → Start with Tick
   Enabled` → coché.

2. Class Defaults de `BP_Item_Crystals` → array Tags : reset de
   l'override pour ré-hériter du `Item` du parent (préféré à un ajout
   manuel pour rester en cohérence avec l'intention de la classe
   parente).

3. Dans l'Event Graph de `BP_FPSPlayer`, handler
   `IA_Player_Interact (Started)`, branche ungrab : ajout d'un
   `SET HoldingObject = None` entre `Detach From Actor` et
   `SET Should Raycast = true`. La variable est désormais clear à
   chaque relâchement, l'IsValid suivant retourne false comme
   attendu, et le système accepte un nouveau grab.

**Vérification :**
- En PIE la ligne de debug apparaît bien devant la caméra.
- En visant un cristal : il devient blanc (decal `EnableOutline` via
  l'interface) et le prompt "Press 'LMB' to grab" s'affiche au HUD.
- Clic gauche → le cristal s'attache à `hand_r`. Re-clic → relâche.
- Cycle grab → ungrab → re-grab → ungrab → grab d'un autre cristal :
  fonctionne sans verrouillage de la state machine. ✅

**Notes :**
- À vérifier sur `BP_Item_CrystalPedestre` : probablement même
  problème de Tags array vide à reset.
- Le `Display Info Text` du HUD s'affiche correctement, donc
  `WBP_HUD` est bien instancié et la référence `HUDRef` valide.
- Le piège (1) du tick désactivé est sournois car presque rien
  d'autre ne dépend du tick dans ce BP - mouvement (IA_Move) et
  caméra (IA_Look) passent par les events Enhanced Input et
  fonctionnent sans tick. Seule la trace en avait besoin.

---

### Bug 6 - Puzzle ne réagit pas

**Symptôme :**
La vidéo de référence montre que poser les 3 cristaux corrects sur
leurs piédestaux respectifs (un cristal par émotion attendue par le
piédestal) déclenche une cinématique et fait disparaître les rochers
qui bloquent l'accès à la prochaine pièce. En pratique : je peux
poser les 3 cristaux, le print de debug `ENIGMA END` apparaît bien
à l'écran (donc la condition de validité est atteinte côté logique),
mais **rien ne se passe ensuite** : pas de cinématique, pas de
rochers qui disparaissent, la zone reste bloquée.

Symptôme additionnel observé en parallèle : tout cristal placé sur un
piédestal apparaît visiblement **décoloré** (presque blanchâtre)
alors que les cristaux non placés ont leur couleur native (bleu pour
Sadness, vert pour Hope, rose/magenta pour Fear, etc.).

**Diagnostic :**
J'ai remonté la chaîne complète du puzzle, du placement individuel
jusqu'à l'événement final.

Architecture observée :

- Chaque `BP_Item_CrystalPedestre` capte l'arrivée d'un cristal via
  `On Component Begin Overlap (Sphere)`, l'attache à `CrystalPos`
  (Snap to Target), force ungrab côté joueur, puis check
  `Crystal Type == Feeling Type` du piédestal pour déclencher soit
  `GoodCrystal` (lumière verte) soit `WrongCrystal` (lumière rouge).
- La fonction `CheckifEnigmaIsOver` lit `First Enigma Total Valid`
  (compteur stocké côté `BP_FPSPlayer`) et compare à `== 3`.
- En face, le **Level Blueprint** de `L_DeepCave` est censé écouter
  un event dispatcher `EnigmaIsValid` exposé par les piédestaux et
  déclencher un `OpenRockDoor` qui joue `S_FirstEnigmaCompleted`
  puis `Destroy Actor` sur les `SM_Cliff_XX` qui bloquent la pièce.

J'ai trouvé **trois couches de sabotage empilées**.

**Cause :**

1. **`Call EnigmaIsValid` manquant dans `CheckifEnigmaIsOver`.** Sur
   la branche `True` du `Branch (== 3)`, il n'y avait qu'un
   `Print String "ENIGMA END"` - **aucun appel au dispatcher
   `EnigmaIsValid`**. Le dispatcher était bien déclaré dans le BP
   (sinon le Level Blueprint ne pourrait pas s'y binder), mais
   jamais firé. Conséquence : même quand le compteur atteint 3,
   le Level Blueprint n'est jamais notifié → rien ne se déclenche
   dans le monde. Le print `ENIGMA END` était la trace de cette
   détection orpheline.

2. **`Bind Event` mal câblé dans le Level Blueprint.** Sur
   `BeginPlay → Delay 0.2s → Bind Event to EnigmaIsValid`, j'ai
   trouvé **un seul node `Bind Event`** avec **trois références
   piédestal câblées sur le même pin `Target`**. Or `Bind Event`
   n'accepte qu'un seul Target - visuellement les 3 wires donnent
   l'illusion d'écouter les 3 piédestaux, mais à runtime un seul
   est effectivement bindé (probablement le dernier wired).
   Conséquence : bug **non-déterministe**. Chaque piédestal a son
   propre dispatcher `EnigmaIsValid` (per-instance en BP) ; selon
   l'ordre dans lequel le joueur pose les cristaux, le 3ème (celui
   qui déclenche `EnigmaIsValid`) tombe ou non sur le piédestal
   écouté. Sur 6 ordres possibles, ~2 marchent, ~4 échouent
   silencieusement. Particulièrement vicieux car en démo rapide,
   on peut tomber par chance sur un ordre fonctionnel et conclure
   à tort que le puzzle marche.

3. **`Decal_Outline` héritée visible sur le piédestal.**
   `BP_Item_CrystalPedestre` hérite de `BP_MasterItem`, donc
   embarque le composant `Decal_Outline`. Sur les cristaux ce
   composant est correctement caché à `BeginPlay` (cf. fix Bug 3).
   Sur le piédestal il est visible par défaut, sans aucune logique
   pour le manipuler - vérifié par scan du `.uasset` : aucune
   référence à `Decal_Outline` dans l'EventGraph ni le Construction
   Script du piédestal. Le decal projette du blanc sur tout ce qui
   passe en dessous → quand un cristal vient se poser à
   `CrystalPos`, il atterrit sous la projection blanche → cristal
   visuellement blanchi via son matériau translucide. C'est la
   variante "piédestal" du Bug 3 (decal blanc parasite côté
   cristal), même cause racine (composant hérité visible par
   défaut), mais cette fois sans aucun handler à corriger.

**Correction :**

1. Dans `BP_Item_CrystalPedestre.CheckifEnigmaIsOver`, ajout d'un
   node **`Call EnigmaIsValid`** sur la branche `True` du
   `Branch (== 3)`, juste avant le Print `ENIGMA END`. Compile +
   Save.

2. Dans le Level Blueprint de `L_DeepCave`, refonte du bind :
   - Détaché les 3 wires fantômes du pin Target unique du
     `Bind Event` initial.
   - Insertion d'un node **`Sequence`** entre `Delay (Completed)`
     et le Bind Event, avec 3 outputs (`Then 0`, `Then 1`,
     `Then 2`).
   - Duplication du `Bind Event to EnigmaIsValid` pour avoir 3
     instances séparées.
   - Câblage : `Then 0` → Bind #1 (Target =
     `BP_Item_CrystalPedestre2`), `Then 1` → Bind #2 (Target =
     `BP_Item_CrystalPedestre3`), `Then 2` → Bind #3 (Target =
     `BP_Item_CrystalPedestre4`). Les trois Bind Events ont le
     même `Event` input câblé vers le custom event `OpenRockDoor`.
   - Compile + Save.

3. Sur l'instance `Decal_Outline` héritée du piédestal :
   suppression impossible côté enfant (composant hérité de
   `BP_MasterItem`, et le supprimer côté parent casserait les
   cristaux qui l'utilisent légitimement pour leur feedback
   look-at). Solution : dans le panel Components de
   `BP_Item_CrystalPedestre`, sélection de `Decal_Outline` →
   Details → section Rendering → **`Hidden In Game = true`**. Le
   composant reste déclaré (pas d'orphan reference) mais n'est
   plus rendu en jeu.

**Vérification :**

- Place les 3 bons cristaux dans n'importe quel ordre (j'ai testé
  tous les ordres) → cinématique `S_FirstEnigmaCompleted` joue,
  les `SM_Cliff_XX` disparaissent, l'accès à la nouvelle pièce
  s'ouvre. ✅
- Cristal placé sur piédestal → couleur native conservée (bleu pour
  Sadness, vert pour Hope, etc.), plus de blanchiment. ✅

**Notes pour l'oral :**

- Le saboteur n°1 (Call manquant) est typique : tout est en place
  visuellement (le dispatcher est déclaré, la condition est testée,
  le print confirme la détection), mais le node final qui propage
  le signal a été supprimé. Avec juste ce print à l'écran, on peut
  conclure "ah le puzzle marche, il manque juste l'effet visuel"
  sans réaliser que la chaîne est rompue.
- Le saboteur n°2 (3 wires sur 1 Bind) crée un comportement
  **non-déterministe** : selon l'ordre de résolution du joueur, le
  puzzle marche ou pas. C'est exactement le profil de bug latent
  qui peut passer inaperçu en test rapide et exploser en démo si
  l'ordre testé n'est pas le bon. Solution : un `Sequence` + N
  `Bind Event` distincts, un par cible.
- Le saboteur n°3 (decal piédestal) est une variante du Bug 3 mais
  sans handler à corriger. La contrainte d'héritage Blueprint
  (impossible de supprimer un composant hérité côté enfant)
  explique pourquoi `Hidden In Game` est la bonne réponse plutôt
  que la suppression.
- Pour mémoire pendant le diagnostic : j'avais observé un cas
  transitoire où le 3ème cristal refusait de se poser sur son
  piédestal (état physique probablement coincé après plusieurs
  grab/ungrab). Pas reproduit après les fixes ci-dessus, donc je
  ne le compte pas comme une 4ème couche.

---

## Optimisation

### Lag progressif - accumulation de BP_FPSPlayer

**Symptôme :**
Au démarrage PIE le jeu tourne à un framerate médiocre mais stable.
Au bout de quelques secondes ça commence à ramer, et plus la partie
dure plus c'est laggy, jusqu'à devenir totalement injouable
(<10 FPS au bout d'une minute). Pas un baseline lent uniforme,
mais une **dégradation progressive** dans le temps.

**Diagnostic :**
Dégradation progressive ≠ baseline lent. Si c'était juste du
lighting/shaders coûteux, le framerate serait stable bas. Une
dégradation au fil du temps signale typiquement une **accumulation** :
actor / particle / audio component / timer entry qui se crée chaque
frame sans être nettoyé, ou GC saturée.

J'ai ouvert le **World Outliner pendant le PIE** : confirmation,
le compteur d'actors `BP_FPSPlayer` montait en flèche
(~1 instance par frame au pic, soit ~3 600 par minute à 60 fps).
Chaque instance traîne sa CharacterMovement, sa Camera, ses
components → GPU et CPU saturent rapidement.

J'ai cherché les sources via `Find in Blueprints` global sur
`SpawnActor` puis sur `BP_FPSPlayer`. Trois nodes
`Event Tick → SpawnActor BP_FPSPlayer` empilés dans des graphes
différents :

1. `WBP_HUD` (Event Graph)
2. `GM_Level1` (Event Graph)
3. **Level Blueprint de `L_DeepCave`** (Event Graph du level)

Les trois identiques : Spawn à `(0, 0, 5000)` avec
`Collision Handling = Always Spawn, Ignore Collisions`. Pattern
récurrent du projet - comme les couches du bug 4, le dev casseur
a posé le même piège dans plusieurs endroits pour qu'on n'en
élimine qu'une partie au premier passage.

**Cause :**
Ces trois `Event Tick → SpawnActor` font apparaître un nouveau
`BP_FPSPlayer` chaque frame, dans trois contextes différents
(widget tick, gamemode tick, level blueprint tick). Aucun n'est
nettoyé. Aucun n'a de raison gameplay d'exister - c'est de la
sabotage pure. Plus le jeu tourne, plus le level se peuple.

**Sous-piège supplémentaire - recompilation :**
Après suppression des nodes et `File > Save All`, les spawns
continuaient au runtime. Cause : *en supprimant des nodes, le
graphe source est nettoyé mais le **bytecode compilé** dans le
`.uasset` reste obsolète tant qu'on ne **Compile** pas le BP*.
PIE exécute le bytecode, pas le graphe source. Donc tant que le
bouton Compile n'a pas été cliqué après chaque édition, les
spawns persistaient même fichier sauvegardé.

C'est une subtilité Unreal : `Save` écrit le fichier mais ne
force pas `Compile`. Pour les Level Blueprints en particulier,
l'auto-compile au save n'est pas systématique. À retenir : *toute
modification de graphe doit être Compile + Save*, dans cet ordre.

**Correction :**
- Suppression des trois paires `Event Tick + SpawnActor` dans
  `WBP_HUD`, `GM_Level1` et le Level Blueprint de `L_DeepCave`.
- **Compile** explicite sur chaque BP modifié, puis Save.

**Vérification :**
PIE relancé après compile + save : Outliner affiche un seul
`BP_FPSPlayer` (le pawn du joueur) et ne grossit plus. Le
framerate reste stable dans le temps (à hauteur du baseline,
qui lui reste à améliorer). ✅

### Instrumentation pour le profilage

Avant d'attaquer le baseline, j'ai mis en place un mesure
chiffrée pour pouvoir comparer avant/après chaque action.

- Ajout d'une touche console accessible (`²` puis `F1` en
  fallback) dans `Project Settings → Engine → Input → Console
  Keys`. Les défauts (`` ` `` et `Caret`) ne sont pas
  accessibles en AZERTY, sans ça impossible de taper
  `stat fps`/`stat unit` en PIE.
- Mesure baseline avec `stat fps` + `stat unit` +
  `stat scenerendering` :
  - **50.69 FPS** / Frame **20.26 ms**
  - Game `17.24 ms`, **Draw `20.38 ms`** ← bottleneck,
    RHIT `4.63 ms`, GPU `5.59 ms` (très peu sollicité)
  - 766 draws, 1.37 M prims, **114 lights in scene**, 19 decals
    in view
  - Warnings actifs : `LIGHTING NEEDS TO BE REBUILT (533 unbuilt
    objects)`, `REFLECTION CAPTURES NEED TO BE REBUILT
    (9 unbuilt)`, `PROFILING WITH AI LOGGING ON!`

Lecture du diagnostic : le **CPU render thread (Draw) est
saturé** alors que le GPU s'ennuie. C'est typique d'une scène
avec beaucoup de lights dynamiques + lighting non précalculé.
Le levier prioritaire est donc **lighting**, pas la complexité
géométrique ni les shaders.

### Cleanup du `Config/DefaultEngine.ini`

`GameEngine=/Script/MissingPlugin.MagicEngineSubsystem` (ligne 85
avant fix) - référence un module qui n'existe pas dans le projet
(il n'y a même pas de C++). Au démarrage du jeu en standalone,
Unreal essaie de résoudre cette classe → échec → fallback sur
le défaut (`/Script/Engine.GameEngine`). Pas un bloqueur en
éditeur mais pollue les logs et c'est exactement le genre de
résidu de sabotage qu'un évaluateur peut repérer en ouvrant le
`.ini`. Faux jumeau du `MissingPlugin.MagicEngineSubsystem` du
`.uproject` qu'on avait viré pour Bug 1.

**Correction :** suppression complète de la ligne. UE retombe
sur le défaut implicite `/Script/Engine.GameEngine`, plus propre
qu'une réécriture explicite.

### Audit et nettoyage des lights

Le World Outliner révèle des lights mixtes (Static / Stationary
/ Movable) avec beaucoup de **doublons et de Movable invisibles
ou redondantes** - rayons d'attenuation qui se chevauchent
(sphères rouges qui se recouvrent dans l'éditeur), Movable avec
intensité élevée mais sans utilité visible, Movable purement
décoratives qui ne devraient pas l'être (le coût d'une Movable
est multiple d'une Static car ombres dynamiques en permanence).

**Correction :** suppression des Movable dont la zone éclairée
ne correspondait à rien de visible et qui chevauchaient une
autre light déjà présente. Audit visuel light par light dans
l'Outliner avec preview de la zone d'attenuation.

### Build Reflection Captures et mesure éditeur

Avant tout autre changement, j'ai voulu une mesure
avant/après-clean défendable. J'ai exécuté **Build Reflection
Captures** (les 9 sondes unbuilt) puis pris une nouvelle capture
de `stat unit` depuis le spawn :

- **55 FPS** / Frame **18.40 ms**
- Game `17.80 ms` (nouveau bottleneck CPU)
- **Draw `10.09 ms`** ← divisé par 2 vs baseline (20.38 ms)
- GPU `5.70 ms`, RHIT `5.09 ms`
- 833 draws, 1.39 M prims
- **RenderRes 50.0% (676×404)** - point important

Lecture : le Draw thread a chuté de moitié grâce au combo
*lights bakées + reflections buildées + lights inutiles
supprimées*. Le bottleneck a basculé sur le Game thread.

**Mais** : le `RenderRes 50%` révèle que **Dynamic Resolution
Scaling** intervient pour maintenir le FPS. Le moteur rend en
demi-résolution (676×404 upscalé). Tant que je mesure dans
l'éditeur, je ne vois pas la vraie charge GPU.

### Sur le `Draw Debug Type` du `LineTraceByChannel`

J'avais noté dans la version précédente de ce journal qu'il
fallait le passer à `None`. Après réflexion : la ligne rouge
dessinée chaque frame **fait office de viseur visuel** au centre
de l'écran (le projet n'a pas de réticule HUD). J'ai donc choisi
de la **laisser sur `For One Frame`**. Coût marginal, gain UX
concret. Trade-off assumé.

### Sur les warnings `PROFILING WITH AI LOGGING ON!` et `PROFILING WITH GC VERIFY ON!`

Grep complet dans `Config/*.ini` : aucune trace de
`AILogging`, `bEnableAILogging`, `gc.VerifyAssumptions`, etc.
Ces warnings sont activés par défaut **par Unreal en mode
éditeur/PIE** pour valider l'intégrité du runtime. Ils
disparaissent en build standalone Development/Shipping (vérifié
en regardant les screenshots de mes packages - ces warnings ne
sont pas présents).

Ce ne sont **pas des sabotages projet**, juste de l'overhead
éditeur. Aucune action à prendre. À mentionner à l'oral si la
question vient.

### Premier package Windows - révélation de la vraie perf

Pour valider que le projet est sain et avoir une mesure honnête
hors influence Dynamic Res, j'ai fait un **Package Project
Windows / Development** vers `C:\UnrealBuilds\Sprint6\`. Le
build sort propre (aucune erreur de cook).

Lancement du `.exe`, mesure au spawn :

- **22 FPS** / Frame **43.48 ms** ← divisé par 2.5 vs éditeur
- Game `4.40 ms` (très bas - pas d'overhead éditeur)
- **Draw `43.48 ms`** / GPU `42.88 ms` ← le GPU est saturé
- **RenderRes 86.2% (1324×745)** - DRS moins agressif qu'en
  éditeur, on voit la vraie charge
- `Draws: 1771`, `Prims: 2.94 M`
- 30 secondes d'écran noir au démarrage, sauf une light qui
  éclaire la porte
- Warning rouge : **`[VSM] Non-Nanite Marking Job Queue overflow.
  Performance may be affected.`**

Analyse :
- L'éditeur **mentait** via Dynamic Resolution Scaling à 50%.
  La vraie charge GPU est ~43 ms par frame, soit ~23 FPS.
- L'écran noir au démarrage = **shaders/PSO compilés à la volée
  + saturation VSM** au boot (la queue met ~30s à se stabiliser
  avant que la scène devienne visible).
- Le warning VSM pointe vers la cause racine : **trop de meshes
  non-Nanite dans une grosse zone d'ombre**, le système Virtual
  Shadow Maps de UE5 n'arrive pas à indexer assez vite.

### Bascule VSM → Shadow Maps + activation Nanite

Deux fixes liés, à appliquer ensemble :

1. **Désactiver VSM** : `Project Settings → Rendering → Shadow
   Map Method` : `Virtual Shadow Maps (Beta)` → `Shadow Maps`.
   VSM est conçu pour des scènes 100% Nanite ; ici la cave
   n'était pas Nanite, l'overflow était inévitable.

2. **Activer Nanite sur les meshes lourds** : ~110 props dans
   `Content/DeepElderCaves/Meshes/Props/` + dossier
   `Architecture/`. Sélection bulk dans le Content Browser →
   clic droit → `Nanite → Enable`. Nanite réduit drastiquement
   le coût de rendu des meshes haute densité (clusters
   adaptatifs en place des draws traditionnels).

**Re-package + mesure :**
- FPS stable à ~23 (pas mieux), GPU toujours saturé.
- Deux nouveaux warnings rouges : **`LIGHTING NEEDS TO BE
  REBUILT (3,225 unbuilt objects)`** et **`Video memory has
  been exhausted (81,292 MB over budget)`**. 81 **giga** au
  dessus du budget, c'est énorme.

Diagnostic : activer Nanite invalide les lightmaps. Sans VSM
(que j'ai désactivé) et sans lightmaps (qui sont à rebuild),
les 3225 objets passent en **shadow maps dynamiques par
défaut**. Chaque light × chaque mesh alloue sa propre shadow
map → la VRAM explose. Les deux warnings disent la même chose.

### Build Lighting (Production) - fix VRAM

Réponse : **Build → Build Lighting Only → Production quality**.
~25 min sur 3 225 objets. Pendant ce temps j'ai préparé l'audit
textures (voir section suivante).

### Audit textures - leçon sur les Texture Groups

Profitant du temps de build, j'ai ouvert
`Window → Developer Tools → Asset Audit` filtré sur
`Texture2D`, trié par `Dimensions`. Premier constat : **65
textures en 4K** (cliffs, stones, mushrooms, character Manny,
divers).

J'ai ouvert `T_Cliff_04_D` (diffuse) pour mesurer l'impact :

- `Imported: 4096×4096`
- **`Displayed: 1024×1024`**
- **`Max In-Game: 1024×1024`**
- `Combined LOD Bias: 2`

Donc cette diffuse est en fait chargée à 1K, pas 4K. Le
**Texture Group `World`** applique un `LOD Bias = 2` automatique
(skip 2 mips → 4K÷4 = 1K). Côté VRAM, elle ne coûte que 683 KB,
pas 22 MB.

Vérification croisée sur `T_Cliff_05_N` (normal) :

- `Imported: 4096×4096`
- **`Displayed: 4096×4096`** ← chargée full 4K
- `Combined LOD Bias: 0`

Le Texture Group **`WorldNormalMap` n'a pas de LOD Bias par
défaut** (choix Unreal natif pour préserver la qualité des
normales). Donc les ~25 normales d'environnement consomment
~1.4 GB de VRAM à elles seules - une partie significative des
448 MB d'overrun final.

**Correction ciblée :** ouvrir chaque normale concernée et
définir `Maximum Texture Size = 2048` (cap absolu, plus
prévisible qu'un LOD Bias). J'ai traité :

- 7 cliff normales (`T_Cliff_01..07_N`)
- 4 stone tile normales (`T_StoneTile_A..D_N`)
- + quelques diffuses cliff/stone qui échappaient aussi au LOD
  Bias (incohérence projet : certaines `_D` étaient déjà cap à
  2048, d'autres pas du tout)

J'ai laissé les normales de mushrooms et character Manny pour
plus tard si besoin (impact secondaire).

### Mesure après Build Lighting + textures

Re-package + mesure :

- Warning VRAM : **`81,292 MB over budget` → `448 MB over
  budget`** - ordre de magnitude résolu, reste un dépassement
  marginal.
- Démarrage : encore un peu noir mais beaucoup moins long que
  les 30s du premier package.
- FPS : reste autour de 23, GPU toujours saturé.

Donc la VRAM n'est plus le problème dominant. Mais le GPU
travaille toujours trop. J'ai basculé en **viewmode Light
Complexity** (`Optimization Viewmodes → Light Complexity`)
pour voir d'où vient le coût.

### Light Complexity - le vrai bottleneck restant

Capture éloquente : la scène est dominée par du **rouge
("Bad")** et du **rose pâle ("Extremely bad")**, signifiant que
chaque pixel est touché par **6 à 10+ lights simultanément**.
Seules quelques zones sont en vert. Échelle : `Good → Bad →
Extremely bad`.

Lecture rapide via le World Outliner filtré "Light" : seulement
17 lights placées individuellement (`PointLight*`, `SpotLight*`,
`SkyLight`). Le reste vient des **BP de champignons lumineux**
(`BP_Mushroom_Lit_A/D/E/F`), instanciés en nombre dans le level.
Chaque instance porte un `LightComponent` qui éclaire son
environnement → quand 5-10 champignons se rapprochent, leurs
zones d'attenuation se chevauchent et chaque pixel paye le
coût de chaque light.

**Première tentative de correction :** désactiver `Cast Shadows`
sur le `LightComponent0` de chaque `BP_Mushroom_Lit_*` →
constaté en ouvrant les 4 BPs concernés : **les Cast Shadows
sont déjà désactivés**. Donc ce n'est pas le shadow casting le
problème, c'est l'**overlap brut** des contributions
lumineuses (chaque light reste un calcul d'éclairage à faire
même sans ombre).

### À faire ensuite (Bug 7)

1. **Mesurer l'`Attenuation Radius` des `LightComponent0` des
   `BP_Mushroom_Lit_*`** (4 BPs : A, D, E, F). Si > 500, le
   réduire à 200-300. Une lueur de champignon n'a pas besoin
   d'éclairer à 20 m.
2. **Décocher `Affects Translucent Lighting` et `Affects
   Indirect Lighting`** sur ces mêmes light components (gain
   secondaire mais quasi gratuit).
3. **Si insuffisant** : passer la Mobility de Movable →
   Stationary + nouveau Build Lighting Production → contribution
   bakée, light dynamique résiduelle très réduite.
4. **Solution radicale si besoin** : World Outliner →
   `BP_Mushroom_Lit_*` → supprimer 1 instance sur 2 dans les
   zones de plus forte densité.
5. **Audit des 17 lights individuelles** : même critère
   d'attenuation, vérifier si certaines Movable peuvent passer
   en Stationary.

### Bilan provisoire Bug 7

- **Avant tout fix :** ~10-20 FPS, baseline catastrophique
- **Après cleanup lights + ini + build lighting + Nanite +
  caps textures :** ~23 FPS standalone (avec DRS désactivée
  côté moteur), Draw 10 ms en éditeur, VRAM ramenée dans des
  bornes raisonnables (~448 MB over au lieu de 81 GB)
- **Plafonné par Light Complexity overlap** → c'est le prochain
  levier, attaquable par réduction des attenuation radius.

Le scope "fix sabotage" est largement couvert. La marge restante
relève de l'optimisation fine (rayons d'attenuation, dosage du
nombre d'instances). À continuer prochaine session.



### Bug 8 - Porte traversable sans résoudre l'énigme

**Symptôme :**
Le joueur peut passer à travers la porte de la première énigme sans
résoudre quoi que ce soit. La porte est composée de plusieurs cliffs
préfixés `SM_Cliff_*` qui forment une arche bloquant le passage.

**Diagnostic :**
Premier réflexe : sélectionner un cliff de la porte (`SM_Cliff_64`).
L'Outliner affiche son `Type` comme `Geometry`, pas `StaticMeshActor`.
Le composant racine est un `GeometryCollectionComponent` - c'est en
fait un `GeometryCollectionActor` du système Chaos Destruction,
malgré son préfixe `SM_` trompeur. Tous les cliffs de la porte sont
des instances de GC partageant le même asset `GC_Rock_FirstEnigma`,
lui-même généré à partir du StaticMesh source `SM_Cliff_02`.

Viewmode `Player Collision` : la zone porte apparaît en **magenta**
(collision présente mais ne bloque pas le pawn) au milieu de cliffs
cyan environnants (bloquants normaux). Pourtant en inspectant la
collision de l'instance dans Details, tout est correctement configuré :
`Collision Enabled = Query and Physics`, `Object Type = Destructible`,
**toutes les Object Responses sur Block, y compris Pawn**.

J'ai testé plusieurs pistes sans effet :
- `Import Collision from Source` coché sur l'asset GC
- `Always Create Physics State` coché sur l'instance
- `Use Static Mesh Collision for Traces` coché sur l'instance

La piste s'est ouverte avec la commande console `show collision`
lancée en PIE. J'ai remarqué que les cliffs de la porte sont rendus
en **cyan** (collision complexe per-triangle) alors que les autres
murs et meshes de la grotte sont en **bleu foncé** (collision
simple - Box / Sphere / Convex hull). Le `CapsuleComponent` du player
utilise par défaut la **collision simple** pour ses sweeps de
mouvement (c'est rapide et stable). Sans simple collision exploitable,
le pawn capsule traverse comme s'il n'y avait rien.

**Cause :**
Combinaison cassée propre à Unreal 5 :
- Le `StaticMesh` source `SM_Cliff_02` est **Nanite enabled**
- Il est configuré en `Collision Complexity = Use Complex Collision
  As Simple`
- Il a été utilisé pour générer une `GeometryCollection`

Cette combinaison ne produit pas de collision simple exploitable par
le pawn capsule sweep. La pipeline Chaos hérite du source mais ne
sait pas reconstruire de primitive simple à partir d'un Nanite mesh
marqué "use complex as simple" - résultat : uniquement de la complex
collision, qui visuellement remplit l'espace dans le viewmode mais
ne bloque pas les sweeps capsule.

Test de contrôle : drag-and-drop direct de `SM_Cliff_02` dans le
niveau (en tant que `StaticMeshActor` indépendant) → le joueur est
bloqué normalement. C'est bien le wrap Chaos qui casse la pipeline,
pas le mesh source lui-même.

**Correction :**
Plutôt que de tenter de réparer le pipeline Chaos (notoirement
capricieux et hors scope du sprint), j'ai séparé la responsabilité
visuel / collision sur chaque cliff de la porte :

1. Sur chaque `GeometryCollectionActor` du level concerné, ajout
   d'un `StaticMeshComponent` enfant, mesh = `SM_Cliff_02`,
   `Collision Preset = BlockAll`.
2. Sur le `GeometryCollectionComponent` parent : `Visible = off`,
   `Hidden in Game = on`, `Collision Presets = NoCollision`.
   On garde le composant pour ne pas casser les bindings de la
   `Level Sequence` `S_FirstEnigmaCompleted` (qui anime la descente
   progressive de la porte par cliff).
3. Sur l'asset `SM_Cliff_02` : génération d'une `Auto Convex
   Collision` (Hull Count = 8, Verts = 16) et bascule de
   `Collision Complexity` de `Use Complex As Simple` vers
   `Default`. Le SM enfant a maintenant une vraie simple collision
   exploitable.
4. Le `StaticMeshComponent` enfant hérite naturellement des Transform
   animations du parent GC via la `Level Sequence` - la porte
   descend toujours progressivement, mais avec une collision réelle
   qui suit le mouvement.
5. Ajustement du Scale du SM enfant pour matcher l'extent visuel
   du GC d'origine (le SM brut était légèrement plus petit, le
   joueur s'enfonçait un peu dans le rocher avant d'être bloqué).

**Vérification :**
- En viewmode `Player Collision`, les cliffs de la porte
  apparaissent désormais en cyan classique (simple collision
  présente et bloquante pour le pawn).
- PIE : le joueur est bloqué net à l'entrée de la porte avant
  résolution.
- Une fois l'énigme résolue, la `Level Sequence` joue : les cliffs
  descendent comme prévu, les SM enfants descendent avec eux, le
  passage s'ouvre progressivement.

**Notes :**
- Découverte importante : le préfixe `SM_` dans le nom d'un acteur
  ne garantit pas qu'il s'agit d'un `StaticMeshActor`. Toujours
  vérifier le `Type` réel dans l'Outliner ou le composant racine
  dans Details. Ici les cliffs ont visiblement été convertis en
  `GeometryCollection` à un moment sans renommage.
- Le `GeometryCollection` n'apporte aucun bénéfice fonctionnel ici
  puisque la porte ne fracture pas (elle se contente de descendre).
  L'usage du système Chaos était un overkill légué du projet de
  base. Le garder en boîte vide visuelle/collision est le moindre
  mal vu qu'il faut préserver les bindings Sequencer.



### Bug 9 - Piédestaux saisissables comme des cristaux

**Symptôme :**
En jeu, en visant un piédestal avec la trace d'interaction, le prompt
"Press 'LMB' to grab" s'affiche et un clic gauche permet de le
ramasser et le déplacer comme un cristal. Comportement absurde - un
piédestal est censé être un réceptacle fixe, pas un objet portable.

**Diagnostic :**
Le système de grab dans `BP_FPSPlayer.LineTrace` filtre les cibles
via `Actor Has Tag (Item)`. Tout ce qui porte ce tag est considéré
comme saisissable. Or le tag `Item` est défini directement sur
`BP_MasterItem`, la classe parent commune des cristaux et des
piédestaux.

Quand j'avais corrigé Bug 5, j'avais reset l'override Tags sur
`BP_Item_Crystals` pour qu'il ré-hérite du tag `Item` du parent. À
l'époque j'avais flaggé dans mes notes : *"à vérifier sur
`BP_Item_CrystalPedestre` : probablement même problème de Tags array
vide à reset"*. La piste était bonne mais en sens inverse de ce que
je pensais : ce n'est pas que le piédestal a perdu son tag, c'est
qu'il l'a hérité alors qu'il ne devrait pas - sans override, il
prend le tag de son parent et devient détectable comme un objet à
ramasser.

**Cause :**
`BP_Item_CrystalPedestre` n'override pas le tableau Tags de
`BP_MasterItem`. Par défaut, il hérite donc du tag `Item`, ce qui le
rend détectable par le raycast de grab. Aucune intention gameplay
derrière - c'est juste une conséquence non désirée du partage de
classe parent. Variante du pattern *"sabotages par héritage"* déjà
rencontré sur le `Decal_Outline` hérité du piédestal en Bug 6.

**Correction :**
Class Defaults de `BP_Item_CrystalPedestre` → tableau Tags : retrait
du tag `Item` hérité (override avec un tableau vide). Compile + Save.

C'est le pattern inverse de Bug 5 : sur le cristal, j'ai retiré
l'override pour ré-hériter du parent ; sur le piédestal, je rajoute
un override vide pour bloquer l'héritage. Même mécanisme Unreal
exploité dans les deux sens.

**Vérification :**
PIE → vise un piédestal : plus de prompt "Press 'LMB' to grab", plus
de feedback visuel de look-at, le raycast traverse comme attendu
pour un décor fixe. Vise un cristal : feedback et grab fonctionnent
toujours normalement. Le puzzle (Bug 6) reste opérationnel - la
détection de pose d'un cristal sur le piédestal passe par
`On Component Begin Overlap` du `Sphere`, pas par le système de
tags. ✅

**Notes pour l'oral :**
Tout enfant de `BP_MasterItem` mérite un audit de ce qu'il hérite vs
ce qu'il devrait réellement porter. Ce bug aurait pu rester latent
longtemps puisqu'il n'empêche aucune mécanique de fonctionner - il
introduit juste un comportement absurde côté joueur que personne ne
détecte tant qu'il n'essaie pas explicitement d'interagir avec un
piédestal.



### Bug 10 - Piédestal inutilisable après le premier retrait

**Symptôme :**
Cycle qui marche une seule fois. Je pose un cristal sur un piédestal
→ light verte/rouge OK, comportement attendu. Je re-grab le cristal
→ il revient en main. Je tente de le re-poser sur le même piédestal
(ou de poser n'importe quel autre cristal dessus) → **rien ne se
passe** : pas d'attach, pas de force ungrab, pas de feedback
lumineux. Le piédestal est mort après un seul cycle d'usage.

**Diagnostic :**
J'ai inspecté `BP_Item_CrystalPedestre.Event Graph`.

Côté `OnComponentBeginOverlap (Sphere)`, le tout premier node après
l'event est un `Is Valid (CrystalOnPiedestral)` qui continue
uniquement sur la branche **Is Not Valid**. Donc tout le pipeline de
placement (Cast → Attach → ForceUngrab → check type → light →
counter) ne s'exécute que si la variable `CrystalOnPiedestral` est
null. Sinon BeginOverlap retourne sans rien faire.

À la fin du pipeline, la branche True/False (Good/Wrong) finit par
un `SET CrystalOnPiedestral = <le cristal qui arrive>` puis un
`SET AlreadyHasACrystalOn = true`.

J'ai ensuite cherché où ces variables étaient remises à None/false.
Un `OnComponentEndOverlap (Sphere)` existait déjà avec :

- Gate `Other Actor == CrystalOnPiedestral` (pour ne réagir qu'au
  cristal réellement posé, pas à un autre qui passerait)
- Cast vers `BP_Item_Crystals` puis comparaison
  `Crystal Type == Feeling Type`
- Si Good : décrément de `First Enigma Total Valid` côté joueur +
  reset de la light + reset du material `SM_Activated` →
  `BasicShapeMat`
- Si Wrong : reset light + reset material
- `SET AlreadyHasACrystalOn = false` à la toute fin

**Tout ce cleanup est là - sauf le `SET CrystalOnPiedestral = None`
qui est le SET critique** (celui sur lequel s'appuie le gate de
BeginOverlap). C'est exactement le profil de sabotage par omission
déjà vu sur Bug 5 (cleanup partiel sans `SET HoldingObject = None`)
et sur Bug 6 (chaîne complète sauf `Call EnigmaIsValid` final).
Pattern : *"laisse 95 % du code propre pour camoufler l'omission
du SET final"*.

**Cause :**
Dans `OnComponentEndOverlap`, le `SET CrystalOnPiedestral = None`
était absent. Quand le joueur re-grab le cristal et quitte la
sphère, la référence reste valide dans la variable (le cristal
n'est pas détruit, juste détaché et attaché ailleurs) → `IsValid`
retourne toujours `true` au prochain `BeginOverlap` → tout le
pipeline de placement est skippé.

**Correction :**
Dans `BP_Item_CrystalPedestre.OnComponentEndOverlap`, en bout de
chaîne, juste après `SET AlreadyHasACrystalOn = false`, ajout d'un
node `SET CrystalOnPiedestral` avec le pin Object **laissé vide**
(= None). Position en bout de chaîne : exécuté dans tous les cas
(Good ou Wrong), le clear de la référence n'a pas à être
conditionnel. Compile + Save.

**Vérification :**
Cycle complet en PIE :

- Pose un cristal Sadness sur piédestal Sadness → light verte ✓
- Re-grab le cristal → light revient au défaut, material revient
  sur `BasicShapeMat` ✓
- Re-pose le même cristal → light verte de nouveau ✓
- Re-grab → pose un cristal Joy (mauvais type) → light rouge ✓
- Re-grab → re-pose un Sadness → light verte ✓

Le compteur côté joueur reste cohérent (incrémenté sur Good,
décrémenté sur retrait de Good, jamais touché par les Wrong). ✅

**Notes pour l'oral :**
Troisième occurrence du pattern *"sabotage par omission de node
final"* dans le projet (après Bug 5 et Bug 6). À chaque fois, la
chaîne logique semble propre à la relecture : variables déclarées,
events câblés, conditions vérifiées, prints en place. Le saboteur
retire UN node - le plus en aval, celui qui propage l'état final  - 
pour casser l'invariant sans dégrader la lisibilité du graph. À
l'examen visuel rapide, rien ne saute aux yeux.

Heuristique de détection issue de ces trois bugs : *"si une
variable booléenne ou une référence est SET quelque part, chercher
systématiquement le SET inverse (false / None) et vérifier qu'il
existe ET qu'il est sur le bon chemin d'exécution"*.



## Feature ajoutée

Voir [`feature-design.md`](feature-design.md) : conception, scope et suivi du puzzle Simon.

## Notes diverses

*(astuces, fausses pistes, points à surveiller)*
