# baraline.github.io

Personal homepage of Antoine Guillaume built with MkDocs Material.

## 🌐 Live Site

Visit the site at: [https://baraline.github.io](https://baraline.github.io)

## 📚 Structure

- **Professional**: CV, publications, conferences, reviewer work, and public communications
- **Blog**: Technical blog posts on machine learning, data science, and software development
- **Links**: Social media and professional network links

## 🚀 Local Development

### Prerequisites

- Python 3.8+
- pip

### Setup

1. Clone the repository:
```bash
git clone https://github.com/baraline/baraline.github.io.git
cd baraline.github.io
```

2. Install dependencies:
```bash
pip install mkdocs-material mkdocs-blog-plugin
```

3. Serve the site locally:
```bash
mkdocs serve
```

4. Open your browser and visit `http://127.0.0.1:8000`

## 🛠️ Building

To build the static site:

```bash
mkdocs build
```

The built site will be in the `site/` directory.

## 📝 Adding Content

### Adding a Blog Post

Create a new markdown file in `docs/blog/posts/` with the following frontmatter:

```markdown
---
date: YYYY-MM-DD
authors:
  - baraline
categories:
  - Category1
  - Category2
---

# Your Post Title

Your content here...

<!-- more -->

Extended content...
```

### Updating Professional Information

Edit the relevant files in the `docs/professional/` directory:
- `cv.md` - Curriculum Vitae
- `publications.md` - Academic publications
- `conferences.md` - Conference presentations
- `reviewer.md` - Reviewer work
- `communications.md` - Public communications

## 🚢 Deployment

The site automatically deploys to GitHub Pages when you push to the `main` branch via GitHub Actions.

## 📄 License

Content is © Antoine Guillaume. Code is MIT licensed.

## 📧 Contact

- GitHub: [@baraline](https://github.com/baraline)
- LinkedIn: [antoine-guillaume](https://linkedin.com/in/antoine-guillaume)