# Analyse : chargement des places d'apprentissage sur orientation.ch

Analyse du 10.11.2026, faite avec un navigateur headless qui capture les requêtes réseau (équivalent de l'onglet Réseau).

## TL;DR

- Le site est une app **Next.js** (App Router / RSC, Turbopack). `https://www.orientation.ch/dyn/show/2930` redirige vers
  `https://www.orientation.ch/fr/recherche/apprentissages`.
- La liste des résultats **n'est pas dans le HTML** : le rendu serveur ne contient que les filtres (cantons, professions…).
- Les résultats sont chargés **côté client depuis Typesense Cloud** :
  `POST https://h6gjov3kr97m42fzp.a1.typesense.net/multi_search?x-typesense-api-key=<clé de recherche publique>`
  (la clé est une clé *search-only* intégrée au front-end).
- Le **sitemap ne contient aucune place d'apprentissage** : il n'est pas utilisable pour lister les offres.
- Les **pages de détail** `https://www.orientation.ch/fr/apprentissages/{id}` sont rendues côté serveur (HTML complet).

## 1. Sitemap

- `robots.txt` : `Allow: /`, `Sitemap: https://www.orientation.ch/sitemap.xml`.
- `sitemap.xml` : un seul `urlset` (~6 100 URL, ~1,6 Mo), sans sous-sitemaps. Les URL couvrent `/fr/formations/*` (~3 900),
  `/fr/professions/*` (~850), `/fr/video/*`, `/fr/questionnaires-*`, etc.
- **Aucune URL `/fr/apprentissages/{id}`** dans le sitemap.

## 2. Requête de recherche (Typesense)

Envoyée par le navigateur au chargement de la page, puis à chaque changement de filtre. `Content-Type: text/plain`, corps JSON :

```json
{
  "searches": [
    {
      "preset": "apprenticeship",
      "collection": "apprenticeships",
      "q": "*",
      "filter_by": "canton_id:=[`GE`]",
      "facet_by": "canton_id,langcode,open_places,profession_id,school_year",
      "sort_by": "profession_name_fr:asc,locality_name_fr:asc,location_name:asc",
      "max_facet_values": 5000,
      "page": 1
    },
    {
      "preset": "apprenticeship",
      "collection": "apprenticeships",
      "q": "*",
      "facet_by": "canton_id",
      "max_facet_values": 5000,
      "page": 1,
      "per_page": 0
    }
  ]
}
```

- La 1re recherche renvoie la page de résultats (10 par page par défaut) et les facettes. La 2e (`per_page: 0`)
  sert seulement à compter les résultats par canton pour le filtre.
- Paramètres d'URL de la page → `filter_by` Typesense :
  - `?canton=GE` → `canton_id:=[\`GE\`]`
  - `?profession=80104.1` → `profession_id:=[80104.1]`
  - `?page=2` → `page: 2`
- Les codes profession (`80104.1`, `28420.1`…) se trouvent dans le payload RSC de la page (taxonomies `value`/`label`).

### Réponse (extrait, `?canton=GE`)

`found: 148`, `out_of: 19581` (total des offres indexées en Suisse), `per_page: 10`.
La page affiche « 346 places » : c'est la somme de `open_places` sur les 148 offres.

Champs principaux d'un document `apprenticeships` :

| Champ | Exemple | Usage |
|---|---|---|
| `id` / `id_apprenticeship` | `3600782` | → `/fr/apprentissages/3600782` |
| `profession_name_fr` / `_de` / `_it` | `Agent/e de propreté CFC` | métier |
| `profession_id`, `sbfi_number`, `swissdoc` | `["80102.1","80104.1"]`, `80104` | identifiants du métier |
| `school_year` | `2027` | année de début |
| `open_places` | `"4"` | places ouvertes |
| `canton_id`, `locality_name_fr`, `locality_geo` | `["GE"]`, `Genève`, `[[46.21, 6.15]]` | lieu |
| `location_name`, `location_street`, `location_zip_code`, `location_lbx_code` | entreprise / lieu de formation | |
| `application_name`, `application_street`, `application_zip_code`, `application_locality_name`, `application_business_phone`, `application_email`, `application_person_*` | adresse de postulation | |
| `url` | site de l'entreprise / annonce | lien externe |
| `drupal_profession_path_fr` | `/bb/fr/professions/agent-e-de-proprete-cfc` | fiche métier |
| `langcode`, `last_updated`, `index_updated_at` | `["Fr"]`, `2026-06-10T10:44:09Z` | métadonnées |

## 3. Autres appels observés

- `GET /api/draft-status`, `GET /api/translation-paths?path=...` : internes Next.js, sans données d'offres.
- `GET /cookie-check?trg=...` : redirection 302 au premier accès (pose d'un cookie), puis retour sur la page.
- Scripts tiers : Matomo (`piwik.sdbb.ch`), statuspal, widget de chat.

## 4. Pistes pour l'app de postulation

1. **Sitemap** : inutilisable pour les offres.
2. **Typesense** : source structurée (JSON, filtrable par canton, profession et année). C'est une API interne et non
   documentée du site : avant toute utilisation automatisée, vérifier les conditions d'utilisation d'orientation.ch / CSFO
   (SDBB), limiter la fréquence des appels et, idéalement, demander un accès officiel.
3. **Pages de détail** `/fr/apprentissages/{id}` : HTML rendu côté serveur, lisible sans JavaScript.
