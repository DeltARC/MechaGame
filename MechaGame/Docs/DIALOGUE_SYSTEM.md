# Système de dialogue à choix

Tout le système est en Blueprint. Il n'y a aucun C++ à compiler.

Trois couches séparées :

- **Données** : ce que les personnages disent. Des DataTables, des Data Assets, deux structures, des enums.
- **Logique** : les règles du dialogue. Aucun widget, aucun affichage.
- **Interface** : ce que le joueur voit. Tes widgets ne contiennent aucun nœud ; un composant les pilote de l'extérieur.

Les assets sont dans `Content/Blueprints/ClementTest/Dialogue/`, sauf les widgets et les PNJ qui sont restés à leur place.

---

## 1. Vue d'ensemble

### Le flux

```
Le joueur clique sur un PNJ
        │
        ▼
BP_Interact1ClementTest…      (le PNJ : un seul nœud, "Interact")
        │
        ▼
BPC_DialogueNPC               (composant du PNJ)
  · choisit le nœud de départ selon les tags du joueur
  · pose la caméra sur le PNJ, attend la fin du mouvement
        │  StartDialogue(DataTable, nœud de départ, nom du widget)
        ▼
BP_DialogueManager            (LOGIQUE : état, nœud courant, choix filtrés, tags)
        │  OnDialogueStarted · OnNodeChanged · OnChoiceSelected · OnDialogueEnded
        ▼
BPC_DialoguePresenter         (INTERFACE : crée le widget, écrit le texte, écoute les boutons)
        │
        ▼
WBP_DialogueChoix…            (ta mise en page, aucun code)
        │  le joueur clique sur un bouton
        ▼
BPC_DialoguePresenter  ──►  BP_DialogueManager.SelectChoice / Continue / EndDialogue
                                    │
                                    └──► nouveau nœud : la boucle recommence
```

La règle à retenir : **le gestionnaire ne connaît pas l'interface, l'interface ne décide de rien**. Le gestionnaire prévient par des Event Dispatchers ; le composant d'affichage appelle seulement `SelectChoice`, `Continue` et `EndDialogue`.

### Le rôle de chaque asset

| Asset | Couche | Rôle |
|---|---|---|
| `E_ChoiceType` | Données | Type d'un choix : Normal, Persuasion, Intimidation, Leave |
| `E_SpeakerMood` | Données | Humeur du personnage : Neutral, Happy, Angry, Scared |
| `E_DialogueState` | Données | Idle, Playing, WaitingForChoice, Ended. **Pas encore branché**, voir section 2.5 |
| `S_DialogueChoice` | Données | Un choix : texte, nœud suivant, condition, tags donnés et retirés, type |
| `S_DialogueNode` | Données | Une réplique : personnage, humeur, texte, choix, nœud suivant par défaut |
| `BP_DialogueSpeakerData` | Données | Modèle d'un personnage : nom, couleur du nom, portrait par humeur |
| `DA_Speaker_FemmeArme`, `DA_Speaker_Mechano` | Données | Les deux personnages |
| `DT_Dialogue_FemmeArme`, `DT_Dialogue_Mechano` | Données | Les dialogues. Le Row Name d'une ligne est l'identifiant du nœud |
| `BP_DialogueManager` | Logique | Le cerveau. Un seul par niveau, créé automatiquement |
| `BP_DialogueSave` | Logique | Le fichier de sauvegarde : il ne contient que les tags du joueur |
| `BPC_DialogueNPC` | Logique | Composant à poser sur un PNJ : sa DataTable, son nœud de départ, sa caméra |
| `BPC_DialoguePresenter` | Interface | Composant du gestionnaire : pilote le widget |
| `WBP_DialogueChoixFemmeArme`, `WBP_DialogueChoixMechano` | Interface | Tes mises en page. Dans `Content/Widgets/ClementTest/DialogueChoix/` |
| `BP_Interact1ClementTestFemmeArme`, `…Mechano` | PNJ | Tes PNJ. Dans `Content/Blueprints/ClementTest/` |

### Le gestionnaire : ce qu'on peut appeler depuis n'importe quel Blueprint

Pour l'obtenir : **Get Actor Of Class → `BP_DialogueManager`**. Depuis un PNJ, préfère `DialogueNPC → Get Manager`, qui le crée s'il n'existe pas encore.

