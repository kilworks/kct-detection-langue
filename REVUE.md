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
| **dev** | 711 lignes de chat Kick réel annotées dans une langue (hors neutre, autre, mélange), captures du 04/11 | choix du modèle, de la température, des seuils, de k | agrégats seulement |
| **heldout** | 993 lignes de chat Kick annotées, mises de côté | **lecture unique à la fin** | non lu |
| Chat non étiqueté (E9) | voir §7 | auto-apprentissage, simulation de l'adaptation | agrégats seulement |

Garde : aucun texte de dev ni de heldout n'entre à l'entraînement (exclusion au texte normalisé, testée par `test_dataset.py`).

## 4. Fonction de coût (fixée avant toute mesure)

| Événement | Coût |
|---|:---:|
| Message caché à tort | 10 |
| Mauvaise langue source envoyée au cloud | 5 |
| Mauvaise langue source imposée au moteur sur l'appareil | 4 |
| Mauvais drapeau | 1 |
| Silence (le cloud détecte seul) | 0,5 |

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

1. **dev sert à tout** (température, seuils, k, choix du modèle) : les chiffres de dev sont optimistes. Seul heldout tranchera.
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
2. Choisir température, seuils et k sur le même dev : quel risque réel, et faut-il couper dev en deux avant de lire heldout ?
3. Auto-apprentissage : faut-il plafonner par langue, pondérer par chaîne (une chaîne bavarde ne doit pas dominer), ou passer à un second tour (réétiqueter avec le nouveau modèle) ?
4. Romanisé et mélange de langues stagnent : données synthétiques plus riches, classe « mélange » explicite, ou laisser ces lignes au silence ?
5. Le réseau (E7) perd au coût parce qu'il est trop sûr de lui : une calibration plus forte ou un mélange linéaire + réseau mérite-t-il un essai, ou le linéaire suffit-il ?
6. Le contrôle de fuite E9b (Twitch seul) est-il suffisant pour écarter un biais de chaînes communes avec dev ?

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
