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
