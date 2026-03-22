# AGENTS.md

## Project Overview

This is a Hugo-based static blog using the PaperMod theme. The site is deployed at `https://awbuana.github.io/`.

## Build/Lint/Test Commands

### Hugo Commands

```bash
# Development server with live reload
hugo server -D

# Production build
hugo

# Production build with minification
hugo --minify

# Build drafts (including draft: true content)
hugo -D

# Validate and cleanup
hugo --gc

# Check for errors without building
hugo --printI18nWarnings
```

### Running a Single Test

This is a static site generator - there are no unit tests. However, you can validate:

```bash
# Verify all content renders correctly
hugo -D --disableKinds=home,taxonomy,term

# Test specific page
hugo -D --renderToMemory
```

### Git Submodule

```bash
# Update PaperMod theme submodule
git submodule update --init --recursive
```

## Code Style Guidelines

### General Principles

- **Hugo Version**: Requires v0.146.0 or higher (extended version recommended)
- **No Comments**: Do not add comments unless explicitly requested
- **Minimal Changes**: Only modify what is necessary

### Writing Style Guidelines

**Avoid repetitive opening phrases.** Do not use the same story setup across multiple posts. Previously used phrases to avoid:
- "I stared at the whiteboard..."
- "I was three months into building..."
- "I couldn't sleep that night..."

**Vary your hooks.** Each post should have a distinct opening:
- Use different settings (meeting room, debugging session, production outage)
- Vary the emotional tone (confusion, frustration, excitement, anxiety)
- Mix first-person observations with direct statements

**Check existing posts before writing.** Search for similar phrases in `content/posts/` to ensure you're not repeating yourself:
```bash
grep -r "your opening phrase" content/posts/
```

**Prefer specific details over generic setups.** Instead of "I stared at the whiteboard," try:
- "The database query timed out after 30 seconds..."
- "Our CEO raised an eyebrow during the demo..."
- "I was reviewing the logs at 2 AM when..."

### Project Structure

```
.
├── content/           # Blog posts (Markdown)
│   ├── posts/        # Blog articles
│   └── about/        # About page
├── layouts/          # Custom layouts (overrides theme)
│   └── partials/     # Custom partials
├── static/           # Static files (favicon, images)
├── assets/           # Asset files (CSS, JS)
├── themes/           # Hugo themes (PaperMod submodule)
├── hugo.toml         # Site configuration
└── public/           # Built site output (gitignored)
```

### File Conventions

#### Content Files (Markdown)

Front matter fields:
- `title` - Page title (required)
- `description` - Meta description
- `date` - Publication date (YYYY-MM-DD format)
- `draft` - Set to `true` to hide from build
- `tags` - Array of tag strings
- `categories` - Array of category strings
- `cover.image` - Featured image URL
- `showToc` - Enable table of contents
- `searchHidden` - Hide from search

#### Configuration (hugo.toml)

- Use TOML format with double quotes for strings
- Use array syntax `[[params.socialIcons]]` for array items
- Nested sections use `[section.subsection]` syntax

### CSS Conventions

- Use CSS custom properties for colors: `--primary-color`, `--gradient-start`, etc.
- Color scheme: Purple/indigo primary (#6366f1), amber accent (#f59e0b)
- Fonts: Inter for content, JetBrains Mono for code
- Mobile breakpoint: 768px
- Use `!important` sparingly - only when overriding theme defaults
- Include transition animations for interactive elements (0.3s ease)

### HTML/Template Conventions

- Place custom CSS in `layouts/partials/extend_head.html`
- Use semantic HTML5 elements
- Theme uses partial caching - use `partial` not `partialCached` for dynamic content
- Hugo template syntax: `{{ }}` for directives, `{{- }}` for trim-enabled

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Files | kebab-case | `welcome-post.md` |
| Directories | kebab-case | `content/posts` |
| Front matter | camelCase | `showToc`, `searchHidden` |
| CSS classes | kebab-case | `.post-title` |
| Hugo params | camelCase | `homeInfoMode` |

### Error Handling

- Always run `hugo -D` after changes to verify build success
- Check for template errors: `hugo --printUnusedTemplates`
- Validate multilingual: `hugo --printI18nWarnings`
- Use `--verbose` flag for detailed build output

### Working with the Theme

- The PaperMod theme is a git submodule in `themes/hugo-PaperMod`
- To override theme templates, create matching file in `layouts/`
- The theme is read-only - do not modify files in `themes/`
- Custom CSS should be added to `assets/css/custom.css` or `layouts/partials/extend_head.html`
- Theme partials can be overridden by creating files at `layouts/partials/<name>.html`

### Common Patterns

#### Adding a New Blog Post

```bash
hugo new posts/my-new-post.md
```

#### Adding a New Page

```bash
hugo new about/_index.md
```

#### Linking to Content

```markdown
[Link Text]({{ "posts/my-post" | relURL }})
```

#### Including Partial Templates

```go
{{ partial "custom_partial.html" . }}
```

### Deployment

- Built output goes to `public/` directory
- Deploy `public/` contents to GitHub Pages
- Run `hugo --minify` for production deployment

### Checklist Before Committing

- [ ] Run `hugo -D` and verify no errors
- [ ] Test in browser at localhost:1313
- [ ] Check for broken links
- [ ] Verify draft content is excluded (unless testing)
- [ ] Update README.md if adding new features
