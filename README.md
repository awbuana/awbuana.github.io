# awbuana's Blog

Personal blog built with Hugo and PaperMod theme.

## Project Description

A minimalist, fast, and responsive personal blog for sharing thoughts on software development, technology trends, and lessons learned. Features a personalized UI with a gradient-based design, JetBrains Mono for code, and Inter font for content.

## Dependencies

- **Hugo** v0.146.0 or higher (extended version recommended)
- **hugo-PaperMod** theme (included as git submodule)

## Install Dependencies

### 1. Install Hugo

**macOS:**
```bash
brew install hugo
```

**Linux:**
```bash
snap install hugo
# or
sudo apt install hugo
```

**Windows:**
```bash
choco install hugo-extended
```

Verify installation:
```bash
hugo version
```

### 2. Install Git Submodules

If you cloned the repo without submodules:
```bash
git submodule update --init --recursive
```

## Run the Server

### Development Mode (with drafts)

```bash
hugo server -D
```

### Production Build

```bash
hugo
```

## Test the Server

### Local Testing

1. Start the server:
   ```bash
   hugo server -D
   ```

2. Open your browser to `http://localhost:1313`

3. Verify:
   - Homepage loads with profile info
   - Navigation menu works (Posts, Tags, About, Archive)
   - Theme toggle (light/dark) functions
   - Sample blog post displays correctly
   - Code syntax highlighting works

### Test Commands

```bash
# Build and check for errors
hugo

# Build with drafts included
hugo -D

# Validate all content
hugo --gc
```

## Project Structure

```
.
├── content/           # Blog posts and pages
│   ├── posts/         # Blog articles
│   └── about/         # About page
├── layouts/           # Custom layouts
│   └── partials/      # Custom partials (extend_head.html)
├── static/            # Static files (favicon, images)
├── assets/            # Asset files (CSS)
├── themes/            # Hugo themes (PaperMod)
├── hugo.toml          # Site configuration
└── public/            # Built site output (gitignored)
```

## Configuration

Edit `hugo.toml` to customize:

- Site title and description
- Social media links
- Menu items
- Theme settings

## Build for Production

```bash
hugo --minify
```

Output will be in `public/` directory, ready to deploy to GitHub Pages or any static hosting.
