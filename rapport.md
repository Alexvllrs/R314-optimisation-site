# Rapport d'optimisation Web — TP R3.14 (SlowSite)

**Étudiant :** Alexandre Villiers  
**Formation :** BUT MMI, IUT de Béziers  
**Date :** 25 octobre 2026  

- **Mon fork (dépôt GitHub) :** https://github.com/Alexvllrs/R314-optimisation-site
- **Site déployé (après fixes) :** https://alexvllrs.github.io/R314-optimisation-site/
- **Dépôt de référence (départ) :** https://github.com/nico3807/R314-optimisation-site

---

## 1. Ce qu'il faut retenir

Au départ, le site de test était impraticable sur mobile : **17,5 Mo** téléchargés, **22 requêtes** HTTP, un score Lighthouse Mobile désastreux de **29/100**, et surtout un LCP bloqué à **67,4 secondes** à cause d'images gigantesques et de scripts parasites.

En travaillant sur l'optimisation des assets et du code JS, j'ai réussi à réduire le poids total à **631 ko** (soit 96 % de moins) et à tomber à **15 requêtes**. 

Résultats sur les audits :
- **Lighthouse Mobile :** passage de 29/100 à **79/100** (LCP ramené à 3,7s, TBT réduit de 90 %).
- **Lighthouse Bureau :** passage de 49/100 à **99/100** (LCP à 0,8s, TBT quasi nul).
- **PageSpeed Insights :** monte à **86/100** en mobile et **98/100** en desktop.

Le tout a été réalisé sans dénaturer le visuel ni supprimer des fonctionnalités pour l'utilisateur.

---

## 2. Protocole de mesure

Pour avoir des chiffres fiables et éviter les biais :
1. Les tests DevTools ont été faits sur Chrome en **navigation privée**, avec le cache désactivé et rechargement forcé (`Ctrl+Maj+R`).
2. Les mesures réelles ont toutes été relevées sur la version hébergée GitHub Pages (le serveur local ne compressant pas le texte, ça aurait faussé les poids).
3. Sur Lighthouse, j'ai fait **3 passages par profil** pour retenir les médianes, vu que le TBT varie pas mal d'un lancement à l'autre.
4. Pour le panneau Performances, j'ai appliqué un **bridage CPU ×4** en simulant un scroll d'au moins 20 secondes.

*Note : le site n'ayant pas de trafic réel, il n'y a pas de données CrUX/INP disponibles. L'audit repose donc à 100 % sur les données de labo.*

---

## 3. Comparatif avant / après

### Relevés DevTools (Lighthouse)

| Métrique | Mobile (Avant) | Mobile (Après) | Bureau (Avant) | Bureau (Après) |
|---|---|---|---|---|
| **Poids total** | 17,5 Mo | **631 ko** | 14,2 Mo | **631 ko** |
| **Requêtes** | 22 | **15** | 21 | **15** |
| **FCP** | 3,6 s | **2,3 s** | 1,1 s | **0,8 s** |
| **LCP** | 67,4 s | **3,7 s** | 9,9 s | **0,8 s** |
| **TBT** | 3 150 ms | **320 ms** | 410 ms | **0 ms** |
| **CLS** | 0 | **0,033** | 0,048 | **0,048** |
| **Score global** | **29** | **79** | **49** | **99** |

### Tests PageSpeed Insights (Serveurs Google)

| Profil | Site de base (nico3807) | Mon site optimisé (alexvllrs) |
|---|---|---|
| **Mobile** | 58 / 100 | **86 / 100** |
| **Bureau** | 59 / 100 | **98 / 100** |

---

## 4. Détail des problèmes corrigés

### Réseau & Ressources (Gain immédiat)

1. **Double chargement d'image inutilisé (`index.html`) :**
   Un `<link rel="preload">` appelait `img6.jpg?v=2` inutilement tout en bas de page.
   *Fix :* Suppression du preload. Gain immédiat de **3,3 Mo** et une requête en moins.

2. **Images beaucoup trop lourdes (`/img`) :**
   Les visuels originaux de la galerie et du header pesaient entre 800 ko et 3,4 Mo chacun, alors qu'ils étaient affichés en petit. La favicon pesait à elle seule près de 800 ko.
   *Fix :* Création de miniatures redimensionnées en 800px max pour la grille (`img1-thumb.jpg`, etc.), compression de l'image de fond `hero-bg.jpg` (140 ko au lieu de 3,4 Mo) et nettoyage de la favicon.