| Fonction | Ce qu'elle fait |
|---|---|
| `StartDialogue(Table, StartNodeID, WidgetName)` | Lance un dialogue. Ignoré si un dialogue est déjà en cours |
| `SelectChoice(Index)` | Valide un choix. L'index est celui de la liste **filtrée** |
| `Continue()` | Passe au `NextNodeID` du nœud courant |
| `EndDialogue()` | Termine le dialogue |
| `GetAvailableChoices()` | Les choix visibles du nœud courant, conditions déjà appliquées |
| `AddFlag(Tag)`, `RemoveFlag(Tag)`, `HasFlag(Tag)` | Lire et modifier les tags du joueur, un par un |
| `AddFlags(Tags)`, `RemoveFlags(Tags)` | Pareil avec plusieurs tags d'un coup |
| `ValidateTable(Table, StartNodeIDs)` | Vérifie une DataTable, voir section 5 |
| `ResetFlags()` | Efface tous les tags et la sauvegarde. Aussi sur la touche **F6** |
| `SaveFlags()`, `LoadFlags()` | Écrire et relire la sauvegarde. Appelées automatiquement, tu n'as pas à t'en servir |

Variables utiles en lecture : `PlayerFlags` (tous les tags du joueur), `bIsActive`, `CurrentNodeID`, `CurrentNode`, `LastChoice`.

Event Dispatchers : `OnDialogueStarted`, `OnNodeChanged`, `OnChoiceSelected`, `OnDialogueEnded`, `OnFlagsChanged`. Aucun n'a de paramètre : celui qui écoute lit ensuite les variables du gestionnaire.

### Pourquoi chaque fonction est écrite comme ça

Les outils que j'ai utilisés ne savent pas poser de commentaires dans les graphes. Les explications sont donc ici.

**`BP_DialogueManager`**

| Fonction | Le pourquoi |
|---|---|
| `StartDialogue` | Appelle `OnDialogueStarted` **avant** d'aller au premier nœud : le widget doit exister avant de recevoir la première réplique |
| `GoToNode` | `None` veut dire « fin du dialogue », c'est la convention de la DataTable. Si l'identifiant n'existe pas, on affiche une erreur et on ferme le dialogue au lieu de laisser le joueur bloqué |
| `RefreshAvailableChoices` | Recalcule la liste à chaque nœud, car le choix précédent a pu changer les tags. Une condition vide veut dire « toujours visible » : dans Unreal, une requête de tags vide ne réussit jamais, d'où le test `Is Tag Query Empty` |
| `SelectChoice` | Copie le choix dans `LastChoice` tout de suite, car la liste des choix est réécrite dès qu'on change de nœud. Retire les tags **avant** d'en donner : un même choix peut remplacer un tag par un autre |
| `EndDialogue` | Vérifie `bIsActive` d'abord : le bouton Leave et la fin d'un nœud peuvent l'appeler tous les deux, l'interface ne doit être prévenue qu'une fois |
| `ValidateTable` | Parcourt le dialogue depuis ses points d'entrée, comme un joueur. Elle utilise ses propres variables locales pour ne jamais toucher à un dialogue en cours |
| `SaveFlags` / `LoadFlags` | La sauvegarde ne contient que les tags : tout l'état des dialogues et des quêtes en découle. Chaque changement de tag est écrit tout de suite, il n'y a pas de bouton « Sauvegarder » à oublier. Le chargement se fait au démarrage du gestionnaire, donc aussi à chaque changement de niveau |
| `ResetFlags` | Outil de test : sans lui, un dialogue de première rencontre ne pourrait plus être rejoué une fois la sauvegarde écrite |

**`BPC_DialogueNPC`**

| Fonction | Le pourquoi |
|---|---|
| `GetManager` | Le premier PNJ qui en a besoin crée le gestionnaire, les autres le retrouvent. C'est ce qui évite de placer quoi que ce soit dans le niveau ou sur le joueur |
| `ResolveStartNodeID` | Le premier point d'entrée dont le joueur possède le tag gagne : l'ordre du tableau fait la priorité |
| `Interact` | Point d'entrée unique : un clic, une touche ou une zone n'ont qu'à l'appeler. Retient la vue actuelle pour la rendre à la fin. `bOwnsCurrentDialogue` bloque le double-clic pendant le mouvement de caméra |
| `HandleDialogueEnded` | Tous les PNJ reçoivent la fin du dialogue ; seul celui qui l'a lancé rend la caméra |

