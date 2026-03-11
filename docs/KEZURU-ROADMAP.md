# Kezuru — Roadmap

A lightweight PHP starter kit for developers who want to write PHP, not "framework".
Built on Slim 4, inspired by Japanese craft tooling philosophy: remove what's unnecessary to reveal what's underneath.

Base: [samuelgfeller/slim-starter](https://github.com/samuelgfeller/slim-starter)

---

## What's Already Working

- [x] Slim 4 framework with PHP-DI dependency injection
- [x] SQLite database (swappable to MariaDB/PostgreSQL via config)
- [x] User CRUD with validation
- [x] PHP-View templating (plain PHP, no Twig)
- [x] CakePHP query builder (not ORM — just queries)
- [x] Phinx database migrations
- [x] Monolog logging
- [x] CSRF protection
- [x] Dark/light theme toggle
- [x] ES module JS architecture (no jQuery, no build step)
- [x] PHPUnit test suite
- [x] Tailwind CSS v4 via standalone CLI (no Node, no npm)
- [x] Strapi-inspired colour theme with light/dark mode tokens
- [x] `bin/setup.php` — one-command project setup (directories, database, Tailwind CLI download, initial CSS build)
- [x] Three-zone template architecture (admin / front / account / shared)
- [x] Admin layout shell — Strapi-inspired sidebar + topbar + content area grid
- [x] Ionicons (MIT, inline SVG) adopted as icon set

---

## Phase 1 — Content Engine (MVP)

The core of Kezuru. A schema-driven content management system that auto-generates admin CRUD screens from simple PHP config files.

### 1.1 Content Modelling Philosophy

**The PDC hierarchy.** Kezuru content is organised into three levels:

- **Primitives** — single fields (`text`, `image`, `toggle`, etc.)
- **Data structures** — repeatable groups of primitives (a benefits box, a gallery item, a testimonial row)
- **Content types** — the top-level definition containing both

This is universal and non-prescriptive. A "benefits box", a "card", a "hero" — these are all just data structures. The names are yours. The engine doesn't care what you call them.

**Content types, not collections.** Kezuru uses the term "content type" — simple, unambiguous, no trademark conflicts.

**Schema as PHP files, stored in git.** Content type definitions live in `config/content/` as plain PHP arrays. They are the canonical source of truth. This keeps schema changes version-controlled, diffable, and deployable — avoiding the classic WordPress problem where "your database is everything and nothing is portable."

**Developer names the fields; developer owns the structure.** There is no generic `body` field. There is no assumed page structure. The developer looks at the actual template and names fields to match it — `hero_heading`, `intro_text`, `section_1_image`. Named data flows into named places. No magic, no `the_content()`.

**Fixed structure for MVP.** The developer defines the shape entirely. The client fills in the values. The page structure never changes at runtime. This is the right starting point.

**Flexible content (page builder) is a post-MVP layer.** The architecture does not prevent it, but it is not built now. See Phase 3+.

**GUI schema editor is deferred.** A browser-based tool that writes PHP schema files to disk (GUI + git-trackable) is a viable future addition. For MVP, developers edit schema files directly. If friction emerges among developer users, revisit.

**The engine reads what's there; it never assumes.** The FormRenderer, ListRenderer, and all engine components read field definitions from the schema and apply sensible defaults for anything absent. This means new field-level metadata keys can be added at any time without breaking existing schemas.

**Phinx is the single tool that touches the database.** The schema file is the source of truth. The CLI migration generator reads it and outputs a Phinx migration file. Phinx applies it. No competing systems, no auto-applying schema changes behind your back. Every DB change is explicit, reviewable, and git-tracked.

```
PHP schema file → php bin/kezuru generate:migration {type} → Phinx migration file → phinx migrate
```

Phinx is also used for core schema (users, settings) — anything that isn't content-type-driven.

### 1.2 Schema Definition Format

Define content types as PHP arrays in `config/content/`. One file per type. The array key becomes the database column name — keep it snake_case.

Two canonical examples ship with Kezuru. Use them as-is, copy and adapt them, or write your own from scratch.

**Example 1 — primitives only** (`config/content/team_members.php`):

```php
return [
    'label'        => 'Team Members',
    'label_single' => 'Team Member',
    'icon'         => 'people',
    'fields'       => [
        'full_name' => [
            'type'     => 'text',
            'label'    => 'Full Name',
            'required' => true,
        ],
        'job_title' => [
            'type'  => 'text',
            'label' => 'Job Title',
        ],
        'headshot' => [
            'type'  => 'image',
            'label' => 'Photo',
        ],
        'bio' => [
            'type'  => 'richtext',
            'label' => 'Biography',
        ],
        'is_published' => [
            'type'  => 'toggle',
            'label' => 'Published',
        ],
    ],
];
```

**Example 2 — primitives + data structure** (`config/content/pages.php`):

```php
return [
    'label'        => 'Pages',
    'label_single' => 'Page',
    'icon'         => 'document-text',
    'fields'       => [

        // Primitives
        'hero_heading' => [
            'type'     => 'text',
            'label'    => 'Hero Heading',
            'required' => true,
        ],
        'hero_image' => [
            'type'  => 'image',
            'label' => 'Hero Image',
        ],
        'intro_text' => [
            'type'  => 'richtext',
            'label' => 'Introduction',
        ],

        // Data structure — a repeatable group of primitives
        'benefits' => [
            'type'   => 'repeater',
            'label'  => 'Benefits',
            'fields' => [
                'icon' => [
                    'type'  => 'image',
                    'label' => 'Icon',
                ],
                'headline' => [
                    'type'     => 'text',
                    'label'    => 'Headline',
                    'required' => true,
                ],
                'body_text' => [
                    'type'  => 'richtext',
                    'label' => 'Body',
                ],
                'cta_text' => [
                    'type'  => 'text',
                    'label' => 'Button Label',
                ],
                'cta_url' => [
                    'type'  => 'url',
                    'label' => 'Button Link',
                ],
            ],
        ],

        'is_published' => [
            'type'  => 'toggle',
            'label' => 'Published',
        ],
    ],
];
```

The template then does:
```php
// Primitives
<?= $page['hero_heading'] ?>
<?= $page['intro_text'] ?>

// Data structure
<?php foreach ($page['benefits'] as $benefit): ?>
    <?= $benefit['headline'] ?>
    <?= $benefit['body_text'] ?>
    <a href="<?= $benefit['cta_url'] ?>"><?= $benefit['cta_text'] ?></a>
<?php endforeach ?>
```

No magic. No assumed structure. Named data into named places.

**MVP field types:**

| Type       | Renders as                        | DB column                                    |
|------------|-----------------------------------|----------------------------------------------|
| `text`     | `<input type="text">`             | VARCHAR(255)                                 |
| `textarea` | `<textarea>`                      | TEXT                                         |
| `richtext` | Textarea + simple editor          | TEXT                                         |
| `number`   | `<input type="number">`           | INTEGER                                      |
| `date`     | `<input type="date">`             | DATE                                         |
| `url`      | `<input type="url">`              | VARCHAR(500)                                 |
| `email`    | `<input type="email">`            | VARCHAR(254)                                 |
| `image`    | File upload + preview             | VARCHAR(500)                                 |
| `select`   | `<select>`                        | VARCHAR(255)                                 |
| `toggle`   | Checkbox/switch                   | BOOLEAN                                      |
| `repeater` | Repeatable group of sub-fields    | No column — generates a child table instead  |

A `repeater` field named `benefits` on the `pages` content type generates a child table `content_pages_benefits` with columns for `parent_id`, `sort_order`, and one column per sub-field.

**MVP field-level flags:**

| Flag       | Type    | Purpose                                      |
|------------|---------|----------------------------------------------|
| `required` | bool    | Validates presence before save               |
| `label`    | string  | Human-readable label shown in admin form     |

**Deferred field-level flags (YAGNI — add when needed, non-breaking):**

`is_editable`, `is_movable`, `group_id`, `group_order` — hints at future flexible content and drag-and-drop UI. Intentionally absent from MVP. Because the engine reads what's present and ignores what's absent, adding them later requires no architectural change.

### 1.3 Migration Generator (SchemaToMigration)

The schema file is the source of truth. The migration generator reads it and outputs a Phinx migration file. Phinx applies it. There is no system that modifies the database directly from schema — every DB change goes through an explicit, reviewable migration file.

**CLI usage:**
```bash
php bin/kezuru generate:migration pages
# Outputs: db/migrations/20240311120000_create_content_pages.php
```

**What it generates:**
- A `content_{type}` table with one column per primitive field
- A `content_{type}_{repeater_name}` child table for each `repeater` field, with `parent_id`, `sort_order`, and one column per sub-field
- Standard `id`, `created_at`, `updated_at` columns on every table

**Rules:**
- Re-running on an existing type generates an `alter` migration, not a fresh `create`
- The developer reviews the generated file before running `phinx migrate`
- Generated migrations are committed to git — colleagues run `phinx migrate` on pull
- Must produce valid migrations for SQLite, MariaDB, and PostgreSQL

### 1.4 Schema-to-Form (FormRenderer)

- Reads content type definition
- Outputs correct HTML form fields per type
- Handles validation rules (required, max length, etc.)
- FieldTypeRegistry maps type string to renderer class
- New field types added by registering one class

### 1.5 Generic Admin CRUD

All content types share one set of actions/routes:

```
GET    /admin/content/{type}             → List view
GET    /admin/content/{type}/create      → Create form
POST   /admin/content/{type}             → Store
GET    /admin/content/{type}/{id}        → Edit form
PUT    /admin/content/{type}/{id}        → Update
DELETE /admin/content/{type}/{id}        → Delete
```

Components:
- `ContentListAction` — generic list with sortable columns from schema
- `ContentCreateAction` / `ContentStoreAction` — form + save
- `ContentEditAction` / `ContentUpdateAction` — form + save
- `ContentDeleteAction` — soft or hard delete
- `ContentRepository` — generic CRUD using query builder
- `ContentValidator` — validates input against schema rules

### 1.6 File/Image Upload

- `ImageUploadService` handles uploads to `storage/uploads/`
- Resize/thumbnail generation
- Returns stored path for DB
- Abstracted via Flysystem for future S3/CDN support (optional)

### 1.7 Admin Navigation

- Auto-generated from content type definitions
- Sidebar lists each content type with its icon and label
- "Users" module remains as-is alongside content types
- Sidebar nav items array is ready to accept injected collection entries from SchemaLoader

---

## Phase 2 — Admin UI Overhaul

### Goals
- Modern, clean, professional admin feel
- Clients see it and feel confident
- Not over-designed — just polished

### Decision: Tailwind CSS via Standalone CLI
Tabler was considered but felt too Bootstrap-heavy. Tailwind gives us modern utility-first styling without introducing Node or npm into the project. The standalone CLI binary lives at `bin/tailwindcss` and is downloaded automatically by `bin/setup.php`.

- **Source CSS:** `resources/css/app.css` (theme tokens defined here)
- **Compiled CSS:** `public/assets/css/app.css`
- **Dev workflow:** `./bin/tailwindcss -i resources/css/app.css -o public/assets/css/app.css --watch`
- **Dark mode:** Uses existing `data-theme` attribute via `@variant dark` override
- **Colour system:** Strapi-inspired tokens — `primary`, `surface`, `body`, `muted`, `danger`, etc.

### Decision: Three-Zone Template Architecture

Templates are organised into three distinct zones plus a shared directory:

```
templates/
├── account/    # Front-facing account/profile area (skeleton — rip it out and build your own)
├── admin/      # The product. Polished, fixed, works out of the box
├── front/      # Public pages (skeleton — rip it out and build your own)
└── shared/     # Partials any zone can explicitly include
```

**Admin** is "the product" — it ships polished and complete. You can customise it, but you don't need to. **Front** and **Account** are skeletons — starting points that developers are expected to replace with their own designs. **Shared** holds partials (like the asset renderer) that any zone can explicitly pull in.

Each zone has its own layout. Page templates declare their layout on line one via `$this->setLayout('admin/layout.php')`. This is deterministic — open any template, you immediately know what zone it belongs to.

### Decision: Route Groups = Zones

Zones map to Slim route groups. `/admin/*` routes use the admin layout, `/account/*` routes use the account layout, top-level routes use the front layout. The Action specifies the layout explicitly — no middleware magic, no fallbacks, no guessing.

### Decision: Icons — Ionicons (MIT, Inline SVG)

Icons are from the Ionicons set (MIT licensed), used as inline SVGs in templates. No CDN, no JS dependency, no icon font. SVGs use `stroke="currentColor"` so they inherit text colour automatically. Optimised via SVGOMG before inclusion.

### Completed

- [x] Admin layout shell (`templates/admin/layout.php`)
  - CSS Grid: fixed 260px sidebar + fluid main area
  - Main area splits into topbar (auto height) + content (fills remaining)
  - Loads zone-specific assets via shared asset renderer
  - Dark mode script fires before paint
- [x] Admin sidebar (`templates/admin/partials/sidebar.php`)
  - Dark background (`bg-sidebar`), Strapi-style
  - Brand/app name at top
  - Icon + label navigation with active state detection via route names
  - Nav items array ready for content collection injection
  - Settings link anchored at bottom
- [x] Admin topbar (`templates/admin/partials/topbar.php`)
  - Page title on the left (set by page template via `$pageTitle` variable)
  - Dark/light mode toggle with sun/moon Ionicons
  - User avatar placeholder on the right
- [x] Admin dashboard (`templates/admin/dashboard.php`)
  - Calls `$this->setLayout('admin/layout.php')` — demonstrates the zone pattern
  - Quick-link cards to available sections (Users)
  - No charts, no analytics — this is a content tool, not an ERP

### Remaining

- [ ] Wire up route (`GET /admin`) and `AdminDashboardAction` to serve the dashboard
- [ ] Move Samuel's asset renderer to `templates/shared/assets.php`
- [ ] Wire up dark mode toggle JS for admin
- [ ] Responsive sidebar (collapsible on tablet/mobile)
- [ ] Gradually replace existing Gfeller CSS with Tailwind equivalents
- [ ] Colour identity refresh (move away from Strapi's exact palette to something unique)

### Inspiration
- Strapi's clean content editing experience
- Statamic's field UI polish
- NOT WordPress. Never WordPress.

---

## Architecture Decision: Content Model — Collections, Not Hierarchies

Every content type in Kezuru is a flat collection. This is a deliberate departure from WordPress's approach where registering a custom post type auto-generates archive pages, tag pages, JSON API routes, and other URLs you never asked for.

In Kezuru:
- You define a collection and what fields it contains
- You query that collection in your template
- You decide what pages exist and what routes serve them
- Nothing is auto-generated. Every route is explicit

**No automatic archive pages. No automatic tag pages. No automatic JSON routes. No automatic anything.**

The developer defines the data shape. The developer defines the routes. The developer writes the template. The system is fully deterministic — if a URL exists, it's because someone deliberately created it.

This approach also means no complex parent/child hierarchies governing content. Collections are flat. If you need nesting or relationships, you model them explicitly — not through framework magic that quietly spawns pages and feeds behind your back.

---

## Phase 3 — Live Preview (The Dream)

Split-pane editing: fields on the left, front-end preview on the right.

### Architecture
- `ContentPreviewAction` — receives form data via AJAX POST
- Renders through the actual front-end template with unsaved data
- Returns HTML to an iframe in the right pane
- Debounced onChange fires preview refresh as client types

### Notes
- Does NOT need to be in MVP
- The admin content area has been designed with space to accommodate a preview pane alongside the edit form
- Preview uses same template files as the live site

---

## Phase 4 — Packaging & Distribution

### Composer create-project
```bash
composer create-project kezuru/kezuru mysite
cd mysite
php bin/setup.php
```

### bin/setup.php CLI Installer
Already implemented:
- Creates `storage/`, `storage/uploads/`, `logs/` directories
- Initialises SQLite database file
- Downloads Tailwind CSS standalone CLI (platform-detected)
- Runs initial Tailwind CSS build

Still to add:
- Run SchemaToTable against all content definitions
- Prompt for admin email + password
- Create first admin user

### Deployment
- Single directory, single process
- PHP-FPM + Caddy (auto SSL)
- SQLite for small sites, MariaDB/PostgreSQL via one config change
- No Docker required (but Dockerfile provided for those who want it)
- No Node, no build step, no compilation

---

## Phase 5 — Community Release

### Before release
- Clean README ("This is not a CMS. It's a starter kit.")
- Example content definition shipped (basic pages type)
- MIT licence
- Contributing guide
- Changelog

### Positioning
For PHP developers who:
- Want to write PHP, not "Laravel"
- Need a clean admin for client content
- Don't want WordPress's React-driven complexity
- Value simplicity, control, and independence from private equity

---

## Architecture Reference

```
src/
├── Core/
│   ├── Database/
│   ├── Exception/
│   ├── Middleware/
│   ├── Responder/
│   ├── Settings/
│   └── Utility/
│
└── Module/
    ├── Home/
    ├── User/
    │   ├── Create/
    │   ├── Read/
    │   ├── Update/
    │   ├── Delete/
    │   ├── List/
    │   ├── Validation/
    │   └── Data/
    │
    └── ContentEngine/           # Phase 1
        ├── Schema/
        │   ├── SchemaLoader.php
        │   ├── SchemaToTable.php
        │   └── FieldTypeRegistry.php
        ├── Admin/
        │   ├── Action/
        │   └── Service/
        │       ├── FormRenderer.php
        │       ├── ListRenderer.php
        │       └── PreviewRenderer.php  (Phase 3)
        ├── Repository/
        │   └── ContentRepository.php
        ├── Validation/
        │   └── ContentValidator.php
        └── FileHandling/
            └── ImageUploadService.php

config/
├── routes.php
├── container.php
├── middleware.php
├── settings.php
├── env/
│   └── env.php
├── content/                     # Content type definitions (Phase 1)
│   ├── page.php
│   ├── team_member.php
│   └── testimonial.php

resources/
├── css/
│   └── app.css                  # Tailwind theme tokens + imports

storage/
├── database.sqlite
└── uploads/

templates/
├── admin/                       # The product — polished, out-of-the-box
│   ├── layout.php               # Outer shell: head, CSS grid, partial includes
│   ├── dashboard.php            # Admin landing page
│   ├── partials/
│   │   ├── sidebar.php          # Dark sidebar with Ionicon nav
│   │   └── topbar.php           # Page title, theme toggle, user menu
│   └── content/                 # Auto-generated CRUD screens (Phase 1)
│       ├── list.php
│       └── form.php
├── front/                       # Skeleton — rip it out, build your own
│   ├── layout.php
│   └── components/
├── account/                     # Skeleton — rip it out, build your own
│   └── layout.php
└── shared/                      # Partials any zone can explicitly include
    └── assets.php               # CSS/JS asset tag renderer

public/
├── index.php                    # Front controller
└── assets/
    └── css/
        └── app.css              # Compiled Tailwind output
```

---

## Tech Stack

| Layer            | Tool                     | Why                                    |
|------------------|--------------------------|----------------------------------------|
| Framework        | Slim 4                   | Micro-framework, no opinions, just PHP |
| DI Container     | PHP-DI                   | Clean, annotation-free                 |
| Database         | SQLite / MariaDB / PgSQL | One config change to switch            |
| Query Builder    | cakephp/database         | Standalone, multi-driver, not an ORM   |
| Migrations       | Phinx                    | Standard PHP migration tool            |
| Templating       | PHP-View                 | Plain PHP templates, no Twig           |
| JS               | Vanilla ES modules       | No jQuery, no bundler, no build step   |
| Icons            | Ionicons                 | MIT, inline SVG, no JS dependency      |
| CSS              | Tailwind CSS v4          | Standalone CLI, no Node, utility-first |
| Logging          | Monolog                  | Industry standard                      |
| Testing          | PHPUnit                  | Industry standard                      |

---

## Design Principles

1. **You write PHP.** Not framework DSL, not YAML incantations, not React.
2. **Template-first, API-available.** Server-rendered by default, JSON endpoints alongside for those who need them.
3. **Clients edit content.** Clean fields, clear labels, nothing they don't need.
4. **One server, one directory.** No microservices, no Docker required, no Node.
5. **Schema defines everything.** One config file per content type. The engine does the rest.
6. **Collections, not hierarchies.** Flat, deterministic content. No auto-generated pages, tags, or routes.
7. **Swap what you want.** SQLite today, PostgreSQL tomorrow. Tailwind today, custom UI tomorrow.
8. **No private equity.** MIT licensed. No "free tier" that turns paid. No rug pulls.