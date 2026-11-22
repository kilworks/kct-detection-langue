# Recherche : identification de langue par inférence (KCT)

Branche `feat/lid-inference`. Spec et plan dans le dépôt (docs/superpowers). Ce journal : une ligne par expérience, chiffre et verdict.

## Protocole

- Entraînement : Tatoeba (42 langues, 20 000 phrases max par langue), augmenté (minuscules sans ponctuation, fenêtres de 1 à 4 mots). Exclus : dev, heldout, banc 5040.
- Sélection : dev (chat réel, 711 lignes étiquetées dans une langue, hors neutral/other/mixed), coût de la spec moyenné sur 3 chemins (sans API, Chrome CLD, Firefox CLD). Banc Tatoeba rapporté, pas utilisé pour choisir.
- Contrainte : aucune hausse des suppressions à tort, par lecteur et par chemin.
- heldout lu une fois, à la fin.
- Biais connu : dev sert à la fois à la température et aux seuils, donc les chiffres dev sont optimistes.

## Expériences

| # | Hypothèse | Modèle | Résultat | Verdict |
|---|---|---|---|---|
| E1 | taille des cases | b14/b15/b16, 2 époques, lr 0.5 | banc 97.5/97.9/98.2 %, dev argmax 84.5/85.1/85.2 %, ECE cal 0.035/0.032/0.017 | gain faible au-delà de 2^15 |
| E2 | bogue de banc | answers.mjs sans modele installe | `pre` contenait franc : 1er chiffre (-20 %) FAUX, garde ajoutee (refus sans modele) | corrige |
| E3 | lineaire b15 vs master, dev 711 | base-b15, det 0.8, sl 0.95, silence | cout sans API 0.531 -> 0.329 (IC95 [-0.213,-0.192]), Chrome 0.439 -> 0.344, Firefox 0.424 -> 0.354 ; reponses fausses sans API 121 -> 23 (corrige 107, casse 9, McNemar 81) | gain reel sans API |
| E3b | meme chose sans sl | det 0.8, sl inf | sans API 0.382, Chrome 0.423 (-3.5 %), Firefox 0 changement | sur Chrome/Firefox le gain vient presque tout de sl |
| E3c | sl en plus | seuil 0.95 | sl donne 361 -> 483 lignes, sl faux 7 -> 12 (ajouts : 4 % d erreurs, es/it, pt/ca) | rentable SEULEMENT si sl ameliore la traduction : a mesurer sur vraies traductions (le poids 0.5 du silence est une hypothese) |
| E4 | analyse d erreurs b15 (sans API) | 23 faux (14 modele, 7 table, 2 regle), 155 muets | faux : arabe/persan sur lignes courtes, argot anglais tres court, espagnol/italien, portugais/catalan ; muets : hindi et bengali romanises (21 lignes hindi), tcheque sans accents ; BOGUE master : une table sure repond tcheque sur une ligne arabe | pistes E6 a E8 |
| E1b | b16 vs b15 au cout | base-b16 | meilleur respectant la contrainte : 0.3176/0.3206/0.3388 (b15 : 0.329/0.344/0.354) | b16 retenu |
| E5 | frequences globales du chat (EM Saerens) | b15/b16 + biais | EM instable (tamoul 9 % estime, classe rare a 598 phrases) ; comptage robuste b16 : 0.292 | REJETE : les captures ont les memes chaines que dev, la frequence apprise est celle de l echantillon, pas de Kick (fuite de distribution) |
| E6 | adaptation par chaine en direct (comptage des reponses sures, k=50 pseudo-lignes) | b16, simulee sans voir la ligne ni son etiquette | 0.326 -> 0.221 (sans API 0.214, Chrome 0.214, Firefox 0.236) ; demarrage : 10 lignes 0.281, 25 0.261, 50 0.244, 100 0.225 ; sl faux 15 -> 9 avec plus de sl donnes (523 -> 576) | GROS GAIN, a porter en TS (etat par chaine) |
| E7 | reseau de neurones facon CLD3 (embedding 32, cache 128, ReLU) | nn-b16-d32-h128, 3 epoques, lr 0.1 | banc 98.3 % (lineaire 98.2), dev 83.3 % (lineaire 85.2) ; cout 0.410 (lineaire 0.326) : plus d erreurs avec assurance, sl force a 0.995 | REJETE pour l instant (essai plus large prevu) |
| E8a | variante sans accents | acc-b15 | cout 0.338 (b15 0.343) ; adapte 0.223 | petit gain, a combiner en b16 |
| E6b | CORRECTION E6 : adaptation par VRAIE chaine (cle capture:canal, comme l extension), E6 groupait par langue declaree (plusieurs chaines melangees, prior trop fort) | base-b16 et accrom-b16, k 10 et 50, sure 0.9 | accrom-b16 : 0.306 -> 0.209 (k=10), 0.238 (k=50) ; base-b16 : 0.326 -> 0.251 (k=10) ; sl faux 15 -> 10 ; 515 chaines, mediane 22 lignes | gain confirme mais plus petit qu en E6 ; reste non causal (lignes futures de la chaine comptees) et k choisi sur dev |
| E6c | adaptation CAUSALE par vraie chaine : seules les lignes captees AVANT la ligne jugee comptent (comme l extension en direct) | accrom-b16, sure 0.9 | k=10 : 0.306 -> 0.210 (none 0.203, chrome 0.205, firefox 0.222), sl faux 15 -> 11 ; k=50 : 0.258 | gain confirme en causal ; k=10 retenu (choisi sur dev, a confirmer sur heldout) |
| E9a | auto-apprentissage, 1er essai : 28 759 lignes de chat (Kick+Twitch du 22/11, ~20 min de collecte) pseudo-etiquetees par accrom-b16 (p>=0.9 ET accord avec la langue declaree), x5, ajoutees a Tatoeba | st1-b16 | dev 89.3 % (accrom 87.8) ; banc 98.06 % (98.17) ; cout seul 0.284 (0.306) ; adapte causal k=10 0.189 (0.210) ; sl faux seul 17 (15), adapte 9 (11) | gain net sur le chat, petite perte Tatoeba ; biais possible : memes grosses chaines Kick que les captures du 04/11 (dev), a verifier en excluant les chaines du dev |
| E9-analyse | erreurs de st1-b16 par type de ligne (dev) | accrom-b16 -> st1-b16 | tout 88 -> 89 % ; anglais de chat 76 -> 84 % ; argot 80 -> 82 % ; <=2 mots 83 -> 85 % ; melange de langues 69 % et romanise 57 % INCHANGES ; hindi romanise lu anglais 8/23 ; persan romanise (finglish) jamais vu a l entrainement | filtre d accord : 'English (India)' retire (112/381 lignes Kick 'en' venaient de ces chaines, hinglish renvoye comme anglais) |
| E9b | controle de fuite : pseudo-etiquettes d une seule plateforme (dev = captures Kick, donc Twitch ne partage aucune chaine avec dev) | st-twitch-b16 (42 641 lignes x5), st-kick-b16 (13 251 x5) | twitch : seul 0.275, adapte 0.200, dev 89.2 %, banc 98.5 % ; kick : seul 0.331 (PIRE que accrom 0.306), adapte 0.234, dev 88.7 % | le gain d E9a n est PAS une fuite : Twitch seul fait mieux que le melange ; les pseudo-lignes Kick nuisent au cout (hinglish etiquete anglais, ja/ar dominants) -> filtre corrige, a remesurer sur la collecte complete |

