# claude-skills

Marketplace de plugins Claude Code : des skills génériques partagés entre projets.

## Plugins

| Plugin | Source | Skills |
| :- | :- | :- |
| `dev-workflow` | ce dépôt (`plugins/dev-workflow`) | `/dev-workflow:review` |
| `security-audit` | [cloudflare/security-audit-skill](https://github.com/cloudflare/security-audit-skill), épinglé sur un commit | `/security-audit:security-audit` |

## Installation

Pour soi, dans tous ses projets :

```bash
claude plugin marketplace add johnkrovitch/claude-skills
claude plugin install dev-workflow@johnkrovitch-skills
claude plugin install security-audit@johnkrovitch-skills
```

Puis activer la mise à jour automatique : `/plugin` → **Marketplaces** → **Enable auto-update**.
Sans elle, `claude plugin marketplace update johnkrovitch-skills` récupère les changements à la demande.

Pour toute l'équipe d'un projet, dans son `.claude/settings.json` versionné :

```json
{
  "extraKnownMarketplaces": {
    "johnkrovitch-skills": {
      "source": { "source": "github", "repo": "johnkrovitch/claude-skills" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "dev-workflow@johnkrovitch-skills": true
  }
}
```

Le marketplace est proposé à l'ouverture du projet, après le dialogue de confiance du dossier.
`enabledPlugins` au niveau projet suffit pour `dev-workflow`, dont les fichiers sont dans ce dépôt. Un plugin
à source externe comme `security-audit` n'est pas téléchargé par les seuls réglages du projet : chacun
l'installe avec `claude plugin install`.

## Mises à jour

- Les plugins ne déclarent pas de `version` : la version est le commit, donc chaque commit poussé sur ce dépôt
  est une mise à jour. Ne pas ajouter de `version` dans `plugin.json` sans la faire évoluer à chaque release,
  sinon les utilisateurs restent bloqués sur l'ancienne copie.
- Les plugins tiers sont épinglés par `sha`. Pour monter de version : relire le diff amont (y compris les
  scripts que le skill fait exécuter), puis changer le `sha` dans `.claude-plugin/marketplace.json`.

## Adapter un skill générique à un projet

Les skills de ce dépôt ne contiennent rien de propre à un projet. Ils lisent le `CLAUDE.md` et `.claude/rules/`
du projet pour les commandes, l'architecture et les conventions. Le skill `review` lit en plus
`.claude/rules/review.md` quand il existe : c'est là qu'un projet décrit ses points de review spécifiques
(identifiants sensibles pour la passe IDOR, commandes de couverture, stratégie de migrations…).

## Ajouter un skill

1. Créer `plugins/<plugin>/skills/<skill>/SKILL.md` (frontmatter `name` + `description`), dans un plugin
   existant ou un nouveau plugin avec son `.claude-plugin/plugin.json`.
2. Pour un nouveau plugin, l'ajouter dans `.claude-plugin/marketplace.json` ; le `name` de l'entrée doit être
   celui de `plugin.json`.
3. Valider : `claude plugin validate .`
