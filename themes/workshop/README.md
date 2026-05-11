# Workshop — Hugo theme

Editorial Hugo theme for a builder's notebook. Paper-and-ink palette with an ember accent, Inter + JetBrains Mono. Designed for [brianhengen.us](https://brianhengen.us).

## What's in the box

- **Layouts** — home, single post, posts list, series index, single series, tags, term, about, search, 404
- **Shortcodes** — `tldr`, `insight`, `metric`, `learn`, `author`, `series-nav`
- **Search** — client-side, generated `/index.json`, no external deps
- **Single CSS file** — `assets/css/workshop.css`, ~600 lines

## Install

```bash
# from your Hugo site root
git clone <this> themes/workshop
# or copy the themes/workshop/ directory in
```

Then in `hugo.toml`:

```toml
theme = "workshop"

[outputs]
  home = ["HTML", "RSS", "JSON"]

[taxonomies]
  tag = "tags"
  series = "series"

[params]
  brandName = "Your Name"
  brandInitials = "YN"
  domain = "YOURSITE.COM"
  email = "you@example.com"
  linkedin = "https://linkedin.com/in/you/"
```

## Content conventions

- **Posts** live in `content/posts/`. Front-matter: `title`, `description`, `date`, `tags`, optional `series`.
- **Series** live in `content/series/<slug>/`. The `_index.md` is the series landing; child `.md` files appear in the timeline (sorted by date). Front-matter on `_index.md`: `title`, `subtitle`, `description`, `status`, `startDate`, `code`.
- **About** at `content/about.md` with `layout: "about"`.
- **Search** at `content/search.md` with `layout: "search"`.

## Shortcode reference

```markdown
{{</* tldr */>}}
One-paragraph summary at the top of the post.
{{</* /tldr */>}}

{{</* insight */>}}
A single sentence that is the whole point. Use _emphasis_ for the key word.
{{</* /insight */>}}

{{</* metric num="10×" label="LATENCY IMPROVEMENT" */>}}

{{</* learn */>}}
What I'd do differently. **Bold the lesson.**
{{</* /learn */>}}

{{</* author */>}}

{{</* series-nav name="Multi-Modal RAG" */>}}
```

## Brand parameters (`params`)

| Key | What it controls |
|---|---|
| `brandName` | Footer + nav brand text |
| `brandInitials` | Two-letter mark in nav square |
| `domain` | Footer right side |
| `version` | Footer right side (after slash) |
| `author` | Default author shortcode name |
| `email` | Author shortcode link |
| `linkedin` | Author shortcode link |
| `description` | `<meta description>` fallback |

## Migration from PaperMod

1. Drop `themes/workshop/` into your `themes/` directory
2. Change `theme = "PaperMod"` to `theme = "workshop"` in `hugo.toml`
3. Add `home = ["HTML", "RSS", "JSON"]` under `[outputs]`
4. Remove any `[params]` keys specific to PaperMod (`profileMode`, `homeInfoParams`, `socialIcons`, etc.) — replace with the workshop keys above
5. Existing posts/series/tags work as-is. Shortcodes have new names — search/replace if you used PaperMod's

## License

MIT.