3. **Absence de lazy-loading et dimensions manquantes :**
   Toutes les cartes chargeaient leurs images dès l'arrivée sur la page sans attributs `width`/`height`.
   *Fix :* Ajout des attributs de taille, passage en `loading="lazy"` sur les cartes 2 à 6 et attribution d'une `fetchpriority="high"` uniquement sur la première image (LCP).

4. **Fichier 404 et préchargement casé (`index.html`) :**
   Un script `polyfills.js` renvoyait une 404 et une police préchargée levait un warning console (`crossorigin` manquant).
   *Fix :* Nettoyage de la balise du polyfill inutile et suppression du preload de la police inutilisée.

5. **Chaîne de requêtes CSS :**
   Les styles s'appelaient en cascade via des `@import` (`styles.css` -> `extra.css` -> `theme-legacy.css`).
   *Fix :* Regroupement de toutes les règles CSS dans un seul fichier `styles.css` propre (2,1 ko) pour supprimer la chaîne d'appels.

---

### JavaScript & Performances CPU

6. **Injection DOM en boucle (`js/app.js` - `buildKeywordIndex`) :**
   Le script faisait un `innerHTML +=` dans une boucle de 400 mots-clés, ce qui forçait le navigateur à recalculer l'arbre DOM à chaque élément.
   *Fix :* Concaténation de la chaîne globale en mémoire puis injection unique dans le DOM.

7. **Forced Reflow au scroll (`js/app.js` - `revealCards`) :**
   À chaque cran de molette, le script lisait `getBoundingClientRect` puis modifiait le style dans la même boucle, créant un reflow forcé massif sur les cartes et le calendrier (plus de 9 secondes de calculs cumulés sur le profil).
   *Fix :* Séparation nette des lectures de positions et des écritures de styles pour stopper le recalcul en boucle.

8. **Script tiers volumineux (`js/vendor/ui-kit.min.js`) :**
   Presque 3 secondes de CPU perdues au chargement à cause d'une boucle active (`_cal`), d'un tableau massif en mémoire (`_pool`) et d'un `MutationObserver` agressif qui observait tout le DOM sans être exploité.
   *Fix :* Élimination des fonctions mortes tout en gardant les helpers essentiels (`qs`, `qsa`, `ready`).

9. **Tracking parasite (`js/analytics.js`) :**
   Un listener lourd tournait sans arrêt en tâche de fond.
   *Fix :* Nettoyage du script (poids divisé par deux) et passage en `defer` sur `index.html`.

---

## 5. Vérification du rendu (Iso-rendu)

J'ai contrôlé l'ensemble du site après déploiement pour m'assurer que rien n'était cassé :
- La galerie d'images et les animations au scroll fonctionnent normalement.
- Le widget calendrier s'affiche et réagit bien au défilement.
- Le filtre par mot-clé de la photothèque est fonctionnel avec sa liste de suggestions.
- Les pages de détail (`img1.html` à `img6.html`) continuent d'afficher les visuels originaux en haute qualité.

---

## 6. Axes d'amélioration pour aller plus loin

Si le site devait être mis en prod sur un vrai serveur :
- **Compression serveur :** Activer Brotli ou Gzip sur le serveur HTTP pour gagner encore sur le CSS/JS.
- **Cache HTTP :** Configurer des en-têtes `Cache-Control` agressifs avec du *cache busting* pour les fichiers statiques.
- **Formats récents :** Convertir les visuels JPEG en WebP ou AVIF pour gratter 20 à 30 % de poids en plus.

---

## Annexe — Fichiers de preuves (`audit/`)

- `reseau-avant.png` : Bilan réseau initial (17,5 Mo / 22 requêtes).
- `reseau-apres-images.png` : Bilan réseau final (631 ko / 15 requêtes).
- `lighthouse-mobile-1.png` à `-3` : Relevés initiaux Mobile.
- `lighthouse-mobile-apres-image-1.png` à `-3` : Relevés finaux Mobile.
- `performances-frise.png` & `performances-scroll.png` : Preuves des blocs rouges et du reflow sur le CPU.