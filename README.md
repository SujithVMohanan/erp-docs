# erp-docs

A comprehensive business solution and ERP knowledge base covering business processes, workflows, terminology, data requirements, and operational requirements for different types of companies and industries.

## Local setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

## GitHub Pages deployment

This project is configured for MkDocs and GitHub Pages.

### Local build

```bash
mkdocs build
```

### GitHub deployment

1. Push the repository to GitHub.
2. Open the repository in GitHub.
3. Go to Settings → Pages.
4. Select GitHub Actions as the source.
5. The workflow in [.github/workflows/deploy.yml](.github/workflows/deploy.yml) will build and publish the site automatically on pushes to `main`.

After deployment, the site will be available at:

```text
https://<username>.github.io/<repository>/
```

> This is the standard MkDocs GitHub Pages URL. It is not served from a `/docs` folder by default.
