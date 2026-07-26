# Roadmap d'amélioration — refuges.info

Plan incrémental issu de [`AUDIT.md`](AUDIT.md), classé du plus rentable au plus coûteux.

**Contraintes structurantes :**
- Pas de big-bang. Le site est vivant et alimenté par des contributeurs bénévoles : **chaque lot est livrable et déployable indépendamment.**
- HTML natif et simplicité privilégiés. **Aucun framework front, aucun moteur de template introduit.**
- **Contenu et données non modifiés.** Aucun lot ci-dessous ne touche aux tables de contenu.

**Effort exprimé en jours-personne**, pour un développeur familier de PHP et du dépôt. Total : **~37 à 53 j** répartis sur 8 lots, dont **1,5 j** produisant l'essentiel des gains mesurables.

---

## Cadrage retenu

Cette version intègre les réponses aux questions ouvertes de la première itération. Quatre décisions structurent le plan :

1. **Usage mixte, mais non recouvrant : mobile = en montagne, desktop = en préparation.** Le mobile *est* le cas de connexion dégradée. → La compression (0.1) est verrouillée en priorité absolue, et le **confort mobile passe devant les chantiers de contribution**.
2. **Aucune obligation légale RGAA.** Le référentiel sert de **levier d'ergonomie et de lisibilité**, pas de cible de conformité déclarable. → **Retirés du périmètre** : audit formel avec lecteur d'écran, déclaration d'accessibilité, mise en conformité du forum phpBB. En pratique cela ne change presque rien aux Lots 0 à 3 — les mêmes corrections servent les deux objectifs — mais cela **autorise à s'arrêter quand le bénéfice utilisateur est atteint**, sans chercher la couverture critère par critère.
3. **Statut de maintenance de `myol` : inconnu, à instruire sur le forum du projet.** → Le Lot 6 (unification cartographique) reste **bloqué en attente**.
4. **Les formulaires de contribution peuvent évoluer librement**, fusion d'écrans incluse. → Lot 5 à périmètre complet.

**Un lot a été ajouté suite à une vérification en production** : les formulaires d'export et de flux RSS comptent **631 et 636 cases à cocher** sur un seul écran (mesuré, cf. AUDIT §3.3). L'environnement local, avec 7 polygones, masquait totalement ce problème. C'est aujourd'hui le pire point d'ergonomie du site sur mobile → **Lot 3bis**.

---

## Vue d'ensemble

| Lot | Thème | Effort | Impact | Risque | Statut |
|---|---|---|---|---|---|
| **0** | Quick wins | **1,5 j** | ⭐⭐⭐⭐⭐ | quasi nul | prêt |
| **1** | Structure sémantique & clavier | 3-4 j | ⭐⭐⭐⭐⭐ | faible | prêt |
| **2** | Formulaires accessibles | 3-4 j | ⭐⭐⭐⭐ | faible | prêt |
| **3** | Confort mobile | 4-5 j | ⭐⭐⭐⭐⭐ | moyen | **remonté** (Q1) |
| **3bis** | Sélecteurs de massifs (631 cases) | 5-6 j | ⭐⭐⭐⭐⭐ | moyen | **nouveau** |
| **4** | Tests et intégration continue | 6-9 j | ⭐⭐⭐⭐ *(indirect)* | nul | **nouveau** |
| **5** | Robustesse des formulaires de contribution | 6-8 j | ⭐⭐⭐⭐ | moyen | périmètre complet (Q4) |
| **6** | Carte accessible & unification carto | 8-15 j | ⭐⭐⭐ | élevé | **6c bloqué** (Q3) |

**Séquencement recommandé :** `0 + 4a → 1 → 2 → 4b/4c → 3 → 3bis → 4d/4e → 5 → 6a/6b`, avec `6c` conditionné à la réponse sur myol.

Les Lots 0 à 2 restent en tête : ils sont sans risque, ils portent la majorité des nœuds mesurés, et ils préparent le terrain des suivants (le lien d'évitement du Lot 1 est un prérequis d'ergonomie du Lot 3bis). Le Lot 3 remonte devant les formulaires de contribution parce que le mobile est le cas d'usage terrain.

**Le Lot 4 est le seul lot transverse, et il est volontairement fractionné plutôt que séquencé d'un bloc.** Le numéro 4, libéré quand le Lot 4 initial est devenu le Lot 5 pour laisser la place au 3bis, est réutilisé ici. Son sous-lot `4a` (lint + CI minimale, 0,5 j) se livre avec le Lot 0 ; les tests de non-régression `4b`/`4c` doivent être en place **avant le Lot 3**, qui est le plus exposé aux régressions de toute la roadmap. Le Lot 4 n'a aucun impact utilisateur direct — son impact est de rendre les lots 3, 3bis, 5 et 6 livrables sans pari.

---

## Lot 0 — Quick wins

**Objectif :** supprimer les non-conformités et bugs dont la correction est mécanique, sans aucune refonte. Chaque item est indépendant et tient en moins d'une journée.

**Effort total : 1,5 j.** **Risque : quasi nul.** Aucun item ne modifie de structure HTML ni de logique métier.

### 0.1 — Activer la compression sur HTML, CSS et JS ⭐ le plus rentable du projet

- **Fichier :** `.htaccess:65`
- **Action :** ajouter `text/html text/css application/javascript text/javascript image/svg+xml` à la directive `AddOutputFilterByType DEFLATE` existante.
- **Impact mesuré :** **−72 % sur les assets, soit −575 Ko** par visite non cachée. Sur Edge à ~50 Ko/s, le chargement des assets de la fiche point passe de ~16 s à ~4,5 s.
- **Effort :** 15 min. **Risque : nul** (le module est déjà chargé et déjà utilisé sur 4 types MIME).
- **Vérification :** `curl -sI -H "Accept-Encoding: gzip" https://.../myol/dist/myol.js | grep -i content-encoding` doit renvoyer `gzip`.

### 0.2 — Rendre les menus du bandeau ouvrables au clavier

- **Fichier :** `vues/bandeau.css:62-68`
- **Action :** ajouter les variantes `:focus-within` à la liste de sélecteurs qui porte déjà `.menu-touch` et `.menu-hover` :
  ```css
  .menu-bouton:focus-within:not(.menu-liste)>ul,
  .menu-bouton:focus-within:not(.menu-liste)>form,
  .menu-bouton:focus-within:not(.menu-liste)>p { opacity: 1; z-index: 2000; }
  ```
