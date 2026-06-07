# Masthead theme template

A starter [Masthead](https://github.com/JoeriDijkstra/masthead) theme.
Clone it, customize the templates and stylesheet, zip the result, and
upload it through the **Themes** page on any Masthead instance.

This template is a copy of Masthead's built-in `default` theme — minimal,
readable, opinionated about typography. It's the simplest of the three
shipped looks and the easiest to fork.

## Layout

```
manifest.json              required — theme metadata + customizable tokens
theme.css                  required — the stylesheet inlined into <style>
templates/
  layout.liquid            required — wraps every render target
  index.liquid             required — site homepage (post list)
  post.liquid              required — single blog post
  page.liquid              required — standalone page
  blog.liquid              required — "blog" format page (intro + post list)
  not_found.liquid         required — site-scoped 404
assets/                    optional — images, fonts, additional CSS/JS
```

Every file listed as `required` must be present in the zip Masthead
extracts, or the upload will fail validation.

## The manifest

`manifest.json` declares the theme's identity and two kinds of
customization surfaces: **tokens** (site-wide knobs) and **metadata**
(per-page knobs).

```json
{
  "name": "Default",
  "slug": "my-theme",
  "version": "1.0.0",
  "author": "Masthead",
  "description": "Minimal, readable, opinionated about typography.",
  "tokens": [
    { "key": "accent", "label": "Accent color", "type": "color",
      "default": "#0066cc", "category": "Colors" },
    { "key": "logo", "label": "Logo", "type": "file",
      "default": "", "category": "Branding" },
    { "key": "font", "label": "Body font", "type": "select",
      "options": ["ui-sans-serif, system-ui, sans-serif", "ui-serif, Georgia, serif"],
      "default": "ui-sans-serif, system-ui, sans-serif", "category": "Typography" }
  ],
  "metadata": [
    { "key": "layout", "label": "Page layout", "type": "select",
      "options": ["contained", "wide"], "default": "contained" }
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
uploaded asset.

- `tokens[].type` is one of:
  - `color` — `<input type="color">`; value is a `#rrggbb` string
  - `string` — free-text input (e.g. a font stack)
  - `length` — CSS length string (`880px`, `60ch`, `4rem`)
  - `number` — numeric input, stored as a string for CSS embedding
  - `file` — a picker over the site's existing uploads; the stored value
    is the chosen upload's id, **resolved to a public URL at render time**.
    Good for logos, favicons, header/background images.
  - `select` — a `<select>` over a fixed `options` array (required,
    non-empty); value is the chosen option string. Use for switches like
    a body-font or layout choice.
- `tokens[].default` is the value used when a site has no override.
- `tokens[].options` is **required for `select`** — a non-empty array of
  strings. Ignored for other types.
- `tokens[].category` (optional) groups the token under a labelled
  accordion on the Settings screen. Tokens with no category fall under
  **"General"**. Use categories (`"Colors"`, `"Typography"`, `"Layout"`,
  …) to keep a large token set navigable.

To expose a new customizable value, add a token here **and** use it in
`theme.css` as `var(--my-key)`. Masthead emits the effective tokens as CSS
custom properties at render time, *after* `theme.css`, so any
`var(--accent)` reference picks up the overridden value:

```css
:root { --accent: <user value or default>; }
```

**`file` tokens in CSS.** A `file` token is emitted as a CSS `url(...)`,
so reference it with `var()` where a URL is expected:

```css
.site-header { background-image: var(--hero); }
```

An empty `file` token (no upload chosen) resolves to an empty string, so
guard with a sensible default or let the rule no-op.

### Metadata (per-page)

`metadata` declares per-page settings that show up on the **page editor's
"Page settings"** section. The active theme drives the form — every
field you declare becomes an input the user can fill in per page. The
values reach your templates as `page.metadata.<key>`.

- `metadata[].type` is one of:
  - `string` — single-line text input
  - `text` — multi-line textarea
  - `boolean` — checkbox (template sees `true` / `false`)
  - `color` — color picker (returns `#rrggbb`)
  - `url` — text input validated as a URL
  - `number` — numeric input (template sees `123` or `1.5`, not a string)
  - `select` — dropdown; requires a non-empty `options` array of strings
- `metadata[].default` is required and used when the page has no override.
- `metadata[].description` (optional) is shown as helper text below the input.

Example use inside `page.liquid`:

```liquid
<article{% if page.metadata.layout == "wide" %} class="wide"{% endif %}>
  {% if page.metadata.subtitle != "" %}
    <p class="subtitle">{{ page.metadata.subtitle | escape }}</p>
  {% endif %}
  {{ body_html }}
</article>
```

Example use inside `layout.liquid` (gating the nav per page):

```liquid
{% unless page.metadata.hide_navigation %}
  <nav class="site-nav">…</nav>
{% endunless %}
```

**Theme-switch resilience.** Unknown metadata keys are *preserved* on
save — a page authored under theme A and edited under theme B will keep
A's keys even though the form doesn't show them, so switching back to A
restores everything. Tokens behave differently: unknown overrides are
dropped, because tokens are inert without a matching CSS variable.

**Type coercion.** Form posts always arrive as strings; the renderer
coerces declared fields to their type before the template sees them.
That means you can branch directly on `{% if page.metadata.hide_navigation %}`
without the `"true"` / `"false"` footgun.

**Empty values fall back to defaults.** Blanking an input on save strips
that key, so the next render reads the manifest default again.

## Templates

Templates are [Liquid](https://shopify.github.io/liquid/) — the safe,
sandboxed templating language used by Jekyll and Shopify. They have **no
access** to Elixir, the filesystem, or the database — the rendering
context is a fixed set of plain maps.

### Render flow

1. The inner template (e.g. `post.liquid`) renders against the page's
   context.
2. The result is passed to `layout.liquid` as the variable `content`.
3. The final HTML is sent to the browser.

### Variables available

| Variable | Always | index | post | page | blog | not_found |
|---|---|---|---|---|---|---|
| `site` | ✓ | | | | | |
| `pages` (nav list) | ✓ | | | | | |
| `theme.tokens.<key>` | ✓ | | | | | |
| `theme.css` (in layout) | ✓ | | | | | |
| `content` (in layout) | ✓ | | | | | |
| `posts` | | ✓ | | | ✓ | |
| `post` | | | ✓ | | | |
| `page` | | | | ✓ | ✓ | |
| `page.metadata.<key>` | | | | ✓ | ✓ | |
| `body_html` | | | ✓ | ✓ | ✓ | |

Each shape:

```
site     { name, title, description, slug, css_overrides, homepage_slug }
post     { title, slug, excerpt, published_at, url }
page     { title, slug, format, url, metadata }
```

`site.homepage_slug` is the slug of the page chosen as the site's
homepage (empty when the homepage is the default post list) — handy for
marking the active nav item.

`page.metadata` is the merge of your manifest's `metadata` defaults and
whatever the user filled in on the page editor. See **Metadata** above.

`body_html` arrives **pre-sanitized** by Masthead's HTML allowlist — it is
the one variable that intentionally emits raw HTML. Drop it in directly:

```liquid
<div class="post-body">{{ body_html }}</div>
```

### Escaping user-supplied strings

Liquid does **not** auto-escape output. Any string a site owner can edit
(site name, page title, post title, ...) must be piped through `escape`:

```liquid
<h1>{{ site.title | default: site.name | escape }}</h1>
```

This is the single most important footgun to remember when authoring a
theme. If you forget, a site name like `<script>alert(1)</script>` will
execute in every visitor's browser.

### Filters

Masthead ships the standard Liquid filter set (`escape`, `default`, `date`,
`size`, ...) plus three additions:

| Filter | Use | Example |
|---|---|---|
| `strftime` | Calendar-style date formatting on Elixir `DateTime` values | `{{ post.published_at | strftime: "%B %-d, %Y" }}` |
| `iso8601` | Safe-on-nil ISO-8601 for `<time datetime="...">` | `<time datetime="{{ post.published_at | iso8601 }}">` |
| `asset_url` | Build a URL to a theme-bundled asset | `<img src="{{ 'logo.png' | asset_url: theme.asset_base }}">` |

## Customizing

A typical workflow:

1. Set a `slug` in `manifest.json` that doesn't clash with the built-ins
   (`default`, `studio`, `tailwind`).
2. Bump `version` whenever you ship a meaningful change (uploads only
   update an installed theme when the version is strictly newer).
3. Edit `theme.css` — the cascade is `theme.css` first, then your token
   overrides, then the site's free-text CSS overrides (set in the
   settings UI).
4. Edit the templates. Always `| escape` user-supplied strings.
5. Add or remove tokens as you go, grouping them with `category`. The
   settings UI rebuilds the customization form from the manifest, so every
   declared token automatically gets an input under its category accordion.

## Packaging and uploading

```bash
zip -r my-theme.zip manifest.json theme.css templates assets
```

Then in the Masthead admin: **Themes → + Upload theme**, drop the zip in
the modal, click Install. The validator checks size caps, path safety,
manifest schema, and parses every Liquid template before promoting the
files into object storage.

Once installed, the theme shows up in the **Theme** dropdown on each
site's settings page. Select it, tweak the tokens, save.

## Limits

- Archive: max **5 MB** compressed, **25 MB** uncompressed, **200 files**.
- Asset extensions allowed: `.css .png .jpg .jpeg .gif .webp .svg .woff .woff2 .ttf .otf .ico .json`.
- Per-site CSS overrides: max **50 KB**.
- No `..` / absolute paths / Windows-style separators in zip entries.
- Partials (`{% include %}`, `{% render %}`) are not supported in v1.

## License

MIT — do whatever you want with this template.
