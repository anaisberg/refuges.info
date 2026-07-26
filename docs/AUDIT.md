# Audit refuges.info — accessibilité, UX, mobile

**Date :** 26 juillet 2026
**Périmètre :** code applicatif du site (`controlleurs/`, `modeles/`, `vues/`, `routes/`, `includes/`, `myol/`, `leaflet/`).
Le forum phpBB est hors périmètre d'audit détaillé (voir §0.2).
**Aucun fichier de code, contenu ou donnée n'a été modifié** dans le cadre de cet audit.

---

## 0. Conditions de mesure et limites

### 0.1 Environnement

Tout a été mesuré sur la stack Docker locale du dépôt (`make up` + `make seed`), et non en production :

| Élément | Valeur locale | Production (annoncée) |
|---|---|---|
| PHP | 8.4 (`docker/Dockerfile`) | 8.2 / 7.3 / 7.4 (`INSTALL.md`) |
| PostgreSQL / PostGIS | 15 / 3.4 | 15 / 3 |
| Serveur | Apache 2.4, `mod_deflate` + `mod_expires` actifs | Apache 2.4 |
| Points en base | 18 | plusieurs dizaines de milliers |
| Polygones en base | **7** (5 massifs + 2 zones) | **> 630** (mesuré, §3.3) |
| Types de points affichés | 8 (dont `sommet`, absent en prod) | **7** (dont `grotte`, absent du dump public) |
| Clés API cartes | **vides** | IGN, Mapbox, Thunderforest, Bing |