**`BPC_DialoguePresenter`**

| Fonction | Le pourquoi |
|---|---|
| `CreateDialogueWidget` | L'adaptateur. C'est le **seul** endroit qui connaît le nom des éléments de chaque mise en page. Tout le reste passe par `LineText`, `Portrait`, `ChoiceButtons`, `ChoiceTexts`, `LeaveButton` |
| `ShowLine` | Convertit le texte en chaîne une seule fois, car l'effet machine à écrire découpe une chaîne. Coupe l'ancien timer avant d'en lancer un, sinon deux répliques rapprochées s'écriraient deux fois plus vite |
| `ShowSkipButton` | Pendant la frappe, un seul bouton « Passer » est visible : cliquer sur un vrai choix avant d'avoir lu serait trop facile par erreur. Les boutons inutilisés sont `Hidden` et non `Collapsed`, pour que la mise en page ne saute pas |
| `ShowChoices` | Donne le focus au premier bouton : c'est ce qui rend la navigation clavier et manette possible. Prévient si un nœud a plus de choix que le widget n'a de boutons |
| `GetClickedIndex` | Tous les boutons partagent un seul gestionnaire de clic, sans paramètre. On retrouve le bouton activé en cherchant celui qui a le focus, puis celui sous la souris. C'est ce qui permet de garder les widgets sans code |
| `GetChoiceLabel` | Le type de choix se voit par un préfixe dans le libellé. Pour passer à des icônes, c'est ici |
| `HandleDialogueStarted` / `HandleDialogueEnded` | Retient l'état du curseur au début et le rend tel quel à la fin, car ton niveau joue déjà avec le curseur visible |

---

## 2. Mise en place dans l'éditeur

Tout est déjà créé et branché sur tes deux PNJ. Cette section sert à tester, puis à refaire la même chose pour un nouveau personnage.

### 2.1 Tester

1. Ouvre `Content/Levels/Lvl_ClementTest`.
2. Lance le jeu (Play).
3. Clique sur le PNJ au pistolet. La caméra se pose sur lui, la réplique s'écrit lettre par lettre.
4. Clique sur « Passer » pour afficher tout le texte d'un coup, puis sur « Ca marche ». L'animation de quête se joue.
5. Clique sur « Sortir ». La caméra revient.
6. Reclique sur le même PNJ : il démarre sur une autre réplique, car tu as maintenant le tag `Quete.FemmeArme.Acceptee`.
7. Va voir la mécanicienne et choisis « quel est votre but dans la vie ? » : un choix `[Persuasion]` apparaît, visible seulement si la quête est acceptée. Choisis-le : elle te donne l'indice.
8. Retourne voir le PNJ au pistolet : il clôt la quête. Appuie sur **J** : le journal l'affiche « Terminée ».

Les tags sont sauvegardés : en relançant le jeu, tu reprends où tu en étais. **Appuie sur F6 en jeu pour tout remettre à zéro** et rejouer la première rencontre.

### 2.2 Ajouter un PNJ qui parle

1. Ouvre le Blueprint de ton acteur.
2. Dans le panneau Components, clique sur **Add** et choisis `BPC_DialogueNPC`.
3. Sélectionne ce composant. Dans Details, catégorie **Dialogue** :
   - `Dialogue Table` : la DataTable du personnage.
   - `Start Node ID` : le Row Name de la première réplique.
   - `Widget Name` : `FemmeArme` ou `Mechano`, selon la mise en page voulue.
   - `Focus Camera On Owner` : coche-le si l'acteur a un composant Camera.
4. Dans l'Event Graph, relie ton déclencheur à **`Interact`** du composant. Pour un clic souris : `Event ActorOnClicked` → glisse le composant dans le graphe → `Interact`.

Le projet a aussi l'interface `BPI_Interact` (touche d'interaction du joueur). Pour un PNJ qui l'implémente, relie `Event OnInteract` à `Interact` de la même façon.

### 2.3 Points d'entrée conditionnels

Toujours sur le composant, deux tableaux qui vont **par paires, même index** :