## Protocole de collecte E9 (chat non etiquete, 22/11)

- But : beaucoup de chat reel non etiquete, dans un maximum de langues, pour (a) l auto-apprentissage (pseudo-etiquettes du modele, seuil haut) et (b) une simulation causale de l adaptation par chaine sur beaucoup plus de chaines. Jamais pour evaluer : dev et heldout restent les seuls juges.
- Kick : kick_capture.mjs (points publics du site, navigateur visible), toutes les langues declarees de l annuaire, 40 a 80 chaines max par langue, un tour par minute ; repli automatique (intervalle double) au-dela de 10 % de refus 403/429. Une montee a ~1 100 chaines et 20 s d intervalle a fait limiter l IP (78 a 90 % de refus) : revenu a 40 chaines par langue et 60 s.
- Twitch : annuaire public lu dans le navigateur, tri spectateurs decroissants, chacune des 32 langues de diffusion (hors langue des signes et autre), ~30 chaines par langue au depart (863 chaines, 32 langues), puis une liste elargie de 2 400 chaines nouvelles (13 langues, jusqu a 250 par langue) lue sur plusieurs machines en parallele ; lecture par IRC anonyme officiel (justinfan, lecture seule). L API interne GQL a ete refusee par le classifieur : non utilisee.
- Plafond par langue cote chaines pour que l anglais n ecrase pas le reste ; reequilibrage par langue au moment de l entrainement.
- Hygiene : emotes Twitch retirees par les tags IRC, @mentions -> @u<n>, chaines renumerotees, aucun pseudo ni nom de chaine ecrit ; lignes dedoublonnees au texte normalise ; tout texte de dev/heldout exclu (captures.mjs) ; donnees hors git.
- Biais connus : langue declaree de la chaine != langue de chaque ligne (melange, anglais partout) ; gros streams surrepresentes ; une seule fenetre horaire (fuseaux).