> **Mise à jour du 26/07/2026 :** après rédaction initiale, six constats ont été **vérifiés directement sur `https://www.refuges.info`** (requêtes en lecture seule sur le site et l'API publics). Résultat : **cinq confirmés en production, un infirmé.** Les sections concernées portent la mention « ✅ confirmé en production » ou « ❌ infirmé ». Cela ne remplace pas un audit en conditions réelles, mais lève les principaux doutes signalés en §0.2.

### 0.2 Ce que je n'ai **pas** pu mesurer, et pourquoi

Ces limites sont importantes : plusieurs conclusions ci-dessous sont des **extrapolations explicitement signalées** plutôt que des mesures.

1. **Le forum et l'authentification.** `docker/config_privee.docker.php` force `$config_wri['forum_desactive']=true` (les dumps SQL publics fournissent un schéma phpBB 3.0/3.1 incompatible avec le code phpBB 3.3.17 du dépôt). Conséquence : **le parcours de connexion, l'espace compte et les pages du forum n'ont pas été audités.** Or toute l'authentification est déléguée à phpBB (`modeles/identification.php`). C'est un **angle mort de cet audit**, et il est significatif.
2. **Le poids réel des cartes est sous-estimé.** Les clés API étant vides, les tuiles IGN/Mapbox/Thunderforest ne se chargent pas. Les chiffres de poids page ci-dessous **excluent donc les tuiles raster**, qui constituent en pratique l'essentiel du trafic sur la carte.
3. ~~**Les volumes de formulaire sont sous-estimés.**~~ **Levé** : mesuré depuis la production, cf. §3.3. Avec 7 polygones locaux, les formulaires « une case par massif » apparaissaient quasi vides ; ils comptent en réalité **631 et 636 cases à cocher**.
4. **Je n'ai pas testé avec un lecteur d'écran réel** (NVDA/JAWS/VoiceOver). Les constats d'accessibilité reposent sur axe-core, Lighthouse, analyse statique et parcours clavier automatisé. Un audit RGAA formel exigerait une validation humaine avec technologies d'assistance — **hors périmètre retenu** (cf. §6).
5. **Je n'ai pas mesuré axe-core ni Lighthouse sur la production**, seulement en local. Les vérifications de production listées ci-dessus portent sur des constats ponctuels (en-têtes HTTP, CSS servi, HTML généré), pas sur une nouvelle campagne de mesure.

### 0.3 Le drapeau `debug` faussait les premières mesures

`docker/config_privee.docker.php` définit `$config_wri['debug']=true`. Or ce drapeau pilote le choix des bundles JS (`vues/_page.html:24-70`) :

```php
case 'myol':
    if(empty($config_wri['debug'])) {
      add_lib('myol-min.css', 'chemin_ol');
      add_lib('myol.js', 'chemin_ol');          // 744 Ko minifié
    } else {
      add_lib('myol.css', 'chemin_ol');
      add_lib('myol-debug.js', 'chemin_ol');    // 2,8 Mo non minifié
    }
```

En `debug=true`, `/nav` pèse **3 169 Ko** et le LCP mobile monte à **15,7 s**. Toutes les mesures de performance de ce document ont donc été **refaites avec `debug=false`** pour être représentatives de la production. Le fichier `config_privee.php` a été restauré à son état d'origine après mesure.

> À noter au passage : dans la branche `leaflet` du même `switch`, **les commentaires `// Libs en mode debug` et `// Libs en mode prod` sont inversés** par rapport à la condition (`vues/_page.html:42` et `:49`). La logique est correcte, les commentaires mentent. Source de confusion garantie pour le prochain contributeur.

### 0.4 Outils

- **axe-core 4.x** via `@axe-core/puppeteer`, tags `wcag2a, wcag2aa, wcag21a, wcag21aa, best-practice`, sur 6 pages × 2 viewports (1280×900 et 375×667).
- **Lighthouse 12** en profil mobile, `--throttling-method=simulate` (Slow 4G, CPU ×4).
- Sondes maison : parcours clavier réel (`Tab` × 220), calcul de contraste WCAG sur couleurs **calculées** (et non déclarées), mesure d'opacité effective par produit des ancêtres, dimensions de cibles tactiles.
- Analyse statique du HTML généré (`curl`) et du code source.

---

## 1. Cartographie technique

### 1.1 Stack

| Couche | Technologie | Version | Gestion |
|---|---|---|---|
| Langage serveur | PHP | 8.4 en local, 8.2 annoncé en prod | — |
| Base | PostgreSQL + PostGIS | 15 / 3.4 | — |
| Serveur | Apache 2.4 + `.htaccess` | — | — |
| Templating | **PHP brut + `include()`** | — | aucun moteur |
| Front JS | **Vanilla ES6+**, aucun framework | — | aucun build |
| Carto n°1 | Leaflet + 7 plugins | Leaflet **1.9.4** | copie manuelle |
| Carto n°2 | OpenLayers via `myol` | `ol` **10.7.0**, myol **1.1.2.dev** | vendoré, dist committé |
| Forum | phpBB | **3.3.17** | Composer, `vendor/` committé (15 Mo) |
| i18n | **aucune** | — | français en dur |

**Aucun gestionnaire de dépendances à la racine** : pas de `composer.json`, pas de `package.json`, pas d'autoloader. Tout passe par `require_once`/`include`. **Aucune CI** : `.github/` ne contient que `FUNDING.yml` — donc aucun garde-fou automatisé (ni lint, ni test, ni build vérifié).

### 1.2 Arborescence et points d'entrée

Point d'entrée unique. `.htaccess:38-40` redirige tout ce qui n'est pas un fichier réel vers `index.php` :

```
index.php (22 l.)
  └── includes/config.php + includes/gestion_erreur.php
  └── routes/generales.routes.php
        ├── switch sur le 1er segment d'URL — whitelist de 16 routes
        ├── api/*      → routes/api.routes.php   (court-circuit, pas de bandeau)
        ├── gestion/*  → routes/gestion.routes.php (exige modérateur)
        ├── controlleurs/bandeau.php     ← inclus pour TOUTES les pages
        ├── controlleurs/<type>.php
        └── vues/_page.html
              ├── vues/bandeau.html
              ├── vues/<type>.html
              └── vues/_pied.html
```

Servis **hors MVC** (fichiers réels) : `forum/`, `gps/` (PWA GPS hors-ligne), `leaflet/`, `myol/`, `images/`, `ressources/`.

Le pipeline d'assets tient en deux fonctions, `includes/gestion_erreur.php:10-36`. Cache-busting par `filemtime`, ce qui est correct et déjà généralisé :

```php
return $config_wri['url_'.$chemin].$nom_fichier_vue.'?'
  .filemtime($config_wri[$chemin].$nom_fichier_vue);
```

### 1.3 Ce qui contraint une refonte

**Atouts réels, à ne pas casser :**

1. **Variables CSS déjà en place** — 10 variables sur `:root` (`vues/style.css.php:62-73`), 74 usages, mode sombre fonctionnel via `prefers-color-scheme`. Commentaire d'intention à `:61` : `/* DOM 09/2025 on passe des constantes PHP aux constantes CSS */`. **C'est le socle de theming dont un chantier accessibilité a besoin, et il existe déjà.**
2. **JS vanilla ES6 sans jQuery ni framework** côté site (jQuery 3.7.1 n'existe que dans phpBB). Migration sans dette de librairie.
3. **Volume applicatif faible** : ~43 Ko de CSS + ~46 Ko de JS non minifiés. C'est petit — donc traitable.
4. **Séparation contrôleur/vue lisible**, un seul point d'entrée, routes en whitelist explicite.
5. `style_formulaire.css` montre déjà des motifs modernes et corrects : `:focus-visible` avec `outline: 2px solid` + `outline-offset` (lignes 496-498, 537-539, 608-610). Quelqu'un a commencé le travail.

**Points durs, par ordre de gravité structurelle :**

1. **Double stack cartographique.** Leaflet 1.9.4 (page d'accueil) *et* OpenLayers 10.7 via myol (`/nav`, fiche point, formulaire de modification). Deux catalogues de fonds de carte maintenus en parallèle : `vues/_cartes.js` (20 Ko, OpenLayers) et `vues/_cartes_leaflet.js` (7,6 Ko, Leaflet) définissent les mêmes couches en double. **Toute évolution cartographique doit être faite deux fois.** C'est le choix à trancher avant tout le reste.
2. **JS mêlé au PHP — bloque tout bundling.** `vues/_pied.html:24-37` ouvre un `<script>` et y `include()` du PHP. Conséquence : **7 des 16 fichiers `.js` applicatifs contiennent du PHP** (`bandeau.js`, `index.js`, `nav.js`, `point.js`, `edit.js`, `point_formulaire_recherche.js`, `point_formulaire_modification.js`) et ne sont donc pas des fichiers JS servables statiquement.

   ```js
   // vues/index.js:9-14
   const map = initLeafletMap(
     'carte-accueil',
     'https://<?=$_SERVER["SERVER_NAME"]?>',
     <?=$vue->version_features?>,
     <?=json_encode($config_wri['mapKeys'])?>
   );
   ```
3. **`myol` vendoré : 7,3 Mo de `dist/` committé** (`myol.js` 744 Ko, `myol-debug.js` 2,8 Mo, `myol.js.map` 4,0 Mo), version `1.1.2.dev` (non publiée), avec une dépendance `"@myol/geocoder": "latest"` → **build non reproductible**.
4. **`leaflet/` sans manifeste** : 7 plugins en copie manuelle, versions documentées dans des fichiers `LIS_MOI.txt`. Une désynchronisation est déjà constatée (leaflet-gps : 1.7.7 embarqué, 1.7.8 annoncé dans le `.txt`).
5. **Pas d'échappement automatique.** Tout passe par `<?=$var?>` brut, y compris des superglobales (`vues/bandeau.html:38` `redirect=<?=$_SERVER['REQUEST_URI']?>`). Certaines valeurs sont du HTML volontairement non échappé. Introduire un moteur avec auto-escaping demanderait un audit champ par champ.
6. **20 handlers HTML inline** (`onclick` ×17, `onchange` ×2, `onmouseover` ×1) + **5 blocs `<script>` inline** → une CSP sans `unsafe-inline` est impossible en l'état.
7. **CSS : 0 `rem`, 137 `px`, 8 breakpoints incohérents, 42 `!important`.** Détail en §2.6 et §3.4.
8. **Contraintes d'environnement héritées** : `short_open_tag = On` requis (`vues/bandeau.html:69` utilise `<? }`), `output_buffering` requis pour le CSS généré en PHP, phpBB sur Symfony 3.4 tournant sur PHP 8.4, `strftime` déprécié encore utilisé (`modeles/commentaire.php:210`) — supprimé en PHP 9.
9. **Le diagnostic est déjà posé dans le code lui-même**, `vues/style.css.php:20-24` :
   > *« ce style a évolué au fil des années et je suis sûr qu'il y a plusieurs classes qui ne servent nulle part, pas mal de redondance… Un support parfois médiocre des petites écrans, des adressages par id, par class. Bref, ça mériterait vraiment un coup de neuf. »*

### 1.4 Parcours utilisateurs identifiés

| Parcours | URL | Contrôleur | Écrans/clics |
|---|---|---|---|
| Recherche rapide | `/point_recherche?nom=` (form du bandeau) | `point_recherche.php` | 2 écrans |
| Recherche avancée | `/point_formulaire_recherche` → `/point_recherche` | idem | **4 écrans / 3-4 clics** |
| Recherche carto | `/nav`, `/nav/<id_polygone>` | `nav.php` | 1 écran, filtres client |
| Fiche refuge | `/point/<id>/<type>/<nom>/` | `point.php` | 1 écran, **22 sections** |
| Ajout de point | `/point_ajout` → `/point_formulaire_modification` → `/point_modification` | 3 contrôleurs | **3 écrans** |
| Modification de point | `/point_formulaire_modification?id_point=` → `/point_modification` | 2 contrôleurs | 2 écrans |
| Ajout commentaire/photo | `/point_ajout_commentaire/<id>` | `point_ajout_commentaire.php` | 1 écran |
| Avis sur commentaire | `/avis_internaute_commentaire/<id>/` | idem | 1 écran |
| Édition de polygone | `/edit`, `/edit/<id>` | `edit.php` (inclut `nav.php`) | modérateurs |
| Wiki | `/wiki/<page>` | `wiki.php` | lecture publique, écriture modérateurs |
| Nouvelles | `/nouvelles` | `nouvelles.php` | 1 écran |
| Export GPS | `/formulaire_exportations` | idem | **3 clics pour un fichier** |
| Compte / connexion | `/forum/ucp.php?mode=login` | **phpBB** | non audité (§0.2) |
| Gestion | `/gestion/{moderation,historique_modifications,…}` | `controlleurs/gestion/` | modérateurs |
| API | `/api/{bbox,point,massif,commentaires,contributions,polygones}` | `controlleurs/api/` | — |

**Contribution sans compte : oui**, sur trois formulaires (ajout de point, ajout de commentaire, avis sur commentaire), protégés par un **captcha statique unique** — `includes/config.php:112-113` : question « Entrez la lettre **g** », réponse « g ». La *modification* d'un point ou d'un commentaire exige en revanche d'être connecté.

---

## 2. Audit accessibilité (RGAA 4.1 / WCAG 2.1 AA)

### 2.1 Résultats bruts

**axe-core** — 6 pages, viewport desktop 1280×900. Nombre de nœuds en défaut :

| Règle | Impact | WCAG | index | /nav | fiche | recherche | ajout | commentaire | **Total** |
|---|---|---|---|---|---|---|---|---|---|
| `region` (contenu hors landmark) | moderate | — | 33 | 37 | 71 | 52 | 26 | 42 | **261** |
| `color-contrast` | serious | 1.4.3 | 58 | 8 | 23 | 8 | 9 | 18 | **124** |
| `button-name` | **critical** | 4.1.2 | — | 6 | 3 | — | — | — | **9** |
| `landmark-one-main` | moderate | — | 1 | 1 | 1 | 1 | 1 | 1 | **6** |
| `label` | **critical** | 4.1.2 | — | 1 | — | 1 | — | 3 | **5** |
| `page-has-heading-one` | moderate | — | 1 | 1 | — | 1 | 1 | 1 | **5** |
| `list` | serious | 1.3.1 | — | 2 | — | — | — | — | **2** |
| `scrollable-region-focusable` | serious | 2.1.1 | — | 1 | 1 | — | — | — | **2** |
| `image-alt` | **critical** | 1.1.1 | — | 1 | — | — | — | — | **1** |
| `select-name` | **critical** | 4.1.2 | — | — | 1 | — | — | — | **1** |
| `document-title` | serious | 2.4.2 | — | — | — | — | — | 1 | **1** |
| `heading-order` | moderate | — | — | — | 1 | — | — | — | **1** |
| `empty-heading` | minor | — | — | 1 | — | — | — | — | **1** |
| **Total** | | | **93** | **59** | **101** | **63** | **37** | **66** | **419** |

13 règles en défaut, **419 nœuds**, dont **16 nœuds d'impact `critical`**. En viewport mobile 375 px, les mêmes 13 règles se déclenchent (les compteurs de nœuds sont plus bas car moins de contenu est rendu) : **aucune non-conformité n'est spécifique à un viewport**.

**Lighthouse mobile** (`debug=false`, Slow 4G simulé) :

| Page | Perf | Access. | Best pract. | SEO | FCP | LCP | TBT | Poids* |
|---|---|---|---|---|---|---|---|---|
| `/` | 83 | 91 | 92 | 92 | 3,1 s | 3,7 s | 0 ms | 518 Ko |
| `/nav` | 80 | **61** | 81 | 75 | 1,2 s | 5,4 s | 0 ms | 1 136 Ko |
| `/point/37` | 80 | **77** | 100 | 83 | 1,2 s | 5,4 s | 10 ms | 944 Ko |
| `/point_formulaire_recherche` | **100** | 86 | 92 | 92 | 1,2 s | 1,5 s | 0 ms | 55 Ko |

\* hors tuiles raster (clés API vides, cf. §0.2) — le poids réel de `/nav` et de la fiche est **supérieur**.

Résultats complets conservés hors dépôt (`axe-results.json`, `kbd-results.json`, `lh/*.json`, `lh-prod/*.json`).

### 2.2 ✅ Bloquant, confirmé en production — 18 contrôles de navigation focusables mais invisibles

**Le constat le plus grave de cet audit.** Les menus déroulants du bandeau sont masqués par `opacity: 0` + `z-index: -2000` et **non** par `display: none` — `vues/bandeau.css:33-48` :

```css
.menu-bouton:not(.menu-liste)>p,
.menu-bouton:not(.menu-liste)>ul,
.menu-bouton:not(.menu-liste)>form {
  position: absolute;
  opacity: 0;
  z-index: -2000;
}
```

Or un élément en `opacity: 0` **reste dans l'ordre de tabulation**. Et l'ouverture ne se déclenche que sur `mouseover` / `click` (`vues/bandeau.js:4-6`), **sans aucune règle `:focus-within`** — vérifié : aucune feuille de style du site ne contient `focus-within` appliqué au bandeau.

Parcours clavier réel mesuré sur `/point/37` (220 tabulations depuis le début du document) :

```
tab #  1  op=0  "passer en HTTPS"        (invisible)
tab #  2  op=1  "Refuges.info"
tab #  3  op=1  "🔔Nouvelles"
tab #  4  op=1  "📌Ajouter"
tab #  5  op=1  "🌍Cartes"
tab #  6  op=1  "💬Forum"
tab #  7  op=0  ""                       (invisible)
tab #  8  op=0  "🔍"                     (invisible)
tab #  9  op=0  "Recherche avancée"      (invisible)
tab # 10  op=0  "Accueil"                (invisible)
tab # 11  op=0  "Qui sommes nous ?"      (invisible)
tab # 12  op=0  "Donation € ❤️"          (invisible)
…
tab # 24  op=0  "Liens"                  (invisible)
```

`op` = opacité effective (produit des opacités de tous les ancêtres). **24 contrôles dans les menus du bandeau, dont 24 mesurés à opacité 0 hors les 6 liens de premier niveau.** Un utilisateur au clavier traverse donc **18 éléments consécutifs totalement invisibles** avant d'atteindre le contenu, sans jamais pouvoir ouvrir un menu.

**✅ Vérifié en production** — `https://www.refuges.info/vues/bandeau.css` sert bien `opacity: 0` + `z-index: -2000` (lignes 46-48) et **ne contient aucune règle `focus-within`**. Le blocage est actif sur le site en ligne.

- **RGAA 10.7** (focus visible), **RGAA 12.8** (ordre de tabulation cohérent), **RGAA 7.3** (contrôlable au clavier)
- **Gravité : bloquant**
- **Coût : très faible.** Ajouter `:focus-within` à la liste de sélecteurs de `vues/bandeau.css:62-68` rend les menus ouvrables au clavier. C'est une modification de 3 lignes de CSS.

### 2.3 Bloquant — même problème sur les panneaux de carte myol

Sur `/point/37`, **17 des 45 premières tabulations** aboutissent sur un élément invisible : le sélecteur de fond de carte (14 boutons radio « OpenHikingMap », « IGN TOP25 », « SwissTopo »…) et le panneau de téléchargement.

```
   5. op=1    BUTTON  ""
   6. op=0    INPUT   "0"                    <<< INVISIBLE
   9. op=0    DIV     "Cliquer sur un format…" <<< INVISIBLE
  15. op=0    INPUT   "OpenHikingMap"        <<< INVISIBLE
  …
  28. op=0    INPUT   "Photo Maxar"          <<< INVISIBLE
```

Corroboré par axe : `scrollable-region-focusable` sur `.myol-button-download > div`.

- **RGAA 10.7**, **RGAA 12.8**, **RGAA 7.3**
- **Gravité : bloquant**
- **Coût : moyen** — le CSS est dans `myol/dist/myol.css`, donc généré depuis `myol/src/`. Correction en amont dans myol, ou surcharge CSS locale en attendant.

### 2.4 Bloquant — la carte est inutilisable au clavier

Les conteneurs de carte n'ont **ni `tabindex`, ni `role`, ni `aria-label`** :

| Page | Élément | tabindex | role | aria-label |
|---|---|---|---|---|
| `/nav` | `div#carte-nav` (960×737) | `null` | `null` | `null` |
| `/nav` | `div.ol-viewport` | `null` | `null` | `null` |
| `/nav` | `canvas` (960×737) | `null` | `null` | `null` |
| `/point/37` | `div#carte-point` (640×640) | `null` | `null` | `null` |

La carte étant rendue dans un `<canvas>`, son contenu est **intégralement invisible aux technologies d'assistance** et aucun déplacement/zoom n'est possible au clavier.

- **RGAA 7.1** (script compatible avec les technologies d'assistance), **RGAA 7.3** (contrôlable au clavier), **WCAG 2.1.1**
- **Gravité : bloquant**
- **Coût : élevé** pour une vraie alternative clavier sur la carte. **Mais** : une alternative textuelle équivalente existe déjà partiellement — `controlleurs/nav.php:106-119` génère une liste HTML de 150 liens en bas de `/nav` (initialement pour le SEO / le no-JS). **La rendre explicite et annoncée est un chantier faible coût / fort impact**, bien plus rentable qu'une navigation clavier dans le canvas.

### 2.5 ✅ Majeur, confirmé en production — contraste insuffisant sur 2 des 3 palettes saisonnières

`vues/style.css.php:31-57` change la palette selon la saison calendaire. J'ai calculé les ratios WCAG sur les couleurs réellement produites :

| Palette | Période | lien / fond (clair) | texte / fond (clair) | lien / fond (sombre) |
|---|---|---|---|---|
| Hiver | 21/12 → 21/03 | **5,58** ✅ | 18,76 ✅ | 7,26 ✅ |
| Printemps-Été | 21/03 → 22/09 | **3,83 ❌** | 20,10 ✅ | 11,86 ✅ |
| Automne | 22/09 → 21/12 | **3,27 ❌** | 17,24 ✅ | 5,28 ✅ |

Le seuil AA pour du texte normal est 4,5:1. **Le site est donc non conforme sur le contraste des liens 6 mois par an** (printemps-été et automne), et conforme en hiver.

Mesure croisée : sur l'accueil, une **seule** paire de couleurs explique les 124 nœuds `color-contrast` relevés par axe — `rgb(95, 140, 17)` sur `rgb(245, 253, 232)`, ratio **3,83**, **81 occurrences**. Autrement dit : *un* changement de variable corrige l'essentiel du problème.

Corrections calculées (assombrissement minimal atteignant 4,5:1, teinte préservée) :

| Palette | Actuel | Proposé | Ratio obtenu |
|---|---|---|---|
| Printemps-Été | `#5f8c11` | `#538005` | 4,51 |
| Automne | `#cf5d32` | `#b44217` | 4,62 |
| Hiver | `#006699` | *(inchangé)* | 5,58 |

**✅ Vérifié en production le 26/07/2026** — `https://www.refuges.info/vues/style.css.php` sert actuellement la palette été :

```css
--couleur_lien: #5f8c11;
--couleur_fond: #f5fde8;   /* ratio 3,83:1 — non conforme */
```

Le site est donc **non conforme au contraste des liens en ce moment même**. Le mode sombre servi en production (`#9ae31c` sur `#161210`, ratio 11,86) est en revanche conforme.

- **RGAA 3.2** (contraste texte)
- **Gravité : majeur**
- **Coût : très faible** — 2 valeurs hexadécimales dans `vues/style.css.php`. À valider visuellement avec les mainteneurs, la charte saisonnière étant un choix identitaire assumé.

> Effet de bord relevé : `vues/style.css.php` envoie `Cache-Control: max-age=86000` mais son URL n'est bustée que par le `filemtime` du `.php`, **qui ne change pas au passage d'une saison à l'autre**. Le changement de palette ne se propage donc pas aux navigateurs ayant le fichier en cache.

### 2.6 Majeur — aucune structuration sémantique

**Zéro landmark sur les 6 pages auditées** : `<main>` = 0, `<nav>` = 0, `<header>` = 0, `<footer>` = 0, `<aside>` = 0. D'où 261 nœuds `region` et 6 nœuds `landmark-one-main`.

**Titres `h1` absents sur 5 pages sur 6** — seule la fiche point en a un :

| Page | `<h1>` | Structure de titres relevée |
|---|---|---|
| `/` | **aucun** | `h4` → `h5` → `h4` → `h4` → `h4` |
| `/nav` | **aucun** | `h3` **vide** (`#titrepage`) |
| `/point/37` | ✅ | `h1` → **`h5`** → `h5` (saut de niveau) |
| `/point_formulaire_recherche` | **aucun** | `h3` → `h4` |
| `/point_ajout` | **aucun** | `h5` seul |
| `/point_ajout_commentaire/37` | **aucun** | `h4` seul |

**Aucun lien d'évitement** vers le contenu principal — ce qui, combiné à §2.2, signifie qu'un utilisateur au clavier doit traverser ~24 contrôles de bandeau à chaque page.

- **RGAA 9.1** (titres), **RGAA 12.6** (zones de regroupement), **RGAA 12.7** (lien d'évitement)
- **Gravité : majeur**
- **Coût : faible à moyen.** Les landmarks se posent presque entièrement dans 3 fichiers mutualisés (`vues/_page.html`, `vues/bandeau.html`, `vues/_pied.html`) — donc **une seule intervention couvre tout le site**. Les `h1` demandent en revanche une passe page par page (~10 vues).

### 2.7 Bloquant — page sans titre, et titres/labels manquants

**`<title>` vide sur `/point_ajout_commentaire` — ✅ confirmé en production.** Cause : `controlleurs/point_ajout_commentaire.php` n'affecte `$vue->titre` **que dans la branche d'erreur** (ligne 135). Sur le chemin nominal, aucun titre.

```
# local
/point_ajout_commentaire/37     <!doctype html>~<html lang="fr">~  <head>~    <title></title

# production, sur un point réel existant
$ curl -s https://www.refuges.info/point_ajout_commentaire/7885 | grep -o "<title>[^<]*</title>"
<title></title>
```

Détail révélateur du mécanisme : sur un point **inexistant**, la branche d'erreur s'exécute et le titre est correctement rempli (`<title>Impossible d'ajouter un commentaire car : Le numéro de point demandé 1 est introuvable…</title>`). **Le titre n'est donc présent que quand la page échoue.**

- **RGAA 8.5** (titre de page) — **bloquant**, **coût : 1 ligne.**

**Champs de formulaire sans étiquette** (ni `<label for>`, ni `aria-label`) :

| Page | Champs sans étiquette | Exemples |
|---|---|---|
| `/point_formulaire_recherche` | **10** | 8 × `champs_null[]` (cases rouges), `places_matelas_minimum`, champ `nom` du bandeau |
| `/point_ajout_commentaire` | **4** | `texte` (textarea principal !), `auteur_commentaire`, `lettre_verification` |
| `/nav` | 3 | `gcd-input-query` (géocodeur), `input[type=file]` (import GPX/KML) |
| `/point/37` | 2 | `select#cadre-select` (format de coordonnées) |

Le champ `nom` du formulaire de recherche du bandeau est sans étiquette **sur toutes les pages** (il est dans le bandeau mutualisé).

- **RGAA 11.1** (étiquette de champ), **RGAA 11.2**
- **Gravité : bloquant** — le textarea principal du formulaire de commentaire n'a pas d'étiquette.
- **Coût : faible** (~20 `<label>` à poser).

Point positif à souligner : `/point_formulaire_recherche` compte **8 `<fieldset>` + 8 `<legend>`** et `/formulaire_exportations` 5 de chaque. Le regroupement de champs (RGAA 11.5) est donc **déjà correctement fait** sur les formulaires les plus complexes. En revanche `/point_ajout_commentaire` n'a **aucun** `fieldset` et seulement 3 `<label>` pour 9 champs.

### 2.8 Bloquant — 9 boutons sans nom accessible

| Page | Bouton | Fonction probable |
|---|---|---|
| `/nav` | `button#gcd-button-control.gcd-gl-btn` | géocodeur / recherche d'adresse |
| `/nav` | 5 × `<button>` sans id ni classe | contrôles de carte myol |
| `/point/37` | 3 × `<button>` sans id ni classe | contrôles de carte myol |

Plus `select#cadre-select` sans nom accessible sur la fiche.

- **RGAA 7.1** / **RGAA 11.1** (selon le contrôle), **WCAG 4.1.2**
- **Gravité : bloquant**
- **Coût : faible** pour le géocodeur (`aria-label` à poser côté `leaflet/geocoder/`) ; **moyen** pour les boutons myol (correction en amont dans `myol/src/` ou surcharge).

### 2.9 ❌ Infirmé en production — l'icône « sommet » n'est cassée qu'en local

**Ce constat, initialement classé majeur, ne se manifeste pas en production.** Vérification faite sur `https://www.refuges.info/nav` : la légende affiche **7 types, tous avec une icône valide**, et le type `sommet` n'y figure pas du tout.

```html
<img id="icone_7"  src="/images/icones/cabane.svg"                 alt="icone de cabane non gardée">
<img id="icone_10" src="/images/icones/cabane_red.svg"             alt="icone de refuge gardé">
<img id="icone_9"  src="/images/icones/cabane_green.svg"           alt="icone de gîte d'étape">
<img id="icone_29" src="/images/icones/grotte.svg"                 alt="icone de grotte">
<img id="icone_23" src="/images/icones/pointdeau.svg"              alt="icone de point d'eau">
<img id="icone_3"  src="/images/icones/triangle_a33.10.svg"        alt="icone de passage délicat">
<img id="icone_28" src="/images/icones/cabane_white_black_a63.svg" alt="icone de bâtiment en montagne">
```

**Cause de l'écart :** le dump SQL public (`docker/init/refuges-local.sql.gz`) est désynchronisé de la production. Il contient un type `sommet` (id 6) que la production n'affiche pas, et **ne contient pas** le type `grotte` (id 29) que la production affiche — alors que `grotte` est bien présent dans le mapping de `includes/config.php`. Le mapping de config est donc **cohérent avec la production**, pas avec le dump.

**Ce qui reste valable, en gravité mineure :**

1. **Absence de garde-fou.** `vues/nav.html:51` concatène directement une entrée de tableau sans vérifier son existence :
   ```html
   src="<?=$config_wri['url_chemin_icones'].$config_wri['correspondance_type_icone'][replace_url($type_affichable->nom_type)]?>.svg"
   ```
   Si un modérateur ajoute un type de point en base sans compléter `includes/config.php`, l'icône casse silencieusement (`src="/images/icones/.svg"`). C'est exactement ce qui se produit avec le dump public. **Durcissement souhaitable, pas urgence.**
2. **Alternatives textuelles redondantes — ✅ confirmé en production.** Les 7 icônes portent `alt="icone de <type>"` alors que le libellé du type est déjà en texte juste après l'image (`vues/nav.html:54`). Ce sont des icônes décoratives qui devraient porter `alt=""` : un lecteur d'écran annonce aujourd'hui « icone de cabane non gardée, cabane non gardée ». **RGAA 1.2, gravité mineure, coût très faible.**
3. **Le dump public mérite d'être resynchronisé**, sinon tout contributeur qui monte l'environnement local voit une légende cassée et peut « corriger » un bug qui n'existe pas.

<details>
<summary>Constat initial (mesuré en local, conservé pour traçabilité)</summary>

Le type de point **« sommet » (id 6) existe dans le dump local mais est absent de la table de correspondance des icônes** (`includes/config.php:49-58`).

`vues/nav.html:51` concatène donc une valeur inexistante dans un `src` :

```html
<img id="icone_6"
     src="<?=$config_wri['url_chemin_icones'].$config_wri['correspondance_type_icone'][replace_url($type_affichable->nom_type)]?>.svg"
     alt="icone de <?=$type_affichable->nom_type?>">
```

Sortie réelle observée :

```html
src="<br />
<b>Warning</b>:  Undefined array key "sommet" in <b>/var/www/html/vues/nav.html</b> on line <b>51</b><br />
```

> **Nuance importante :** la fuite du warning dans le HTML est un artefact de `display_errors=1` en local. En production (`display_errors` off) le `src` vaudrait `/images/icones/.svg` → **image cassée**, sans warning visible. Le bug fonctionnel est réel dans les deux cas ; seule sa manifestation diffère.

L'accueil compte par ailleurs 12 `alt=""` sur 14 images (bonne pratique respectée) ; c'est `/nav` et `/point_ajout` qui décrivent des icônes redondantes.

</details>

### 2.10 Mineur — divers

| Constat | Preuve | RGAA | Gravité | Coût |
|---|---|---|---|---|
| `<ul>` contenant du texte direct hors `<li>` | `/nav`, `<ul>Ctrl+click: nouvel onglet</ul>` | 9.3 | mineur | très faible |
| Titre `h3` vide | `/nav`, `<h3 id="titrepage">   </h3>` | 9.1 | mineur | très faible |
| Saut de niveau `h1` → `h5` | `/point/37`, « Discussions sur… » | 9.1 | mineur | très faible |
| Variable non initialisée → warning + `null` retourné | `modeles/utilisateur.php:57` : `return $utilisateurs;` sans `$utilisateurs=[]` préalable. Produit un warning **avant le `<!doctype>`** sur `/point_formulaire_recherche` quand la requête ne renvoie aucune ligne, ce qui invalide le document | 8.1 | mineur en prod (masqué), majeur en dev | 1 ligne |
| Second `<meta viewport>` avec `user-scalable=no` | `vues/_page.html:81-83`, conditionné à `isset($vue->css)`. **Vérifié : `$vue->css` n'est assigné nulle part dans le dépôt** → code mort. Piège latent, pas une non-conformité active | 10.4 | mineur | très faible (supprimer) |
| CDATA XHTML résiduel dans le JS inline | `vues/_pied.html:25` `//<![CDATA[` | — | cosmétique | très faible |

### 2.11 Ce qui va bien

À dire clairement, parce que cela change le plan : **le socle n'est pas si mauvais.**

- **`lang="fr"` présent** sur toutes les pages (RGAA 8.3 ✅), `<!doctype html>` correct (RGAA 8.1 ✅ sur 5 pages / 6).
- **`<meta name="viewport" content="width=device-width, initial-scale=1.0">` présent partout** (RGAA 13.x ✅).
- **Le focus est visible partout où il est visible.** Mesuré sur 60 tabulations × 3 pages : **0 élément focusé sans indicateur**. Le `outline: none` de `vues/style_formulaire.css:35` est **correctement remplacé** par `border-color` + `box-shadow` :
  ```css
  #form_point input[type=text]:focus, … {
    outline: none;
    border-color: var(--couleur_lien);
    box-shadow: 0 0 0 3px color-mix(in srgb, var(--couleur_lien) 22%, transparent);
  }
  ```
  Ce n'est **pas** une non-conformité RGAA 10.7 en soi. Réserve : un focus par `box-shadow` disparaît en mode contraste forcé (Windows High Contrast) ; ajouter `outline: 2px solid transparent` en complément serait plus robuste. Gravité mineure.
- **Aucun `tabindex` positif** sur les 6 pages (RGAA 12.8 ✅ sur ce point).
- **Un seul handler `onclick` dans le HTML généré** des 6 pages auditées (20 dans les sources, mais concentrés sur des formulaires peu visités).
- **`<fieldset>`/`<legend>` déjà en place** sur les 2 formulaires les plus complexes.
- **Mode sombre fonctionnel** via `prefers-color-scheme`, avec des contrastes **tous conformes** (5,28 à 11,86).

---

## 3. Audit UX et mobile

### 3.1 Responsive : la base est saine

**Aucun débordement horizontal** sur les 6 pages en 375 px (`scrollWidth == innerWidth == 375` partout). C'est un bon point, souvent absent des sites de cette génération.

18 media queries au total, mais **8 breakpoints différents et non harmonisés** : 640, 641, 699.9, 700, 800, 1000, 1200, 1300 px — dont un `699.9px` (`vues/bandeau.css:172`) et un exotique `@media screen and (min-aspect-ratio: 1/1) and (min-width: 365px) and (max-height: 360px)` (`vues/style_pages.css:507`). Il y a aussi un `@media (pointer:coarse)` (`:590`) et un `@media print` (`:596`) — donc une conscience du sujet.

| Fichier | Taille | `@media` | `!important` | `px` | `rem` | `var(--` |
|---|---|---|---|---|---|---|
| `style_formulaire.css` | 14,3 Ko | 3 | **30** | 42 | **0** | 38 |
| `style_pages.css` | 13,6 Ko | 11 | 6 | 92 | **0** | 19 |
| `style.css.php` | 5,5 Ko | 1 | 0 | 3 | **0** | 2 |
| `bandeau.css` | 3,6 Ko | 3 | 3 | 24 | **0** | 5 |
| `style_forum.css` | 2,4 Ko | 0 | 3 | 0 | **0** | 10 |

**Zéro `rem` dans tout le CSS applicatif.** Conséquence d'accessibilité : les tailles fixées en `px` ne suivent pas le réglage de taille de police du système/navigateur. À noter cependant que `em` est très utilisé (56 occurrences dans `style_formulaire.css`, 23 dans `style_pages.css`), ce qui atténue partiellement le problème sur les composants concernés.

### 3.2 Cibles tactiles trop petites

Mesuré en viewport 375×667, éléments interactifs dont une dimension < 24 px :

| Page | Cibles < 24 px |
|---|---|
| `/nav` | **72** |
| `/` | **68** |
| `/point/37` | **63** |
| `/point_formulaire_recherche` | 39 |
| `/point_ajout` | 31 |
| `/point_ajout_commentaire` | 28 |

Le cas dominant est structurel : **tous les liens de texte du bandeau font 19 px de haut** (« Accueil » 56×19, « Nouvelles » 77×19, « Ajouter un point » 131×19, « Cartes » 52×19). Et le bouton de recherche `🔍` fait **17×22 px**.

> **Précision de conformité :** la taille de cible minimale (2.5.8, 24×24 px) est un critère **WCAG 2.2 AA**, absent de WCAG 2.1 AA et donc du RGAA 4.1. Ce n'est **pas** une non-conformité RGAA à ce jour — mais c'est un vrai problème d'usage mobile, et un critère qui entrera dans les référentiels à venir. Je le classe en UX/mobile, pas en accessibilité réglementaire.

**Gravité : majeur (UX mobile).** **Coût : faible** — augmenter le `padding` vertical des liens de menu dans `vues/bandeau.css` traite l'essentiel du volume d'un coup.

### 3.3 Densité d'information et navigation

**La fiche refuge empile 22 sections** (`vues/point.html`, 311 lignes) sans hiérarchisation visuelle forte : bandeaux modérateur, H1, carte, localisation par catégorie de polygone, propriétaire, accès, remarques, informations complémentaires, points à proximité, réglementation, coordonnées en 4 systèmes + 4 formats d'export, métadonnées, forum, commentaires, avertissement, licence. Avec 3 niveaux de titres seulement (`h1`, `h5`, `h5`) et aucun landmark, **la structure est plate** : l'utilisateur mobile scrolle beaucoup pour trouver l'information utile (accès, état d'ouverture, places).

**Formulaires « une case par massif » — ✅ mesuré en production, et pire que mon estimation.** `formulaire_exportations` et `formulaire_rss_et_nouvelles` génèrent une case à cocher par massif. En local (7 polygones) le problème est invisible. Comptage réel sur le site en ligne :

```
$ curl -s https://www.refuges.info/formulaire_rss_et_nouvelles | grep -oE 'type=.checkbox' | wc -l
631
$ curl -s https://www.refuges.info/formulaire_exportations   | grep -oE 'type=.checkbox' | wc -l
636
```

**631 et 636 cases à cocher sur un seul écran** (j'avais estimé ~480 : l'estimation était basse). Conséquences concrètes :
- Sur mobile, un formulaire de 631 contrôles est en pratique inutilisable : le défilement à lui seul demande des centaines de gestes.
- Au clavier, atteindre le bouton de validation exige de traverser 631 tabulations — il n'y a pas de lien d'évitement (§2.6).
- Au lecteur d'écran, la liste est annoncée intégralement sans structure de regroupement suffisante pour s'y repérer.

Le problème est d'ailleurs reconnu en commentaire dans le code (`controlleurs/formulaire_rss_et_nouvelles.php:35`). Les 6 boutons « tout cocher / tout décocher » (`checkboites()`) sont un pansement, et ils reposent sur des handlers `onclick` inline dupliqués entre les deux formulaires.

**C'est un problème d'ergonomie de premier ordre que l'environnement local masque totalement** — et il ne figurait pas dans mon plan initial. Il justifie un lot dédié (cf. ROADMAP, Lot 3bis).

**Aucune pagination nulle part.** Confirmé sur l'ensemble des parcours :
- Recherche : limite dure à **40 résultats** (`includes/config.php:164`), ou 100 000 si la case « ne pas limiter » est cochée. La troncature est détectée par heuristique — `controlleurs/point_recherche.php:76-78` : *« si on a pile le nombre de points, c'est qu'on a atteint la limite »*.
- Nouvelles : plafond dur à 2 000 (`controlleurs/nouvelles.php:20-21`, pour contenir un crawler Google), « pagination » par lien « Plus » qui fait `$nombre+100` — donc **rechargement complet et croissant** de la page.

**Messages d'erreur : le bouton Retour comme stratégie.** Il n'y a **aucun ré-affichage de formulaire avec le champ en erreur**. Les contrôleurs basculent sur la vue `page_simple` et invitent explicitement l'utilisateur à revenir en arrière — `vues/point_modification.html:16-18` porte même un FIXME assumé. Sur un formulaire d'ajout de point qui compte une trentaine de champs, **une erreur de validation = ressaisie potentielle de tout le formulaire.** C'est le point UX le plus coûteux pour les contributeurs.

Les messages eux-mêmes sont en revanche **de bonne qualité, précis et parfois savoureux** (`modeles/point.php:785` : « c'est plus que l'Everest, vous avez dû vous tromper ? »). Le problème est le *véhicule*, pas le contenu.

**Nombre de clics par parcours clé :**

| Parcours | Écrans | Commentaire |
|---|---|---|
| Trouver un refuge par nom | 2 | efficace (form du bandeau) |
| Recherche avancée | **4** | formulaire → liste → fiche |
| Ajouter un point | **3** | dont un écran ne servant qu'à choisir le type |
| Obtenir un export GPX par massif | **3** | le dernier écran ne fournit qu'un lien à cliquer |
| Ajouter un commentaire | 1 | bon |

Deux frictions évitables : l'écran `/point_ajout` ne sert **qu'à choisir un type de point** (il pourrait être fusionné avec le formulaire), et `/formulaire_exportations` produit un écran intermédiaire dont le seul contenu est un lien à cliquer.

### 3.4 Performance en connexion dégradée (cas d'usage montagne)

**Le constat le plus rentable de tout l'audit.**

`.htaccess:63-67` n'active `mod_deflate` que sur 4 types MIME :

```apache
<IfModule mod_deflate.c>
  <IfModule mod_filter.c>
    AddOutputFilterByType DEFLATE application/rss+xml application/xml application/json application/vnd.google-earth
  </IfModule>
</IfModule>
```

**Ni `text/html`, ni `text/css`, ni `application/javascript`.** Vérifié par requête directe, **en local et ✅ en production** :

```
$ curl -sI -H "Accept-Encoding: gzip, br" http://localhost:8080/myol/dist/myol.js
  -> AUCUN Content-Encoding : servi non compressé
  Content-Length: 743676

$ curl -sI -H "Accept-Encoding: gzip, br" https://www.refuges.info/myol/dist/myol.js
  content-length: 743676
  -> AUCUN content-encoding : identique en production

$ curl -sI -H "Accept-Encoding: gzip, br" https://www.refuges.info/vues/style_pages.css
  content-length: 13643 | cache-control: max-age=21600
  -> AUCUN content-encoding
```

**Le `.htaccess` de production a donc la même lacune que celui du dépôt.** Le gain ci-dessous est immédiatement disponible en ligne.

Le module est chargé (`deflate_module (shared)`), la configuration est simplement incomplète. Gain mesuré en ajoutant les 3 types manquants :

| Fichier | Actuel | gzip -9 | Gain |
|---|---|---|---|
| `myol/dist/myol.js` | 743 676 o | 210 968 o | **−72 %** |
| `myol/dist/myol.css` | 19 834 o | 4 323 o | −79 % |
| `vues/style_pages.css` | 13 643 o | 4 261 o | −69 % |
| `vues/style_formulaire.css` | 14 346 o | 3 258 o | −78 % |
| `vues/_cartes.js` | 20 096 o | 4 877 o | −76 % |
| `vues/bandeau.css` | 3 613 o | 1 285 o | −65 % |
| `vues/autocomplete.js` | 4 356 o | 1 431 o | −68 % |
| **Total** | **819 564 o** | **230 403 o** | **−72 % (−575 Ko)** |

Le HTML n'est pas compté ici et bénéficierait du même ordre de gain (les pages font 11 à 23 Ko).

**Sur une connexion Edge à ~50 Ko/s, cela fait passer le chargement des assets de la fiche point de ~16 s à ~4,5 s.** Pour un site dont le cas d'usage central est la consultation en montagne, c'est **le meilleur rapport impact/effort du projet** : 1 ligne dans `.htaccess`, aucun risque de régression.

**Nombre de requêtes.** La page d'accueil charge **23 fichiers d'assets** (12 CSS + 11 JS) parce qu'elle utilise la stack Leaflet non bundlée :

```
/leaflet/leaflet/dist/leaflet-src.js, /leaflet/markercluster/…, /leaflet/fullscreen/…,
/leaflet/Coordinates/…, /leaflet/gps/…, /leaflet/FeatureGroup.SubGroup/…,
/leaflet/geocoder/…, /leaflet/src/MarkerCompass.js, /leaflet/src/tileLayers.js,
/leaflet/src/vectorLayers.js, /vues/_cartes_leaflet.js
```

`/nav` n'en charge que 7 (grâce au bundle myol) mais bien plus lourds. **Aucun `defer` ni `async`** sur les `<script>` (`vues/_pied.html:19-22`) — atténué par le fait qu'ils sont en fin de `<body>`.

**Cache.** `.htaccess:53-60` positionne `ExpiresByType` à 6 h pour CSS/JS et 12 h pour PNG. Comme le cache-busting par `filemtime` est déjà en place, ces durées pourraient être portées à un an (`immutable`) sans risque — gain net sur les visites répétées, typiques d'un usage de préparation de course.

**Erreurs JS en production.** 9 exceptions identiques sur la page d'accueil :

```
TypeError: Cannot read properties of undefined (reading 'json')   × 9
```

Signalé aussi par Lighthouse (`errors-in-console` en échec sur `/` et `/point_formulaire_recherche`). Origine probable : les clés API vides en local (§0.2) — **à confirmer en production avant d'en tirer une conclusion.**

---

## 4. Synthèse par gravité

### Bloquant (rend une fonction inaccessible)

| # | Constat | RGAA | Coût |
|---|---|---|---|
| B1 | 18 contrôles de navigation focusables et invisibles ; menus non ouvrables au clavier | 10.7, 12.8, 7.3 | **très faible** |
| B2 | 17/45 tabulations sur éléments invisibles (panneaux carte myol) | 10.7, 12.8 | moyen |
| B3 | Carte sans `tabindex`/`role`/`aria-label` : inutilisable au clavier et aux lecteurs d'écran | 7.1, 7.3 | élevé (mais alternative existante à valoriser) |
| B4 | 9 boutons + 1 `select` sans nom accessible | 7.1, 11.1 | faible à moyen |
| B5 | Textarea principal du formulaire de commentaire sans étiquette (+ 19 autres champs) | 11.1 | faible |
| B6 | `<title>` vide sur `/point_ajout_commentaire` | 8.5 | **1 ligne** |

### Majeur (gêne fortement)

| # | Constat | RGAA | Coût |
|---|---|---|---|
| M1 | Contraste des liens 3,83:1 (été) et 3,27:1 (automne) — 124 nœuds, 1 seule cause | 3.2 | **très faible** |
| M2 | Aucun landmark ; aucun `h1` sur 5 pages/6 ; aucun lien d'évitement | 9.1, 12.6, 12.7 | faible à moyen |
| M3 | Assets non compressés : −72 % / −575 Ko disponibles immédiatement ✅ | — | **très faible** |
| M4 | **Formulaires à 631 et 636 cases à cocher** sur un écran ✅ | 12.7, 11.5 | élevé |
| M5 | Cibles tactiles < 24 px (jusqu'à 72 par page) | WCAG 2.2 (hors RGAA 4.1) | faible |
| M6 | Zéro `rem` : le texte ne suit pas les préférences système | 10.4 | moyen |
| M7 | Erreurs de formulaire sans ré-affichage : ressaisie complète possible | 11.10 | élevé |

✅ = confirmé par mesure directe en production.

### Mineur

`<ul>` mal formé, `h3` vide, saut `h1`→`h5`, `$utilisateurs` non initialisé, `user-scalable=no` en code mort, CDATA résiduel, focus par `box-shadow` fragile en contraste forcé, cache-busting saisonnier inopérant, commentaires debug/prod inversés, `alt` redondants sur les icônes de légende, absence de garde-fou sur le mapping d'icônes, dump SQL public désynchronisé de la production.

### Infirmé après vérification

**L'icône « sommet » n'est pas cassée en production** (§2.9). C'était un artefact du dump SQL public. Le constat initial était classé majeur ; il descend en mineur (durcissement du code + resynchronisation du dump).

---

## 5. Ce qui relève de la retouche progressive et ce qui nécessite une réécriture

**Retouche progressive — la grande majorité.** Trois propriétés du code le permettent :
1. Le bandeau, le pied de page et le squelette HTML sont **mutualisés dans 3 fichiers**. Une correction y porte sur tout le site (landmarks, lien d'évitement, cibles tactiles, étiquette du champ de recherche).
2. Les **variables CSS existent déjà** : le theming, le contraste et le mode sombre se pilotent depuis `vues/style.css.php`.
3. Le JS est du **vanilla ES6 sans framework** : aucune dette de migration.

**Réécriture nécessaire — deux chantiers seulement, et aucun n'est urgent :**

1. **L'unification de la stack cartographique.** Maintenir Leaflet *et* OpenLayers, avec deux catalogues de fonds de carte dupliqués (`_cartes.js` / `_cartes_leaflet.js`), est le vrai coût structurel. Ma recommandation : **migrer la page d'accueil de Leaflet vers myol** plutôt que l'inverse. Raisons : myol est déjà utilisé sur 3 des 4 pages cartographiques, il est bundlé (7 requêtes contre 23), il est maintenu par un contributeur actif du projet, et cela supprime 3,4 Mo de `leaflet/` non versionné. Ce n'est pas un lot de démarrage : à faire **après** avoir sécurisé les gains d'accessibilité, car c'est le seul chantier avec un vrai risque de régression fonctionnelle.

2. **Le découplage JS/PHP**, prérequis à tout bundling ou à toute CSP stricte. 7 fichiers `.js` contiennent du PHP. Le remède est connu et incrémental : passer les données par `<script type="application/json">` ou des attributs `data-`, un fichier à la fois. **À ne lancer que si un besoin le justifie** (CSP, minification) — ce n'est pas un objectif en soi.

**Ce que je recommande de ne PAS faire :**
- **Ne pas introduire de framework front.** Le JS applicatif fait 46 Ko, il est lisible, et la communauté de contributeurs est composée de bénévoles. Un framework augmenterait la barrière à l'entrée sans résoudre un seul des constats ci-dessus.
- **Ne pas introduire de moteur de template.** L'absence d'auto-escaping est un vrai défaut, mais le migrer demanderait un audit champ par champ (certaines valeurs sont du HTML volontairement non échappé). Le rapport coût/bénéfice est mauvais face aux priorités d'accessibilité.
- **Ne pas toucher au forum phpBB** dans ce cadre. C'est un périmètre autonome, et il n'a pas été audité (§0.2).

---

## 6. Cadrage retenu

Après échange avec la personne qui porte le sujet, quatre décisions encadrent la suite. Elles sont reportées dans [`ROADMAP.md`](ROADMAP.md).

1. **Profil d'usage : mobile en montagne, desktop en préparation.** Les deux comptent, mais ils ne se recouvrent pas : le mobile *est* le cas de connexion dégradée. Cela verrouille la compression (§3.4) en priorité absolue et fait remonter le confort mobile (§3.1-3.2) au-dessus des chantiers de contribution.
2. **Aucune obligation légale RGAA.** Le référentiel est utilisé comme **levier d'ergonomie et de lisibilité**, pas comme cible de conformité déclarable. Conséquences directes sur le périmètre : **pas d'audit formel avec lecteur d'écran**, **pas de déclaration d'accessibilité à produire**, **pas de mise en conformité du forum phpBB**. On priorise ce qui se voit à l'usage plutôt que la couverture critère par critère — ce qui, en pratique, ne change presque rien aux Lots 0 à 3 : les mêmes corrections servent les deux objectifs.
3. **Statut de maintenance de `myol` : inconnu à ce stade**, à instruire sur le forum du projet. L'unification cartographique (§5, Lot 5c) reste donc **bloquée en attente de cette information** — c'est une décision de gouvernance avant d'être technique.
4. **Les formulaires de contribution peuvent évoluer librement**, y compris en fusionnant des écrans. Le Lot 5 de la roadmap conserve son périmètre complet.

---

## 7. Reproduire les mesures

```bash
make up && make seed
```

Puis, dans un répertoire de travail hors du dépôt :

```bash
npm i puppeteer @axe-core/puppeteer lighthouse
node axe-run.mjs      # axe-core, 6 pages x 2 viewports
node kbd-probe.mjs    # parcours clavier, contrastes calculés, captures
npx lighthouse http://localhost:8080/point/37 --form-factor=mobile --output=html
```

Deux précautions pour obtenir des chiffres représentatifs :
- Mettre `$config_wri['debug']=false` dans `config_privee.php` (sinon `myol-debug.js`, 2,8 Mo, est servi — cf. §0.3).
- Renseigner les clés dans `$config_wri['mapKeys']` pour mesurer le poids réel des tuiles.