- **Impact :** corrige le constat **B1**, le plus grave de l'audit — 18 contrôles de navigation aujourd'hui focusables mais invisibles deviennent visibles quand le focus les atteint. RGAA 10.7, 12.8, 7.3.
- **Effort :** 2 h (dont test croisé souris/tactile/clavier, la logique JS de `menu-hover`/`menu-touch` devant continuer à fonctionner).
- **Risque : faible.** À surveiller : que le menu ne s'ouvre pas de façon parasite quand le focus traverse la page.
- **Vérification :** tabuler depuis le haut de la page ; chaque élément de menu atteint doit être visible. Script de contrôle fourni en §Vérification.

### 0.3 — Corriger le contraste des palettes été et automne

- **Fichier :** `vues/style.css.php:45` et `:53`
- **Action :** `$couleur_lien="5f8c11"` → `"538005"` (ratio 3,83 → 4,51) ; `$couleur_lien="cf5d32"` → `"b44217"` (3,27 → 4,62). Palette hiver inchangée (déjà à 5,58).
- **Impact :** corrige **M1** — 124 nœuds `color-contrast` relevés par axe, tous imputables à une seule paire de couleurs. Rend le site conforme au contraste **6 mois de l'année où il ne l'est pas**. RGAA 3.2.
- **Effort :** 1 h. **Risque : nul techniquement**, mais **à valider avec les mainteneurs** : la charte saisonnière est un choix identitaire assumé (`vues/style.css.php:9-12`).
- **Vérification :** forcer la date système ou les variables, puis contrôler au vérificateur de contraste (cible ≥ 4,5:1 pour texte normal).

### 0.4 — Ajouter le titre manquant sur le formulaire de commentaire

- **Fichier :** `controlleurs/point_ajout_commentaire.php`
- **Action :** affecter `$vue->titre` sur le chemin nominal (il ne l'est aujourd'hui qu'en branche d'erreur, ligne 135). Ex. : `« Ajouter une information sur <nom du point> »`.
- **Impact :** corrige **B6** (`<title></title>` en production). RGAA 8.5.
- **Effort :** 15 min. **Risque : nul.**
- **Vérification :** `curl -s .../point_ajout_commentaire/37 | grep '<title>'`.

### 0.5 — Icônes de légende : `alt` redondants et absence de garde-fou

> **Révisé.** La première version de ce lot annonçait une icône cassée en production pour le type « sommet ». **Vérification faite sur `https://www.refuges.info/nav` : c'est faux.** La légende de production affiche 7 types, tous avec une icône valide, et `sommet` n'y figure pas. Le bug ne se manifeste qu'en local, parce que le dump SQL public est désynchronisé de la production (il contient `sommet` mais pas `grotte`). Ce qui reste est réel mais mineur.

- **Fichiers :** `vues/nav.html:51-54`, `includes/config.php:49-58`, `docker/init/refuges-local.sql.gz`
- **Actions :**
  1. Passer les `alt` des icônes de légende à `""` — les 7 icônes portent aujourd'hui `alt="icone de <type>"` alors que le libellé du type est déjà en texte juste après. Un lecteur d'écran annonce « icone de cabane non gardée, cabane non gardée ». **Confirmé en production.** RGAA 1.2.
  2. Ajouter un garde-fou sur le mapping (`?? 'defaut'`) pour qu'un type ajouté en base sans entrée de config ne produise plus un `src` vide silencieusement.
  3. *(hors Lot 0, à signaler aux mainteneurs)* **resynchroniser le dump SQL public** : en l'état, tout contributeur qui monte l'environnement local voit une légende cassée et risque de « corriger » un bug inexistant.
- **Impact :** mineur en accessibilité, **réel en confort de contribution** (le point 3 fait perdre du temps à chaque nouvel arrivant).
- **Effort :** 1 h pour 1 et 2. **Risque : nul.**
- **Vérification :** aucun `src="/images/icones/.svg"` dans le HTML généré ; les `alt` des icônes de légende sont vides.

### 0.6 — Initialiser `$utilisateurs`

- **Fichier :** `modeles/utilisateur.php:48-58`
- **Action :** `$utilisateurs = [];` avant la boucle `while`.
- **Impact :** supprime un warning émis **avant le `<!doctype>`** (document invalide) quand la requête ne renvoie aucune ligne, et un `return null` qui casse le `foreach` de `vues/point_formulaire_recherche.html:172`. Masqué en production par `display_errors=off`, mais réel.
- **Effort :** 10 min. **Risque : nul.**

### 0.7 — Nettoyages sans risque

