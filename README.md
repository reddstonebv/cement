# Cement

Reddstone's project system: a design-token core, a WordPress shell, and per-project canvases. This repo is the entry point — it holds the setup guide ([GitHub Pages](https://reddstonebv.github.io/cement/)) that walks a new project through its first choices.

## The system

```
┌──────────────────────────────────────────────────────────┐
│  cement-wp-plugin           "the core"                   │
│  Process flow, brand tokens, CDN, workflow tracker       │
└───────────────────┬──────────────────────────────────────┘
                    │ serves brand.css, exposes hooks
┌───────────────────▼──────────────────────────────────────┐
│  cement-wp-theme            "the shell"                  │
│  WordPress integration, admin tooling, child support     │
└───────────────────┬──────────────────────────────────────┘
                    │ inherits, overrides, or replaces
┌───────────────────▼──────────────────────────────────────┐
│  cement-wp-theme-{project}  "the canvas"                 │
│  Brand-specific templates, tokens, project logic         │
└──────────────────────────────────────────────────────────┘
```

## Repos

| Repo | Role |
|---|---|
| [cement-wp-plugin](https://github.com/reddstonebv/cement-wp-plugin) | Core — `/cement/*` workflow routes, brand.css CDN, intake, exports |
| [cement-wp-theme](https://github.com/reddstonebv/cement-wp-theme) | Parent theme — WP plumbing, token foundation, admin enhancements |
| [cement-wp-theme-child](https://github.com/reddstonebv/cement-wp-theme-child) | Starter child theme — fork this for every new WordPress project |
| [cement-wp-theme-rsr](https://github.com/reddstonebv/cement-wp-theme-rsr) | Worked example — full visual replacement (Rotterdam Ship Repair) |
| [cement-tooling](https://github.com/reddstonebv/cement-tooling) | Shared lint/format config — prettier, stylelint, eslint, phpcs |

## Starting a new project

Open the **[setup guide](https://reddstonebv.github.io/cement/)** and follow the flow — it starts with the one question that decides everything else:

**WordPress or static?**

- **WordPress** → full stack: plugin + parent theme + a fork of the child starter. The guide covers Local setup, cloning, activation, pages and tooling. Deep-dives live in each repo's README ([child bootstrap](https://github.com/reddstonebv/cement-wp-theme-child#using-as-a-base-for-a-new-client), [full fresh-site walkthrough](https://github.com/reddstonebv/cement-wp-theme-rsr#readme)).
- **Static** → no WordPress install; consume the design tokens from the CDN (`/cement/css/brand.css`) and pull in [cement-tooling](https://github.com/reddstonebv/cement-tooling) for lint/format.

## This repo

- `README.md` — this file
- `docs/index.html` — the setup guide, published via GitHub Pages (Settings → Pages → deploy from `main`, `/docs`)
