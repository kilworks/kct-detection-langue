# Le processus, en entier

Ce document raconte **comment** la recherche a été menée : d'où vient l'idée, quelles options ont été pesées, pourquoi chaque décision a été prise, ce qui a été changé en route, et où le travail s'écarte de son propre plan. Il est volontairement long et franc, pour qu'un relecteur puisse critiquer le raisonnement et pas seulement les chiffres.

Les chiffres eux-mêmes sont dans [REVUE.md](REVUE.md) et [journal.md](journal.md). Ici, on parle de méthode.

**Qui a fait quoi.** Le propriétaire du projet (développeur de l'extension) a fixé le but, les contraintes et les arbitrages. Le travail d'exécution (lecture du code, conception, scripts, entraînements, mesures, collecte, rédaction) a été fait par un agent IA (Claude, dans Claude Code), sous sa direction, en une longue session le 22/11/2026. Ce point compte pour la critique : un agent a tendance à optimiser ce qu'il mesure, et à mesurer ce qui est facile ; plusieurs erreurs racontées plus bas en viennent.

---

## Table

1. [D'où vient l'idée](#1-doù-vient-lidée)
2. [Reformuler le problème : ce qu'une erreur coûte vraiment](#2-reformuler-le-problème--ce-quune-erreur-coûte-vraiment)
3. [Les options pesées pour le modèle](#3-les-options-pesées-pour-le-modèle)
4. [Les données : ce qu'on avait, ce qu'il manquait](#4-les-données--ce-quon-avait-ce-quil-manquait)
5. [Le protocole de mesure](#5-le-protocole-de-mesure)
6. [Déroulé des expériences, dans l'ordre](#6-déroulé-des-expériences-dans-lordre)
7. [L'adaptation par chaîne : l'idée la plus rentable](#7-ladaptation-par-chaîne--lidée-la-plus-rentable)
8. [L'auto-apprentissage et la collecte massive](#8-lauto-apprentissage-et-la-collecte-massive)
9. [Écarts entre le plan écrit et ce qui a été fait](#9-écarts-entre-le-plan-écrit-et-ce-qui-a-été-fait)
10. [Erreurs de raisonnement commises](#10-erreurs-de-raisonnement-commises)
11. [Ce que je ferais autrement](#11-ce-que-je-ferais-autrement)
12. [Points où la critique est la plus utile](#12-points-où-la-critique-est-la-plus-utile)

---

## 1. D'où vient l'idée

### Le contexte produit

Kick Chat Translator traduit le chat de Kick (et, dans une extension sœur, de Twitch) en direct, dans la langue de chaque spectateur. Le même jour, un chantier de performance venait de se terminer (version 3.6.2) : il avait montré que sur un chat rapide (20 à 25 messages par seconde), le budget de traduction par chaîne refuse 66 à 73 % des envois. Le propriétaire a alors listé des pistes de traitement de la langue « au service de la lecture du spectateur » (pas d'outils pour streamers) :

1. écarter les lignes sans contenu avant traduction (récupérer du budget) ;
2. résumer une rafale ;
3. humeur du chat ;
4. un modèle de sentiment (exclu : poids, dépendance).

Il a ajouté une cinquième piste, qui est devenue ce projet : **les faux positifs de langue**. Quand l'extension se trompe de langue, deux choses arrivent, et les deux sont visibles par l'utilisateur :

- un message est **caché** alors qu'il ne devait pas l'être (l'extension croit qu'il est déjà dans la langue du lecteur, ou dans une langue que le lecteur a exclue, ou en anglais alors que l'option « ignorer l'anglais » est cochée) ;
- le moteur de traduction reçoit une **langue source fausse** et rend du charabia, ou le texte inchangé.

### Ce que disait le code avant

Lecture du code de `master` (3.6.2) :

- la détection repose sur `franc-min` (statistique de trigrammes, pensée pour des textes longs) et sur des tables de mots écrites à la main ;
- quand le navigateur en a un, son détecteur intégré (`chrome.i18n.detectLanguage`, CLD, sur Chrome et Firefox) sert de contre-vérification ; Brave n'en a pas ;
- un banc existant (`langMatrix`, 5 040 phrases Tatoeba, 42 langues) donnait déjà le chiffre : **franc se trompe sur 24 % des phrases, et 29 % des phrases courtes**. Confusions typiques : danois/suédois, catalan/espagnol, bulgare ou ukrainien lus comme russe ;
- une langue source (`sl`) n'est imposée au moteur cloud que si la détection est jugée « sûre », et une traduction identique à l'original est jetée.

### Première version du plan, rejetée

Une première version du plan proposait des règles supplémentaires et de brancher l'API LanguageDetector de Chrome comme colonne de banc. Le propriétaire l'a rejetée (« prompt de débutant, je veux de l'inférence »). Une deuxième version proposait un élève distillé à partir d'un grand modèle local (Qwen) servant de professeur, avec un modèle d'agrégation d'annotateurs (Dawid-Skene). Elle a été recadrée par une contrainte forte : **rien d'externe côté utilisateur**, pas d'Ollama, pas de Qwen. Le chemin retenu :

> **A puis B si résidu.** A = un modèle entraîné, livré dans l'extension, qui tourne chez chaque utilisateur sur les trois navigateurs. B = les modèles intégrés de Chrome (LanguageDetector, puis Gemini Nano), seulement si une mesure montre qu'il reste un écart que A ne comble pas.

**Point de critique possible** : la contrainte « rien d'externe » vise l'utilisateur. Un professeur lourd utilisé **hors ligne, à l'entraînement seulement**, ne l'aurait pas violée (l'utilisateur ne reçoit que des poids). Cette porte a été fermée tôt et jamais rouverte ; l'auto-apprentissage (§8) a pris sa place, avec un professeur beaucoup plus faible : le modèle lui-même.

---

## 2. Reformuler le problème : ce qu'une erreur coûte vraiment

### Pourquoi pas la précision

La métrique naturelle serait la précision (pourcentage de lignes bien classées). Elle a été écartée comme critère de choix, pour trois raisons :

1. **Les erreurs n'ont pas le même prix.** Cacher un message que le lecteur voulait lire est bien pire qu'afficher un mauvais drapeau.
2. **Se taire est une option.** Si le modèle n'est pas sûr, l'extension peut ne rien décider et laisser le service cloud détecter la langue lui-même. La précision ne sait pas récompenser un silence prudent.
3. **Le même modèle n'a pas le même effet selon le navigateur** : sur Chrome et Firefox, CLD contre-vérifie ; sur Brave, rien.

### La table de coût

Fixée **avant toute mesure** (dans la spécification, avec la mention « non modifiée après avoir vu les résultats ») :

| Événement | Coût | Raisonnement |
|---|:---:|---|
| Ligne cachée à tort | 10 | perte d'information irréversible pour le lecteur |
| Mauvaise langue source envoyée au cloud | 5 | traduction absurde, mais le lecteur voit au moins l'original |
| Mauvais drapeau, rien de caché | 1 | gêne visuelle |
| Silence (le cloud détecte) | 0,5 | on perd un éventuel gain de qualité à donner `sl` |
| Ligne gardée alors que sa langue est hors liste blanche (lecteur à liste blanche seulement) | 1 | le lecteur a demandé à ne pas la voir |
| Bonne réponse | 0 | |

La spécification prévoyait aussi un coût de 4 pour une mauvaise langue source imposée au moteur sur l'appareil. Le simulateur (`cost.py`) ne l'applique pas : le mode par défaut de l'extension est « cloud d'abord » depuis la 3.5.0, ce cas n'arrive donc pas avec les réglages simulés. Le coût 1 de la ligne gardée hors liste blanche ne concerne que le lecteur à liste blanche.

Le coût est simulé sur un panel de lecteurs (langues cibles fr, es, pt, de, en, ru, tr, ar, réglages par défaut, plus un lecteur avec une liste blanche de langues sources), puis moyenné par ligne, pour chacun des trois chemins (aucun détecteur, Chrome, Firefox).

### La contrainte dure

> Le nombre de lignes cachées à tort ne doit monter **pour aucun lecteur** et **sur aucun chemin**.

Raison : un gain moyen peut cacher une régression pour une minorité (par exemple les lecteurs arabophones). On préfère un gain plus petit mais sans perdant.

### L'hypothèse la plus fragile

Le poids **0,5 du silence** suppose que donner la bonne langue source au cloud améliore la traduction. Si ce n'est pas le cas (si le cloud détecte aussi bien seul), le silence ne coûte rien, et toute la stratégie « parler quand on est sûr » perd une partie de sa raison d'être. L'expérience E3b a montré que **sur Chrome et Firefox, presque tout le gain mesuré vient de l'envoi de `sl`**. Ce poids n'a jamais été mesuré sur de vraies traductions. C'est probablement le point le plus important à critiquer.

---

## 3. Les options pesées pour le modèle

| Option | Pour | Contre | Décision |
|---|---|---|---|
| Garder franc, ajouter des règles | zéro poids | plafond bas, règles fragiles, rejeté par le propriétaire | non |
| `chrome.i18n` / CLD partout | déjà là | absent de Brave, faible sur le texte court | reste en contre-vérification |
| LanguageDetector (Chrome) | modèle récent, sur l'appareil | Chrome seul, asynchrone | réservé à B |
| Gemini Nano (Chrome) | comprend le contexte, le mélange | Chrome seul, lent, jamais un appel par ligne à 25 msg/s | réservé à B, en lots |
| fastText officiel (lid.176) | référence du domaine | 126 Mo (ou 917 Ko compressé, précision moindre), licence CC BY-SA incompatible avec un dépôt MIT, runtime WASM | non |
| CLD3 | petit, réseau éprouvé | port JS, entraîné sur du texte propre | idée reprise en E7 (réseau maison) |
| **Linéaire sur n-grammes hachés, entraîné ici** | petit (1,5 à 3 Mo), TypeScript pur, rapide, calibrable, entraînable sur nos propres augmentations | plafond sur le mélange et le romanisé | **retenu** |
| Réseau façon CLD3 (plongements + couche cachée) | plus expressif | plus lent, plus sûr de lui à tort | testé en E7, rejeté au coût |

### Détails du modèle retenu, et pourquoi

- **N-grammes de caractères 1 à 4 + mots entiers** : les caractères capturent l'orthographe (accents, digrammes typiques), les mots aident sur les phrases courtes du chat où chaque mot compte.
- **Hachage (FNV-1a sur UTF-8, 2^15 ou 2^16 cases)** : taille fixe, pas de vocabulaire à livrer. Collisions acceptées ; E1 montre que passer de 2^15 à 2^16 apporte peu en précision mais un peu au coût.
- **Régression logistique multinomiale** : convexe, stable, facile à calibrer, rapide.
- **Température** (un seul scalaire) : rend les probabilités honnêtes, nécessaire parce que les décisions se prennent sur des seuils de probabilité.
- **int8 par ligne** : divise le poids par 4 sans perte mesurable.
- **Format binaire avec somme de contrôle** : si les poids sont illisibles, l'extension retombe exactement sur l'ancien comportement (testé).
- **Parité Python / TypeScript** : le hachage doit donner les mêmes cases dans les deux langages, y compris sur les caractères hors du plan de base (emoji, CJK étendu) ; un fichier de référence de 500 lignes le vérifie.

---

## 4. Les données : ce qu'on avait, ce qu'il manquait

### Ce qu'on avait

- **Tatoeba** : phrases propres, bien étiquetées, 42 langues, licence compatible. Mais ce n'est pas du chat : ponctuation, majuscules, phrases complètes.
- **Des captures de chat Kick du 04/11** (environ 28 000 lignes), sans étiquette fiable : seule la langue **déclarée par la chaîne** est connue.
- **Un jeu annoté** (environ 2 000 lignes de ces captures, annotées ligne à ligne), prévu par capture (pour éviter qu'une même conversation soit des deux côtés), en réalité coupé **ligne par ligne** dans chaque langue (REVUE §3 bis), donc une même conversation est des deux côtés, en :
  - **dev** : sert à tout choisir ; 711 lignes y sont étiquetées dans une seule langue (les autres sont neutres, « autre » ou mélangées, et n'entrent pas dans le coût) ;
  - **heldout** : 993 lignes, empreinte sha256 vérifiée, **à lire une seule fois**, à la fin.

### Ce qu'il manquait, et comment on l'a compensé

| Manque | Compensation | Limite |
|---|---|---|
| Registre du chat | augmentations : minuscules, sans ponctuation, fenêtres de 1 à 4 mots | synthétique, ne crée pas l'argot |
| Texte sans accents (très courant en tchèque, slovaque, portugais tapé vite) | variante « sans accents » de chaque phrase | double les données |
| Hindi et bengali écrits en lettres latines (hinglish) | translittération synthétique (`romanize.py`, table de correspondance, voyelle inhérente finale retirée) | E9-analyse : le romanisé reste à 57 %, la synthèse ne suffit pas |
| Persan romanisé (finglish) | rien | jamais vu à l'entraînement |
| Mélange de langues dans une ligne | rien | 69 %, inchangé |
| Chat réel en quantité | collecte massive (E9, §8) | sans étiquettes |

---

## 5. Le protocole de mesure

1. **Banc Tatoeba (5 040 phrases)** : rapporté, jamais utilisé pour choisir. Ses phrases sont exclues de l'entraînement par empreinte du texte normalisé ; un test échoue si l'une d'elles est injectée.
2. **dev** : température ajustée sur la moitié paire des lignes étiquetées, ECE rapportée sur la moitié impaire ; choix du modèle, des seuils `tau_det` (au-dessus : on répond) et `tau_sl` (au-dessus : on envoie la langue source au cloud), et du paramètre `k` de l'adaptation.
3. **Simulateur de coût** (`cost.py`) : rejoue la décision de l'extension (cacher, drapeau, `sl`, silence) pour chaque lecteur et chaque chemin. La spécification prévoyait de le vérifier contre la vraie fonction de l'extension sur 50 lignes avant de lui faire confiance.
4. **Statistiques** : intervalle de confiance du gain (bootstrap), test de McNemar sur les erreurs appariées (même ligne, avant/après).
5. **heldout** : une seule lecture, à la fin, sur la configuration figée.

**Faiblesse assumée** : dev sert à la fois aux seuils, à `k` et au choix du modèle (seule la température est ajustée sur une moitié). Les chiffres de dev sont donc optimistes ; seul heldout dira le vrai gain.

---

## 6. Déroulé des expériences, dans l'ordre

Chaque étape a été décidée en regardant la précédente. L'ordre compte pour juger du risque de sur-ajustement à dev.

1. **E1, taille des cases** (2^14, 2^15, 2^16). Le banc monte peu (97,5 → 98,2 %). Choix provisoire : 2^15.
2. **E2, bogue de banc.** Le premier chiffre comparant le modèle à `master` donnait -20 % d'erreurs. Il était faux : le banc tournait sans modèle installé et mesurait en partie franc. Chiffre retiré, garde ajoutée (le banc refuse de tourner sans modèle).
3. **E3, linéaire 2^15 contre `master`** sur dev, avec `tau_det` 0,8 et `tau_sl` 0,95 : coût sans détecteur 0,531 → 0,329 ; réponses fausses 121 → 23. Gain net et significatif.
4. **E3b et E3c, décomposer le gain.** Sans envoi de `sl`, le gain sur Chrome tombe à 3,5 % et à zéro sur Firefox : sur ces navigateurs, presque tout le gain vient de `sl`. Envoyer `sl` sur plus de lignes ajoute 4 % d'erreurs sur ces ajouts. D'où l'hypothèse fragile du §2.
5. **E4, analyse d'erreurs.** Erreurs : arabe/persan sur lignes courtes, argot anglais très court, espagnol/italien, portugais/catalan. Silences : hindi et bengali romanisés, tchèque sans accents. Un **bogue de `master`** trouvé au passage : une table de mots « sûre » répond tchèque sur une ligne arabe. Ces constats ont orienté E6 à E8.
6. **E1b, 2^16 au coût** : légèrement meilleur sous la contrainte. 2^16 retenu.
7. **E5, fréquence globale des langues** (algorithme EM de Saerens : réestimer la fréquence des langues sur le chat non étiqueté, puis corriger les probabilités). Instable (le tamoul estimé à 9 %). Et surtout faux dans son principe : les captures non étiquetées viennent **des mêmes chaînes** que dev, donc la fréquence apprise est celle de l'échantillon, pas celle de Kick. Rejeté.
8. **E6, adaptation par chaîne** (§7). Gros gain annoncé (0,326 → 0,221). Erreur de mise en œuvre découverte plus tard (§10).
9. **E7, réseau de neurones** façon CLD3 (plongements de 32, couche cachée de 128). Même précision sur Tatoeba (98,3 %), mais coût bien pire (0,410) : il se trompe plus souvent **avec assurance**, ce qui force les seuils très haut. Rejeté.
10. **E8, variantes sans accents et romanisées** : `accrom-b16`, nouvelle référence (dev 87,8 %, coût seul 0,306).
11. **E6b, correction d'E6** par vraie chaîne : 0,306 → 0,209.
12. **E6c, adaptation causale** : seules les lignes vues avant la ligne jugée comptent, comme en direct : 0,306 → 0,210. `k` = 10 retenu.
13. **E9a, premier auto-apprentissage** sur 20 minutes de collecte : coût seul 0,284, adapté 0,189.
14. **E9-analyse** : le gain vient de l'anglais de chat (76 → 84 %), pas du mélange ni du romanisé (inchangés). Les chaînes « English (India) » injectaient du hinglish étiqueté anglais : retirées du filtre.
15. **E9b, contrôle de fuite** (§8) : Twitch seul fait mieux que le mélange, Kick seul fait pire.
16. **Collecte massive** (en cours) pour refaire E9 sur environ un million de lignes.

---

## 7. L'adaptation par chaîne : l'idée la plus rentable

### L'intuition

Dans une chaîne donnée, la plupart des messages sont dans une ou deux langues. Une ligne ambiguë (« jajaja », « kkkk », « ok bro ») est bien plus probablement dans la langue dominante de **cette** chaîne que dans une langue au hasard. Le modèle, lui, a appris des fréquences de langues uniformes (Tatoeba équilibré).

### Le calcul

Pour chaque chaîne, on compte les lignes que le modèle classe avec assurance (probabilité ≥ 0,9). On en tire une fréquence de langues lissée vers celle de l'entraînement :

```
pi_chaine = (comptes_sûrs + k · pi_entraînement) / (n_sûrs + k)
```

puis on corrige chaque prédiction par la règle de Bayes (changement de loi a priori) :

```
p_corrigée(langue) ∝ p_modèle(langue) · pi_chaine(langue) / pi_entraînement(langue)
```

`k` règle la confiance donnée à la chaîne : petit `k`, on suit vite la chaîne ; grand `k`, on reste proche du modèle.

### Pourquoi c'est défendable en production

- L'extension a déjà un état par chaîne ; il suffit d'y ajouter un compteur de 42 cases.
- On ne compte que les réponses **sûres** du modèle, jamais les étiquettes : rien de magique, aucune donnée externe.
- Simulé en **causal** (E6c) : la ligne jugée et les lignes postérieures ne sont jamais comptées.

### Ses limites

- Une chaîne qui change de langue (invité étranger, raid) sera en retard ; `k` petit limite le retard.
- Si la langue dominante est mal reconnue au départ, l'adaptation peut renforcer l'erreur (boucle).
- `k` a été choisi sur dev.
- Les chaînes de dev sont courtes (médiane 22 lignes) ; en direct, les chaînes sont plus longues. Mesuré depuis : sur le chat non étiqueté (pool E9b, 313 989 lignes, 2 050 couples fichier:chaîne, accrom-b16, k = 10, `review_checks.py` partie B), la part des lignes qui reçoivent une langue source contraire au choix du modèle seul passe de 0,09 % (moins de 10 lignes sûres vues avant dans la chaîne) à 1,76 % (10 à 50, le cas de dev), 3,65 % (50 à 200), 5,20 % (200 à 1 000) et 7,17 % (plus de 1 000) ; les lignes dont le modèle seul était sûr (p ≥ 0,9, justes à 98,4 % sur dev) sont retournées dans 0,16 % puis 1,24 % des cas. Une chaîne longue n'aide donc pas d'elle-même : la loi a priori pèse de plus en plus. Juste ou faux n'est pas mesurable sans étiquettes ; à trancher avant le portage (fenêtre glissante, plancher, adaptation tenue hors de `sl`).

---

## 8. L'auto-apprentissage et la collecte massive

### L'idée

Le modèle se trompe surtout sur le registre du chat, qu'il n'a vu qu'en synthèse. Or on peut récolter du chat réel en quantité, sans étiquette. **L'auto-apprentissage** : on fait étiqueter ce chat par le modèle actuel, on ne garde que les lignes où il est très sûr **et** d'accord avec la langue déclarée de la chaîne, et on les ajoute à l'entraînement (cinq copies de chaque, pour qu'elles pèsent face à Tatoeba, plafond de 30 000 par langue).

Le double filtre (confiance + accord) vise à limiter le principal danger de l'auto-apprentissage : apprendre ses propres erreurs. Le prix : les lignes difficiles (mélange, romanisé) sont justement celles qui ne passent pas le filtre, donc le modèle ne peut pas progresser dessus par cette voie (constaté en E9-analyse).

### Le contrôle de fuite (E9b)

Danger : dev vient de captures Kick. Si la collecte Kick contient les mêmes chaînes, le modèle apprendrait le style de ces chaînes, et dev serait flatté. Contrôle : entraîner avec des pseudo-lignes **Twitch seulement** (aucune chaîne en commun avec dev). Résultat : Twitch seul fait **mieux** que le mélange (coût 0,275 contre 0,284). Le gain n'est donc pas une fuite. Kick seul fait **pire** (0,331), ce qui a mené à découvrir le problème du hinglish étiqueté anglais.

### Pourquoi collecter massivement

Le premier essai reposait sur 20 minutes de chat (28 759 pseudo-lignes). Pour savoir si le gain grandit avec les données, et pour couvrir les langues rares, le propriétaire a demandé de « récupérer le max de data », en maximisant le nombre de langues, avec méthode.

### Comment la collecte a été conçue

- **Kick** : les points publics que le site appelle lui-même (liste des directs, messages récents d'une chaîne), dans un vrai navigateur visible (le pare-feu de Kick répond différemment à un navigateur sans fenêtre). On parcourt l'annuaire trié par spectateurs, on garde les N plus grosses chaînes **par langue déclarée** (plafond par langue, pour que l'anglais n'écrase pas tout), puis on relève leurs messages chaque minute.
- **Twitch** : l'IRC anonyme officiel (lecture seule, sans compte). Les chaînes viennent de l'annuaire public filtré par chacune des 32 langues de diffusion et trié par spectateurs. Une API interne de Twitch a été envisagée puis écartée.
- **Hygiène** : émotes retirées grâce aux métadonnées IRC, mentions remplacées par `@u<n>`, chaînes renumérotées, aucun pseudo ni nom de chaîne écrit, dédoublonnage au texte normalisé, exclusion de tout texte de dev et heldout, données hors de git.
- **Montée en charge** : à la demande du propriétaire, la collecte a été répartie sur plusieurs machines (le PC, un serveur cloud à 2 cœurs plafonné en priorité basse pour ne pas gêner son service principal, deux VM cloud), avec un moteur unique qui répartit les chaînes, rapatrie les fichiers chaque minute et un tableau de bord en console pour suivre le tout ([tui-dash](https://github.com/kilworks/tui-dash)).

### Ce que la collecte a appris en route

- Le premier relevé de l'annuaire Twitch était mauvais (mauvais filtre, tri « recommandé », donc petites chaînes) : refait.
- L'annuaire ne chargeait que 30 chaînes par langue ; un défilement correct en donne jusqu'à 250.
- **Kick a limité l'adresse IP** après une montée trop brutale (environ 1 100 chaînes, un tour toutes les 20 secondes) : 78 à 90 % de refus ensuite. Un repli automatique (intervalle doublé au-delà de 10 % de refus) a été ajouté, mais après coup.
- Conséquence : la collecte récente est surtout Twitch (740 000 lignes contre 250 000 pour Kick), alors que le produit sert Kick. E9b suggère que ce n'est pas un problème, mais c'est un biais à garder en tête.

---

## 9. Écarts entre le plan écrit et ce qui a été fait

La spécification et le plan ont été écrits **avant** le travail (dans la branche, `docs/superpowers/`). Voici où l'exécution s'en écarte. C'est probablement la section la plus utile pour une critique.

| Plan écrit | Fait | Conséquence |
|---|---|---|
| Température ajustée sur une moitié de dev, vérifiée sur l'autre | fait ainsi (`train.py` : moitié paire, ECE sur l'impaire) ; mais seuils, `k` et modèle sont choisis sur dev entier | chiffres dev des seuils plus optimistes |
| Seuils par langue quand une langue a 50 lignes de dev ou plus | seuils globaux (`tau_det` 0,8, `tau_sl` 0,95) | plus simple, peut-être sous-optimal sur certaines langues |
| Chat réel ajouté seulement s'il bat « Tatoeba seul » sur dev | remplacé par l'auto-apprentissage (E9), qui le bat | même logique, autre forme |
| Coût du moteur sur l'appareil (4) dans la table | absent de `cost.py` (mode par défaut cloud d'abord) ; ajout d'un coût 1 « gardée hors liste blanche » non prévu | la table simulée n'est pas celle de la spécification, écart à juger |
| Simulateur vérifié contre la vraie fonction de l'extension sur 50 lignes | 9 cas communs (`cost-cases.json`) testés des deux côtés pour les règles de suppression seulement | drapeau, `sl` et silence crus sur parole (REVUE §10 bis) |
| Comparaison linéaire / fastText à égalité de coût, le plus simple gagne | linéaire contre réseau maison (E7), puis fastText lid.176 en référence le 22/11 : 0,390 contre 0,306 | fastText non réentraîné sur nos données (licence, et pas d'entraînement en ce moment) |
| `franc` retiré s'il est battu dans chaque tranche de longueur | pas encore tranché | |
| p95 sous 0,3 ms par ligne en build de production | p95 0,044 ms en Node sur le PC de développement (REVUE §10 bis) | navigateur réel et machine modeste pas mesurés |
| B conditionné à un résidu chiffré | pas encore mesuré | |
| Coupe dev/heldout par capture, jamais par ligne | coupe ligne par ligne par langue (`real-bench.mjs`, 04/11), deux fichiers de capture seulement | heldout partage chaînes et conversations avec dev : il ne mesure pas la généralisation |
| Sortie : 42 langues plus `und` | 42 labels, pas de `und` : le silence passe par les seuils | pas de classe de rejet apprise (romanisé, langues hors liste) |
| Test de fumée de 15 minutes sur kick.com, PR sans fusion | pas encore | la recherche n'est pas finie |

---

## 10. Erreurs de raisonnement commises

1. **Mesurer sans le modèle (E2).** Un chiffre spectaculaire a été annoncé avant de vérifier que le modèle était bien chargé. Leçon : un banc doit refuser de tourner quand son sujet est absent.
2. **Adapter par langue déclarée au lieu de par chaîne (E6).** La clé de regroupement était la langue déclarée, donc des dizaines de chaînes mélangées en une seule « chaîne » géante, avec une loi a priori bien trop forte. Le chiffre (0,214) avait même été publié dans le README avant correction. Leçon : simuler exactement ce que le produit verra (une chaîne, dans l'ordre du temps).
3. **Fréquence globale sur les mêmes chaînes que dev (E5).** La fuite de distribution n'a été vue qu'après avoir codé l'EM.
4. **Pseudo-étiquettes « anglais » sur des chaînes indiennes.** Une langue déclarée (« English (India) ») a été traitée comme « en », alors que ces chaînes écrivent en hinglish. Vu seulement grâce au contrôle Kick seul / Twitch seul.
5. **Collecte trop agressive.** L'adresse a été limitée par Kick parce que le rythme a été poussé avant d'avoir un garde-fou sur les refus.
6. **Ne pas utiliser toutes les machines d'emblée.** La collecte a d'abord été poussée sur une seule machine, alors que d'autres étaient disponibles ; c'est le propriétaire qui l'a fait remarquer.
7. **Affirmer la coupe par capture sans relire le script.** La spécification exigeait une coupe par capture ; le dossier l'a affirmée sans relire `real-bench.mjs` (04/11), qui coupait ligne par ligne. Relevé par la revue du 22/11 au soir (REVUE §3 bis).

---

## 11. Ce que je ferais autrement

- **Couper dev en deux dès le début** : une moitié pour la température et les seuils, l'autre pour choisir le modèle. Ou passer à une validation croisée par capture.
- **Mesurer le poids du silence avant tout** : quelques centaines de vraies traductions avec et sans `sl` auraient dit si la moitié du gain mesuré existe vraiment.
- **Garder un professeur hors ligne** (grand modèle utilisé seulement à l'entraînement) pour étiqueter les lignes difficiles (mélange, romanisé) que l'auto-apprentissage ne peut pas apprendre.
- **Pondérer par chaîne** dans l'auto-apprentissage : quelques chaînes très bavardes pèsent aujourd'hui autant que des centaines de petites.
- **Mettre le garde-fou de rythme avant de monter la collecte**, pas après.

---

## 12. Points où la critique est la plus utile

Par ordre d'importance, à mon avis :

1. **Le poids 0,5 du silence** et la dépendance du gain Chrome/Firefox à l'envoi de `sl` (§2, E3b).
2. **Le sur-ajustement à dev** : seuils, `k` et choix du modèle sur le même jeu (§5, §9), et un heldout déjà mesuré pour la base `master` (REVUE §3 bis).
3. **La boucle de l'auto-apprentissage** : apprendre ses propres certitudes ne corrige pas les erreurs de fond (§8).
4. **L'adaptation par chaîne** : risque de renforcer une erreur initiale, comportement sur un changement de langue (§7).
5. **Le choix linéaire contre réseau** : E7 a-t-il été un test juste (calibration, régularisation) ou un réseau mal réglé ?
6. **La représentativité de la collecte** : grosses chaînes, une seule soirée, Twitch majoritaire pour un produit Kick.
7. **Les écarts au plan** du §9, en particulier le simulateur de coût non vérifié contre l'extension.
