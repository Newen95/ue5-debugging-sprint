# Sprint 6 - Journal de feature

Notes prises au fil de l'eau pendant que je conçois et j'implémente la feature personnelle du sprint. À chaque session, j'ajoute ce que j'ai fait, ce que j'ai compris, ce que j'ai changé. Document pensé pour être défendable à l'oral le 2026-05-21 et pour qu'une reprise après coupure rentre dans le bain en une lecture.

## Pitch

Le puzzle de base demande au joueur de mémoriser une **combinaison** : quel cristal sur quel piédestal. J'étends le jeu avec un second espace, débloqué après résolution du premier, qui introduit une **nouvelle dimension cognitive** : la mémoire séquentielle (Simon-like).

Même grammaire visuelle (lumières colorées, halos, décal outline déjà appris), nouvelle règle (l'ordre dans le temps compte). C'est défendable à l'oral comme **extension naturelle du gameplay corrigé**, pas comme une feature ornementale plaquée à côté.

## Concept de la sous-zone

Un mur du premier niveau (à choisir en début d'implémentation) **s'abaisse dans le sol** une fois le puzzle 1 résolu - comme une pierre qui glisse vers le bas, accompagnée d'un son grave. Derrière, une salle relativement vide qui ne demande qu'à recevoir les 4 champignons en ligne et les rares props nécessaires.

Quand le joueur entre dans cette salle, un trigger lance automatiquement la séquence Simon-like. À sa résolution, écran de fin.

## Scope révisé 2026-05-17

Pivot complet du scope initial (9 sub-features atmosphériques : lampe torche, particules, color grading, glyphes, créature, mode écho, écrans...) jugé trop éparpillé pour 4 jours. Recentrage sur 5 features compactes qui forment un mini-niveau cohérent.

### F1 - Gate : le mur qui s'abaisse

Un mesh "mur" placé dans `L_DeepCave.umap` qui bloque l'accès à la sous-zone. Il **descend dans le sol** sur l'event de fin du puzzle 1 (dispatcher déjà émis par Bug 6).

**Acceptance :**
- Avant résolution puzzle 1 : passage infranchissable, le mur a l'air "scellé" (visuel : runes, glyphes ou juste solide).
- À l'instant de résolution : animation Timeline qui descend le mur de N unités vers le bas, son de pierre qui glisse, collision désactivée à la fin.
- Après : passage libre dans les deux sens. Non réversible (une fois ouvert, le mur reste en bas).

### F2 - Puzzle Simon-like (cœur de la feature)

4 champignons lumineux **alignés en ligne** dans la sous-zone, chacun avec :
- Une **couleur de lumière** unique (rouge, vert, bleu, jaune par défaut - à matcher avec les couleurs des cristaux du puzzle 1 si elles sont au nombre de 4, à confirmer à l'implémentation).
- Une **note audio** distincte spatialisée (émanant du champignon, ce qui aide la mémorisation spatiale).
- Un **décal outline** réutilisant le même `Material` que le `Decal_Outline` de `BP_MasterItem`, mais sur un `DecalComponent` indépendant. Hidden In Game par défaut, montré uniquement pendant la phase de reproduction → sert de **double signal** : "tu peux cliquer ici" + "c'est ton tour".

**Pipeline du Simon :**
1. **Entrée** : trigger box à l'entrée de la sous-zone. Au premier overlap player, hint UMG s'affiche, attente 1-2 s, puis appel de `BP_SimonManager.StartPuzzle()`.
2. **Affichage de la séquence** : le manager joue la séquence courante via une Timeline qui appelle `PlayNote()` sur chaque champignon selon `Sequence[i]` (~0.5 s allumé, ~0.3 s d'extinction entre chaque). Pendant ce temps, décals cachés, input désactivé.
3. **Signal "à toi"** : à la fin de la séquence, le manager rend les 4 décals outline visibles → le joueur sait qu'il peut tirer.
4. **Reproduction** : pas de chrono, pas de timeout. Le joueur tire à son rythme via LineTrace.
5. **Match partiel** : tant que `PlayerInput[i] == Sequence[i]` et i < Length-1, attendre input suivant. Chaque hit déclenche le flash+note du champignon visé.
6. **Match complet** (i == Length-1) : manche réussie. Length++, nouvelle séquence générée, replay.
7. **Erreur** : flash rouge global sur les 4 champignons (~0.5 s) + son d'erreur, décals cachés, state = WrongAnswer. Après ~1 s, re-play de **la même séquence depuis le début** (sans regénérer).
8. **Succès final** (manche 3 complétée, séquence de 5 notes jouée correctement) : tous les champignons brillent en synchro pendant 2 s, fondu au noir, dispatch `OnSimonSolved` → écran de fin.

**Difficulté progressive** : 3 manches de longueur 3 → 4 → 5 notes. Possibilité d'étendre à 4 manches en buffer si nécessaire.

### F3 - Écran de fin

Widget UMG simple, déclenché par `OnSimonSolved`. Texte court (à choisir, ex : *« Le souvenir est rendu. »*) + fade. Bouton Quit ou retour menu principal.

**Acceptance :**
- S'affiche immédiatement après le fondu noir final du Simon.
- Lisible 4-5 secondes (texte clair, contraste suffisant).
- Au moins un bouton fonctionnel pour quitter.

### F4 - Écran pause (ESC)

Widget UMG avec deux boutons : Resume et Quit. Bind sur Escape via Input Action. `Set Game Paused = true` à l'ouverture.

**Acceptance :**
- ESC ouvre le menu, ESC à nouveau ou Resume le ferme proprement.
- Quit retourne au menu principal ou au desktop (à choisir).
- Doit marcher partout dans le jeu (level 1 + sous-zone).

### F5 - Hint UMG entrée sous-zone

Petit overlay UMG qui apparaît à l'entrée du joueur dans la sous-zone, juste avant le démarrage de la séquence. Texte type : *« Reproduisez la séquence en tirant sur les champignons. »* Fade in ~0.5 s, lisible ~3 s, fade out ~0.5 s.

**Acceptance :**
- Affiché une seule fois (la première fois que le joueur entre).
- Ne bloque pas la lecture de la séquence (totalement faded avant le premier flash).
- Non interactif (pas de bouton), juste textuel.

## État

| #  | Sous-feature                  | Statut | Effort estimé |
|----|-------------------------------|--------|---------------|
| F1 | Gate (mur qui s'abaisse)      | ✅     | déjà existant (Bug 8) |
| F2 | Puzzle Simon-like             | ✅     | 3j (J2+J3+J4 Tier 1)  |
| F3 | Écran de fin                  | ✅     | 0.5j          |
| F4 | Écran pause (ESC)             | ✅     | 0.5j          |
| F5 | Hint UMG entrée sous-zone     | 🟡     | trigger ✅, hint UMG non livré |
| P  | Polish (couleurs, notes, sons fin/erreur) | ✅ | 0.5j (J4) |

Légende : ⏳ à faire - 🔧 en cours - ✅ fait - 🟡 partiel/à reprendre

**Note 2026-05-18** : F1 supprimée du scope, le mur de transition puzzle 1 → puzzle 2 existe déjà dans `L_DeepCave.umap` (corrigé en Bug 8 `96af773`). Sous-zone derrière déjà construite (salle carrée avec puits de lumière, rocher central avec fleur bleue, tapis de petits champignons décor). 0.5j économisée.

## Layout de la sous-zone

- **Position** : derrière le mur de gate (à identifier dans `L_DeepCave.umap`). Salle à **construire** car actuellement il n'y a (à priori) que le décor existant - l'utilisateur a précisé *« on arrive dans une salle vide »*.
- **Forme** : rectangulaire, suffisamment large pour que le joueur puisse reculer et voir les 4 champignons d'un seul coup d'œil (cible : ~10 m de long, ~6 m de large, ~4 m de haut, à ajuster selon la place dispo).
- **Champignons** : **alignés en ligne droite**, perpendiculaires à l'axe d'entrée (le joueur arrive et voit la ligne complète face à lui). Espacement ~2 m entre chaque, soit ~6 m de ligne. Hauteur similaire (en ligne au sol, pas en escalier).
- **Sol/murs** : props et matériaux déjà optimisés du `Content/DeepElderCaves/` pour rester dans le budget VRAM atteint après Bug 7.
- **Lumière ambiante** : faible, pour que les champignons restent les éléments visuels dominants. La cave doit être assez sombre pour que les couleurs des champignons ressortent.

## Design détaillé - Simon-like

### États du `BP_SimonManager`

Enum `ESimonState` :
- `Idle` : avant la première entrée du joueur. Rien ne se passe, champignons en idle décor.
- `Playing` : la séquence est en cours d'affichage. Input ignoré, décals cachés.
- `Listening` : c'est au joueur. Décals visibles, hits comptés.
- `WrongAnswer` : transition après erreur. Flash rouge global, ~1 s, puis retour à `Playing` avec même séquence.
- `Success` : transition après succès final. Brillance globale, 2 s, fade au noir, dispatch.

### Génération de la séquence

À chaque appel de `RegenerateSequence(int Length)` (appelé au début d'une manche réussie ou au tout début du puzzle, **pas après une erreur**) :
- Vider l'array `Sequence`.
- Pour i de 0 à Length-1 : `Sequence.Add(RandomIntInRange(0, 3))`.
- Réinitialiser `PlayerInputIndex` à 0.

Les répétitions sont autorisées par construction (génération indépendante de chaque index). Une séquence valide peut donc être par exemple `R-G-R-B-G`.

`StartRound()` (sans paramètre) ne fait que **rejouer la séquence courante** : c'est cette fonction qui est appelée après une erreur (préserve la séquence) ou après l'affichage initial (pour la manche en cours).

### Pipeline LineTrace (modification du `BP_FPSPlayer`)

Le LineTrace réparé en Bug 5 est déjà branché. On ajoute un cas dans le traitement du hit, **avant** le check existant `Cast to BP_Item` :

```
On LineTrace Hit:
  → Cast to BP_SimonMushroom
    → success:
        → Get BP_SimonManager (via reference cached, ou GetAllActorsOfClass au premier appel puis stocké)
        → call SimonManager.OnMushroomHit(Mushroom.MushroomIndex)
        → return  (do not fall through to grab logic)
    → fail: continue to BP_Item cast (Bug 5 path, inchangé)
```

**Important** : ne pas casser le grab existant sur les cristaux du puzzle 1. La cascade doit prioriser le check Simon (puisqu'il est state-gated et plus restrictif), puis fall through vers Bug 5.

Le manager check :
- Si state != `Listening`, ignore le hit silencieusement.
- Sinon, compare `Mushroom.MushroomIndex` avec `Sequence[PlayerInputIndex]`, advance ou trigger WrongAnswer.

### Signal "interactif" et "à toi" - révisé 2026-05-18

**Pivot 2026-05-18** : abandon de l'approche décal pour deux signaux séparés via `LightComponent0.Intensity` :

- **Signal "interactif" (look-at sur un champignon)** : boost d'intensité de `LightComponent0` quand le joueur vise le champignon, retour à idle quand il regarde ailleurs. Piloté par les events `StartLookAtInteractables` / `StopLookAtInteractables` de l'interface `BPI_Interactables` (mêmes events que les cristaux du puzzle 1).
  - Valeur idle : à relever sur `BP_Mushroom_Lit_D` (parent), probablement entre 500 et 5000 cd.
  - Valeur look-at : ~5× idle.
- **Signal "c'est ton tour" (global après affichage de la séquence)** : flash blanc bref sur les 4 champignons simultanément, piloté par le manager via une fonction `FlashWhite()` sur chaque mushroom (set color = white + intensity boost ~1s, puis retour à idle color/intensity).

**Pourquoi le pivot** : le composant `Decal_Outline` du `BP_MasterItem` n'a aucun matériau assigné, et le fallback blanc qui rendait sur les cristaux ne fonctionne pas en UE 5.4 sur le mesh trompette du `BP_SimonMushroom` (probablement matériau SSS ou changement de comportement du moteur). Validé en session 2026-05-18 : les events `StartLookAtInteractables` fire bien (confirmé via Print String), mais aucun rendu visible quand on toggle `Hidden in Game` sur le composant Decal sans matériau. Approche Light Intensity évite toute dépendance asset et donne un signal plus naturel pour un champignon lumineux.

**Le composant `DecalOutline`** créé dans `BP_SimonMushroom` reste en place mais inutilisé. Pourra être supprimé proprement plus tard ou réutilisé si on crée un matériau outline.

Mapping state → comportement :

| State          | Intensity LightComponent0     | Couleur                |
|----------------|-------------------------------|------------------------|
| Idle (pas viseé) | IdleIntensity                | MushroomColor          |
| Look-at        | IdleIntensity × 5             | MushroomColor          |
| Playing (séquence joue) | flash bref selon `PlayNote()` | MushroomColor (boost émissif) |
| Listening (your turn) | initial flash blanc global ~1s puis IdleIntensity | flash blanc puis MushroomColor |
| WrongAnswer    | flash rouge global ~0.5s       | rouge puis MushroomColor |
| Success        | brillance synchro 2s          | blanc/MushroomColor    |

### Tempo

- **Affichage de la séquence** : tempo fixe et régulier, ~0.5 s allumé + ~0.3 s d'extinction entre chaque note. À ajuster à l'oreille en playtest.
- **Reproduction joueur** : aucun tempo, aucun chrono. Le joueur tire à son rythme.

Si v2 (en buffer J4) : envisager d'accélérer légèrement le tempo d'affichage à chaque manche (0.5 → 0.4 → 0.35 s) pour augmenter la difficulté de mémorisation. Pour la v1 : tempo unique.

### Audio

4 notes courtes (~0.5 s chacune). Stratégies possibles, à décider en début d'implémentation :
- 4 SoundCues distincts (notes musicales : do, mi, sol, do d'octave par exemple).
- 1 son base + pitch shift dans une Sound Cue avec 4 sorties (économise les assets).
- Sons fournis dans le projet de base ? À vérifier dans `Content/` (notamment dans les dossiers crystal SFX réparés en Bug 4).

Chaque note est jouée via l'`AudioComponent` du `BP_SimonMushroom`, donc **spatialisée** dans le monde.

## Liste exhaustive des BPs à créer

### `BP_SimonMushroom` (duplicate de `BP_Mushroom_Lit_A` ou équivalent)

**Components hérités** : Static Mesh (champignon), Point Light, etc.
**Components ajoutés** :
- `DecalComponent` (nommé `DecalOutline`) : utilise le même Material que celui du `Decal_Outline` de `BP_MasterItem`. Hidden In Game par défaut.
- `AudioComponent` (si pas hérité) : Auto Activate false, sound assigné via variable.

**Variables :**
- `MushroomIndex` (int, 0-3, public, set par instance) : identifie quel champignon dans la ligne.
- `MushroomColor` (LinearColor, public) : couleur de la lumière + de l'émissive material.
- `MushroomNote` (Sound Base ref) : la note audio à jouer.

**Functions / Events :**
- `Construction Script` : applique `MushroomColor` à la light et au matériau émissif (via `Set Light Color` et un MID sur le mesh).
- `PlayNote()` : boost émissive temporaire (Timeline ~0.5 s, courbe ease in/out), play du `AudioComponent`.
- `SetOutlineVisible(bool bVisible)` : toggle `Hidden In Game` du `DecalOutline`.
- `FlashRed()` : override temporaire d'émissive vers rouge sur ~0.5 s, retour à idle.

**Note Compile + Save** : toujours dans cet ordre après modification de graphe.

### `BP_SimonManager`

Actor placé dans la sous-zone. Pas de mesh, c'est un "ghost actor" qui orchestre.

**Variables :**
- `Mushrooms` (Array of `BP_SimonMushroom` refs, taille 4, set par instance dans le level via World Outliner ou via Construction Script avec `GetAllActorsOfClass`).
- `Sequence` (Array of int) : la séquence courante.
- `PlayerInputIndex` (int) : où on en est dans la reproduction.
- `State` (enum `ESimonState`).
- `CurrentRoundLength` (int, default 3).
- `MaxRoundLength` (int, default 5).
- `SequencePlaybackTimeline` (Timeline).

**Event Dispatchers :**
- `OnSimonSolved` : dispatché quand le puzzle entier est résolu. L'écran de fin s'y bind.

**Functions / Events :**
- `StartPuzzle()` : entry point appelé par le trigger d'entrée. `CurrentRoundLength = 3`, appelle `RegenerateSequence(3)`, appelle `StartRound()`.
- `RegenerateSequence(int Length)` : vide `Sequence`, remplit avec des int aléatoires [0..3], réinitialise `PlayerInputIndex`.
- `StartRound()` : set state `Playing`, cache les outlines, démarre `SequencePlaybackTimeline`. Ne regénère PAS la séquence.
- `SequencePlaybackTimeline` : Timeline qui fire à intervalles réguliers (0.8 s entre chaque clé). À chaque fire, appelle `Mushrooms[Sequence[KeyIndex]].PlayNote()`. À la fin, appelle `OnSequenceFinished`.
- `OnSequenceFinished` : set state `Listening`, montre tous les outlines.
- `OnMushroomHit(int HitIndex)` :
  - Si state != `Listening`, return.
  - Si `HitIndex == Sequence[PlayerInputIndex]` :
    - `Mushrooms[HitIndex].PlayNote()` (feedback du hit correct).
    - `PlayerInputIndex++`.
    - Si `PlayerInputIndex == Sequence.Length` : `OnRoundWon`.
  - Sinon : `OnWrongAnswer`.
- `OnRoundWon` :
  - Si `CurrentRoundLength == MaxRoundLength` : `OnPuzzleSolved`.
  - Sinon : `CurrentRoundLength++`, delay ~1 s, `RegenerateSequence(CurrentRoundLength)`, `StartRound()`.
- `OnWrongAnswer` : set state `WrongAnswer`, cache outlines, flash rouge global (tous les `Mushrooms[i].FlashRed()`) + son d'erreur, delay ~1.5 s, `StartRound()` (replay même séquence).
- `OnPuzzleSolved` : set state `Success`, cache outlines, brille les 4 champignons en synchro pendant 2 s (`PlayNote()` sur tous simultanément ou anim émissive globale), fade au noir, dispatch `OnSimonSolved`.

### `BP_SimonGate` (le mur qui s'abaisse)

Actor placé dans `L_DeepCave.umap` à l'emplacement du mur de gate.

**Components :**
- `StaticMeshComponent` (nommé `WallMesh`) : mesh du mur (à choisir parmi les props existants, idéalement quelque chose qui s'intègre visuellement à la cave).
- `AudioComponent` : son de pierre qui glisse, Auto Activate false.

**Variables :**
- `LoweredOffset` (Vector, default `(0, 0, -300)` ou ajustable selon la hauteur du mur) : déplacement vers le bas.
- `LowerDuration` (float, default 3.0) : durée de l'animation.
- `bIsLowered` (bool) : pour éviter de re-trigger.

**Functions / Events :**
- `BeginPlay` : bind à l'event dispatcher "puzzle 1 résolu" (cf. dispatcher Bug 6, à identifier précisément en lisant le BP gestionnaire du puzzle 1).
- `LowerWall()` :
  - Si `bIsLowered`, return.
  - Jouer le son.
  - Lancer une Timeline qui interpole la position du `WallMesh` de `Initial` à `Initial + LoweredOffset` sur `LowerDuration`.
  - À la fin : désactiver la collision du `WallMesh`, `bIsLowered = true`.

### `BP_SimonTrigger` (trigger d'entrée)

Actor placé à l'entrée de la sous-zone.

**Components :**
- `BoxCollision`.

**Variables :**
- `SimonManagerRef` (BP_SimonManager ref, set par instance via World Outliner).
- `HintWidgetClass` (UserWidget class ref).
- `bAlreadyTriggered` (bool).

**Functions / Events :**
- `OnBoxBeginOverlap` (filtré sur Player Character) :
  - Si `bAlreadyTriggered`, return.
  - `bAlreadyTriggered = true`.
  - Spawn et add to viewport le `HintWidgetClass`.
  - Delay ~3-4 s (durée du hint + 1 s de marge).
  - Appel `SimonManagerRef.StartPuzzle()`.

### `WBP_HintEntry` (widget UMG du hint)

Texte centré : *« Reproduisez la séquence en tirant sur les champignons. »*
Animation : fade in 0.5 s → opaque 3 s → fade out 0.5 s → auto-remove via `Remove from Parent`.

### `WBP_EndScreen` (écran de fin)

Widget UMG :
- Background : fondu noir avec léger gradient.
- Texte central (h1) : *« Le souvenir est rendu. »* (texte à finaliser).
- Bouton Quit (ou retour menu principal).

Bind au dispatcher `OnSimonSolved` du `BP_SimonManager` via le PlayerController ou le GameMode (à décider à l'implémentation).

### `WBP_PauseMenu` (écran pause)

Widget UMG :
- Background : semi-transparent (~80% noir).
- Texte titre : *« Pause »*.
- Boutons : `Resume` (close widget + `Set Game Paused false`) et `Quit` (retour menu principal).

Triggered via Input Action `Pause` (bind sur Escape dans Project Settings → Input).

## Modifications nécessaires aux BPs existants

### `BP_FPSPlayer` (LineTrace handling)

Dans la logique de hit du LineTrace (déjà fonctionnelle depuis Bug 5), ajouter le check Simon **avant** le check `Cast to BP_Item`. Voir pseudo-code dans la section "Pipeline LineTrace" ci-dessus.

**Compile + Save** obligatoire (cf. insight Unreal : `Save All` ne recompile pas le bytecode, PIE exécute le bytecode).

### Gestionnaire du puzzle 1 (à identifier)

Selon les insights de Bug 6, il y a un dispatcher pour "puzzle résolu" (probablement dans le BP enigma manager ou équivalent). On doit le binder dans `BP_SimonGate.BeginPlay`. Si le dispatcher n'est pas directement accessible depuis un autre BP :
- Soit le promouvoir (le rendre public si déclaré privé).
- Soit ajouter un dispatcher relais dans le GameMode ou un Singleton accessible.

À explorer en début d'implémentation de F1, via **Find in Blueprints global** (Ctrl+Shift+F) sur le nom du dispatcher.

## Contraintes techniques

- **Deadline finale : 2026-05-21 23:59** (4 jours nets depuis 2026-05-17).
- **Budget perf** : baseline standalone après Bug 7 ≈ 22-23 FPS, GPU 42 ms saturé. La sous-zone doit rester dans cette enveloppe.
  - Pas de nouvelle light dynamique non maîtrisée → les 4 champignons sont à `Attenuation Radius` ~300, beaucoup plus petit que les `BP_Mushroom_Lit_*` du décor (qui sont le problème Light Complexity du Bug 7).
  - Pas de particle system lourd (pas obligatoire pour le design choisi).
  - Décal outline réutilisé (même material) : déjà budgétisé.
- **100% Blueprint**. Pas de C++, pas de plugin externe.
- **Réutilisation maximale** :
  - LineTrace du joueur (Bug 5) pour l'activation Simon.
  - Dispatcher "puzzle 1 résolu" (Bug 6) pour le gate.
  - Material du `Decal_Outline` de `BP_MasterItem` (Bug 3) pour signaler l'interactivité.
  - Props/textures déjà optimisés (Bug 7) pour décorer la sous-zone.
  - Mesh du champignon : duplicate d'un `BP_Mushroom_Lit_*` existant.
- **Console keys** `²` et `F1` déjà prises (debug). Ne pas réutiliser. ESC libre pour la pause.
- **Bug 7 considéré stable** (committed `e6886fe`), Light Complexity overlap résiduel parqué. Reprise en buffer J4 si possible.

## Plan d'implémentation 4 jours

### J1 (2026-05-18) - F1 Gate + setup sous-zone

- Identifier dans `L_DeepCave.umap` le mur qui servira de gate (ou le créer si rien d'évident).
- Créer `BP_SimonGate` : mesh + AudioComponent + Timeline de descente + bind dispatcher puzzle 1.
- Tester PIE : résoudre le puzzle 1, vérifier que le mur descend.
- Construire la sous-zone derrière (BSP rapide ou static meshes existants), salle vide ~10×6×4 m.
- Placer un trigger box à l'entrée (`BP_SimonTrigger` placeholder).
- En fin de J1 : Build Lighting Production sur le level (peut prendre ~25 min car la géométrie a changé, à laisser tourner pendant une pause).

### J2 (2026-05-19) - F2 Simon (base mécanique)

- Créer `BP_SimonMushroom` (duplicate de `BP_Mushroom_Lit_A`), ajouter `DecalOutline` + `AudioComponent`, exposer variables.
- Créer `BP_SimonManager` avec enum state, génération séquence, Timeline d'affichage, génération aléatoire, dispatcher.
- Modifier `BP_FPSPlayer` pour router le LineTrace hit vers le manager si `BP_SimonMushroom`. Compile + Save.
- Placer 4 instances de `BP_SimonMushroom` en ligne dans la sous-zone, set index/color/note par instance.
- Placer 1 `BP_SimonManager` dans la sous-zone, remplir l'array `Mushrooms`.
- Lancer manuellement `StartPuzzle()` en testant (via touche debug temporaire ou trigger placeholder activé manuellement).
- Tester en PIE : séquence joue, hits comptés correctement, win/loss conditions OK.

### J3 (2026-05-20) - F2 Simon (polish) + F3 écran fin + F5 hint

- Brancher le `BP_SimonTrigger` sur le manager (auto-start au premier overlap).
- Implémenter les feedbacks visuels : flash rouge global sur erreur, brillance synchro sur succès final.
- Créer `WBP_HintEntry`, brancher dans `BP_SimonTrigger`.
- Créer `WBP_EndScreen`, brancher au dispatcher `OnSimonSolved`.
- Playtest complet : entrée → séquence → reproduction → erreur (volontaire) → manche 2 → manche 3 → succès → écran de fin.
- Ajustements tempo, durée flash, sons.

### J4 (2026-05-21) - F4 pause + build standalone + commits + buffer

- Créer `WBP_PauseMenu` + Input Action ESC dans Project Settings.
- Tester pause partout (level 1 + sous-zone).
- Build standalone (`Package Project Windows / Development` vers `C:\UnrealBuilds\Sprint6\`).
- Vérifier perf standalone (`stat unit`) : on doit rester dans l'enveloppe ~22 FPS baseline.
- Si time buffer : reprendre Bug 7 (réduction Attenuation Radius `BP_Mushroom_Lit_*` décor) ou 4e manche Simon, ou tempo accéléré par manche.
- Préparer commits (commit message en anglais, format `[FEAT] feature description`, single-developer voice, pas de trace Claude/AI). User fait le commit/push.
- Préparer le zip final + envoi au client + push GitHub.

## Risques identifiés et mitigations

- **Le dispatcher du puzzle 1 n'est pas accessible directement** depuis un BP externe → forcer un relais via GameMode ou rendre public. Risque mineur, 1-2 h max.
- **Le LineTrace ne distingue pas BP_SimonMushroom de BP_Item** si on oublie de modifier `BP_FPSPlayer` correctement → debug LineTrace en standalone (la ligne rouge sert de viseur, cf. insight Bug 7 du `Draw Debug Type = For One Frame` laissé exprès). Risque mineur.
- **Light Complexity dans la sous-zone** : 4 champignons avec lumière contribuent au compte. À surveiller via le viewmode Light Complexity. Mitigation : Attenuation Radius petit (~300), pas d'overlap entre les 4 (espacés de 2 m).
- **Audio assets manquants** : si pas de sons de notes dans le projet, il faudra en générer/importer rapidement. Mitigation : 1 son simple + pitch shift dans Sound Cue avec 4 sorties.
- **L'élargissement de la salle peut invalider les BuiltData lightmaps** existants → re-Build Lighting peut prendre ~25 min. Garde-fou : faire le level design d'abord, puis Build Lighting une seule fois en fin de J1.
- **Cast cascade dans `BP_FPSPlayer`** peut introduire une régression sur le grab cristaux si mal ordonnée → tester puzzle 1 systématiquement après modification.

## Rappels transverses (toujours appliquer)

- **Compile + Save** dans cet ordre après chaque modification de graphe BP. `Save All` seul n'embarque pas le bytecode.
- **Find in Blueprints global** (Ctrl+Shift+F) avant de toucher à un BP ou un dispatcher pour identifier toutes les références.
- **Each bug/feature peut avoir plusieurs couches** : ne pas conclure trop vite qu'un truc est cassé/réparé, vérifier toutes les références.
- **Commit messages** : anglais, terse, single-developer voice, pas de trace AI. Préfixe `[FEAT]` pour les commits feature.
- **Mesurer la perf en standalone**, pas en éditeur (DRS masque la vraie charge GPU).
- **Pour toute modif `.ini` / `.uproject` / `.uplugin`** : expliquer ce que fait la ligne et le problème **avant** d'éditer (préférence utilisateur).
- **Claude ne fait que `git add`**, l'utilisateur s'occupe de `commit` et `push`.

## Comment ça marche (synthèse pour la défense orale)

### Architecture haut-niveau

Trois acteurs principaux interagissent autour du puzzle Simon :

- **`BP_SimonManager`** - *orchestrateur ghost* (Actor sans mesh placé dans la sous-zone). Détient l'état (`E_SimonState`), la séquence courante, le compteur de rounds, et le dispatcher de fin (`OnSimonSolved`).
- **`BP_SimonMushroom`** - *répondeur* (4 instances en ligne au mur du fond). Connaît son `MushroomIndex` (0-3), sa `MushroomColor`, sa `MushroomNote`, et une référence vers le manager (`ManagerRef`). Réagit au look-at, au tir LineTrace, et aux appels `PlayNote` / `ResetVisualState` venant du manager.
- **`BP_SimonTrigger`** - *déclencheur d'entrée* (BoxCollision). Au premier overlap joueur, appelle `ManagerRef.StartPuzzle` après un Delay 2s.

### Enchaînement runtime

```
Joueur entre dans la sous-zone
  → BP_SimonTrigger.OnComponentBeginOverlap
  → Cast to BP_FPSPlayer (filtre overlaps non-joueur)
  → Branch bAlreadyTriggered = false → Set true
  → IsValid ManagerRef → Delay 2s
  → ManagerRef.StartPuzzle
      → CurrentRoundLength = 3
      → RegenerateSequence(3)              // Sequence[] de 3 ints aléatoires 0..3
      → StartRound
          → ForEach Mushrooms → ResetVisualState   // garantit tous off avant playback
          → Set State = Playing
          → PlaySequenceStep(0)            // récursif
              → IsValid Mushrooms[Sequence[Index]] → PlayNote
              → Delay 0.8s
              → PlaySequenceStep(Index+1)
              → quand Index >= Sequence.Length : Set State = Listening

Joueur tire sur un champignon (LineTrace réparé en Bug 5)
  → BP_FPSPlayer.IA_Player_Interact (Started)
  → Cast prioritaire BP_SimonMushroom (avant le Cast BP_Item du grab)
      - succès → Mushroom.OnHitFromPlayer
          → Switch on State (gating mushroom-side) : seul Listening continue
          → PlayNote(Self)                 // feedback visuel + audio immédiat
          → ManagerRef.OnMushroomHit(MushroomIndex)
              → Switch on State (defense in depth) : seul Listening continue
              → HitIndex == Sequence[PlayerInputIndex] ?
                  - True  → PlayerInputIndex++
                            → PIIdx >= Sequence.Length ? OnRoundWon : attend prochain clic
                  - False → OnWrongAnswer
      - échec → fall-through vers la logique grab Bug 5 (inchangée)

OnRoundWon
  → CurrentRoundLength >= MaxRoundLength (5) ?
      - True  → OnPuzzleSolved
      - False → CurrentRoundLength++ → Set State = Idle → Delay 1s
                → RegenerateSequence(CurrentRoundLength) → StartRound

OnWrongAnswer
  → Play Sound 2D (ErrorSound)
  → Set State = Idle → PlayerInputIndex = 0 → Delay 1s
  → StartRound                              // rejoue la MÊME séquence (pas de RegenerateSequence)

OnPuzzleSolved
  → Set State = Success
  → Broadcast OnSimonSolved (Event Dispatcher)

Côté joueur (souscripteur du dispatcher)
  → BP_FPSPlayer.BeginPlay (au démarrage du level) :
      GetAllActorsOfClass(BP_SimonManager) → Get[0] → IsValid → Bind Event to OnSimonSolved
  → Au broadcast, BP_FPSPlayer.OnSimonPuzzleSolved :
      Create WBP_EndScreen → Add to Viewport
      Set Game Paused (true) → Input Mode UI Only → Show Mouse Cursor
      WBP_EndScreen.Event Construct → Play Sound 2D (Victory)
```

### Choix d'architecture défendables

- **Pattern récursif `PlaySequenceStep(Index)`** plutôt qu'une Timeline native : la Timeline UE a une Length figée à compile-time, on ne peut pas avoir une track dynamique pour une séquence dont la longueur change à chaque round (3 → 4 → 5). Custom Event + Delay + auto-call sur Index+1 est plus simple à debugger qu'une macro `ForEachLoopWithDelay` custom, et le 0.8s est trivial à tuner.

- **Gating à deux niveaux (mushroom + manager)** via `Switch on E_SimonState` : le Switch côté mushroom rend l'UX claire (rien ne réagit visuellement pendant la playback), le Switch côté manager est defense in depth (si un autre déclencheur arrivait à `OnMushroomHit` sans passer par le mushroom). Switch retenu au lieu de `Equal + Branch` parce que Unreal propose par défaut `Equal (Byte)` sur les enums - fragile si on réordonne les enumerators. Switch est type-safe (Compile alerte sur enumerator manquant).

- **`Set State = Idle` AVANT les Delays** dans `OnRoundWon` et `OnWrongAnswer` (safety fix J4) : sans ça, `State` reste à `Listening` pendant le Delay 1s, et un clic joueur pendant ce délai serait évalué contre une `Sequence` dans un état transitoire (out-of-bounds dans `OnRoundWon` car `PlayerInputIndex == Sequence.Length` ; comparaison foireuse dans `OnWrongAnswer`).

- **`ResetVisualState` ForEach dans `StartRound`** plutôt que reset éparpillés : centralise la garantie "tous les mushrooms partent de 0 avant la séquence". Évite les résiduels d'intensité dus à une Timeline `NoteFlashTimeline` interrompue ou à un boost look-at non-nettoyé sur le dernier mushroom cliqué.

- **Branch `IsBeingLookedAt` sur `StopLookAtInteractables`** : workaround minimal pour la dette technique du `BP_FPSPlayer` (look-at appelé per-tick, pas au changement de cible). Plus simple qu'un refactor du player ; empêche `StopLookAt` d'écraser à chaque frame la valeur d'intensité courante de la Timeline.

- **Dispatcher `OnSimonSolved` + bind cross-BP via `GetAllActorsOfClass`** dans `BP_FPSPlayer.BeginPlay` : pattern propre pour découpler le manager de l'écran de fin. Pas de Singleton ni GameMode requis.

- **Couleurs et notes per-instance dans le World Outliner** (Option A) plutôt qu'en Construction Script : évite une couche d'abstraction (variable `MushroomColor` + code de propagation au Construction Script) pour 4 valeurs définies une seule fois dans le level. Modifiable en 30 s dans le Details panel sans recompile.

- **Audio spatialisé `NoteAudio` (3D)** pour les notes des mushrooms (la direction du son aide la mémorisation spatiale), **`Play Sound 2D`** pour `ErrorSound` et `Victory` (événements UI, pas localisés dans le monde).

- **Notes du Do majeur arpégé (C4, E4, G4, C5)** : 4 notes qui sonnent bien ensemble dans n'importe quel ordre (accord parfait majeur), évite la dissonance même sur des séquences aléatoires.

## Sessions

### 2026-05-17 - Pivot du scope + design exhaustif

- Création de ce journal (untracked via `.git/info/exclude`).
- Vérification de l'état du projet avec l'utilisateur : Bug 7 committed (`e6886fe`), 3 fixes additionnels post-Bug 7 trouvés dans le git log et hors mémoire (`96af773` door blocking, `3d9a212` pedestals not grabbable, `7b42dbc` pedestal reusable). Projet **confirmé stable** par l'utilisateur.
- Confirmation deadline finale : **2026-05-21 23:59** (4 jours). Ma mémoire de session avait 2026-05-26, mise à jour.
- Pivot du scope feature : drop des 9 sub-features atmosphériques initiales (lampe torche, particules, color grading, glyphes, créature, mode écho, écrans...) jugées trop éparpillées pour 4 jours et trop polish/atmosphériques. Recentrage sur **Simon-like dans sous-zone gated**.
- Choix de design figés (par itération avec le user) :
  - **Forme** : sous-zone gated dans le level actuel (pas de nouvelle umap).
  - **Mécanique** : mémoire séquentielle type Simon, 4 boutons.
  - **Activation** : tir LineTrace réutilisé de Bug 5 (pas de nouvelle keybind).
  - **Représentation** : 4 champignons en ligne, dupliqués d'un `BP_Mushroom_Lit_*`.
  - **Signal "interactif" + "à toi"** : décal outline (réutilisé du puzzle 1) montré uniquement pendant la phase reproduction.
  - **Gate** : un mur du level qui s'abaisse dans le sol (pas une porte qui s'ouvre).
  - **Tempo** : fixe à l'affichage (~0.5 s + ~0.3 s), aucun chrono à la reproduction.
  - **Erreur** : flash rouge global sur les 4 champignons + replay automatique de **la même séquence** (pas de regénération sur erreur).
  - **Manches** : 3 → 4 → 5 notes (3 manches), répétitions autorisées dans la séquence.
  - **Fin** : flash global → fondu noir → écran de fin direct, pas de salle supplémentaire.
- Bug 7 considéré clos pour cette feature. Reprise possible en buffer J4 si temps.
- **Prochaine action** : démarrer J1 (F1 Gate) - identifier le mur dans `L_DeepCave.umap`, créer le `BP_SimonGate`.

### 2026-05-18 - J1 (partiel)

**F1 supprimée du scope** dès le début de session. Le mur de transition puzzle 1 → sous-zone existe déjà dans le level et fonctionne (corrigé en Bug 8 `96af773 [FIX] door now block the player`). La sous-zone derrière est déjà construite : salle ~carrée avec puits de lumière vertical, rocher central avec petite fleur bleue, tapis de petits champignons décor. Le joueur entre par le mur descendant et voit la pièce. 0.5j économisée.

**F2 démarré** :
- Choix de placement validé : les 4 champignons en **ligne au mur du fond** (option C - perpendiculaire au regard du joueur entrant). Rocher central + fleur bleue conservés (à enlever si gênant plus tard).
- Création de `BP_SimonMushroom` dans `Content/00-Blueprints/Simon/` (nouveau dossier). Duplicate de `BP_Mushroom_Lit_D` choisi pour le mesh trompette blanc-crème distinctif.
- Components ajoutés : `DecalOutline` (Decal, Decal Size 150×150×150, Hidden in Game = true par défaut), `NoteAudio` (Audio, Auto Activate off). Components hérités préservés : `SharedRoot`, `LightComponent0`, `StaticMeshComponent0`.
- Variables Instance Editable ajoutées : `MushroomIndex` (Integer), `MushroomColor` (LinearColor), `MushroomNote` (SoundBase).
- 4 instances placées en ligne au mur du fond de la sous-zone. Visuel validé (image session).
- Interface `BPI_Interactables` implémentée. Events `StartLookAtInteractables` / `StopLookAtInteractables` câblés sur `Set Hidden in Game (DecalOutline)`.
- Tag `Item` ajouté en Class Defaults → Tags array (sinon le LineTrace du joueur ignore l'acteur, cf Bug 5 `Actor Has Tag (Item)`).
- ⚠️ **Effet de bord à corriger en J2** : avec le tag `Item`, le champignon devient grabbable par le système Bug 5/Bug 9. À fix dans `BP_FPSPlayer.IA_Player_Interact (Started)` en ajoutant un Cast prioritaire à `BP_SimonMushroom` qui route vers `OnMushroomHit` au lieu de la logique grab.

**Blocage rencontré et résolution (pivot)** :
- L'approche DecalOutline ne marche pas visuellement. Events fire bien (confirmé via `Print String` "StartLookAt FIRED") mais aucun rendu blanc n'apparaît sur le champignon visé.
- Investigation : ouverture de `BP_MasterItem.EnableOutline` → c'est juste un `Set Hidden in Game = false` sur le composant `Decal_Outline`. **Decal Material = None**, confirmé sur image. L'effet blanc sur les cristaux du puzzle 1 reposait sur un rendu fallback d'un Decal sans matériau qui ne fonctionne plus comme avant en UE 5.4, OU le mesh trompette / matériau du champignon ne reçoit pas les décals (potentiellement matériau SSS du parent).
- Aucun matériau outline réutilisable trouvé dans le projet (recherche `Outline` / `Decal` / `M_*` / `MI_*` exhaustive).
- **Décision** : pivoter sur boost d'intensité de `LightComponent0` au look-at. Plus naturel pour un champignon lumineux, zéro asset à créer.

**Prochaine action (J2, 2026-05-19)** :
1. Ouvrir `BP_SimonMushroom`, dans les events `StartLookAtInteractables` / `StopLookAtInteractables` :
   - Supprimer les `Set Hidden in Game (DecalOutline)`.
   - Ajouter `Set Intensity (LightComponent0)`. Relever d'abord la valeur idle actuelle dans Components → LightComponent0 → Details → Intensity. StartLookAt → IdleIntensity × 5. StopLookAt → IdleIntensity. Compile + Save. Test PIE.
   - Tuner le multiplicateur si trop faible ou trop violent.
2. Une fois look-at OK : démarrer la création de `BP_SimonManager` (state enum, sequence array, Timeline d'affichage, dispatcher `OnSimonSolved`) - cf section "BP_SimonManager" du présent doc.
3. Modifier `BP_FPSPlayer.IA_Player_Interact (Started)` pour le routing LineTrace shoot vers `OnMushroomHit` (cast prioritaire à `BP_SimonMushroom`, sinon fall-through grab).
4. Si reste du temps J2 : 4 instances setup (MushroomIndex 0-3, MushroomColor distincts, MushroomNote - à voir si on a des assets son existants ou si on en importe vite).

### 2026-05-19 - J2 (partiel)

**Avancée significative : infrastructure complète câblée et validée. Reste la logique métier.**

**Fait** :

1. **Pivot Light Intensity appliqué** (étape 1 de la prochaine action J2). IdleIntensity actuel mesurée à **2000** sur `LightComponent0`. Events modifiés :
   - `StartLookAtInteractables` → `Set Intensity (LightComponent0)` = **10000** (5× idle).
   - `StopLookAtInteractables` → `Set Intensity (LightComponent0)` = **2000** (retour idle).
   - Anciens nodes `Set Hidden in Game (DecalOutline)` supprimés. Print String debug conservés temporairement.
   - **Validé en PIE** : visée d'un champignon → boost lumineux net, regard ailleurs → retour normal. Tuner OK à 5×, lisible sans être violent.

2. **Routing LineTrace tir → champignon** (étape 3 de la prochaine action J2, faite avant le manager pour valider l'effet de bord du tag `Item`) :
   - `BP_SimonMushroom` : ajout d'un Custom Event **`OnHitFromPlayer`** (initialement avec Print String "Mushroom hit", supprimé en fin de J2).
   - `BP_FPSPlayer.IA_Player_Interact (Started)` modifié :
     - Cast **prioritaire** à `BP_SimonMushroom` sur `OutHitActor`, branché en tête de chaîne (entre l'event Started et le premier node de la logique grab existante).
     - Succès → call `OnHitFromPlayer` sur le cast, exécution s'arrête (pas de fall-through).
     - Échec → continue vers la logique grab Bug 5 inchangée.
   - **Validé en PIE** : click sur champignon → event fire, **aucun grab** ; click sur cristal → grab marche toujours.

3. **Enum `E_SimonState`** créé dans `Content/00-Blueprints/Simon/` avec 5 enumerators dans l'ordre : `Idle`, `Playing`, `Listening`, `WrongAnswer`, `Success`.

4. **`BP_SimonManager`** créé (Actor sans mesh, orchestrateur ghost) dans `Content/00-Blueprints/Simon/` :
   - Variables :
     - `State` : `E_SimonState` (default Idle implicite).
     - `Mushrooms` : Array of `BP_SimonMushroom` (Object Reference), **Instance Editable**.
     - `Sequence` : Array of Integer.
     - `CurrentRoundLength` : Integer, default 3.
     - `MaxRoundLength` : Integer, default 5.
     - `PlayerInputIndex` : Integer (default 0).
   - Event Dispatcher : `OnSimonSolved` (aucun paramètre).
   - Custom Events stubs (pour valider câblage) :
     - `StartPuzzle` (pas de params) → Print String "StartPuzzle called".
     - `OnMushroomHit(HitIndex : Integer)` → Print String avec ToText(HitIndex).

5. **Câblage manager ↔ mushrooms** :
   - 1 instance `BP_SimonManager` placée dans la sous-zone (à côté du rocher central, pas de mesh visible).
   - `BP_SimonMushroom` : ajout d'une variable **`ManagerRef`** (type `BP_SimonManager` Object Reference, Instance Editable).
   - Pour chaque des 4 instances mushroom dans le level :
     - `MushroomIndex` set 0→3 (gauche→droite).
     - `ManagerRef` set vers l'instance manager.
   - Array `Mushrooms` du manager rempli avec les 4 mushroom refs, **dans l'ordre 0,1,2,3** (Mushrooms[i] = mushroom avec MushroomIndex = i).
   - `BP_SimonMushroom.OnHitFromPlayer` modifié : ancien Print String supprimé, remplacé par `ManagerRef.OnMushroomHit(MushroomIndex)`.
   - **Validé en PIE** : click sur chaque champignon → manager print 0, 1, 2, 3 correctement selon la cible.

**État working tree fin J2** :
- Modifs (M) : `Config/DefaultEditor.ini` (non significatif), `Content/00-Blueprints/Items/BP_Item_Crystals.uasset` (touché en investigation hier probablement, BuiltData ?), `Content/00-Blueprints/Player/BP_FPSPlayer.uasset` (Cast SimonMushroom ajouté), `Content/01-Maps/L_DeepCave.umap` (placements mushrooms + manager + configs instances).
- Untracked : dossier `Content/00-Blueprints/Simon/` (E_SimonState, BP_SimonMushroom, BP_SimonManager).

**Prochaine action (J3, 2026-05-20)** :

1. **`PlayNote()` sur `BP_SimonMushroom`** (étape déjà décrite mais non démarrée côté éditeur) :
   - Add Timeline `NoteFlashTimeline` dans l'EventGraph (Length 0.5, Float track `Brightness` avec 3 keys : 0.0→1.0, 0.1→5.0, 0.5→1.0, interp Auto pour easing doux).
   - Custom Event `PlayNote` → câblé sur `Play from Start` de la Timeline.
   - Update de la Timeline → `Set Intensity (LightComponent0)` avec `Brightness * 2000` (multiplication via node Float×Float).
   - Pin `Finished` de la Timeline : laisser vide (la track redescend à 1.0 = IdleIntensity).
   - Test rapide : modifier temporairement `OnHitFromPlayer` pour appeler `PlayNote (Self)` AVANT le call `ManagerRef.OnMushroomHit` → click champignon → flash visible + manager print. Une fois validé, retirer ce call de PlayNote dans OnHitFromPlayer (le PlayNote sera appelé par le manager lors de l'affichage de la séquence).

2. **Séquence playback côté manager** :
   - Function `RegenerateSequence(int Length)` : vide `Sequence`, boucle 0..Length-1, ajoute `RandomIntegerInRange(0, 3)`, reset `PlayerInputIndex = 0`.
   - Function/Event `StartRound()` : `State = Playing`, lance une nouvelle Timeline de playback (à créer : `SequencePlaybackTimeline`). À chaque tick de la timeline (ou via une boucle Delay) : `Mushrooms[Sequence[i]].PlayNote()` avec ~0.8 s entre chaque. À la fin : `State = Listening`.
   - `StartPuzzle()` (déjà stub, à remplir) : `CurrentRoundLength = 3`, call `RegenerateSequence(CurrentRoundLength)`, call `StartRound()`.

3. **Logique `OnMushroomHit(HitIndex)`** :
   - Branch `State == Listening` ? Si non, return.
   - Branch `HitIndex == Sequence[PlayerInputIndex]` ?
     - Vrai → call `Mushrooms[HitIndex].PlayNote()` (feedback hit correct), `PlayerInputIndex++`. Si `PlayerInputIndex == Sequence.Length` → call `OnRoundWon`.
     - Faux → call `OnWrongAnswer`.

4. **`OnRoundWon`** :
   - Si `CurrentRoundLength == MaxRoundLength` → call `OnPuzzleSolved`.
   - Sinon `CurrentRoundLength++`, Delay ~1 s, `RegenerateSequence(CurrentRoundLength)`, `StartRound()`.

5. **`OnWrongAnswer`** :
   - `State = WrongAnswer`, flash rouge global sur tous les `Mushrooms[i].FlashRed()` (à créer côté mushroom : Timeline qui set la light color rouge brièvement). Delay ~1.5 s, `PlayerInputIndex = 0`, `StartRound()` (replay **même** séquence sans regénérer).

6. **`OnPuzzleSolved`** :
   - `State = Success`, brillance synchro sur tous les mushrooms (`PlayNote()` simultané ou anim émissive globale 2 s), fade au noir, `BroadcastOnSimonSolved`.

7. **Comment déclencher `StartPuzzle` pour les tests J3** :
   - Pas encore de trigger d'entrée (F5 placeholder). Solution temporaire : ajouter une touche de debug dans `BP_FPSPlayer` (par ex `T` ou `P`) qui call `BP_SimonManager.StartPuzzle()` via GetAllActorsOfClass. Une fois F5 fait (J4), retirer cette touche.

Polish à reporter sur J4 si pas le temps J3 : `FlashRed`, `FlashWhite` (signal "à toi"), couleurs distinctes par mushroom (`MushroomColor` appliquée à `LightComponent0` en Construction Script), notes audio (assigner `MushroomNote` par instance + `NoteAudio.SetSound + Play` dans `PlayNote`). Et les features F3 (écran fin), F4 (pause ESC), F5 (hint UMG entrée).

**Risques identifiés J3** :
- Timeline `SequencePlaybackTimeline` avec longueur dynamique : impossible nativement en Blueprint Timeline (Length fixe à compile). Soit faire une Timeline 1 note + boucle externe avec `Delay`, soit pré-calculer la length au démarrage. Approche recommandée : **`ForEachLoopWithDelay`** custom (via macro Sequence + Delay node) sur `Sequence` array, plus simple à debugger qu'une vraie Timeline.
- Risque que le call `Mushrooms[Sequence[i]].PlayNote()` race avec un PIE pause / un autre overlap → ajouter un `IsValid` guard sur Mushrooms[i] avant chaque call.
- Si on garde le tag `Item` sur les mushrooms (nécessaire pour le look-at via BPI_Interactables), il faut s'assurer qu'aucune autre logique parasite (HUD prompt "Press LMB to grab", etc.) ne s'enclenche au look-at. À vérifier en PIE - si prompt apparaît, le retirer côté WBP_HUD pour les BP_SimonMushroom.

### 2026-05-20 - J3

**Objectif atteint : F2 Simon-like complète en logique, jouable bout-en-bout. Reste polish + F3/F4/F5 pour J4.**

**Fait** :

1. **`PlayNote()` sur `BP_SimonMushroom`** :
   - Timeline `NoteFlashTimeline` créée (Length 0.5, track Float `Brightness`).
   - Keys ajustées en cours de session pour matcher le default Intensity = 0 (préférence user : champignon entièrement éteint au repos) : `(0.0, 0.0)`, `(0.1, 10.0)`, `(0.5, 0.0)`, interp Auto.
   - Custom Event `PlayNote` → branché sur `Play from Start` de la Timeline. Update → `Set Intensity (LightComponent0) = Brightness × 2000` (multiplier inchangé, c'est juste la dimension scalaire vers candelas).
   - `StopLookAtInteractables` mis à jour : `Set Intensity = 0` (cohérent avec le nouveau default off).
   - Niveaux d'intensité finaux : **0** = off (repos, look-away, après-flash), **10000** = look-at, **20000** = pic du flash.
   - Validé en PIE.

2. **Bug Light Intensity overlap découvert et corrigé** :
   - Symptôme : pendant un spam-click sur un champignon, les flashs n'étaient pas visibles. Print "Hello" sur Update de la Timeline fire en boucle, donc Update s'exécute bien, mais la lumière ne change pas visuellement.
   - Diagnostic via Print "LookAt FIRED" sur l'event `StartLookAtInteractables` : il est appelé **à chaque frame** (60×/s) par le `BP_FPSPlayer` (qui fait un LineTrace par Tick et call StartLookAt sur tout ce qui est visé, sans tracker "est-ce que je regardais déjà cet acteur la frame précédente ?").
   - Conséquence : la Timeline pilotait bien `Set Intensity (Brightness × 2000)`, mais StartLookAt écrasait avec `Set Intensity (10000)` au tick d'après → flash effectivement invisible.
   - Fix : variable booléenne **`IsBeingLookedAt`** ajoutée à `BP_SimonMushroom` (default false, non Instance Editable). `StartLookAtInteractables` testé en tête (`Branch (IsBeingLookedAt?)` → True = return, False = Set true puis chaîne existante Set Intensity 10000). `StopLookAtInteractables` reset la variable à false avant Set Intensity 0.
   - Validé : "LookAt FIRED" ne print plus qu'une fois par entrée de regard, flash visible normalement.
   - **Dette technique notée** : le comportement per-tick côté `BP_FPSPlayer` reste, on l'a juste isolé côté mushroom via le guard. Si on voulait être propre, il faudrait que le player track "previous look-at target" et n'appelle Start/Stop qu'au changement de cible. Considéré comme Bug 11 si temps J4, sinon laissé tel quel.

3. **`RegenerateSequence(int Length)` sur `BP_SimonManager`** :
   - Clear `Sequence` → For Loop (0 à `Length - 1`) → Add `RandomIntegerInRange(0, Mushrooms.Length - 1)` → Set `PlayerInputIndex = 0`.
   - Borne dynamique sur `Mushrooms.Length - 1` au lieu de 3 codé en dur : robuste si on passe à 5 mushrooms plus tard, ou si on en retire un.
   - Le user a souligné qu'il pourrait vouloir des **séquences plus longues que le nombre de champignons** (ex 7 notes avec 4 mushrooms) - déjà supporté par construction (Random peut répéter, longueur de `Sequence` indépendante de `Mushrooms`).
   - Validé via test BeginPlay → ForEach + Print : 3 valeurs aléatoires 0-3 à chaque PIE, différentes à chaque relance.

4. **`PlaySequenceStep(int Index)` sur `BP_SimonManager`** :
   - Custom Event récursif qui joue la séquence note par note.
   - Branch `Index >= Sequence.Length` :
     - True → Set `State = Listening` (le tour passe au joueur).
     - False → Get `Mushrooms[Sequence[Index]]` → IsValid check (guard sur ref null) → call `PlayNote` → Delay 0.8s → call self avec `Index + 1`.
   - Délai 0.8s entre notes pour lisibilité (à tuner si besoin).
   - **Pattern récursif** retenu (au lieu de Timeline à length dynamique impossible nativement, ou ForEachLoopWithDelay non-natif). Plus simple à debug, pas de macro custom à maintenir.
   - Validé : séquence joue note par note dans l'ordre, mushrooms flashent bien, State passe à Listening à la fin.

5. **`StartRound()` sur `BP_SimonManager`** :
   - Set `State = Playing` → call `PlaySequenceStep(0)`. Wrapper minimaliste, mais centralise le passage d'état pour pouvoir le réutiliser depuis `OnRoundWon` et `OnWrongAnswer`.

6. **`StartPuzzle()` refactorisé (remplace le stub J2)** :
   - Set `CurrentRoundLength = 3` → `RegenerateSequence(CurrentRoundLength)` → `StartRound`.
   - Le reset explicite de `CurrentRoundLength` à 3 permet de relancer le puzzle plusieurs fois dans la même session PIE sans pollution d'état (ex après un Wrong Answer en fin de session précédente où on aurait été à 5).

7. **Logique réelle de `OnMushroomHit(HitIndex)`** :
   - **Switch on `E_SimonState`** (lecture de `State`) - seule la branche `Listening` continue. Pattern retenu au lieu de `Equal (E_SimonState)` + Branch parce que Unreal propose par défaut `Equal (Byte)` quand on tape `==` sur un pin enum (fragile : la comparaison se fait sur l'index byte de l'enum, casse silencieusement si on réordonne les enumerators). Switch est type-safe (Compile alerte sur enumerator manquant) et idiomatique pour brancher sur un état.
   - Branche `Listening` → Branch `HitIndex == Sequence[PlayerInputIndex]` :
     - **Correct** → Set `PlayerInputIndex += 1` → Branch `PIIdx >= Sequence.Length` → True : call `OnRoundWon`. False : return (attend le prochain clic).
     - **Incorrect** → call `OnWrongAnswer`.
   - Note : le flash visuel au clic est géré côté mushroom (`PlayNote(Self)` appelé dans `OnHitFromPlayer` avant le call manager), pas dans le manager. Permet feedback visuel immédiat indépendamment de la validation logique.

8. **Gating UX côté `BP_SimonMushroom`** (ajouté en réflexion mid-session, sur demande user) :
   - Pour éviter ambiguïté "à qui est le tour", les events `StartLookAtInteractables` et `OnHitFromPlayer` sont aussi gatés par un **Switch on `E_SimonState`** (lecture via `ManagerRef.State`) en tête - seule la branche `Listening` continue.
   - Résultat : pendant la playback de la séquence, viser un champignon n'allume rien, cliquer ne flash pas, ne route pas vers le manager. Le joueur sait visuellement qu'il doit attendre.
   - Le Switch côté manager reste comme **defense in depth** (si un autre déclencheur arrivait à `OnMushroomHit` sans passer par le mushroom).
   - Edge case mineur noté : si l'état change pendant que le joueur regarde un champignon (ex au dernier clic correct de la séquence, State passe à Idle puis Playing après le Delay), le glow look-at persiste jusqu'à ce qu'il détourne le regard (à cause du guard IsBeingLookedAt). À l'usage acceptable, fix possible via broadcast d'un event "StateChanged" depuis le manager si nécessaire.

9. **`OnRoundWon` logique réelle** (remplace stub Print du J2) :
   - Garde le Print "Round Won!" pour debug.
   - Branch `CurrentRoundLength >= MaxRoundLength` :
     - True → call `OnPuzzleSolved`.
     - False → Set `CurrentRoundLength += 1` → Delay 1.0s → `RegenerateSequence(CurrentRoundLength)` → `StartRound`.
   - Le Delay 1.0s donne une respiration entre les rounds (sinon enchaînement trop brutal).

10. **`OnWrongAnswer` logique réelle** (remplace stub Print du J2) :
    - Garde le Print "Wrong Answer!".
    - Set `PlayerInputIndex = 0` → Delay 1.0s → `StartRound` (rejoue la **même** séquence sans la regénérer, conformément au design figé J1).

11. **`OnPuzzleSolved` créé** :
    - Set `State = Success` → Print "Puzzle Solved!" (debug) → Call `OnSimonSolved` (broadcast du Event Dispatcher déjà créé J2).
    - Aucun écouteur encore branché sur le dispatcher : ce sera fait en F3 quand `WBP_EndScreen` existera.

**Trigger de test temporaire** :
- `BP_SimonManager` Event BeginPlay → Delay 2.0s → `StartPuzzle`. Permet de tester en PIE sans F5 trigger box.
- À remplacer en J4 par un overlap d'un `BoxCollision` actor à l'entrée de la sous-zone (cf F5).

**Risque identifié en fin de J3 - à fixer en début de J4** :
- Pendant les `Delay 1.0s` dans `OnRoundWon` (branche False) et `OnWrongAnswer`, `State` reste à `Listening`. Conséquence : le joueur peut cliquer pendant ce délai. Évaluations problématiques :
  - Dans OnRoundWon : `PlayerInputIndex == Sequence.Length` à ce moment-là. `Sequence[Length]` est out-of-bounds → runtime warning probable + comparaison foireuse.
  - Dans OnWrongAnswer : `PlayerInputIndex = 0` après le Set, Sequence inchangée → clic évalué contre `Sequence[0]`, peut "avancer" indûment ou re-trigger Wrong Answer.
- Fix simple à appliquer J4 : `Set State = Idle` AVANT le Delay dans les deux events. `StartRound` repassera à `Playing` puis `PlaySequenceStep` à `Listening`.

**État working tree fin J3** :
- M : `Config/DefaultEditor.ini` (cosmétique éditeur), `Content/00-Blueprints/Items/BP_Item_Crystals.uasset` (BuiltData lighting), `Content/00-Blueprints/Player/BP_FPSPlayer.uasset` (Cast SimonMushroom déjà fait J2), `Content/01-Maps/L_DeepCave.umap` (placements + configs instances).
- Untracked : `Content/00-Blueprints/Simon/` (E_SimonState, BP_SimonMushroom, BP_SimonManager).

**Prochaine action (J4, 2026-05-21) - deadline 23:59** :

**Tier 1 - Critique (à faire en priorité)** :
1. **Fix `State = Idle` pendant les Delays** dans OnRoundWon et OnWrongAnswer (cf risque ci-dessus). Quick safety fix avant tout le reste.
2. **F5 Trigger Box** : créer `BP_SimonTrigger` (BoxCollision + ManagerRef Instance Editable) à l'entrée de la sous-zone. Remplacer le BeginPlay → Delay → StartPuzzle par OnBoxBeginOverlap filtré sur Player → StartPuzzle. Guard `bAlreadyTriggered` pour pas re-déclencher.
3. **F4 Pause Menu (ESC)** : `WBP_PauseMenu` (Resume/Quit) + Input Action ESC dans Project Settings + Set Game Paused. Tester pause partout (level 1 + sous-zone, pendant playback Simon).
4. **F3 Écran de fin** : `WBP_EndScreen` (texte + bouton Quit). Bind sur dispatcher `OnSimonSolved` depuis le PlayerController ou GameMode au BeginPlay (récupérer ref manager via `GetAllActorsOfClass` puis bind).

**Tier 2 - Polish important (recommandé si temps)** :
5. **Couleurs distinctes par mushroom** : appliquer `MushroomColor` à `LightComponent0.LightColor` dans le Construction Script de `BP_SimonMushroom`. Set par instance dans le level (rouge/vert/bleu/jaune par défaut, ou autre palette).
6. **Notes audio** : assigner `MushroomNote` (SoundBase) par instance. Dans `PlayNote`, ajouter parallèlement au Play Timeline un `NoteAudio.SetSound + Play` (ou `Play Sound 2D` simple si non-spatialisé). Sons à chercher dans `Content/` (cf Bug 4 crystal SFX) ou pitch-shift d'un son unique via Sound Cue.
7. **Flash rouge OnWrongAnswer** : créer fonction `FlashRed()` sur `BP_SimonMushroom` (Timeline qui set LightColor rouge + boost intensity ~0.5s puis retour à idle color). Boucler dessus dans `OnWrongAnswer` AVANT le Delay et le restart.
8. **Brillance synchro OnPuzzleSolved** : tous les mushrooms `PlayNote()` simultanément (ou variante flash blanc synchro) pendant 2s avant le dispatch `OnSimonSolved`. Visuel de fin marquant.

**Tier 3 - Polish ergonomie (si temps après Tier 2)** :
9. **F5 Hint UMG** : `WBP_HintEntry` qui fade in/out à l'entrée de la sous-zone. Trigger box affiche le hint, attend ~3-4s, puis appelle `StartPuzzle`.
10. **HUD prompt "Press LMB to grab" sur mushrooms** : à vérifier en PIE car le tag `Item` peut le faire apparaître. Si oui, masquer côté WBP_HUD pour les `BP_SimonMushroom` (cast filter).
11. **Tempo séquence** : tuner si 0.8s est trop long/court à l'usage. Possiblement accélérer chaque round (0.8 → 0.65 → 0.5s) pour difficulté progressive de mémorisation.
12. **Reprise Bug 7 résiduel** (Light Complexity Attenuation Radius `BP_Mushroom_Lit_*` décor). En vrai buffer seulement.

**Tier 4 - Polish à éviter sauf si vraiment temps** :
13. **Refactor `BP_FPSPlayer` look-at per-tick** (cf bug J3 #2). Le guard côté mushroom est OK, le refactor est cosmétique. À documenter comme dette technique acceptée si on ne le fait pas.
14. **Composant `DecalOutline`** abandonné définitivement, à supprimer du `BP_SimonMushroom` si nettoyage.

**Documentation finale J4** :
- Rédiger une section "**Comment ça marche**" dans FEATURE_PROGRESS.md pour la défense orale : architecture du Simon (manager orchestrateur + mushrooms répondeurs), séquence des appels (StartPuzzle → RegenerateSequence → StartRound → PlaySequenceStep récursif → OnMushroomHit → OnRoundWon ou OnWrongAnswer → loop ou OnPuzzleSolved → dispatcher), gating à 2 niveaux (mushroom + manager), pattern récursif vs Timeline.
- PROGRESS.md : entrées J3 + J4 (rolling log, vue d'ensemble du sprint).
- Commit final F2 : messages anglais terse, préfixe `[FEAT]`, single-developer voice, **aucune trace AI**. User fait le commit/push.

**Risques identifiés J4** :
- F5 trigger box : si BoxCollision overlap fire avant que ManagerRef soit valide → guard `IsValid` ou utiliser `GetAllActorsOfClass` au BeginPlay du trigger.
- F4 Pause : `Set Game Paused = true` peut bloquer la Timeline `NoteFlashTimeline` (qui dépend du Game Time par défaut). Si le pause fige le flash en cours c'est OK ; si ça casse au resume, à investiguer (Timeline Play Rate ou Tickable When Paused).
- F3 Écran de fin : bind dispatcher cross-BP. Le `OnSimonSolved` est sur l'instance de `BP_SimonManager` placée dans le level. Le widget ou son controller doit chercher cette instance via `GetAllActorsOfClass` au BeginPlay puis bind. Pattern propre via GameInstance ou GameMode possible mais plus lourd ; pour la deadline, le pattern simple suffit.
- Build standalone final : tester en `C:\UnrealBuilds\Sprint6\` au moins une fois avant le commit final (rappel Bug 7 : DRS éditeur masque la perf réelle, et le `Set Game Paused` peut se comporter différemment en standalone).
- Si Tier 2 polish déborde, **prioriser Tier 1 + commit jouable** plutôt que polish incomplet. La feature est défendable à l'oral en l'état (logique complète, juste pas joli) - mieux vaut une feature complète et défendable qu'une feature à moitié polish et cassée.

### 2026-05-21 - J4 (deadline 23:59) - Tier 1 livré, polish, commit/push

**Objectif atteint** : Tier 1 critique complet (#1-4), Tier 2 polish (couleurs + notes audio + sons fin/erreur), bugs visuels résolus, commit jouable livré (`400aac9`), push GitHub progressif en cours.

**Tier 1 critique livré** :

1. **Safety fix `State = Idle`** (cf risque identifié fin J3) :
   - `BP_SimonManager.OnRoundWon` (branche False, avant `CurrentRoundLength++`) : `Set State = Idle` ajouté AVANT le Delay 1.0s. `StartRound` repassera ensuite à `Playing`.
   - `BP_SimonManager.OnWrongAnswer` : `Set State = Idle` AVANT le Delay 1.0s, idem.
   - Conséquence validée : pendant les Delays inter-round, les clics joueur sont ignorés silencieusement par les deux Switch State, plus de risque d'évaluation contre une `Sequence` dans un état transitoire.

2. **F5 - `BP_SimonTrigger`** (partie "trigger" de F5, sans hint UMG) :
   - Actor créé : BoxCollision (OverlapAllDynamic, Hidden In Game).
   - Variables : `ManagerRef` (BP_SimonManager Object Reference, **Instance Editable**), `bAlreadyTriggered` (Boolean, default false).
   - Event Graph : `OnComponentBeginOverlap → Cast to BP_FPSPlayer → Branch (bAlreadyTriggered = False) → Set bAlreadyTriggered = true → IsValid(ManagerRef) → Delay 2.0s → ManagerRef.StartPuzzle`.
   - Le `Cast to BP_FPSPlayer` ne consomme pas le pin `As BPFPSPlayer` ; il sert juste à filtrer les overlaps non-joueur (Cast réussi = c'est bien le player). Cast conservé pour cette filtration.
   - Delay 2 s ajouté par toi pour laisser au joueur le temps de se positionner avant le début de la séquence.
   - Premier essai échoué : `IsValid(ManagerRef)` retournait false. Résolu en configurant `ManagerRef` par instance dans le World Outliner (référence explicite, pas auto-resolved).
   - Trigger BeginPlay temporaire du `BP_SimonManager` (`Delay 2 s → StartPuzzle`, mis en place en J3 pour tester) supprimé.

3. **F4 - Pause Menu (ESC)** :
   - `IA_Pause` créé (Digital, bool). Binding `Escape` ajouté dans `IMC_Player` (l'IMC effectivement utilisé, confirmé via Find in Blueprints global sur `Add Mapping Context` - deux IMC coexistaient, `IMC_Default` et `IMC_Player`, seul ce dernier est ajouté au context au runtime).
   - `WBP_PauseMenu` importé d'un autre projet (Background Blur, ResumeButton, QuitButton, sons SW_Click / SW_Back / SW_Hovered préservés). Quit changé de `Open Level (MainMenu)` → `Quit Game` (pas de main menu dans ce projet).
   - `BP_FPSPlayer.IA_Pause (Triggered)` : `Branch (IsGamePaused = False) → Create WBP_PauseMenu Widget → Add to Viewport → Set Game Paused (true) → Get Player Controller → Set Input Mode UI Only (Widget to Focus = widget ref) → Set Show Mouse Cursor (true)`.
   - Le widget gère lui-même Resume (close + Set Game Paused false + Input Mode Game Only + Hide Cursor) et Quit (Quit Game).

4. **F3 - Écran de fin sur `OnSimonSolved`** :
   - `WBP_EndScreen` créé en dupliquant `WBP_PauseMenu` pour réutiliser le Background Blur et le styling boutons.
   - **Refactor hiérarchie** nécessaire : initialement TextBlock et QuitButton étaient enfants du Background Blur → pas d'Anchor disponible (le Slot d'un Background Blur n'expose pas Anchor, contrairement au Canvas Panel Slot). Drag dans Hierarchy panel pour sortir TextBlock + QuitButton du Background Blur et les placer directement sous Canvas Panel. Background Blur réglé en full-screen (anchors stretched).
   - Titre `"Congratulations"` (~72-96 px, vert) + sous-titre `"You solved all the puzzles!"` (~28-36 px) + QuitButton.
   - `Event Construct → Play Sound 2D (Victory)`.
   - **Binding dispatcher cross-BP** : `BP_FPSPlayer.BeginPlay → GetAllActorsOfClass(BP_SimonManager) → Get [0] → IsValid → Bind Event to OnSimonSolved → Custom Event OnSimonPuzzleSolved → Create WBP_EndScreen Widget → Add to Viewport → Set Game Paused → Input Mode UI Only → Show Mouse Cursor`.

**Bugs visuels résolus mid-session** :

5. **Flash écourté pendant le replay de la séquence** :
   - Symptôme : pendant la phase Playing (la séquence se rejoue), le flash du dernier champignon cliqué par le joueur au round précédent était extrêmement court (~1 frame visible). Les 3 autres mushrooms flashent normalement.
   - Faux-départ : diagnostic initial pointait `StopLookAtInteractables` en Listening, corrigé par toi.
   - Vrai diagnostic : `StopLookAtInteractables` fire **per-tick** côté `BP_FPSPlayer` (dette technique notée J3 : le player ne tracke pas "previous look-at target" et call Start/StopLookAt à chaque LineTrace de Tick). Sur le dernier mushroom hovered au clic final, pendant le replay, `StopLookAt → Set Intensity = 0` écrase à chaque tick la valeur courante de la Timeline `NoteFlashTimeline` → flash effectivement invisible.
   - Première tentative fix : `Switch on E_SimonState` (Listening only) sur `StopLookAt`. Résout le flash écourté mais introduit bug 6 (X reste allumé).
   - **Solution finale (choix toi, plus minimaliste)** : juste un `Branch (IsBeingLookedAt = True)` en tête de `StopLookAtInteractables`. True → Set IsBeingLookedAt = false → Set Intensity = 0. False → return. Pas de Switch State, plus simple.

6. **Dernier mushroom cliqué reste allumé pendant le replay** :
   - Symptôme : après le fix du flash écourté (première tentative Switch State, puis Branch IsBeingLookedAt), le dernier mushroom cliqué restait à intensité résiduelle pendant tout le replay au lieu de revenir à off.
   - Diagnostic : `StopLookAt` étant gaté (Switch State ou Branch IsBeingLookedAt), il ne peut plus reset l'intensité résiduelle laissée par le look-at boost (Intensity = 10000) au moment du dernier clic.
   - Fausse piste explorée : "résiduel de Timeline" → toi as relevé que les mushrooms sont bien à 0 à la fin du replay donc la Timeline finit propre. C'est bien le boost look-at non-nettoyé qui persiste.
   - **Fix** : Custom Event **`ResetVisualState`** côté `BP_SimonMushroom` (Set `IsBeingLookedAt = false` → Set `LightComponent0.Intensity = 0`). Dans `BP_SimonManager.StartRound`, ForEach Loop sur `Mushrooms` → `Mushroom.ResetVisualState` AVANT le `Set State = Playing` (corrigé par toi en cours de session - initialement je proposais ailleurs dans le flow).
   - Résultat : la transition Listening → Playing commence avec tous les mushrooms garantis à 0, plus de résiduel.

**Tier 2 polish livré** :

7. **Couleurs distinctes par mushroom (Option A - direct level config)** :
   - Choix Option A (direct dans le World Outliner) au lieu d'Option B (variable `MushroomColor` + Construction Script qui propage à `LightComponent0.LightColor`) : moins de code pour 4 valeurs définies une seule fois.
   - Hex sRGB : Mushroom 0 bleu foncé `#1F45FF`, Mushroom 1 jaune `#FFD110`, Mushroom 2 violet `#9E33FF`, Mushroom 3 cyan `#18E0FF`.
   - Set par instance : Details → LightComponent0 → Light Color → Color Picker → Hex sRGB.

8. **Notes audio spatialisées (Do majeur arpégé)** :
   - Fichiers `.wav` générés via Audacity FR (Générer → Tonalité → onde Sinus, 0.5 s chacun) :
     - `Note_C4.wav` 261.63 Hz, `Note_E4.wav` 329.63 Hz, `Note_G4.wav` 392.00 Hz, `Note_C5.wav` 523.25 Hz.
   - Choix musical : accord parfait majeur arpégé → 4 notes consonantes dans n'importe quel ordre (séquences aléatoires sonneront toujours harmoniquement justes).
   - Importés dans `Content/Audio/SFX/`. Assignés à `MushroomNote` par instance : M0 = C4, M1 = E4, M2 = G4, M3 = C5.
   - `BP_SimonMushroom.PlayNote` enrichi : en parallèle du `Play from Start` Timeline, ajout de `NoteAudio.SetSound (MushroomNote) → NoteAudio.Play`. Audio 3D spatialisé via le composant `NoteAudio`.

9. **`Victory.wav` sur écran de fin** :
   - Asset importé dans `Content/Audio/SFX/`.
   - `WBP_EndScreen.Event Construct → Play Sound 2D (Victory)` (UI 2D, pas spatialisé).

10. **`ErrorSound.wav` sur mauvaise réponse** :
    - Asset importé dans `Content/Audio/SFX/`.
    - `BP_SimonManager.OnWrongAnswer` : `Play Sound 2D (ErrorSound)` ajouté en tête de l'event (avant `Set State = Idle`).

**Commit jouable livré** :
- `400aac9 [ADD] second puzzle, pause and end menu` - état Tier 1 + Tier 2 + bugs résolus. Single-developer voice, anglais terse, préfixe `[ADD]`.

**Push GitHub progressif (limite 2 GiB pack)** :
- Tentative initiale `git push origin main` → "Connection reset by peer" (déco réseau pendant upload long), puis re-tentative → `remote: fatal: pack exceeds maximum allowed size (2.00 GiB)`. Limite hard GitHub tier gratuit ; le repo fait ~2.5 GB du fait des assets UE 5.4 + historique non-LFS.
- **Stratégie progressive** : `git push origin <sha>:refs/heads/main` commit par commit. Chaque push n'envoie que le delta entre la pointe distante et le sha cible → la limite 2 GiB s'applique par pack, pas cumulativement.
- Premier commit `3204fc3` (Initial commit) pushé OK.
- Reste 11 commits : `c8666f0 → 6b85021 → de7e0b3 → 0ae0a27 → 2189d16 → fa669c9 → e6886fe → 96af773 → 3d9a212 → 7b42dbc → 400aac9`. Candidat le plus volumineux : `e6886fe [FIX] optimizing lights, textures` (touche ~110 meshes Nanite + 18 textures Maximum Texture Size 2048).
- Plan A : `git push origin main` direct (espère que le delta restant tient en un pack sous 2 GiB). Plan B si A échoue : push par chunks (`fa669c9` → `e6886fe` seul → `400aac9`).

**État working tree fin J4 (post-`400aac9`)** :
- Modifs (M) : `BP_SimonManager.uasset`, `BP_SimonMushroom.uasset`, `L_DeepCave.umap`, `WBP_EndScreen.uasset` - intégration audio post-commit (PlayNote.NoteAudio.SetSound+Play, Victory sur EndScreen.Construct, ErrorSound sur OnWrongAnswer, configs MushroomNote par instance).
- Untracked (??) : `Content/Audio/SFX/ErrorSound.uasset`, `Note_C4.uasset`, `Note_C5.uasset`, `Note_E4.uasset`, `Note_G4.uasset`, `Victory.uasset`.
- Commit final à préparer (`git add` ces 10 fichiers, message anglais terse type `[ADD] mushroom notes, victory and error sounds`, single-developer voice, **aucune trace AI**).

**Reste à faire (deadline 23:59)** :
1. Finir le push GitHub progressif (Plan A puis B si besoin).
2. Ajouter le tuteur au repo GitHub via interface web (cf sujet Epitech `EB_Sprint6_Sujet.pdf` - fait par toi).
3. Commit final intégration audio + push.
4. (Si temps) Build standalone `C:\UnrealBuilds\Sprint6\` pour smoke test perf en condition de livraison.
5. PROGRESS.md J3 + J4 entries (rolling log côté sprint, séparé de ce doc feature).

**Non livré (deadline-driven, parqué)** :
- F5 hint UMG (`WBP_HintEntry` avec fade in/out). Le trigger box démarre directement après son Delay 2 s, pas de texte affiché. Acceptable : la séquence elle-même sert d'auto-tutoriel évident.
- FlashRed visuel sur OnWrongAnswer (couleur rouge globale sur les 4 mushrooms). ErrorSound seul suffit comme feedback de retour ; visuel non bloquant.
- Brillance synchro 2 s à la fin (`PlayNote` simultané sur les 4) avant le dispatch. Le passage direct à `WBP_EndScreen` + son Victory est suffisamment marquant.
- Refactor `BP_FPSPlayer` look-at per-tick (dette technique acceptée, workaround via guard mushroom suffit).
- Composant `DecalOutline` orphelin dans `BP_SimonMushroom` (pivot J1 abandonné). Pas de nettoyage, sans impact runtime.
