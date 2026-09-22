# Agent Guidelines for Hugo Zen Demo Site

## Build Commands
- **Build site**: `hugo` (outputs to `public/` directory)
- **Build with custom baseURL**: `hugo --baseURL "https://example.com/"`
- **Serve locally**: `hugo server` (starts dev server at http://localhost:1313)
- **Clean build**: `rm -rf public/ && hugo`

> Requires Hugo **v0.158.0 or newer** (the Zen v7 theme uses `css.Build`). Hugo is managed via `mise.toml` (aqua backend) — run `mise install` to get the right version.

## Development Workflow
- **Update theme**: `hugo mod get -u` (updates Hugo theme dependencies; theme is the Go module `github.com/frjo/hugo-theme-zen/v7`)
- **Watch mode**: `hugo server --watch` (auto-rebuild on file changes)

## Code Style Guidelines

### Hugo Templates
- Use `{{ define "main" }}` for main content blocks
- Use `{{-` and `-}}` for whitespace control in templates
- Follow Hugo's naming conventions for partials and layouts
- Use `.Params` for accessing frontmatter variables

### Frontmatter
- Use YAML format for content frontmatter
- Required fields: `title`, `date` (ISO 8601 format)
- Optional: `description`, `tags`, `categories`, `draft`

### Content Structure
- Content files use `.md` extension
- Organize by language: `content/en/`, `content/sv/`
- Use Hugo shortcodes for reusable components (e.g., `{{< figure >}}`)
- Blog posts always end with a `## References` section listing every external
  source cited in the post, formatted as
  `- [Title](url) — Author/Site`; add it to existing posts when editing them

### Assets
- Vanilla CSS (with nesting and cascade layers) in `assets/css/` — theme v7 has **no Sass pipeline**; site overrides live in `assets/css/_custom.css` (loaded last by the theme's `styles.css`)
- JavaScript in `assets/js/` served as-is
- Static files in `static/` copied to root of built site

### Naming Conventions
- Files: kebab-case (e.g., `post-title.md`, `custom-style.css`)
- Directories: lowercase (e.g., `blog/`, `notes/`)
- Hugo variables: camelCase in templates
- CSS classes: kebab-case (e.g., `main-content`, `sidebar-nav`)

### Error Handling
- Hugo build fails on template syntax errors
- Check build output for missing variables or malformed frontmatter
- Use `{{ with .Param "fieldname" }}` for optional parameters

### Git Workflow
- Main branch auto-deploys to GitHub Pages
- Use descriptive commit messages
- Keep content and code changes separate when possible