# Agent Guidelines for Hugo Zen Demo Site

## Build Commands
- **Build site**: `hugo` (outputs to `public/` directory)
- **Build with custom baseURL**: `hugo --baseURL "https://example.com/"`
- **Serve locally**: `hugo server` (starts dev server at http://localhost:1313)
- **Clean build**: `rm -rf public/ && hugo`

## Development Workflow
- **Update theme**: `hugo mod get -u` (updates Hugo theme dependencies)
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

### Assets
- SASS files in `assets/sass/` compiled automatically by Hugo
- JavaScript in `assets/js/` served as-is
- Static files in `static/` copied to root of built site

### Naming Conventions
- Files: kebab-case (e.g., `post-title.md`, `custom-style.scss`)
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