- `Entry Point Tags` : le tag que le joueur doit avoir.
- `Entry Point Node IDs` : le nœud de départ dans ce cas.

Le premier tag trouvé gagne. Si aucun ne correspond, le dialogue démarre à `Start Node ID`.

Exemple sur la femme à l'arme :

| Index | Entry Point Tags | Entry Point Node IDs |
|---|---|---|
| 0 | `Quete.FemmeArme.Terminee` | `FA_Merci` |
| 1 | `Quete.FemmeArme.Indice` | `FA_Rapport` |
| 2 | `Quete.FemmeArme.Acceptee` | `FA_DejaAccepte` |
| 3 | `Quete.FemmeArme.Refusee` | `FA_DejaRefuse` |

L'ordre compte : un joueur qui a fini la quête possède les quatre tags, c'est donc le plus avancé qui doit être en premier.

### 2.4 Ajouter un personnage (speaker)

1. Clic droit dans le Content Browser → Miscellaneous → Data Asset → choisis `BP_DialogueSpeakerData`.
2. Remplis `Display Name` et `Name Color` (laisse l'opacité A à 1).
3. Dans `Portraits`, ajoute une ligne par humeur et choisis l'image.

Une humeur sans portrait garde l'image précédente à l'écran.

### 2.5 Brancher `E_DialogueState` (optionnel, une étape à la main)

Les outils ne savent pas créer une variable de type enum, donc l'état n'est pas encore exposé sous forme d'enum. En attendant, `bIsActive` répond à « un dialogue est-il en cours ? ».

Pour l'avoir : dans `BP_DialogueManager`, ajoute une variable `State` de type `E_DialogueState`. Il reste ensuite à la mettre à jour dans `StartDialogue`, `GoToNode` et `EndDialogue`.

---

## 3. Écrire un dialogue

Tu as deux façons de travailler : modifier la DataTable dans l'éditeur (le plus simple), ou modifier le CSV et le réimporter.

Les CSV sont dans `Docs/Dialogues/`. **Un réimport écrase les modifications faites à la main dans la DataTable** : choisis une seule des deux méthodes par table.

### Les colonnes

```
---,Speaker,Mood,Line,Choices,NextNodeID
```

| Colonne | Contenu |
|---|---|
| `---` | L'identifiant du nœud (Row Name). Unique dans la table |
| `Speaker` | Chemin du Data Asset, par exemple `/Game/Blueprints/ClementTest/Dialogue/DA_Speaker_Mechano.DA_Speaker_Mechano` |
| `Mood` | `Neutral`, `Happy`, `Angry` ou `Scared` |
| `Line` | La réplique |
| `Choices` | La liste des choix, voir plus bas. Vide = pas de choix |
| `NextNodeID` | Le nœud suivant quand il n'y a pas de choix. `None` = fin |

### Chaîner deux répliques, sans choix

Laisse `Choices` vide et mets l'identifiant suivant dans `NextNodeID`. Le joueur voit un bouton « Continuer ».

```
ME_Intro,"/Game/…/DA_Speaker_Mechano.DA_Speaker_Mechano","Neutral","Attendez.","","ME_Suite"
ME_Suite,"/Game/…/DA_Speaker_Mechano.DA_Speaker_Mechano","Happy","Voilà, c'est réparé.","","None"
```

### Proposer des choix

Chaque choix est entre parenthèses, les choix sont séparés par des virgules, le tout entre parenthèses. Dans un CSV, les guillemets à l'intérieur d'une cellule se doublent.

```
"((Text=""Ca marche"",NextNodeID=""FA_Accepte"",ChoiceType=Normal),(Text=""Non"",NextNodeID=""FA_Refuse"",ChoiceType=Normal))"
```

Tes mises en page ont deux boutons de choix : au-delà de deux choix visibles, les suivants ne s'affichent pas et un avertissement apparaît.

### Donner ou retirer un tag

Ajoute `TagsToGrant` ou `TagsToRemove` dans le choix :

```
(Text=""Ca marche"",NextNodeID=""FA_Accepte"",TagsToGrant=(GameplayTags=((TagName=""Quete.FemmeArme.Acceptee""))),ChoiceType=Normal)
```

Le tag doit exister dans Project Settings → Gameplay Tags.

### Un choix conditionnel

Le choix n'apparaît que si le joueur remplit la condition. **Fais-le dans l'éditeur de DataTable** : ouvre la ligne, déplie le choix, clique sur `Condition` et construis la règle avec l'interface.

En CSV, c'est possible mais peu lisible. Pour « le joueur a le tag X » :

```
Condition=(TokenStreamVersion=0,TagDictionary=((TagName=""Quete.FemmeArme.Acceptee"")),QueryTokenStream=(0,1,2,1,0),UserDescription="""",AutoDescription="""")
```

### Terminer un dialogue

- Un choix avec `NextNodeID=""None""` termine le dialogue quand on le choisit.
- Un nœud sans choix avec `NextNodeID` à `None` se termine sur « Continuer ».
- Le bouton `ButtonLeave` de ta mise en page termine le dialogue à tout moment.

### Réimporter un CSV

Clic droit sur la DataTable → **Reimport**. Si Unreal demande le fichier, pointe vers le CSV dans `Docs/Dialogues/`.

Pour une nouvelle table : glisse le CSV dans le Content Browser, choisis **DataTable** et la structure `S_DialogueNode`.

### Traduction

Tous les textes sont des `Text` localisables. À l'import, Unreal leur donne une clé stable par ligne (`FA_Accueil_Line`, `FA_Accueil_Choices_Index0_Text`), ce qui permet de les retrouver dans le Localization Dashboard.

---

## 4. Si je veux changer…

### …ajouter un type de choix ou une humeur

**Une humeur** : ouvre `E_SpeakerMood` et ajoute une valeur **à la fin**. Ajoute ensuite le portrait correspondant dans chaque Data Asset de personnage. Rien d'autre à faire.

**Un type de choix** : ouvre `E_ChoiceType` et ajoute une valeur **à la fin**. Pour qu'il se voie à l'écran, ouvre `BPC_DialoguePresenter` → fonction `GetChoiceLabel` et ajoute une branche comme celles de Persuasion et Intimidation. Les préfixes actuels sont les variables `PersuasionPrefix` et `IntimidationPrefix`.

Ne supprime pas et ne réordonne pas les valeurs existantes d'un enum : les graphes et les DataTables s'y réfèrent par leur position.

### …changer l'apparence ou l'animation de l'interface

Les widgets sont à toi : change les images, les polices, les tailles et les positions librement.

**Les noms à ne pas changer**, car le composant d'affichage les cherche : `TextPNJ`, `ButtonChoix1`, `ButtonChoix2`, `Choix1`, `Choix2`, `ButtonLeave`, `Image_283`. Si tu en renommes un, `BPC_DialoguePresenter` ne compilera plus : corrige alors le nom dans sa fonction `CreateDialogueWidget`.

| Ce que tu veux changer | Où |
|---|---|
| Couleur du texte au survol | Dans le widget : sélectionne `ButtonChoix1` → Style → `Hovered Foreground`. Le texte hérite de la couleur du bouton |
| Vitesse de la machine à écrire | `BP_DialogueManager` → composant `DialoguePresenter` → `Typewriter Interval` (secondes par lettre) |
| Les mots « Continuer » et « Passer » | Même endroit : `Continue Label`, `Skip Label` |
| Un troisième bouton de choix | Ajoute `ButtonChoix3` et `Choix3` dans le widget, puis deux nœuds `Add` de plus dans `CreateDialogueWidget` |
| Une nouvelle mise en page | Crée le widget, puis ajoute une branche dans `CreateDialogueWidget` en copiant celle de `Mechano`. Le `Widget Name` du PNJ sert à choisir la branche |
| Afficher le nom du personnage | Ajoute un TextBlock dans le widget et affecte-le à `NameText` dans `CreateDialogueWidget`. Le nom s'affichera tout seul. La couleur `NameColor` n'est pas encore appliquée |
| Une animation à l'ouverture | Crée l'animation dans le widget, puis joue-la avec `Play Animation` à la fin de `HandleDialogueStarted` |

### …déclencher un événement de gameplay depuis un choix

Ne modifie pas le système : **écoute-le**. L'exemple complet est dans `BP_Interact1ClementTestFemmeArme`, qui joue l'animation de quête quand le joueur accepte.

1. Dans le Blueprint qui doit réagir (une porte, un gestionnaire de quêtes), sur `Event BeginPlay` : récupère le gestionnaire, puis **Bind Event to On Choice Selected**.
2. Dans l'événement relié : lis `Last Choice` du gestionnaire, fais un **Break** dessus, et teste `Tags To Grant` avec **Has Tag**.
3. Si le tag est là, fais ton action : ouvrir la porte, donner l'objet, lancer la quête.

Réagis au **tag**, pas au texte du choix : le texte peut changer ou être traduit.

Variante plus simple pour un objet qui dépend d'un état durable : écoute `OnFlagsChanged` et teste `Has Flag`.

### …ajouter des conditions plus complexes que des tags

Toutes les conditions passent par un seul endroit : `BP_DialogueManager` → fonction `RefreshAvailableChoices`.

- **Le plus simple** : traduis ta donnée en tag. Quand le joueur ramasse la clé, appelle `AddFlag(Inventaire.Cle)`. Aucun changement au système.
- **Une vraie valeur** (réputation minimale, or) : ajoute un champ à `S_DialogueChoice`, par exemple `MinReputation` (Integer), puis ajoute le test dans `RefreshAvailableChoices` à côté du test de tags. Ajoute la colonne dans le CSV.

`ValidateTable` ne connaît que les tags : un choix avec un champ personnalisé est considéré comme toujours disponible par la validation.

### …ajouter du son ou du doublage par réplique

1. Ajoute un champ `Voice` (Sound Base, Object Reference) à `S_DialogueNode`.
2. Dans `BPC_DialoguePresenter` → `ShowLine` : ajoute **Spawn Sound 2D** avec ce champ et garde le résultat dans une variable `CurrentVoice`.
3. Au début de `ShowLine` et dans `HandleDialogueEnded` : appelle **Stop** sur `CurrentVoice`, pour couper la voix si le joueur passe ou quitte.
4. Ajoute la colonne `Voice` dans le CSV (chemin de l'asset son).

### …déplacer la caméra sur le PNJ pendant le dialogue

C'est déjà en place. Sur le composant `BPC_DialogueNPC` du PNJ :

- `Focus Camera On Owner` : active ou coupe le mouvement.
- `Camera Blend Time` : durée du mouvement, en secondes.

Le PNJ doit avoir un composant Camera : c'est lui qui définit le cadrage. La logique est dans l'Event Graph de `BPC_DialogueNPC`, événements `Interact` et `HandleDialogueEnded`.

### …sauvegarder les flags entre les sessions

C'est en place. Les tags du joueur sont écrits dans `Saved/SaveGames/DialogueFlags.sav` à chaque changement, et relus au démarrage du gestionnaire. Comme un gestionnaire est créé dans chaque niveau, les tags suivent aussi le joueur d'un niveau à l'autre.

| Ce que tu veux changer | Où |
|---|---|
| Couper la sauvegarde | `BP_DialogueManager` → Class Defaults → catégorie Sauvegarde → décoche `Persist Flags` |
| Avoir plusieurs emplacements de sauvegarde | Même endroit : change `Save Slot Name` avant le chargement |
| Sauvegarder autre chose (position, inventaire) | Ajoute une variable à `BP_DialogueSave`, remplis-la dans `SaveFlags` et relis-la dans `LoadFlags` |
| Remettre à zéro | Touche **F6** en jeu, ou la fonction `ResetFlags` |

La touche F6 est un outil de test : retire son nœud de l'Event Graph de `BP_DialogueManager` avant de livrer le jeu.

---

## 5. Pièges et dépannage

### Où lire les messages

Le système écrit ses messages à l'écran et dans l'Output Log, tous préfixés par `[Dialogue]`. Dans l'Output Log, tape `[Dialogue]` dans le filtre.

Chaque PNJ vérifie sa DataTable au lancement du jeu :

| Message | Cause | Solution |
|---|---|---|
| `ERREUR : un NextNodeID ou un point d'entrée pointe vers un nœud inexistant` | Faute de frappe dans un identifiant | Corrige le Row Name ou le `NextNodeID` cité |
| `ATTENTION : nœud inaccessible depuis les points d'entrée` | Aucun chemin ne mène à cette ligne | Relie-la, ou supprime-la |
| `ATTENTION : tous les choix de ce nœud sont conditionnels et il n'a pas de NextNodeID` | Si aucune condition ne passe, le dialogue se termine là | Ajoute un choix sans condition ou un `NextNodeID` |
| `ATTENTION : aucune DialogueTable sur le composant` | Le composant n'est pas configuré | Remplis `Dialogue Table` |
| `ERREUR : nœud introuvable dans la DataTable` | Pendant le jeu, un identifiant n'existe pas | Le dialogue se ferme proprement ; corrige l'identifiant |
| `ATTENTION : ce nœud propose plus de choix que le widget n'a de boutons` | Plus de deux choix visibles | Réduis les choix ou ajoute un bouton (section 4) |

### Les erreurs probables

**Rien ne se passe quand je clique sur le PNJ.** Vérifie que le `Event ActorOnClicked` du PNJ est relié à `Interact`, que le curseur est visible, et qu'aucun autre dialogue n'est en cours.

**Le dialogue démarre toujours au même nœud.** Les deux tableaux de points d'entrée doivent avoir la même longueur et le même ordre. Vérifie aussi que le choix donne bien le tag (`TagsToGrant`).

**Un choix conditionnel n'apparaît jamais.** Le tag de la condition doit être exactement celui donné ailleurs. Une condition construite à la main dans le CSV est fragile : refais-la dans l'éditeur de DataTable.

**La caméra ne revient pas.** Elle revient vers la vue active au moment du clic. Si tu changes de vue par un autre Blueprint pendant le dialogue, c'est cette logique qu'il faut revoir (`BPC_DialogueNPC` → `HandleDialogueEnded`).

**`BPC_DialoguePresenter` ne compile plus.** Tu as renommé ou supprimé un élément du widget. Ouvre `CreateDialogueWidget` : le nœud en erreur te dit lequel.

**« Accessed None » dans le log avec une nouvelle mise en page.** Ta branche dans `CreateDialogueWidget` n'affecte pas toutes les variables. `LineText`, `Portrait`, `LeaveButton` et au moins un bouton de choix sont obligatoires.

**Le mauvais choix est validé au clic.** Le système retrouve le bouton cliqué par le focus. Garde `Is Focusable` coché sur les boutons de choix.

**Le PNJ refait toujours son dialogue « déjà vu », je ne revois plus la première rencontre.** C'est la sauvegarde. Appuie sur F6 en jeu.

**J'ai changé les réglages du composant dans le Blueprint du PNJ, mais le PNJ du niveau garde les anciens.** Compile le Blueprint. Si ça ne suffit pas, sélectionne le PNJ dans le niveau, puis son composant `DialogueNPC` : une flèche jaune à côté d'une valeur signale qu'elle a été modifiée sur cet exemplaire, clique dessus pour revenir à celle du Blueprint.

**Mes modifications de DataTable ont disparu.** Tu as réimporté le CSV. Voir section 3.

**J'ai renommé un champ d'une structure et tout est cassé.** Les graphes lisent les champs par leur nom. Renommer `Line` ou `Choices` casse les nœuds Break et les colonnes du CSV. Ajoute des champs, ne renomme pas.

### Limites connues

- Un seul joueur, un seul dialogue à la fois.
- Deux choix visibles maximum avec les mises en page actuelles.
- Le nom du personnage n'est pas affiché : aucune des deux mises en page n'a de champ pour ça.
- Le type de choix s'affiche par un préfixe texte, pas par une icône.
- Les guillemets décoratifs autour de tes répliques d'origine ont été retirés. Remets-les dans le texte si tu les veux.
- Pendant la frappe, les textes des choix ne s'écrivent plus lettre par lettre ; ils apparaissent à la fin de la réplique.

---

## 6. Journal de quêtes

Un écran en deux colonnes : la liste des quêtes à gauche, le détail de la quête choisie à droite. Il s'ouvre et se ferme avec la touche **J**.

Le journal ne stocke aucun état. À chaque ouverture, il relit les tags du joueur dans `BP_DialogueManager` : une quête apparaît dès que le joueur a son `Active Tag`. C'est ce qui relie le journal aux dialogues sans code supplémentaire : le choix « Ca marche » donne `Quete.FemmeArme.Acceptee`, donc la quête s'affiche.

### Les assets

| Asset | Rôle |
|---|---|
| `Content/Blueprints/ClementTest/Quetes/BP_QuestData` | Modèle d'une quête : titre, donneur, lieu, description, tag d'activation, tag de fin |
| `…/Quetes/DA_Quest_FemmeArme` | La quête « Trouver le prisonnier » |
| `…/Quetes/BPC_QuestLog` | Composant du gestionnaire : remplit le widget, gère la sélection, l'ouverture et la fermeture |
| `Content/Widgets/ClementTest/Quetes/WBP_Quest` | La mise en page, sans aucun code |

### Ajouter une quête

1. Clic droit → Miscellaneous → Data Asset → `BP_QuestData`.
2. Remplis `Title`, `Location`, `Description`, et `Giver` (un Data Asset de personnage du système de dialogue).
3. `Active Tag` : le tag qui fait apparaître la quête. Donne-le depuis un choix de dialogue avec `TagsToGrant`, ou depuis n'importe quel Blueprint avec `AddFlag`.
4. Ouvre `BP_DialogueManager`, sélectionne le composant `QuestLog`, et ajoute ton Data Asset au tableau `Quests`. L'ordre du tableau est l'ordre d'affichage.

### Terminer une quête

Renseigne `Completed Tag` dans le Data Asset, puis donne ce tag au joueur quand l'objectif est atteint (`AddFlag` sur le gestionnaire). La quête reste dans la liste avec la mention « Terminée », et le compteur « Quêtes terminées » augmente.

Exemple en place : la quête de la femme à l'arme se termine par le dialogue. La mécanicienne donne le tag `Quete.FemmeArme.Indice` quand on l'interroge ; ce tag fait démarrer la femme à l'arme sur le nœud `FA_Rapport`, dont le choix « Sortir » donne `Quete.FemmeArme.Terminee`. Aucun Blueprint n'a été modifié pour ça : uniquement deux lignes de DataTable, les points d'entrée du PNJ et le `Completed Tag` de la quête.

### Si je veux changer…

| Ce que tu veux changer | Où |
|---|---|
| L'apparence | `WBP_Quest` : tout est à toi, sauf les noms `QuestButton1…5`, `QuestTitle1…5`, `QuestLocation1…5`, `DetailTitle`, `DetailGiver`, `DetailLocation`, `DetailDescription`, `DetailCounter`, `EmptyText`, `CloseButton` |
| La couleur de la ligne sélectionnée | `BP_DialogueManager` → composant `QuestLog` → `Selected Color` et `Normal Color` |
| Les textes fixes (« Quêtes terminées : », « Terminée », « Donnée par : ») | Même endroit : `Counter Label`, `Completed Label`, `Giver Prefix` |
| Plus de cinq quêtes visibles | Ajoute `QuestButton6`, `QuestTitle6`, `QuestLocation6` dans le widget, puis trois nœuds `Add` dans `BPC_QuestLog` → `CreateLogWidget` |
| La touche d'ouverture | `BP_DialogueManager` → Event Graph : remplace le nœud de touche `J`. Pense à changer aussi le texte du bouton « Fermer [J] » dans le widget |
| Ouvrir le journal autrement (bouton à l'écran, Input Action) | Appelle l'événement `ToggleQuestLog` du gestionnaire |

### Pièges

- Le journal ne s'ouvre pas pendant un dialogue : c'est voulu, les deux écrans se disputeraient le curseur.
- La touche J ne répond que si le gestionnaire existe, donc s'il y a au moins un PNJ avec `BPC_DialogueNPC` dans le niveau.
- Une entrée vide dans le tableau `Quests` provoque des messages « Accessed None » : supprime la ligne vide.
- Sans aucune quête, « Aucune quête en cours » s'affiche à gauche et aussi en titre à droite. Pour ne garder que celui de gauche, vide `No Quest Label` sur le composant `QuestLog`.
- L'état des quêtes est sauvegardé avec les tags. Touche F6 pour repartir de zéro.

---

### Pour l'équipe

`MechaGame.uproject` active trois plugins d'éditeur expérimentaux (`ModelContextProtocol`, `MCPClientToolset`, `AllToolsets`) et `.mcp.json` déclare le serveur local. Ils ont servi à construire les Blueprints et **ne sont pas nécessaires pour jouer ni pour modifier le système**. Tu peux ne pas commiter ces deux fichiers.
