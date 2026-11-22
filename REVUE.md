# Dossier pour revue externe

État complet de la recherche au **22/11/2026, 19 h 50**, rédigé pour qu'un relecteur qui n'a vu aucune session de travail puisse juger la méthode, les chiffres et les décisions. Tout ce qui suit est vérifiable dans ce dépôt, dans [journal.md](journal.md) ou dans la branche [`feat/lid-inference`](https://github.com/kilworks/kick-chat-translator/tree/feat/lid-inference) de l'extension.

Le raisonnement complet (d'où vient l'idée, options pesées, écarts au plan, erreurs) est dans [PROCESSUS.md](PROCESSUS.md).

Ce qui est **mesuré** est écrit au présent avec son chiffre. Ce qui est **supposé** est marqué hypothèse. Ce qui **n'est pas fait** est dans « Pas encore fait ».

---

## 1. Le besoin

[Kick Chat Translator](https://github.com/kilworks/kick-chat-translator) (Chrome, Firefox, Brave) traduit le chat de Kick dans la langue de chaque spectateur. Avant de traduire, l'extension doit savoir dans quelle langue est chaque ligne : pour la cacher si elle est déjà dans la langue du lecteur ou dans une langue qu'il a exclue, pour afficher un drapeau, et pour donner (ou non) la langue source au service de traduction.

Le détecteur actuel (branche `master`, version 3.6.2) combine `franc-min`, des tables de mots et, quand le navigateur en a un, son détecteur (CLD sur Chrome et Firefox ; rien sur Brave). Il se trompe sur 24 % des phrases Tatoeba et 17 % des lignes de chat réel annotées.

## 2. Contraintes fixées par le propriétaire du projet

1. **Rien d'externe côté utilisateur** : pas de serveur, pas d'Ollama, pas de modèle à télécharger à part. Le modèle est livré dans l'extension.
2. **Même gain pour tous** : Chrome, Firefox et Brave (Brave n'a aucun détecteur intégré, c'est le chemin le plus exposé).
3. **Ordre** : « A puis B si résidu ». A = modèle entraîné livré dans l'extension. B = LanguageDetector ou Gemini Nano de Chrome, seulement si un écart chiffré le justifie après A.
4. **Recherche, pas de livraison** : le code de l'extension (`src/`) n'est pas modifié pendant la recherche ; tout vit dans `tools/lid/` et dans des données hors git. Sortie, fusion et publication sur les stores restent sa décision.
5. **Vie privée** : aucune ligne de chat, aucun pseudo, aucun nom de chaîne publiés ; seuls des agrégats.

## 3. Données

| Jeu | Contenu | Rôle | Publié |
|---|---|---|---|
| Tatoeba | 42 langues, 20 000 phrases max par langue, CC BY 2.0 FR | entraînement + banc de 5 040 phrases | oui (source publique) |
| Augmentations | minuscules, sans ponctuation, fenêtres de 1 à 4 mots, sans accents, hindi et bengali romanisés synthétiques (`romanize.py`) | rapprocher Tatoeba du registre du chat | code oui |
| **dev** | 1 007 lignes annotées, dont 711 dans une seule langue (les seules comptées au coût), captures du 04/11 | choix du modèle, des seuils, de k ; température sur sa moitié paire | agrégats seulement |
| **heldout** | 993 lignes de chat Kick annotées, mises de côté | **lecture unique à la fin** pour ce modèle | jamais lu ligne à ligne ; déjà mesuré en agrégats pour le détecteur `master` (voir §3 bis) |
| Chat non étiqueté (E9) | voir §7 | auto-apprentissage, simulation de l'adaptation | agrégats seulement |

Garde : aucun texte de dev ni de heldout n'entre à l'entraînement (exclusion au texte normalisé, testée par `test_dataset.py`).

## 3 bis. Comment dev et heldout ont été annotés

- **Échantillon** : 2 000 lignes tirées des captures Kick du 04/11 (`test/e2e/real-bench.mjs sample`), sans la langue déclarée de la chaîne (l'annotateur ne voit que l'id et le texte).
- **Deux annotateurs indépendants, deux consignes différentes** : A « linguiste de terrain » (`consigne-a.md`), B « modérateur d'une équipe de traduction » (`consigne-b.md`). Mêmes champs : langue (40 codes, ou `neutral`, `mixed`, `other`), variante, argot, romanisé, alternance de langues, émotes seules, commande de bot, anglais de chat. Les consignes sont rédigées comme des instructions pour un modèle de langage ; **qui a annoté (humain ou modèle, lequel) reste à confirmer par le propriétaire**.
- **Accord** (kappa de Cohen sur 2 000 lignes) : langue 0,96 (accord brut 96,35 %), variante 0,92, romanisé 0,92, anglais de chat 0,81, alternance 0,73, argot 0,71.
- **Désaccords** : 478 lignes ont au moins un champ en désaccord (`disagreements.jsonl`). **Aucun n'a été tranché.** Un champ n'est retenu que si les deux annotateurs sont d'accord, sinon il vaut `null` et la ligne sort du coût.
- **Coupe par capture** (jamais par ligne) : dev 1 007 lignes, heldout 993 (empreinte sha256 vérifiée).
- **Composition de dev** : 711 lignes dans une seule langue (comptées au coût), 209 neutres, 49 « autre » (langue hors liste : serbe, mongol, malais...), 1 mélange, le reste `null` (désaccord). Les 296 lignes hors coût sont mesurées à part (§ analyses).
- **Heldout n'est pas vierge** : jamais lu ligne à ligne, mais ses totaux ont été calculés pour le détecteur `master` le 04/11 (base : sûr 359 justes, 338 silences, 3 fausses sur 700 lignes à langue d'accord ; brut 428 / 156 / 116), puis le 05/11 une dizaine de fois pour accepter ou refuser des règles de `master` (trois règles refusées sur un écart d'une ligne dans heldout). Rien de cette recherche ne l'a encore touché, mais `master`, la base de comparaison, a été en partie réglée dessus.

## 4. Fonction de coût (fixée avant toute mesure)

| Événement | Coût |
|---|:---:|
| Message caché à tort | 10 |
| Mauvaise langue source envoyée au cloud | 5 |
| Mauvais drapeau | 1 |
| Ligne gardée alors que sa langue est hors de la liste blanche du lecteur | 1 |
| Silence (le cloud détecte seul) | 0,5 |

La spécification prévoyait aussi un coût de 4 pour une mauvaise langue source imposée au moteur sur l'appareil. Le simulateur (`cost.py`) ne l'applique pas : le mode par défaut de l'extension est « cloud d'abord » depuis la 3.5.0, ce cas n'arrive donc pas avec les réglages simulés. Le coût 1 de la ligne gardée hors liste blanche ne concerne que le lecteur à liste blanche.

Coût moyenné par ligne et rapporté sur 3 chemins : aucun détecteur (Brave), Chrome, Firefox.
**Contrainte dure** : le nombre de messages cachés à tort ne doit monter pour **aucun** lecteur (langue du lecteur × langues exclues) ni pour **aucun** chemin. Les seuils (détection, envoi de la langue source) sont choisis pour minimiser le coût sous cette contrainte.

**Hypothèse non vérifiée** : le poids 0,5 du silence suppose que donner la langue source au cloud améliore la traduction. Pas mesuré sur de vraies traductions (voir §9).

## 5. Modèle

- Entrée : n-grammes de caractères 1 à 4 et mots (préfixe `w:`), sur UTF-8, hachés FNV-1a dans 2^B cases (`& (2^B - 1)`), moyenne des plongements.
- Classifieur : régression logistique multinomiale, Adam paresseux, calibration par température (sur dev), poids int8 quantifiés par ligne. Format binaire `LID1`.
- Taille : 1,5 Mo (2^15) à 2,9 Mo (2^16). Inférence TypeScript pur sans dépendance, moins d'une milliseconde par ligne.
- Intégration testée (branche, pas `master`) : 20/20 tests de `src/content/lid` verts, `tsc` propre.
- Démo dans le navigateur : https://kilworks.github.io/kct-detection-langue/ (modèle `base-b15`, plus ancien que le meilleur, à remplacer).

## 6. Résultats (dev ; optimistes par construction)

Coût moyen par ligne, chemin sans détecteur (Brave), `master` = **0,531**.

| Modèle | Coût seul | Coût adapté (causal, k=10) | dev | banc Tatoeba |
|---|:---:|:---:|:---:|:---:|
| base-b15 (E3) | 0,329 | | 85,1 % | 97,9 % |
| base-b16 (E1b) | 0,318 | 0,251 (non causal) | 85,2 % | 98,2 % |
| **accrom-b16** (E8, référence actuelle) | 0,306 | 0,210 | 87,8 % | 98,17 % |
| st1-b16 (E9a, Kick + Twitch pseudo) | 0,284 | **0,189** | 89,3 % | 98,06 % |
| st-twitch-b16 (E9b, Twitch seul) | 0,275 | 0,200 | 89,2 % | 98,5 % |
| st-kick-b16 (E9b, Kick seul) | 0,331 | 0,234 | 88,7 % | |
| nn-b16-d32-h128 (E7, réseau) | 0,410 | | 83,3 % | 98,3 % |

Autres mesures clés :
- E3 : réponses fausses sans détecteur 121 → 23 sur 711 (107 corrigées, 9 cassées, McNemar χ² = 81) ; IC95 du gain de coût [-0,213 ; -0,192].
- E3b : sur Chrome et Firefox, presque tout le gain vient de l'envoi de la langue source (`sl`), d'où l'importance de l'hypothèse du §4.
- E6c : adaptation **causale** par vraie chaîne (seules les lignes captées avant la ligne jugée comptent), 515 chaînes, médiane 22 lignes : 0,306 → 0,210 ; Chrome 0,205, Firefox 0,222 ; langues sources fausses 15 → 11.
- E9b (contrôle de fuite) : dev vient de captures Kick, Twitch ne partage aucune chaîne avec lui. Twitch seul fait **mieux** que le mélange : le gain de l'auto-apprentissage n'est pas une fuite. Les pseudo-lignes Kick **nuisent** (0,331) : les chaînes « English (India) » écrivent en hinglish, étiqueté anglais ; filtre corrigé depuis (112 des 381 lignes Kick « en » retirées).

Analyse d'erreurs (st1-b16 vs accrom-b16, dev) : anglais de chat 76 → 84 %, argot 80 → 82 %, lignes de 2 mots ou moins 83 → 85 %, **mélange de langues 69 % et romanisé 57 % inchangés** ; hindi romanisé lu comme anglais 8 fois sur 23 ; persan romanisé (finglish) jamais vu à l'entraînement.

## 7. Collecte E9 (en cours au moment de la rédaction)

But : beaucoup de chat réel non étiqueté, dans le plus de langues possible, pour l'auto-apprentissage et pour simuler l'adaptation sur bien plus de chaînes. **Jamais pour évaluer.**

État à 19 h 46 : **990 245 lignes brutes**, 45 langues déclarées.

| Source | Lignes | Chaînes |
|---|:---:|:---:|
| Kick (navigateur sur PC) | 249 859 | 492 par tour (40 max par langue) |
| Twitch, lecteur IRC sur PC | 513 270 | 477 |
| Twitch, lecteurs IRC sur un serveur cloud (8 processus) | 89 489 | ~960 |
| Twitch, lecteurs IRC sur 2 VM cloud (6 processus chacune) | 137 627 | ~1 440 |

Répartition par langue **déclarée par la chaîne** (pas la langue de chaque ligne) :
en 317 830 · es 118 840 · de 85 264 · ru 83 273 · ar 66 844 · fr 56 958 · ja 42 643 · pl 42 292 · nl 24 396 · pt 23 969 · it 21 627 · tr 15 673 · en-IN 15 210 · non déclarée 11 734 · zh 8 628 · cs 7 481 · hu 7 406 · th 4 663 · da 4 562 · tl 3 980 · fi 3 619 · ro 3 178 · no 2 471 · fa 2 201 · sv 2 012 · id 2 008 · sk 1 668 · bg 1 414 · uk 1 391 · hi 1 303 · he 1 028 · sr 836 · el 810 · ml 567 · mn 524 · lt 368 · ca 308 · vi 288 · ko 246 · ne 193 · sq 170 · bs 135 · mk 130 · lv 81 · hr 23

Méthode : voir le protocole à la fin de [journal.md](journal.md). Points saillants :
- Kick : points publics du site (ceux que le site appelle lui-même), navigateur visible ; repli automatique quand plus de 10 % des requêtes sont refusées. **Une montée en charge trop forte a fait limiter l'IP** (78 à 90 % de refus depuis) : la part Kick n'avance presque plus.
- Twitch : IRC anonyme officiel, lecture seule ; annuaire public trié par spectateurs, 32 langues de diffusion ; l'API interne GQL n'a pas été utilisée.
- Hygiène : émotes retirées, mentions remplacées par `@u<n>`, chaînes renumérotées, dédoublonnage au texte normalisé, exclusion de dev et heldout.

## 8. Biais et menaces connus

1. **dev sert presque à tout** (seuils, k, choix du modèle ; la température est ajustée sur sa moitié paire et son ECE lue sur la moitié impaire) : les chiffres de dev sont optimistes. Seul heldout tranchera.
2. **Langue déclarée ≠ langue de la ligne** : l'anglais est partout, les chaînes mélangent. Le filtre « accord » (p ≥ 0,9 et langue déclarée d'accord) limite le bruit mais exclut justement les lignes difficiles (mélange, romanisé).
3. **Auto-apprentissage = le modèle apprend ses propres certitudes** : il renforce ce qu'il sait déjà ; il ne peut pas apprendre seul le romanisé ou le mélange (constaté : inchangés).
4. **Gros streams surreprésentés**, une seule fenêtre horaire (fuseaux), quelques langues très minces (hr 23 lignes, lv 81).
5. **Kick sous-représenté dans la collecte récente** (IP limitée), alors que l'extension sert Kick. Twitch seul a pourtant donné le meilleur coût seul (E9b).
6. **Simulation de l'adaptation** sur les captures de dev (515 chaînes, médiane 22 lignes) : l'extension en direct voit des chaînes plus longues, ce qui devrait aider, mais ce n'est pas mesuré.

## 9. Pas encore fait

- Réentraîner sur la collecte complète (3 variantes : Twitch seul, Kick filtré, les deux) et choisir sous la contrainte.
- Mesurer sur de vraies traductions si la langue source aide le cloud (le poids 0,5 du silence).
- Porter l'adaptation par chaîne dans l'extension (état par chaîne en TypeScript).
- Lire heldout, une fois.
- B (Chrome LanguageDetector, Gemini Nano) : seulement si un résidu chiffré reste après A.
- Remplacer le modèle de la démo par le meilleur.

## 10. Questions précises pour le relecteur

1. La fonction de coût et la contrainte « aucune hausse des messages cachés, par lecteur et par chemin » vous semblent-elles bien posées ? Le 0,5 du silence est-il défendable sans mesure de traduction ?
2. Choisir seuils, k et modèle sur le même dev (la température seule est ajustée sur une moitié) : quel risque réel, et faut-il couper dev en deux avant de lire heldout ?
3. Auto-apprentissage : faut-il plafonner par langue, pondérer par chaîne (une chaîne bavarde ne doit pas dominer), ou passer à un second tour (réétiqueter avec le nouveau modèle) ?
4. Romanisé et mélange de langues stagnent : données synthétiques plus riches, classe « mélange » explicite, ou laisser ces lignes au silence ?
5. Le réseau (E7) perd au coût parce qu'il est trop sûr de lui : une calibration plus forte ou un mélange linéaire + réseau mérite-t-il un essai, ou le linéaire suffit-il ?
6. Le contrôle de fuite E9b (Twitch seul) est-il suffisant pour écarter un biais de chaînes communes avec dev ?
7. Avec l'adaptation, le modèle envoie une langue source au cloud pour 145 lignes neutres sur 209, que le coût ne compte pas (§10 bis) : faut-il les compter, ou interdire `sl` sans lettre significative ?
8. Heldout a déjà servi à régler le détecteur de base (`master`) : reste-t-il un juge acceptable pour la comparaison finale, ou faut-il annoter un nouveau jeu ?

## 10 bis. Analyses complémentaires du 22/11 (évaluation seule, aucun entraînement)

Produites par `tools/lid/report.py` et `tools/lid/ft_bench.py` (branche `feat/lid-inference`) sur dev, avec les modèles déjà entraînés. Chaque configuration est la meilleure de la grille sous la contrainte des messages cachés, comme dans `cost.py` et `adapt.py`. Adaptation : causale, k = 10, seuil de sûreté 0,9.

### Intervalles de confiance : le bootstrap de `cost.py` était trop optimiste

`cost.py` rééchantillonnait par fichier de capture. Dev n'en contient que **2** : ses intervalles ne valaient rien. Refaits ici en rééchantillonnant les **232 vraies chaînes** de dev (bootstrap apparié, 2 000 tirages).

| Comparaison (coût par ligne, chemin sans détecteur) | Écart | IC 95 % par chaîne |
|---|---|---|
| accrom-b16 seul contre `master` | 0,531 → 0,296 | [-0,298 ; -0,176] |
| accrom-b16 adapté contre `master` | 0,531 → 0,203 | [-0,393 ; -0,268] |
| accrom-b16 adapté contre accrom-b16 seul | 0,296 → 0,203 | [-0,128 ; -0,063] |
| st1-b16 seul contre accrom-b16 seul | 0,296 → 0,276 | [-0,057 ; +0,021] |
| st-twitch-b16 seul contre accrom-b16 seul | 0,296 → 0,263 | [-0,072 ; +0,008] |
| st-kick-b16 seul contre accrom-b16 seul | 0,296 → 0,313 | [-0,015 ; +0,049] |
| st1-b16 adapté contre accrom-b16 adapté | 0,203 → 0,182 | [-0,045 ; -0,001] |
| st-twitch-b16 adapté contre accrom-b16 adapté | 0,203 → 0,192 | [-0,037 ; +0,018] |

Lecture : le modèle et l'adaptation par chaîne battent `master` nettement, sur les trois chemins (Chrome et Firefox donnent des intervalles du même signe, voir `report.json`). En revanche, **l'auto-apprentissage (E9) n'a pas de gain établi** sur accrom-b16 : seul st1-b16 adapté passe tout juste sous zéro. Les classements E9a et E9b (« Twitch seul fait mieux ») sont dans le bruit de dev.

### Confusion de taille dans E9b

st-twitch-b16 a reçu 42 641 pseudo-lignes, st1-b16 28 759 et st-kick-b16 13 251 (chacune copiée 5 fois). Comparer Twitch et Kick revient donc aussi à comparer trois tailles. Le contrôle propre (sous-échantillonner Twitch à 13 251 lignes) demande un entraînement : **pas fait**, consigne du propriétaire (collecte seulement pour l'instant). Vu les intervalles ci-dessus, la question est de toute façon secondaire.

### Nombre de configurations essayées sur dev

Ce rapport à lui seul en évalue 864 (4 modèles × seul ou adapté × 108 points de grille). Sur toute la recherche : 12 modèles enregistrés, chacun passé dans la grille de 108 points, plus plusieurs valeurs de k (10, 50), deux clés d'adaptation (langue déclarée, chaîne), deux préréglages de loi a priori. **Au moins 1 300 configurations** ont été comparées sur les mêmes 711 lignes ; le chiffre exact n'a pas été tracé. C'est la raison principale de ne croire que heldout.

### Messages cachés à tort, par lecteur et par chemin (accrom-b16 adapté)

| Chemin | Lecteur fr | Lecteur en | Lecteur fr à liste blanche (en, es, pt) | Autres lecteurs |
|---|---|---|---|---|
| Sans détecteur | 1 → 1 | 6 → 1 | 68 → 39 | 0 → 0 |
| Chrome | 1 → 1 | 1 → 1 | 95 → 38 | 0 → 0 |
| Firefox | 1 → 1 | 12 → 8 | 70 → 34 | 0 → 0 |

(`master` → modèle.) La contrainte tient partout ; l'essentiel du gain vient du lecteur à liste blanche.

### Par langue (accrom-b16 adapté, sans détecteur, dev)

| Langue | Lignes | Justes | Silences |
|---|---|---|---|
| en | 91 | 69 | 21 |
| ja | 76 | 74 | 2 |
| th | 61 | 61 | 0 |
| fi | 48 | 42 | 5 |
| tr | 47 | 42 | 5 |
| pl | 46 | 42 | 4 |
| ar | 45 | 42 | 1 |
| pt | 39 | 28 | 10 |
| he | 32 | 32 | 0 |
| ru | 28 | 17 | 9 |
| es | 26 | 20 | 5 |
| id | 22 | 14 | 8 |
| cs | 21 | 15 | 5 |
| hi | 21 | 8 | 13 |
| sk | 19 | 18 | 1 |
| ro | 15 | 12 | 2 |

Les réponses fausses sont rares et dispersées (11 sur 711 ; la plus fréquente : ru lu bg, 2 fois). Le point faible est le **silence** : anglais (21), hindi (13, presque tout du romanisé), portugais (10), russe (9, souvent translittéré).

### Ligne courte (20 caractères ou moins) et calibration

- Coût sur les lignes courtes, sans détecteur : `master` 0,513, accrom-b16 seul 0,359, adapté 0,233.
- ECE (moitié impaire de dev, 10 classes) : 0,205 avant température, 0,022 après (accrom-b16, T = 0,564). Même ordre pour les autres modèles (0,016 à 0,029 après).

### Les 296 lignes de dev hors du coût

209 neutres, 49 « autre », 1 mélange, 37 sans accord des annotateurs. Le coût ne les voit pas ; voici ce que la décision en fait (sans détecteur) :

| | `master` | accrom-b16 adapté |
|---|---|---|
| Neutres : réponse donnée | 47 / 209 | 147 / 209 |
| Neutres : langue source envoyée au cloud | 4 | **145** |
| Neutres : réponse dans une langue de lecteur (cachées pour ce lecteur) | 38 | 10 |
| « Autre » (serbe, mongol, malais...) : réponse donnée | 25 / 49 | 12 / 49 |
| « Autre » : réponse dans une langue de lecteur | 9 | 4 |

Deux lectures. En bien : le modèle cache moins de lignes neutres et de langues hors liste. **En mal : avec l'adaptation, il envoie une langue source au cloud pour 145 lignes neutres** (« gg », « lol », émotes : la loi a priori de la chaîne les pousse au-dessus du seuil). Le coût est nul par construction sur ces lignes ; l'effet réel sur la traduction n'est pas mesuré. À corriger probablement par une règle « pas de `sl` sans lettre significative », ou à compter dans le coût.

### Base de comparaison forte : fastText lid.176

Le modèle public `lid.176.ftz` (176 langues, CC BY-SA 3.0, jamais destiné à être livré) branché à la place du nôtre, mêmes seuils cherchés, même contrainte :

| | Langue juste (argmax brut, 711 lignes) | Coût moyen sous contrainte |
|---|---|---|
| fastText lid.176 | 565 (79 %) | 0,390 |
| accrom-b16 | 624 (88 %) | 0,306 |
| st-twitch-b16 | 634 (89 %) | 0,275 |
| `master` | | 0,465 |

Le modèle entraîné pour le chat bat la référence générique ; fastText reste meilleur que `master`. Texte passé en minuscules aux deux. Le détecteur de Chrome (LanguageDetector) n'est pas mesuré ici.

### Coût dans l'extension

Mesuré en Node 22 sur le PC de développement (Ryzen 9 9900X3D), code de `src/content/lid/model.ts` non modifié, 1 007 lignes de dev × 5 :

| Modèle | Taille dans le paquet (base64) | Décodage au chargement | Par ligne p50 / p95 / p99 | Tas JS ajouté |
|---|---|---|---|---|
| base-b15 (installé aujourd'hui) | 2,0 Mo | 26 ms | 0,014 / 0,046 / 0,118 ms | 13 Mo |
| accrom-b16 | 4,0 Mo | 37 ms | 0,013 / 0,044 / 0,088 ms | 15 Mo |

La spécification demandait un p95 sous 0,3 ms : tenu avec une large marge sur cette machine. Non mesuré : navigateur réel, machine modeste, Firefox. Le passage à 2^16 cases double le poids du paquet (2 → 4 Mo).

### Le simulateur face à l'extension

Contrairement à ce que disait une première version de PROCESSUS §9, une vérification existe : `src/content/lid/costParity.test.ts` et `tools/lid/test_cost.py` lisent les mêmes 9 cas (`cost-cases.json`) et vérifient que les **règles de suppression** du simulateur sont celles du pipeline réel. Ce n'est pas le contrôle prévu (50 lignes réelles), et les autres coûts (drapeau, `sl`, silence) ne sont pas vérifiés contre l'extension.

## 10 ter. Reproduire

Données hors git, dans un dossier `<data>` (ici `C:\Users\kil\kct-data`). Python 3.10, numpy 2.2, Node 22. Graines fixes : `dataset.py` (`random.Random(0)`), `train.py` (`default_rng(0)`), `pseudo.py` (`default_rng(0)`).

```
python tools/lid/fetch_tatoeba.py --data <data>          # export per_language de downloads.tatoeba.org
python tools/lid/dataset.py --data <data> --accents --romanized
node --import tsx tools/lid/featurize.mjs --data <data>
python tools/lid/train.py --data <data> --bits 16 --save accrom-b16
node --import tsx tools/lid/answers.mjs --data <data>
python tools/lid/cost.py --data <data> --model accrom-b16 [--pick 0.8,0.9,silence]
python tools/lid/adapt.py --data <data> --model accrom-b16 --causal --k 10
python tools/lid/report.py --data <data> --out report.json
python tools/lid/ft_bench.py --data <data> --ft lid.176.ftz
```

La date de l'export Tatoeba n'a pas été notée au premier téléchargement (les fichiers locaux portent une date de réextraction). Les captures Kick du 04/11 et le jeu annoté ne sont pas publiés : la reproduction complète demande le dossier de données du propriétaire.

## 10 quater. Données collectées et cadre

- **Volume** : environ 1,25 million de lignes de chat brutes (112 Mo) au 22/11, plus les 28 000 lignes des captures du 04/11.
- **Stockage** : uniquement sur le PC du propriétaire, hors de tout dépôt git. Les machines distantes (serveur, VM) ne gardent rien : chaque cycle efface son dossier après rapatriement.
- **Kick** : points d'accès publics du site, lus par un navigateur classique, sans compte, rythme ralenti dès que le site refuse (taux de refus élevé, environ 78 %).
- **Twitch** : IRC officiel en lecture anonyme (`justinfan`), sans compte, sans l'API interne.
- **Ce qui est publié** : agrégats seulement. Aucun pseudo, aucun nom de chaîne, aucune ligne de chat.
- **Conditions d'utilisation de Kick et Twitch** : pas examinées juridiquement pour cet usage (entraînement d'un modèle).
- **Durée de conservation** : **non fixée, à décider par le propriétaire.**

## 11. Où est le code

| Quoi | Où (branche `feat/lid-inference`) |
|---|---|
| Spécification et plan | [`docs/superpowers/specs`](https://github.com/kilworks/kick-chat-translator/tree/feat/lid-inference/docs/superpowers/specs), [`plans`](https://github.com/kilworks/kick-chat-translator/tree/feat/lid-inference/docs/superpowers/plans) |
| Inférence et intégration | [`src/content/lid/`](https://github.com/kilworks/kick-chat-translator/tree/feat/lid-inference/src/content/lid) |
| Jeu d'entraînement, romanisation | `tools/lid/dataset.py`, `romanize.py`, `fetch_tatoeba.py` |
| Hachage et entraînement | `features.py`, `featurize.mjs`, `train.py`, `train_nn.py`, `blob.py` |
| Coût, seuils, adaptation | `cost.py`, `adapt.py` (`--causal`), `prior.py`, `answers.mjs` |
| Collecte et auto-apprentissage | `kick_capture.mjs`, `twitch_irc.py`, `remote_collect.py`, `pool.mjs`, `pseudo.py`, `captures.mjs` |
| Suivi | `watch_collect.py` sur [tui-dash](https://github.com/kilworks/tui-dash) |
| Page de démo et ses données | [`docs/`](docs) de ce dépôt (`viz_export.py`) |

## 12. Erreurs commises en route (et corrigées)

- **E2** : le premier banc tournait sans modèle installé et mesurait en partie `franc` ; chiffre retiré, une garde refuse maintenant de mesurer sans modèle.
- **E6** : l'adaptation était simulée par langue déclarée (plusieurs chaînes mélangées) et annonçait 0,214 ; refaite par vraie chaîne (E6b), puis en causal (E6c, 0,210).
- **Collecte Kick** : intervalle trop court et trop de chaînes à la fois, IP limitée par Kick ; repli automatique ajouté après coup.
- **Annuaire Twitch** : le premier relevé utilisait un mauvais filtre et un tri « recommandé », d'où des petites chaînes ; refait par case de langue et tri par spectateurs.
