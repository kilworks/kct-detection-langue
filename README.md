# Détection de langue par inférence pour Kick Chat Translator

Recherche en cours pour remplacer la détection de langue heuristique de l'extension [Kick Chat Translator](https://github.com/kilworks/kick-chat-translator) par un modèle entraîné, livré dans l'extension et exécuté chez chaque utilisateur (Chrome, Firefox, Brave), sans service externe.

Démo en direct et graphiques : https://kilworks.github.io/kct-detection-langue/ (le modèle tourne dans la page, tapez une phrase).

## Le problème

L'extension doit savoir dans quelle langue est écrit chaque message du chat avant de le traduire. Une erreur coûte cher :

- elle peut **cacher un message** au lecteur (l'extension croit qu'il est déjà dans sa langue, ou qu'il vient d'une langue qu'il a exclue) ;
- elle peut **envoyer une mauvaise langue source** au moteur de traduction, qui rend alors du charabia ou le texte inchangé.

Le détecteur actuel (`franc-min` et des tables de mots) se trompe sur 24 % des phrases du banc Tatoeba, et sur 17 % des lignes de chat réel annotées.

## La méthode

- **Modèle** : classifieur linéaire sur des n-grammes de caractères (1 à 4) et des mots, hachés dans 2^15 ou 2^16 cases (même famille que fastText). Poids quantifiés en int8, probabilités calibrées par une température. Inférence en TypeScript pur, sans dépendance, moins d'une milliseconde par ligne.
- **Données d'entraînement** : Tatoeba (42 langues, CC BY 2.0 FR), augmenté vers le registre du chat (minuscules, sans ponctuation, fenêtres de 1 à 4 mots, variantes sans accents, hindi et bengali romanisés synthétiques). Aucune ligne évaluée n'est vue à l'entraînement, un test le vérifie.
- **Évaluation** : 711 lignes de chat Kick réel annotées par langue (jeu « dev »), plus 993 lignes gardées de côté (« heldout ») qui ne seront lues qu'une fois, à la fin. Ces lignes viennent d'utilisateurs de Kick : elles ne sont pas publiées ici, seuls les agrégats le sont.
- **Coût d'erreur**, fixé avant toute mesure : message caché à tort 10, mauvaise langue source envoyée au cloud 5, drapeau faux 1, silence (le cloud détecte seul) 0,5. Les seuils du modèle minimisent ce coût sous une contrainte : aucune hausse des messages cachés à tort, pour aucun lecteur.

## Résultats provisoires (dev, pas encore heldout)

Ces chiffres servent à choisir le modèle et ses seuils : ils sont donc optimistes. Seul heldout tranchera.

Coût moyen par ligne de chat (plus bas = mieux) :

| Chemin | Avant (master) | Modèle seul | Modèle + adaptation par chaîne |
|---|---|---|---|
| Sans détecteur du navigateur (Brave) | 0,531 | 0,296 | 0,200 |
| Chrome | 0,439 | 0,301 | 0,203 |
| Firefox | 0,424 | 0,320 | 0,223 |

Modèle : `accrom-b16` (2^16 cases, variantes sans accents et hindi/bengali romanisés).

- Sans détecteur du navigateur, les réponses fausses passent de 121 à 23 sur 711 lignes (107 corrigées, 9 cassées, test de McNemar χ² = 81).
- Sur le banc Tatoeba, la chaîne de détection trouve la bonne langue 98 % du temps, contre 82 % avant.
- **Adaptation par chaîne** : l'extension compterait les langues dont elle est sûre dans le chat de la chaîne en cours et ajusterait ses probabilités pour cette chaîne. Simulée sur 515 vraies chaînes (médiane 22 lignes chacune), sans jamais voir la ligne jugée ni son étiquette : coût 0,306 vers 0,209, langues sources fausses 15 vers 10. Limites : la simulation compte aussi les lignes postérieures de la chaîne, et son paramètre k a été choisi sur dev.
- **Correction** : une première version regroupait les lignes par langue déclarée de la chaîne (plusieurs chaînes mélangées) et annonçait 0,214 ; ce n'était pas ce que l'extension verrait.

## Ce qui n'a pas marché

- **Réseau de neurones** (façon CLD3 : embeddings de n-grammes, couche cachée ReLU) : même précision que le linéaire sur Tatoeba (98,3 %), mais moins bon sur le chat (coût 0,410 contre 0,326), car il se trompe plus souvent avec assurance. Le linéaire reste en tête.
- **Fréquence globale des langues sur Kick** (EM de Saerens sur le chat non étiqueté) : instable sur les langues rares, et la fréquence apprise est celle des chaînes capturées, pas de Kick. Rejeté au profit de l'adaptation par chaîne.
- **Un premier chiffre faux, corrigé** : le banc tournait sans modèle installé et mesurait en partie franc. Une garde refuse maintenant de mesurer sans modèle.

Le détail de chaque expérience est dans [journal.md](journal.md).

## Suite

1. Collecte de chat Kick à grande échelle (non étiqueté) pour l'auto-apprentissage et une simulation causale de l'adaptation.
2. Mesurer sur de vraies traductions si donner la langue source au cloud améliore le résultat (le poids 0,5 du silence est une hypothèse).
3. Porter l'adaptation par chaîne dans l'extension, puis lecture unique de heldout.
4. Gemini Nano (Chrome) en arbitre des lignes ambiguës, seulement s'il reste un écart chiffré.

## Code

Le code vit dans la branche [`feat/lid-inference`](https://github.com/kilworks/kick-chat-translator/tree/feat/lid-inference) de l'extension : inférence dans `src/content/lid/`, entraînement et bancs dans `tools/lid/` (numpy, Node). Spécification et plan : `docs/superpowers/` de la même branche.

## Licences

- Textes et page de ce dépôt : MIT.
- Phrases d'exemple et poids du modèle (`docs/model.b64`) : dérivés de [Tatoeba](https://tatoeba.org), CC BY 2.0 FR, merci à ses contributeurs.
