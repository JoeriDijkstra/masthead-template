# Masthead theme template

A starter [Masthead](https://github.com/JoeriDijkstra/masthead) theme.
Clone it, customize the templates and stylesheet, zip the result, and
upload it through the **Themes** page on any Masthead instance.

This template is an exact copy of Masthead's built-in `default` theme —
minimal, readable, opinionated about typography. It's the simplest of the
shipped looks and the easiest to fork. The only difference from the
built-in is the `slug` (the built-in's `default` slug is reserved, so this
template ships as `starter` — rename it to whatever you like).

## Layout

```
manifest.json              required — theme metadata + customizable tokens
theme.css                  required — the stylesheet inlined into <style>
templates/
  layout.liquid            required — wraps every render target
  index.liquid             required — site homepage (post list)
  post.liquid              required — single blog post
  page.liquid              required — standalone page (markdown / html body)
  not_found.liquid         required — site-scoped 404
  pages/                   optional — theme pages (see below)
    blog.liquid            a theme page template
    blog.json              its settings config (sidecar)
assets/                    optional — images, fonts, additional CSS/JS
```

Every file listed as `required` must be present in the zip Masthead
extracts, or the upload will fail validation. **`blog` is no longer a
required fixed template** — the post-list page now ships as a *theme page*
under `templates/pages/`, and you're free to delete it or add your own.

## The manifest

`manifest.json` declares the theme's identity and two kinds of site-wide
customization surface: **tokens** (knobs that apply to the whole site) and
**metadata** (default per-page settings for ordinary markdown/html pages).

```json
{
  "name": "Default",
  "slug": "my-theme",
  "version": "1.0.0",
  "author": "Masthead",
  "description": "Minimal, readable, opinionated about typography.",
  "tokens": [
    { "key": "accent", "label": "Accent color", "type": "color",
      "default": "#0066cc", "category": "Appearance" },
    { "key": "favicon", "label": "Favicon", "type": "file",
      "default": "", "category": "Appearance" },
    { "key": "show_search", "label": "Show post search", "type": "boolean",
      "default": false, "category": "Posts" }
  ],
  "metadata": [
    { "key": "layout", "label": "Page layout", "type": "select",
      "options": ["contained", "wide"], "default": "contained" },
    { "key": "show_navigation", "label": "Show site navigation",
      "type": "boolean", "default": true }
  ]
}
```

- `slug` is the unique identifier. It must be 1–32 chars; lowercase
  letters, digits, and hyphens; and must start and end with a letter or
  digit. The slugs `default`, `studio`, and `tailwind` are reserved for
  built-ins — pick something else.
- `version` uses semver. Re-uploading a theme **updates it in place only
  when the manifest version is a strictly-newer SemVer** than the
  installed copy; otherwise the upload is rejected. Bump it whenever you
  ship a change you want existing sites to pick up.

### Tokens (site-wide)

`tokens` declares knobs that show up on each site's **Settings** screen.
Most tokens become CSS custom properties; the `file` type resolves to an
uploaded asset. Tokens are always **scalar** — they can't hold nested
objects or lists.

To expose a new customizable value, add a token here **and** use it in
`theme.css` as `var(--my-key)`. Masthead emits the effective tokens as CSS
custom properties at render time, *after* `theme.css`, so any
`var(--accent)` reference picks up the overridden value:

```css
:root { --accent: <user value or default>; }
```

**`file` tokens in CSS.** A `file` token is emitted as a CSS `url(...)`,
so reference it with `var()` where a URL is expected
(`background-image: var(--hero)`). An empty `file` token (no upload
chosen) resolves to an empty string, so guard with a sensible default or
let the rule no-op.

**Categories.** `tokens[].category` (optional) groups the token under a
labelled accordion on the Settings screen. Tokens with no category fall
under **"General"**. Use categories (`"Appearance"`, `"Typography"`,
`"Posts"`, …) to keep a large token set navigable.

See **Field types** below for the full list of scalar types.

### Metadata (per-page defaults for ordinary pages)

`metadata` declares per-page settings for **markdown / html pages** — the
fields show up on the page editor's **"Page settings"** section. The
active theme drives the form: every field you declare becomes an input the
user fills in per page, and the values reach your templates as
`page.metadata.<key>`.

Unlike tokens, top-level metadata fields may be **containers** (`object`
and `list`), so a single page can carry structured settings. See **Field
types** below.

```liquid
<article{% if page.metadata.layout == "wide" %} class="wide"{% endif %}>
  {{ body_html }}
</article>
```

**Theme-switch resilience.** Unknown metadata keys are *preserved* on
save — a page authored under theme A and edited under theme B keeps A's
keys even though the form doesn't show them, so switching back to A
restores everything. Tokens behave differently: unknown overrides are
dropped, because tokens are inert without a matching CSS variable.

**Type coercion.** Form posts always arrive as strings; the renderer
coerces declared fields to their type before the template sees them. You
can branch directly on `{% if page.metadata.show_navigation %}` without
the `"true"` / `"false"` footgun.

**Empty values fall back to defaults.** Blanking an input on save strips
that key, so the next render reads the manifest default again.

## Theme pages

A **theme page** is a full-page template you ship with the theme, used for
pages whose content is *structured settings* rather than a freeform
markdown/html body. The post list (`blog`), a landing page, a contact
page, a pricing table — anything where the author fills in fields instead
of writing prose.

A theme page is two files in `templates/pages/`:

```
templates/pages/landing.liquid   the Liquid template
templates/pages/landing.json     its settings config (the "sidecar")
```

The basename ties them together (`landing.liquid` ↔ `landing.json`). The
`.liquid` file renders the page; the `.json` file declares the editable
settings. A `.liquid` with no matching `.json` is a theme page with no
settings.

### The sidecar config

```json
{
  "label": "Landing",
  "description": "A marketing landing page with a hero and feature grid.",
  "metadata": [
    { "key": "headline", "label": "Headline", "type": "string",
      "default": "Welcome" }
  ]
}
```

- `label` — the human name shown in the page editor's template dropdown
  (falls back to the filename).
- `description` — optional helper text shown under the dropdown.
- `metadata` — the field schema for this page, identical in shape to the
  manifest's `metadata` (see **Field types**). Values reach the template
  as `page.metadata.<key>`.

There is **no `version`** in a sidecar — page configs travel with the
theme and are versioned by the manifest.

### Authoring flow in the admin

In the page editor, the author picks the **Theme page** format, then
chooses one of your theme pages from a dropdown. The page's "settings"
form is built from that page's sidecar `metadata`; there is no body
editor. **The template can't be changed once the page has been saved** —
pick the right one up front. (Switching a *new*, unsaved page between
templates is fine.)

`blog` ships as a theme page in this template: set a page's format to
**Theme page → Blog**, and optionally make it the site homepage, to get
the post list as your front page.

## Field types

Tokens, manifest `metadata`, and theme-page sidecar `metadata` all share
one field model. **Scalar** types are allowed everywhere; **container**
types (`object`, `list`) are allowed only at the *top level* of a
`metadata` schema (manifest or sidecar), never in tokens and never nested
inside another container (one level of nesting only).

### Scalar types

| Type | Editor | Template sees |
|---|---|---|
| `string` | single-line text | the string |
| `text` | multi-line textarea | the string |
| `color` | color picker | `#rrggbb` |
| `length` | text input | CSS length (`880px`, `60ch`) |
| `number` | numeric input | a number (`123`, `1.5`) |
| `url` | text input (URL-validated) | the string |
| `boolean` | checkbox | `true` / `false` |
| `select` | dropdown over `options` | the chosen option string |
| `file` | picker over the site's uploads | a **public URL** (resolved from the stored upload id at render time) |

- `default` is **required for every scalar field** and is used when there
  is no override.
- `options` is **required for `select`** — a non-empty array of strings.
- `description` (optional) renders as helper text under the input.
- `category` (optional) groups the field under a labelled accordion in the
  editor, exactly like token categories.

### `object` — a named group

A group of scalar fields with a single value (a map). Use it to bundle
related settings, e.g. a hero block.

```json
{ "key": "hero", "label": "Hero", "type": "object",
  "fields": [
    { "key": "title", "label": "Title", "type": "string", "default": "Welcome" },
    { "key": "image", "label": "Background", "type": "file", "default": "" }
  ]
}
```

```liquid
<section style="background-image: url({{ page.metadata.hero.image }})">
  <h1>{{ page.metadata.hero.title | escape }}</h1>
</section>
```

- `fields` (required, non-empty) holds the nested **scalar** fields.
- An object's default is derived from its fields' defaults — you don't
  give the object itself a `default`.

### `list` — a repeatable group

A repeatable group whose value is an **array of maps**. The author can
**add, remove, and drag-reorder** items in the editor.

```json
{ "key": "features", "label": "Features", "type": "list",
  "item_label": "Feature",
  "fields": [
    { "key": "title", "label": "Title", "type": "string", "default": "" },
    { "key": "icon",  "label": "Icon",  "type": "file",   "default": "" }
  ],
  "default": [
    { "title": "Fast" },
    { "title": "Simple" }
  ]
}
```

```liquid
<ul class="features">
  {% for f in page.metadata.features %}
    <li>
      {% if f.icon != "" %}<img src="{{ f.icon }}" alt="">{% endif %}
      {{ f.title | escape }}
    </li>
  {% endfor %}
</ul>
```

- `fields` (required, non-empty) holds the nested **scalar** fields each
  item carries.
- `item_label` (optional) names a single row in the editor (the "Add
  <item_label>" button, row headings).
- `default` (optional) is an array of seed items, rendered and editable
  out of the box; omit it (or use `[]`) to start empty.
- `file` fields inside list items resolve to public URLs just like
  top-level file fields.

## Templates

Templates are [Liquid](https://shopify.github.io/liquid/) — the safe,
sandboxed templating language used by Jekyll and Shopify. They have **no
access** to Elixir, the filesystem, or the database — the rendering
context is a fixed set of plain maps.

### Render flow

1. The inner template (e.g. `post.liquid`, or a `pages/<name>.liquid`)
   renders against the page's context.
2. The result is passed to `layout.liquid` as the variable `content`.
3. The final HTML is sent to the browser.

### Variables available

| Variable | layout | index | post | page | theme page | not_found |
|---|---|---|---|---|---|---|
| `site` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `pages` (nav list) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `theme.tokens.<key>` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `theme.css` / `content` | ✓ | | | | | |
| `posts` | | ✓ | | | ✓ | |
| `post` | | | ✓ | | | |
| `page` | | | | ✓ | ✓ | |
| `page.metadata.<key>` | | | | ✓ | ✓ | |
| `page.template` | | | | | ✓ | |
| `body_html` | | | ✓ | ✓ | | |
| `tags` / `current_tag` | | | | | ✓ (blog) | |

Each shape:

```
site     { name, title, description, slug, css_overrides, homepage_slug }
post     { title, slug, excerpt, published_at, url }
page     { title, slug, format, template, url, metadata }
```

- `site.homepage_slug` is the slug of the page chosen as the homepage
  (empty when the homepage is the default post list) — handy for marking
  the active nav item.
- `page.template` is the theme-page name (e.g. `"blog"`) for theme pages,
  empty otherwise.
- `page.metadata` is the merge of the schema defaults (manifest `metadata`
  for ordinary pages, the sidecar `metadata` for theme pages) and whatever
  the author filled in.
- Theme pages receive the full `posts` list, so a `blog` page can render
  it. On a blog theme page, `tags` lists the site's tags and `current_tag`
  reflects the `?tag=` filter, so you can build a tag filter (each `tag`
  has `name`, `slug`, `active`).
- `body_html` arrives **pre-sanitized** by Masthead's HTML allowlist — the
  one variable that intentionally emits raw HTML. Theme pages have no body
  editor, so they don't get `body_html` (the `blog` page guards for it).

### Escaping user-supplied strings

Liquid does **not** auto-escape output. Any string a site owner can edit
(site name, page title, post title, a metadata `string`/`text` field, …)
must be piped through `escape`:

```liquid
<h1>{{ site.title | default: site.name | escape }}</h1>
```

This is the single most important footgun to remember when authoring a
theme. If you forget, a value like `<script>alert(1)</script>` will
execute in every visitor's browser. (`file` fields are URLs, not free
text, so they don't need escaping — but never interpolate one into a
`javascript:`-capable context.)

### Filters

Masthead ships the standard Liquid filter set (`escape`, `default`, `date`,
`size`, …) plus three additions:

| Filter | Use | Example |
|---|---|---|
| `strftime` | Calendar-style date formatting on `DateTime` values | `{{ post.published_at | strftime: "%B %-d, %Y" }}` |
| `iso8601` | Safe-on-nil ISO-8601 for `<time datetime="...">` | `<time datetime="{{ post.published_at | iso8601 }}">` |
| `asset_url` | Build a URL to a theme-bundled asset | `<img src="{{ 'logo.png' | asset_url: theme.asset_base }}">` |

## Previewing locally (recommended)

The [`masthead` CLI](https://github.com/JoeriDijkstra/masthead-cli) renders
your theme through the exact same pipeline as the live platform — no
database, no upload step:

```bash
masthead preview            # serves http://localhost:4010, re-reads on refresh
masthead validate           # parse manifest + templates + page configs
masthead package            # build an installable zip
```

`preview` ships realistic sample content and a live token inspector, so
you can iterate on templates, `theme.css`, and page settings and see the
result on refresh.

## Packaging and uploading

The zip Masthead consumes is exactly:

```
manifest.json
theme.css
templates/layout.liquid  index.liquid  post.liquid  page.liquid  not_found.liquid
templates/pages/*.liquid + *.json     (theme pages + their sidecar configs)
assets/…                              (optional)
```

Build it with the CLI (`masthead package`) or by hand:

```bash
zip -r my-theme.zip manifest.json theme.css templates assets
```

Then in the Masthead admin: **Themes → + Upload theme**, drop the zip in
the modal, click Install. The validator checks size caps, path safety, the
manifest schema, every Liquid template, **and every page sidecar config**
before promoting the files into object storage. Once installed, the theme
shows up in the **Theme** dropdown on each site's settings page.

## Customizing — a typical workflow

1. Set a non-reserved `slug` in `manifest.json`.
2. Bump `version` whenever you ship a meaningful change.
3. Edit `theme.css` — the cascade is `theme.css` first, then your token
   overrides, then the site's free-text CSS overrides.
4. Edit the templates. Always `| escape` user-supplied strings.
5. Add or remove tokens / metadata fields, grouping them with `category`.
   The settings UI and page editor rebuild their forms from the manifest
   and sidecars, so every declared field automatically gets an input.
6. Add theme pages under `templates/pages/` (a `.liquid` + a `.json`) for
   structured, settings-driven pages like landings or the post list.

## Limits

- Archive: max **5 MB** compressed, **25 MB** uncompressed, **200 files**.
- Asset extensions allowed: `.css .png .jpg .jpeg .gif .webp .svg .woff .woff2 .ttf .otf .ico .json`.
- Per-site CSS overrides: max **50 KB**.
- No `..` / absolute paths / Windows-style separators in zip entries.
- Container fields nest **one level** (objects/lists hold scalars only).
- Partials (`{% include %}`, `{% render %}`) are not supported in v1.

## License

MIT — do whatever you want with this template.
