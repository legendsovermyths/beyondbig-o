# bits & beyond

> from contest to codebase

A minimalistic, elegant blog exploring advanced competitive programming algorithms and their real-world applications in software development.

## Features

- **Jekyll-powered** — GitHub Pages compatible
- **Mathematical rendering** — MathJax support for LaTeX equations
- **Syntax highlighting** — Beautiful code blocks with highlight.js
- **Responsive design** — Looks great on all devices
- **Elegant typography** — Inter font family for readability
- **SEO optimized** — Built-in SEO tags and sitemap generation

## Quick Start

### Local Development

1. **Install Ruby and Jekyll**
   ```bash
   # On macOS
   brew install ruby
   gem install jekyll bundler
   
   # On Ubuntu/Debian
   sudo apt-get install ruby-full build-essential zlib1g-dev
   gem install jekyll bundler
   ```

2. **Clone and setup**
   ```bash
   git clone <your-repo-url>
   cd BitAndBeyond
   bundle install
   ```

3. **Run locally**
   ```bash
   bundle exec jekyll serve
   ```
   
   Visit `http://localhost:4000`

### Writing Posts

Create new blog posts in the `_posts` directory with the naming convention:
```
YYYY-MM-DD-title-with-hyphens.md
```

#### Post Template

```markdown
---
layout: post
title: "Your Post Title"
date: YYYY-MM-DD
tags: [algorithms, data-structures, graphs]
author: "Your Name"
excerpt: "Brief description of the post content..."
---

Your content here with full markdown, LaTeX math, and code blocks support.

## Math Example
The time complexity is $O(n \log n)$ for the sorting step.

$$\sum_{i=1}^{n} i = \frac{n(n+1)}{2}$$

## Code Example
```python
def example_function():
    return "Hello, world!"
```
```

### Deployment to GitHub Pages

1. **Create a new repository** on GitHub
2. **Update `_config.yml`** with your details:
   ```yaml
   title: "bits & beyond"
   url: "https://yourusername.github.io"
   author:
     name: "Your Name"
     email: "your.email@example.com"
   ```

3. **Push to GitHub**
   ```bash
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/yourusername/your-repo-name.git
   git push -u origin main
   ```

4. **Enable GitHub Pages**
   - Go to repository Settings
   - Scroll to "Pages" section
   - Select "Deploy from a branch"
   - Choose "main" branch and "/ (root)" folder
   - Save

Your site will be available at `https://yourusername.github.io/your-repo-name`

## Customization

### Fonts
The site uses [Inter](https://rsms.me/inter/) for body text and [JetBrains Mono](https://www.jetbrains.com/mono/) for code. You can change fonts in `_layouts/default.html`.

### Colors
Edit the CSS variables in `assets/css/style.css` to customize the color scheme:

```css
:root {
  --primary-color: #2563eb;
  --text-color: #1a1a1a;
  --background-color: #fefefe;
  /* ... */
}
```

### Navigation
Edit the navigation links in `_layouts/default.html`:

```html
<div class="nav-links">
    <a href="{{ '/' | relative_url }}">Home</a>
    <a href="{{ '/archive/' | relative_url }}">Archive</a>
    <a href="{{ '/about/' | relative_url }}">About</a>
    <!-- Add more links here -->
</div>
```

## Content Guidelines

### Mathematical Notation
- Use `$...$` for inline math: `$O(n \log n)$`
- Use `$$...$$` for display math:
  ```latex
  $$
  \sum_{i=1}^{n} i = \frac{n(n+1)}{2}
  $$
  ```

### Code Blocks
Specify the language for syntax highlighting:
```python
def dijkstra(graph, start):
    # Implementation here
    pass
```

### Images and Diagrams
Place images in `assets/images/` and reference them:
```markdown
![Algorithm visualization](/assets/images/dijkstra-demo.gif)
```

For diagrams, consider using:
- [Mermaid](https://mermaid-js.github.io/) for flowcharts and graphs
- [TikZ](https://tikz.net/) for mathematical diagrams
- [Graphviz](https://graphviz.org/) for graph visualizations

## Performance

The site is optimized for performance with:
- Minimal CSS and JavaScript
- Efficient font loading with `font-display: swap`
- Optimized images (use WebP when possible)
- CDN-delivered libraries

## Browser Support

- Chrome/Chromium 60+
- Firefox 55+
- Safari 12+
- Edge 79+

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally
5. Submit a pull request

## License

This project is open source and available under the [MIT License](LICENSE).

## Acknowledgments

- [Jekyll](https://jekyllrb.com/) for the static site generator
- [Inter](https://rsms.me/inter/) and [JetBrains Mono](https://www.jetbrains.com/mono/) for beautiful typography
- [MathJax](https://www.mathjax.org/) for mathematical rendering
- [highlight.js](https://highlightjs.org/) for syntax highlighting

---

Built with ❤️ for the competitive programming and software engineering community.
