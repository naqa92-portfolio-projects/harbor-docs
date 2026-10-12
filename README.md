# harbor-docs

Étude de cas de [harbor-lab](https://github.com/naqa92-portfolio-projects/harbor-lab), une plateforme de gouvernance des images de conteneurs : architecture, modèle de menace, décisions, ce qui a cassé, limites.

- 🇫🇷 Site : <https://naqa92-portfolio-projects.github.io/harbor-docs/>
- 🇬🇧 English: <https://naqa92-portfolio-projects.github.io/harbor-docs/en/>

Le site est construit avec MkDocs Material et `mkdocs-static-i18n` (français par défaut, traductions en `*.en.md`) et publié sur GitHub Pages à chaque push sur `main`.

```sh
uv venv && uv pip install --require-hashes -r requirements.txt
.venv/bin/mkdocs serve
```