| Action | Fichier | Effort |
|---|---|---|
| Supprimer le `<meta viewport user-scalable=no>` (code mort : `$vue->css` n'est jamais assigné) | `vues/_page.html:81-83` | 15 min |
| Corriger les commentaires debug/prod inversés | `vues/_page.html:42,49` | 10 min |
| Corriger le `<ul>` contenant du texte hors `<li>` | `vues/nav.html` | 20 min |
| Supprimer le `<h3 id="titrepage">` vide, ou le remplir | `vues/nav.html` | 20 min |
| `h5` → `h2` sur « Discussions sur… » (saut de niveau) | `vues/point.html` | 15 min |
| Porter `ExpiresByType` CSS/JS de 6 h à 1 an (le cache-busting par `filemtime` est déjà en place) | `.htaccess:55-56` | 20 min |
| Supprimer le CDATA XHTML résiduel | `vues/_pied.html:25` | 10 min |

**Critères de sortie du Lot 0 :** `curl` confirme `Content-Encoding: gzip` sur HTML/CSS/JS ; les nœuds `color-contrast` d'axe passent de 124 à ~0 ; `document-title` à 0 ; les menus s'ouvrent au clavier ; aucune régression visuelle sur les 6 pages de référence.

> **Les items 0.1 à 0.4 ont été vérifiés comme actifs en production** (compression absente, `opacity:0` sans `focus-within`, palette été à 3,83:1, `<title></title>` sur un point réel). Ce ne sont pas des hypothèses de laboratoire : ce sont quatre corrections déployables qui produisent un effet immédiat en ligne.

---

## Lot 1 — Structure sémantique et parcours clavier

**Objectif :** donner au site une structure de page exploitable par les technologies d'assistance et par le clavier. C'est le lot qui débloque le plus de critères RGAA d'un coup, grâce à la mutualisation du bandeau et du squelette.

**Fichiers concernés :** `vues/_page.html`, `vues/bandeau.html`, `vues/_pied.html` (mutualisés — une intervention couvre tout le site), puis les ~10 vues de premier niveau pour les `h1`.

**Effort : 3-4 j.** **Impact : ⭐⭐⭐⭐⭐** (corrige M2, 267 nœuds axe). **Risque : faible** — modifications de balisage sans changement de logique, mais qui touchent tous les templates : le CSS reposant sur des sélecteurs de descendance devra être vérifié.

### Contenu

1. **Poser les landmarks** dans les 3 fichiers mutualisés : `<header>` autour du bandeau, `<nav>` sur les menus, `<main>` autour de `include(fichier_vue($vue->type.'.html'))`, `<footer>` sur le pied. Attention : `vues/bandeau.html:312` ouvre `<div id="scrolable">` que `vues/_pied.html:14` referme — cette imbrication entre fichiers doit être respectée.
2. **Ajouter un lien d'évitement** vers `<main>` en première position du `<body>` (RGAA 12.7). Il devient particulièrement utile une fois 0.2 livré.
3. **Ajouter un `h1` unique et pertinent** sur les 5 pages qui en manquent : `/`, `/nav`, `/point_formulaire_recherche`, `/point_ajout`, `/point_ajout_commentaire`. Reprendre `$vue->titre` là où il est déjà calculé.
4. **Reprendre la hiérarchie des titres** : l'accueil part aujourd'hui en `h4`, `/point_ajout` n'a qu'un `h5`. Descendre les niveaux d'un cran cohérent après l'ajout du `h1`.
5. **Nommer les 9 boutons sans nom accessible** (B4). Deux périmètres distincts : le géocodeur (`leaflet/geocoder/`, `aria-label` posable localement) et les contrôles myol (voir Lot 6 — une surcharge locale est possible en attendant).

### Risques et précautions

- L'introduction de `<main>`/`<header>` peut modifier la cascade CSS si des règles ciblent `body > div`. **Prévoir une passe de vérification visuelle sur les 6 pages de référence, en clair et en sombre, desktop et mobile.**
- Le CSS des landmarks doit rester neutre (`display: contents` si nécessaire) pour éviter tout décalage de mise en page.

### Vérification

- axe : `region`, `landmark-one-main`, `page-has-heading-one`, `heading-order`, `button-name` à **0**.
- Parcours clavier : le lien d'évitement est la première tabulation et fonctionne.
- Lighthouse accessibilité ≥ 95 sur les 4 pages mesurées (contre 61-91 aujourd'hui).

---

## Lot 2 — Formulaires accessibles

**Objectif :** rendre les formulaires utilisables au lecteur d'écran. Le regroupement par `<fieldset>`/`<legend>` étant **déjà correct** sur les deux formulaires les plus complexes, le travail restant est essentiellement l'étiquetage.

**Fichiers concernés :** `vues/point_formulaire_recherche.html` (10 champs), `vues/point_ajout_commentaire.html` (4 champs dont le textarea principal), `vues/bandeau.html` (champ `nom`, présent sur **toutes** les pages), `vues/point.html` (`select#cadre-select`), `vues/nav.html` (géocodeur, import de fichier).

**Effort : 3-4 j.** **Impact : ⭐⭐⭐⭐** (corrige B5, RGAA 11.1/11.2). **Risque : faible.**

### Contenu

1. **Étiqueter les ~20 champs sans `<label for>` ni `aria-label`.** Priorité au textarea `texte` du formulaire de commentaire (parcours de contribution le plus utilisé) et au champ `nom` du bandeau.
2. **Ajouter `<fieldset>`/`<legend>` à `point_ajout_commentaire`** (0 aujourd'hui, 3 `<label>` pour 9 champs).
3. **Étiqueter les 8 cases `champs_null[]`** du formulaire de recherche (les « cases rouges » réservées aux modérateurs).
4. **Ajouter les attributs `autocomplete`** pertinents sur les champs d'identité du contributeur anonyme (RGAA 11.13).
5. **Reprendre le captcha statique.** `includes/config.php:112-113` : question « Entrez la lettre g », réponse « g ». Ce n'est pas un problème d'accessibilité en soi (c'est du texte, pas une image) mais l'absence d'étiquette sur `lettre_verification` en est un. À traiter avec l'étiquetage ; **sa refonte anti-spam est un sujet distinct, hors périmètre de cet audit.**

### Vérification

- axe : `label`, `select-name` à **0** sur les 6 pages.
- Chaque champ annonce une étiquette pertinente à la navigation au lecteur d'écran (test manuel requis).

---

## Lot 3 — Confort mobile

**Objectif :** rendre le site confortable sur un téléphone, cas d'usage dominant en préparation et en course. La base est saine (aucun débordement horizontal, `<meta viewport>` correct partout) : le travail porte sur les cibles tactiles et l'échelle typographique.

**Fichiers concernés :** `vues/bandeau.css` (l'essentiel du volume), `vues/style_pages.css`, `vues/style_formulaire.css`, `vues/style.css.php`.

**Effort : 4-5 j.** **Impact : ⭐⭐⭐⭐** (corrige M5, M6). **Risque : moyen** — touche le CSS global, donc surface de régression visuelle large.

### Contenu

1. **Agrandir les cibles tactiles à 24×24 px minimum** (44×44 recommandé). Le cas dominant est structurel : **tous les liens du bandeau font 19 px de haut**, le bouton `🔍` fait 17×22. Augmenter le `padding` vertical dans `vues/bandeau.css` traite l'essentiel des 28 à 72 cibles trop petites par page.
2. **Introduire une échelle en `rem`.** Zéro `rem` aujourd'hui, 137 `px`. Poser une taille de base sur `:root` et convertir progressivement les tailles de police, en commençant par `style_pages.css` (92 `px`). Les 79 `em` existants n'ont pas besoin d'être touchés.
3. **Harmoniser les 8 breakpoints** (640 / 641 / 699.9 / 700 / 800 / 1000 / 1200 / 1300) vers 3 ou 4 valeurs déclarées en variables. Chantier de lisibilité autant que de comportement.
4. **Réduire les 42 `!important`** (30 dans `style_formulaire.css`) au fil des interventions — pas comme objectif isolé.

### Risques et précautions

- **C'est le lot le plus exposé aux régressions visuelles.** Recommandation : livrer fichier par fichier plutôt qu'en une passe, et **capturer des références visuelles avant/après** sur les 6 pages × 2 viewports × 2 thèmes (script fourni en §Vérification).
- L'agrandissement des cibles du bandeau modifie la hauteur de l'en-tête : vérifier les `calc(100vh - 33px)` de `vues/bandeau.css:43` qui en dépendent.

### Vérification

- Sonde de cibles tactiles : 0 élément interactif < 24 px sur les 6 pages en 375 px.
- Zoom texte à 200 % sans perte de contenu ni de fonction (RGAA 10.4).
- Aucun débordement horizontal en 320 px (RGAA 10.11) — aujourd'hui déjà conforme en 375 px, à revérifier en 320.

---

## Lot 3bis — Sélecteurs de massifs : 631 cases à cocher sur un écran

**Nouveau lot, issu d'une vérification en production.** Il ne figurait pas dans la première version du plan parce que l'environnement local (7 polygones) rend le problème invisible.

**Le problème, mesuré :**

```
$ curl -s https://www.refuges.info/formulaire_rss_et_nouvelles | grep -oE 'type=.checkbox' | wc -l
631
$ curl -s https://www.refuges.info/formulaire_exportations   | grep -oE 'type=.checkbox' | wc -l
636
```

Conséquences concrètes :
- **Sur mobile, ces formulaires sont en pratique inutilisables** — le défilement seul demande des centaines de gestes. C'est directement le cas d'usage « préparation de course » qui est touché.
- **Au clavier, atteindre le bouton de validation exige 631 tabulations**, sans lien d'évitement (le Lot 1 en fournit un, ce qui atténue mais ne résout pas).
- Les 6 boutons « tout cocher / tout décocher » sont un pansement, et reposent sur des handlers `onclick` inline **dupliqués à l'identique entre les deux formulaires**.

**Fichiers concernés :** `vues/formulaire_exportations.html`, `vues/formulaire_rss_et_nouvelles.html`, `vues/formulaire_exportations.js`, `controlleurs/formulaire_exportations.php`, `controlleurs/formulaire_rss_et_nouvelles.php`.

**Effort : 5-6 j.** **Impact : ⭐⭐⭐⭐⭐.** **Risque : moyen** — ces formulaires produisent des URLs d'API dont des utilisateurs ont pu garder des copies ; **le format des URLs générées doit rester identique.**

### Contenu

1. **Remplacer la liste plate par une sélection hiérarchique** zone → massif, en HTML natif : `<details>`/`<summary>` par zone, replié par défaut. Aucun JS nécessaire, aucune dépendance ajoutée — conforme à la contrainte « HTML natif d'abord ».
2. **Ajouter un champ de filtrage** au-dessus de la liste. Le composant existe déjà dans le dépôt : `vues/autocomplete.js` (4,4 Ko) est utilisé par le formulaire de recherche avancée pour exactement ce besoin. **À réutiliser, pas à réécrire.**
3. **Factoriser le bloc de sélection** entre les deux formulaires, qui le dupliquent aujourd'hui (`vues/formulaire_rss_et_nouvelles.html:2` inclut même le JS de l'autre formulaire pour éviter une troisième copie). Une vue partielle incluse deux fois.
4. **Remplacer les 12 handlers `onclick` inline** par des écouteurs délégués dans le fichier JS factorisé.
5. **Conserver le comportement sans JS** : la liste complète doit rester soumettable si les `<details>` ne sont pas scriptés.

### Vérification

- À sélection identique, l'URL d'API générée est **strictement identique** à celle produite avant le lot (test de non-régression prioritaire — comparer sur une dizaine de combinaisons).
- Formulaire utilisable au pouce sur 375 px : moins de 20 gestes pour sélectionner 3 massifs dans 2 zones différentes.
- Moins de 60 tabulations pour atteindre le bouton de validation depuis le haut du formulaire.
- Fonctionne avec JavaScript désactivé.

---

## Lot 4 — Tests et intégration continue

**Objectif :** transformer le seul facteur de risque transverse de cette roadmap en filet de sécurité. Aujourd'hui, **rien ne détecte une régression avant un utilisateur en montagne** : pas un test, pas un lint, pas un workflow. Chaque lot suivant est un pari sur la relecture humaine.

**État des lieux, vérifié :**

```
$ ls .github/            → FUNDING.yml   (aucun workflow)
$ ls composer.json       → absent        (aucune dépendance de dev, aucun autoload)
$ find . -name '*[Tt]est*.php' -not -path './forum/*'   → aucun résultat
$ find . -name '*.php' -not -path './forum/*' | wc -l   → 80
```

**Ce qui joue en notre faveur :** le harnais d'exécution existe déjà. `make up` monte PostGIS + PHP 8.4 et charge un snapshot, `make seed` injecte un jeu de données déterministe (massifs, points, commentaires). **Un test n'a donc pas besoin d'inventer son environnement — il a besoin d'un `make test` qui s'appuie sur ce qui est déjà là.** C'est ce qui rend ce lot chiffrable à moins de 10 j sur un code sans aucune couverture.

**Effort : 6-9 j.** **Impact : ⭐⭐⭐⭐ mais indirect** (aucun gain utilisateur visible ; conditionne la sûreté des lots 3, 3bis, 5 et 6). **Risque : nul** — aucun fichier du chemin de production n'est modifié. Tout ce lot vit dans `tests/`, `outils/` et `.github/workflows/`.

### Contraintes à respecter

Le projet est alimenté par des **contributeurs bénévoles**. Un dispositif de test qui les ralentit ou les bloque sera contourné, puis abandonné. Trois règles en découlent :

1. **Rien ne devient bloquant avant d'être vert et stable.** Chaque workflow démarre en mode informatif (`continue-on-error`), et ne passe en *required check* qu'après ~2 semaines sans faux positif.
2. **Aucune dépendance ajoutée à la production.** Composer est introduit avec des `require-dev` **uniquement** ; `vendor/` reste hors du dépôt (`.gitignore`) et le déploiement actuel — copie de fichiers, sans étape de build — n'est pas touché.
3. **Le test doit être lançable en une commande, sans lire de documentation** : `make test`. Un contributeur qui ne veut pas savoir ce qu'est PHPUnit doit pouvoir vérifier qu'il n'a rien cassé.

### 4a — Lint et CI minimale (0,5 j, à livrer avec le Lot 0)

Le plus rentable du lot, et sans aucun prérequis.

- **Créer `.github/workflows/lint.yml`** : sur chaque push et chaque PR, `php -l` sur tous les `.php` hors `forum/` et `myol/` (80 fichiers, < 10 s). Un `<?php` mal refermé ou un point-virgule manquant — le mode de panne le plus courant sur ce code sans build — devient impossible à merger.
- **Ajouter au même workflow** : validation JSON/YAML des fichiers de configuration, et détection des fins de ligne CRLF et des marqueurs de conflit (`<<<<<<<`) restés dans un fichier.
- **Cible : < 1 minute, sans Docker, sans base de données.** C'est ce qui permet de l'appliquer à *toutes* les branches sans agacer personne.
- **Cible complémentaire :** ajouter `stylelint` sur `vues/*.css` **est tentant mais reporté** — le CSS existant a 42 `!important` et sortirait des centaines d'avertissements. À reprendre pendant le Lot 3, quand ce CSS est de toute façon en cours de réécriture.

### 4b — Tests de contrat sur l'API (2 j)

**C'est ici que le rapport valeur/coût est le meilleur du lot.** L'API publique (`/api/bbox`, `/api/point`, `/api/massif`, `/api/commentaires`, `/api/polygones`, `/api/contributions`) a des **consommateurs externes qu'on ne contrôle pas** : applications tierces, scripts de contributeurs, URLs conservées par des utilisateurs. Une régression de format y est invisible en revue de code et cassante en aval.

- **Tests de golden file** : pour chaque route d'API, sur les données déterministes de `make seed`, la réponse est comparée à une réponse de référence versionnée dans `tests/fixtures/api/`. Toute évolution de format devient un **diff explicite à valider dans la PR**, au lieu d'un effet de bord.
- **Assertions structurelles** en plus du diff brut : code HTTP, `Content-Type`, JSON parsable, présence des clés documentées dans `vues/api/doc/`, et **absence de toute sortie parasite avant le corps** (le mode de panne du constat 0.6 : un warning PHP émis avant le `<!doctype>` ou avant le JSON).
- **Régression de référence à couvrir immédiatement :** le commit `1634627e` (« Bug api depuis. Les cartes n'affichaient plus ») est exactement la classe de bug qu'un test de contrat aurait attrapé. **Écrire le test qui échoue sur ce bug avant de l'ajouter au harnais** — c'est la meilleure preuve que le dispositif sert à quelque chose.
- **Prérequis du Lot 3bis :** sa vérification prioritaire est « à sélection identique, l'URL d'API générée est strictement identique ». Ces tests de contrat en sont l'outillage direct.

### 4c — Smoke tests HTTP sur les routes (1-2 j)

Couvrir **toutes** les routes en lecture, sans chercher à tester la logique métier. Sur une application PHP sans autoload ni injection, un test de bout en bout via HTTP coûte 10× moins cher qu'un test unitaire et attrape la majorité des régressions réelles.

- **Énumérer les routes** depuis `routes/generales.routes.php`, `routes/api.routes.php` et `routes/gestion.routes.php`, et vérifier pour chacune : HTTP 200 (ou la redirection attendue), aucun `Warning`/`Notice`/`Fatal error` dans le corps ni dans les logs du conteneur, HTML dont les balises se referment, `<title>` non vide.
- **Indicateur de couverture n° 1 : le taux de routes atteintes par au moins un smoke test.** Cible **100 %** des routes en lecture. C'est mesurable, non ambigu, et bien plus parlant sur ce code qu'un pourcentage de lignes.
- **Le `<title>` vide et le warning avant `<!doctype>` corrigés en 0.4 et 0.6 deviennent des tests**, donc non régressables.
- **Ce test devient le garde-fou du Lot 3** : la réécriture du CSS global ne change aucun HTML, donc un smoke test vert reste vert — et s'il rougit, c'est que le Lot 3 a débordé de son périmètre.

### 4d — Tests unitaires et couverture mesurée (2-3 j)

- **Introduire PHPUnit en `require-dev`** (premier `composer.json` du dépôt), avec un `phpunit.xml` limitant le périmètre de couverture à `includes/` et `modeles/` — les seules couches où un pourcentage de lignes veut dire quelque chose. Les contrôleurs et les vues sont couverts par 4b/4c, pas ici.
- **Commencer par `includes/mise_en_forme_texte.php`** (444 lignes, 18 fonctions, dont `bbcode2html`, `bbcode2markdown`, `bbcode2txt`, `protege`, `replace_url`, `lien_inter_fiches`) : **des fonctions quasi pures, sans base de données, sur du texte contribué**. Le meilleur point d'entrée possible — et le code le plus risqué du dépôt, puisqu'il traite de l'entrée utilisateur et produit du HTML.
- **Puis les validations de `modeles/point.php:726-793`** (nom vide, caractères interdits, latitude/longitude hors bornes, altitude non entière ou > 8848 m). Elles sont explicitement le socle du Lot 5 : *« changer le véhicule du message, pas réécrire la validation »*. **Les figer par des tests avant le Lot 5 est ce qui rend cette promesse tenable.**
- **Indicateur de couverture n° 2 : couverture de lignes sur `includes/` + `modeles/`,** mesurée avec `pcov` (plus léger que Xdebug en CI). **Cible de sortie de lot : ≥ 60 % sur `includes/mise_en_forme_texte.php`, ≥ 35 % sur l'ensemble `includes/` + `modeles/`.**
- **Mécanisme de cliquet plutôt que seuil absolu** : la CI échoue si la couverture **baisse** par rapport à la valeur versionnée, pas si elle n'atteint pas un chiffre arbitraire. C'est le seul mode qui fonctionne sur du legacy — un seuil à 80 % d'emblée serait ignoré, un cliquet fait monter la couverture par sédimentation, au rythme des lots.

> **Un pourcentage de couverture global sur ce dépôt serait un indicateur trompeur** et n'est volontairement pas proposé : la majorité des 80 fichiers PHP sont des contrôleurs qui mélangent requête, logique et `include` de vue. Les rendre unitairement testables exigerait de les refactorer — un chantier qui n'est *pas* dans cette roadmap et qui contredirait la contrainte « pas de big-bang ». **Deux indicateurs ciblés (routes couvertes, lignes de la couche testable) valent mieux qu'un chiffre unique flatteur ou décourageant.**

### 4e — Harnais d'accessibilité en CI (1 j)

Reprend et **remplace** la recommandation transverse de 0,5 j de la section « Vérification et non-régression » : les scripts `axe-run.mjs` et `kbd-probe.mjs` de l'audit sont versionnés dans `outils/audit/` avec leur `package.json`, puis exécutés en CI.

- **Sur PR touchant `vues/`** : axe sur les 6 pages de référence, plus la sonde clavier. Le nombre de nœuds est comparé à un budget versionné — même mécanisme de cliquet que la couverture : **le total ne peut plus augmenter.**
- **Verrouille les gains des Lots 0 à 2 :** les 124 nœuds `color-contrast` et les 267 nœuds `region`/`landmark` corrigés ne peuvent plus revenir par inadvertance. Sans ce cliquet, un unique futur ajout de bandeau suffit à les réintroduire.
- **Ajouter la sonde de cibles tactiles** au même harnais quand le Lot 3 la produit.
- Le rapport axe est publié en artefact de la PR, lisible sans installer quoi que ce soit — condition pour que des bénévoles s'en servent.

### Découpage en CI : deux workflows, pas un

| Workflow | Déclencheur | Contenu | Durée cible | Bloquant |
|---|---|---|---|---|
| `lint.yml` | tout push, toute PR | 4a (`php -l`, YAML/JSON, CRLF, marqueurs de conflit) | < 1 min | **oui**, dès le Lot 0 |
| `tests.yml` | PR uniquement | `make up` + `make seed` + 4b, 4c, 4d, 4e | < 8 min | après 2 semaines sans faux positif |

Séparer les deux est ce qui rend le dispositif acceptable : le retour immédiat est gratuit et universel, la vérification lourde est réservée aux PR. **Un `make test` unique fait tourner l'ensemble en local**, à l'identique de la CI, sur la stack Docker déjà documentée dans `docker/README.md`.

### Risques et précautions

- **Le risque principal de ce lot est le faux positif**, pas la panne. Un test instable (dépendance à l'heure système via la charte saisonnière de `vues/style.css.php`, à l'ordre de tri PostgreSQL, à une tuile réseau) détruit la confiance plus vite qu'il n'apporte de sûreté. **Neutraliser explicitement la date dans les tests** — la palette saisonnière change 4 fois par an et rendrait toute comparaison visuelle non déterministe.
- **Le dump SQL public est désynchronisé de la production** (cf. 0.5, point 3) : les fixtures de test doivent s'appuyer sur `make seed`, déterministe et versionné, **jamais** sur `refuges-local.sql.gz`.
- **`forum/` (phpBB) et `myol/dist/` sont exclus de tout périmètre de test**, comme ils le sont de l'audit. Les inclure ferait exploser le coût sans bénéfice.
- **Ne pas viser la couverture pour elle-même.** Chaque test ajouté par ce lot doit correspondre soit à un bug réel déjà survenu, soit à un comportement qu'un lot suivant s'engage à préserver. Les autres sont du coût de maintenance sans contrepartie.

### Critères de sortie du Lot 4

- `make test` passe en local et en CI, sur une machine sans configuration préalable autre que Docker.
- 100 % des routes en lecture atteintes par au moins un smoke test ; 0 route en erreur PHP.
- Toutes les routes d'API couvertes par un test de contrat sur golden file ; le bug du commit `1634627e` a un test de non-régression.
- Couverture ≥ 60 % sur `includes/mise_en_forme_texte.php`, ≥ 35 % sur `includes/` + `modeles/`, avec cliquet actif.
- `lint.yml` est un *required check* ; `tests.yml` est vert de façon stable depuis 2 semaines.
- Le budget de nœuds axe est versionné et ne peut plus augmenter.

---

## Lot 5 — Robustesse des formulaires de contribution

**Objectif :** ne plus perdre la saisie d'un contributeur. C'est le point UX le plus coûteux du site aujourd'hui, et il touche directement la communauté qui alimente la base.

**Le problème :** il n'existe **aucun ré-affichage de formulaire avec le champ en erreur**. Les contrôleurs basculent sur la vue `page_simple` et invitent explicitement à utiliser le bouton Retour du navigateur (`vues/point_modification.html:16-18` porte un FIXME assumé). Sur le formulaire d'ajout de point (~30 champs), **une erreur de validation peut coûter la ressaisie complète.**

**Fichiers concernés :** `controlleurs/point_modification.php`, `controlleurs/point_formulaire_modification.php`, `controlleurs/point_ajout_commentaire.php`, `modeles/point.php` (validations lignes 726-793), `vues/point_formulaire_modification.html`, `vues/point_modification.html`.

**Effort : 6-8 j.** **Impact : ⭐⭐⭐⭐** (corrige M7, RGAA 11.10). **Risque : moyen** — touche le chemin d'écriture des contributions. **Aucune modification de données ni de schéma.**

### Contenu

1. **Ré-afficher le formulaire pré-rempli en cas d'erreur**, plutôt que de basculer sur `page_simple`. Les validations existent déjà et sont de bonne qualité dans `modeles/point.php` — il s'agit de **changer le véhicule du message, pas de réécrire la validation.**
2. **Associer chaque message d'erreur à son champ** via `aria-describedby` + `aria-invalid` (RGAA 11.10).
3. **Ajouter les contraintes HTML natives** (`required`, `type="number"`, `min`/`max`, `pattern`) en miroir des validations serveur, pour un retour immédiat sans JS. Cohérent avec la contrainte « HTML natif d'abord ».
4. **Réduire le parcours d'ajout de 3 à 2 écrans** en fusionnant `/point_ajout` (qui ne sert **qu'à choisir un type de point**) avec le formulaire.
5. **Supprimer l'écran intermédiaire de `/formulaire_exportations`**, dont le seul contenu est un lien à cliquer (3 clics → 2 pour obtenir un fichier).
6. **Rediriger après succès** sur l'ajout de commentaire (aujourd'hui la même page se ré-affiche, sans redirection — motif POST/Redirect/GET absent).

### Précautions

- **Livrer un formulaire à la fois**, en commençant par `point_ajout_commentaire` (le plus simple et le plus utilisé) avant `point_formulaire_modification` (~30 champs).
- Le hook phpBB de traçabilité `refugesinfo.ajout_commentaire` (`controlleurs/point_ajout_commentaire.php:86-101`) doit continuer à être appelé exactement une fois.

### Vérification

- Soumettre chaque formulaire avec chaque erreur de validation connue : le formulaire revient **pré-rempli**, l'erreur est visible et rattachée au champ, aucune donnée saisie n'est perdue.
- Le compteur de contributions ne doit pas régresser après déploiement (indicateur de non-régression fonctionnelle).

---

## Lot 6 — Carte accessible et unification cartographique

**Objectif :** traiter les deux constats bloquants restants sur la carte, et supprimer la dette structurelle de la double stack. **C'est le seul lot qui comporte un vrai risque de régression fonctionnelle** — d'où sa position en dernier.

**Effort : 8-15 j** (fourchette large, cf. questions ouvertes). **Impact : ⭐⭐⭐** (corrige B2, B3). **Risque : élevé.**

### 6a — Alternative textuelle à la carte (2-3 j, ⭐⭐⭐⭐, risque faible)

**À faire en priorité dans ce lot, et éventuellement à remonter en Lot 1.**

La carte est un `<canvas>` sans `tabindex`/`role`/`aria-label` : contenu totalement invisible aux technologies d'assistance et aucune interaction clavier (**B3**). Rendre un canvas navigable au clavier est coûteux et de valeur douteuse.

**Mais une alternative existe déjà** : `controlleurs/nav.php:106-119` génère une liste HTML de 150 liens en bas de `/nav`, initialement pour le SEO et le no-JS. **La rendre explicite, annoncée et correctement structurée est un chantier faible coût / fort impact** — bien plus rentable qu'une navigation clavier dans le canvas, et conforme à l'esprit du RGAA (fournir une alternative équivalente).

Actions : annoncer la carte (`role="application"` + `aria-label`, ou `aria-hidden` assumé avec renvoi vers l'alternative), rendre la liste de points visible et navigable, l'articuler avec les filtres.

### 6b — Panneaux de carte focusables mais invisibles (1-2 j, risque faible)

**B2** : 17 des 45 premières tabulations d'une fiche point aboutissent sur des éléments invisibles (sélecteur de fond de carte, panneau de téléchargement). Même cause que 0.2, mais le CSS est dans `myol/dist/myol.css`, généré depuis `myol/src/`.

Deux options : correction en amont dans `myol` (propre, dépend d'un tiers) ou **surcharge CSS locale** (rapide, à documenter comme temporaire). Recommandation : surcharge locale immédiate, remontée en amont en parallèle.

### 6c — Unification de la stack cartographique (5-10 j, risque élevé)

Aujourd'hui : Leaflet 1.9.4 sur l'accueil, OpenLayers 10.7 via myol sur `/nav`, la fiche point et le formulaire de modification. Deux catalogues de fonds de carte dupliqués (`vues/_cartes.js` 20 Ko / `vues/_cartes_leaflet.js` 7,6 Ko) : **toute évolution cartographique doit être faite deux fois.**

**Recommandation : migrer la page d'accueil de Leaflet vers myol.** Arguments :
- myol couvre déjà 3 des 4 pages cartographiques ;
- il est bundlé : l'accueil passerait de **23 à ~7 requêtes d'assets** ;
- il est maintenu par un contributeur actif du projet ;
- cela permet de supprimer `leaflet/` (3,4 Mo, 7 plugins en copie manuelle, versions suivies dans des `LIS_MOI.txt`, une désynchronisation déjà constatée) ;
- une seule implémentation de fonds de carte à maintenir ensuite.

**Contre-argument à considérer honnêtement :** myol est en version `1.1.2.dev` (non publiée), avec une dépendance `"@myol/geocoder": "latest"` qui rend le build non reproductible, et 7,3 Mo de `dist/` committé. Concentrer 100 % de la cartographie dessus **augmente la dépendance à un contributeur unique**. C'est un choix de gouvernance autant que technique — voir question 3.

**Prérequis avant de lancer 5c :** figer la dépendance `"latest"`, et clarifier le statut de maintenance de myol.

### 6d — Découplage JS/PHP (non chiffré, à ne lancer que si justifié)

7 fichiers `.js` contiennent du PHP, ce qui interdit tout bundling et toute CSP sans `unsafe-inline`. Le remède est incrémental (passer les données par `<script type="application/json">` ou attributs `data-`, un fichier à la fois).

**À ne lancer que si un besoin concret le justifie** (exigence CSP, ou minification devenue nécessaire après mesure). Ce n'est pas un objectif en soi, et le Lot 0.1 (compression) capture déjà l'essentiel du gain de poids que la minification apporterait.

---

## Retouche progressive vs réécriture

**Retouche progressive — Lots 0 à 5, soit ~29 à 38 j.** Quatre propriétés du code le permettent :
1. Bandeau, pied de page et squelette HTML **mutualisés dans 3 fichiers** : une correction porte sur tout le site.
2. **Variables CSS déjà en place** (10 variables, 74 usages, mode sombre fonctionnel) : le theming et le contraste se pilotent depuis un point unique.
3. **JS vanilla ES6 sans framework** : aucune dette de migration.
4. **Un environnement reproductible déjà livré** (`make up`, `make seed`, PostGIS + PHP 8.4) : c'est ce qui rend le Lot 4 chiffrable à 6-9 j au lieu du chantier qu'il serait sur un dépôt sans stack locale.

**Réécriture — Lot 6c uniquement (5-10 j).** L'unification cartographique est le seul chantier qui ne peut pas se faire par retouche : il faut réécrire l'initialisation de la carte d'accueil et retirer une stack entière.

**Ce que je recommande explicitement de ne pas faire :**
- **Pas de framework front.** 46 Ko de JS lisible, contributeurs bénévoles : un framework élèverait la barrière à l'entrée sans résoudre un seul constat de l'audit.
- **Pas de moteur de template.** L'absence d'auto-escaping est un vrai défaut, mais la migration exigerait un audit champ par champ (certaines valeurs sont du HTML volontairement non échappé). Mauvais rapport coût/bénéfice face aux priorités d'accessibilité.
- **Pas de refonte du forum phpBB.** Périmètre autonome, non audité (cf. AUDIT §0.2).
- **Pas de refactoring des contrôleurs au nom de la testabilité.** C'est la dérive naturelle du Lot 4 : les 80 fichiers PHP mélangent requête, logique et vue, donc « rendre le code testable » y signifie tout réécrire. Le Lot 4 fait l'inverse — il teste le code **tel qu'il est**, par HTTP et par golden file, et ne descend en unitaire que sur les fonctions déjà pures. Une couverture globale ambitieuse coûterait plus cher que les six autres lots réunis.
- **Pas de suppression de la charte saisonnière.** C'est un choix identitaire assumé ; seuls 2 codes couleur ont besoin d'être assombris.

---

## Vérification et non-régression

**Il n'existe aucune CI aujourd'hui** (`.github/` ne contient que `FUNDING.yml`) : ni lint, ni test, ni build vérifié. C'était le principal facteur de risque de cette roadmap — **c'est désormais l'objet du [Lot 4](#lot-4--tests-et-intégration-continue)**, à qui cette section délègue son outillage.

**Deux points de séquencement, plus importants que le détail du Lot 4 :**
- **`4a` (lint, 0,5 j) se livre avec le Lot 0.** C'est le seul élément de CI dont le coût est négligeable devant le bénéfice immédiat.
- **`4b`/`4c` (tests de contrat API + smoke tests, 3-4 j) doivent précéder le Lot 3.** Le Lot 3 est explicitement « le lot le plus exposé aux régressions visuelles » et le Lot 3bis promet une URL d'API strictement identique : ni l'un ni l'autre ne peut être validé honnêtement à la main.

La recommandation initiale de cette section — versionner `axe-run.mjs` et `kbd-probe.mjs` dans `outils/audit/` — est conservée et **étendue par le sous-lot 4e**, qui les fait tourner en CI avec un budget de nœuds à cliquet. En attendant le Lot 4, ces deux scripts restent utilisables seuls, en local, comme harnais de mesure avant/après.

Indicateurs de référence à suivre, mesurés dans les conditions de AUDIT §0 (`debug=false`) :

| Indicateur | Aujourd'hui | Après Lot 0 | Après Lots 1-2 | Cible |
|---|---|---|---|---|
| Nœuds axe (6 pages, desktop) | **419** | ~290 | ~20 | < 10 |
| dont `color-contrast` | 124 | ~0 | 0 | 0 |
| dont `region` + `landmark` | 267 | 267 | ~0 | 0 |
| Violations critiques axe | 16 | 14 | 0 | 0 |
| Lighthouse a11y `/nav` | 61 | 68 | ≥ 95 | ≥ 95 |
| Lighthouse a11y fiche | 77 | 84 | ≥ 95 | ≥ 95 |
| Poids assets fiche (hors tuiles) | 944 Ko | **~370 Ko** | ~370 Ko | < 400 Ko |
| Tabulations sur élément invisible (fiche) | 17/45 | ~14/45 | ~14/45 | 0 |
| Cibles tactiles < 24 px (`/nav`) | 72 | 72 | 72 | 0 *(Lot 3)* |
| **Routes couvertes par un smoke test** | **0 %** | 0 % | 0 % | **100 %** *(Lot 4c)* |
| **Routes d'API sous test de contrat** | **0/6** | 0/6 | 0/6 | **6/6** *(Lot 4b)* |
| **Couverture `includes/mise_en_forme_texte.php`** | **0 %** | 0 % | 0 % | **≥ 60 %** *(Lot 4d)* |
| **Couverture `includes/` + `modeles/`** | **0 %** | 0 % | 0 % | **≥ 35 %** *(Lot 4d)* |
| **Workflows CI** | **0** | 1 *(lint, 4a)* | 1 | **2** *(Lot 4)* |

*Les projections « après Lot 0 / Lots 1-2 » sont des estimations dérivées du nombre de nœuds imputables à chaque cause identifiée, pas des mesures.*

*Les quatre indicateurs de test partent tous de zéro et ne bougent qu'au Lot 4 — sauf le lint, livré avec le Lot 0. Les cibles de couverture sont volontairement basses et bornées à la couche testable : voir l'encadré du Lot 4d sur les raisons de ne pas afficher un pourcentage global.*

Pour chaque lot : mesure avant / après avec le même harnais, plus une **vérification visuelle sur 6 pages × 2 viewports × 2 thèmes** (12 à 24 captures) — le Lot 3 en particulier ne peut pas être validé autrement.

---

## Questions de la première itération — réponses et effets

| Question | Réponse | Effet sur le plan |
|---|---|---|
| 1. Profil d'usage ? | Mobile = en montagne, desktop = en préparation. **Les deux, non recouvrants.** | 0.1 verrouillé en priorité absolue ; **Lot 3 remonté** devant les formulaires de contribution |
| 2. Obligation légale RGAA ? | **Non.** RGAA comme levier d'ergonomie et de lisibilité | Audit lecteur d'écran, déclaration et forum phpBB **retirés du périmètre** ; on s'arrête au bénéfice utilisateur |
| 3. Maintenance de `myol` ? | **Inconnue**, à instruire sur le forum | **Lot 6c bloqué en attente** |
| 4. Formulaires modifiables ? | **Oui, tout peut être amélioré** | Lot 5 à **périmètre complet** (fusion d'écrans incluse) |
| 5. Volume en production ? | **Mesuré** : 631 / 636 cases à cocher ; icône « sommet » **non cassée** en prod | **Lot 3bis créé** ; Lot 0.5 corrigé et déclassé |

### Détail de la réponse à la question 5

La question portait sur deux incertitudes que l'environnement local ne pouvait pas lever. Les deux sont désormais tranchées, par simple requête sur le site public — **aucun accès à la base de production n'était nécessaire.**

**(a) Densité des formulaires — confirmé, et pire que mon estimation.**

```bash
curl -s https://www.refuges.info/formulaire_rss_et_nouvelles | grep -oE 'type=.checkbox' | wc -l
```

→ **631** (et 636 sur `/formulaire_exportations`). J'avais estimé ~480. C'est le pire point d'ergonomie mobile du site → Lot 3bis.

**(b) Icône « sommet » — infirmé.** Pour le vérifier soi-même, il suffit d'ouvrir <https://www.refuges.info/nav> et de regarder la légende « Points » : les 7 types y ont tous une icône, et « sommet » n'apparaît pas du tout. En ligne de commande :

```bash
curl -s https://www.refuges.info/nav | tr '\n' ' ' | grep -oE '<img [^>]*icone_[0-9]+[^>]*>'
```

Le bug n'existait qu'en local, parce que le dump SQL public de `docker/init/` est désynchronisé de la production : il contient un type `sommet` que la prod n'affiche pas, et **pas** le type `grotte` qu'elle affiche.

---

## Ce qui reste ouvert

Trois points seulement, dont un seul bloque une décision.

### 1. 🔴 Bloquant — statut de maintenance de `myol`

Seule question qui empêche encore de décider. `myol` est en `1.1.2.dev` (non publiée), avec `"@myol/geocoder": "latest"` (build non reproductible) et 7,3 Mo de `dist/` committé.

- **Si myol reste activement maintenu** → migrer l'accueil de Leaflet vers myol (Lot 6c) : une seule stack, l'accueil passe de 23 à ~7 requêtes d'assets, `leaflet/` et ses 3,4 Mo disparaissent.
- **Si le contributeur qui le porte n'est plus disponible** → la recommandation s'inverse : consolider sur Leaflet, plus largement maintenu, en acceptant l'absence de bundling. **Lot 6c radicalement différent.**

Ce que la recherche sur le forum devrait établir : date du dernier commit sur `Dominique92/myol`, existence d'un plan de publication d'une version stable, et **qui arbitre** une décision technique de cette portée dans le projet. Les Lots 0 à 5 peuvent tous être menés sans cette réponse — il n'y a donc aucune urgence à trancher.

### 2. 🟡 À mesurer — la répartition mobile/desktop réelle

La réponse qualitative suffit à réordonner le plan, et c'est fait. Mais deux arbitrages internes au Lot 3 gagneraient à être chiffrés : faut-il optimiser d'abord la carte (usage terrain) ou les formulaires (usage préparation) ? Et surtout : **la PWA `gps/` est-elle réellement utilisée ?** Si oui, elle est le bon véhicule pour un mode hors-ligne et mérite un lot à elle ; si non, c'est du code mort à documenter comme tel. Les logs Apache ou une instance de statistiques répondraient en quelques minutes.

### 3. 🟢 À valider — la charte saisonnière

Le correctif de contraste (0.3) modifie deux couleurs identitaires, sur une charte assumée comme volontairement gratuite (`vues/style.css.php:9-12` : *« ouais je sais, c'est franchement de la frime »*). Techniquement sans risque, mais **c'est un choix esthétique qui appartient aux mainteneurs** : soit valider `#538005` / `#b44217`, soit leur laisser choisir d'autres teintes atteignant 4,5:1. À ne pas livrer sans accord.
