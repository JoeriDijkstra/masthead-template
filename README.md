# Ledger theme template

A starter [Ledger](https://github.com/JoeriDijkstra/ledger) theme. Clone
it, customize the templates and stylesheet, zip the result, and upload it
through the **Themes** page on any Ledger instance.

This template is a copy of Ledger's built-in `default` theme — minimal,
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

Every file listed as `required` must be present in the zip Ledger
extracts, or the upload will fail validation.

## The manifest

`manifest.json` declares the theme's identity and the **tokens** a site
owner can override per-site from the settings screen:

```json
{
  "name": "Default",
  "slug": "default",
  "version": "1.0.0",
  "author": "Ledger",
  "description": "Minimal, readable, opinionated about typography.",
  "tokens": [
    { "key": "accent", "label": "Accent color", "type": "color", "default": "#0066cc" }
  ]
}
```

- `slug` is the unique identifier. It must be 1–32 chars, lowercase
  letters/digits/hyphens. The slugs `default`, `studio`, and `blank` are
  reserved for built-ins — pick something else.
- `version` bumps trigger a re-parse of cached templates on the next
  request. Use semver.
- `tokens[].type` is one of `color`, `string`, `length`, `number` — the
  setting screen renders a matching `<input>` per token.
- `tokens[].default` is the value used when a site has no override.

To expose a new customizable value, add a token here **and** use it in
`theme.css` as `var(--my-key)`. Ledger emits the effective tokens as CSS
custom properties at render time:

```css
:root { --accent: <user value or default>; }
```

Tokens are emitted *after* `theme.css`, so any `var(--accent)` reference
in your CSS will pick up the overridden value.

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
| `body_html` | | | ✓ | ✓ | ✓ | |

Each shape:

```
site     { name, title, description, slug, css_overrides }
post     { title, slug, excerpt, published_at, url }
page     { title, slug, format, url }
```

`body_html` arrives **pre-sanitized** by Ledger's HTML allowlist — it is
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

Ledger ships the standard Liquid filter set (`escape`, `default`, `date`,
`size`, ...) plus three additions:

| Filter | Use | Example |
|---|---|---|
| `strftime` | Calendar-style date formatting on Elixir `DateTime` values | `{{ post.published_at | strftime: "%B %-d, %Y" }}` |
| `iso8601` | Safe-on-nil ISO-8601 for `<time datetime="...">` | `<time datetime="{{ post.published_at | iso8601 }}">` |
| `asset_url` | Build a URL to a theme-bundled asset | `<img src="{{ 'logo.png' | asset_url: theme.asset_base }}">` |

## Customizing

A typical workflow:

1. Bump the `slug` in `manifest.json` so it doesn't clash with the
   built-in `default`.
2. Bump `version` whenever you ship a meaningful change.
3. Edit `theme.css` — the cascade is `theme.css` first, then your token
   overrides, then the site's free-text CSS overrides (set in the
   settings UI).
4. Edit the templates. Always `| escape` user-supplied strings.
5. Add or remove tokens as you go. The settings UI rebuilds the
   customization form from the manifest, so every declared token
   automatically gets an input.

## Packaging and uploading

```bash
zip -r my-theme.zip manifest.json theme.css templates assets
```

Then in the Ledger admin: **Themes → + Upload theme**, drop the zip in
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
