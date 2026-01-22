# MkDocs Quick Start

From the project root:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

mkdocs serve   # http://127.0.0.1:8000
# or
mkdocs build
```

## Structure

- `mkdocs.yml` – site configuration + navigation
- `docs/index.md` – home page (converted from the original `README.md`)
- `docs/tasks/` – lab task pages
- `docs/images/` – lab images
