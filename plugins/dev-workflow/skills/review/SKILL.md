---
name: review
description: Review complète d'une branche ou d'un ticket — conformité au besoin (relecture à l'œil neuf), architecture et conventions du projet, sécurité avec passe IDOR, tests et couverture, qualité statique, migrations. À utiliser quand l'utilisateur demande une review, une relecture de MR/PR ou une vérification avant merge.
---

# Objectif

Tu réalises une review exhaustive des modifications d'une branche, comme un reviewer senior exigeant. Tu ne te contentes pas de lire le diff : tu vérifies que le besoin est rempli, que le code respecte l'architecture et les conventions du projet, qu'il est sûr, exhaustivement testé, et que la branche est complète.

C'est une **review** : par défaut tu ne modifies pas le code, tu produis un feedback actionnable. Tu ne proposes des corrections que si l'utilisateur le demande.

# Spécificités du projet

Ce skill est générique. Avant toute chose, lis ce que le projet dit de lui-même :

- `CLAUDE.md` (racine et `.claude/`) et `.claude/rules/` : stack, architecture, conventions, commandes de test et de qualité, environnement d'exécution (Docker, `make`…), gestion des tickets ;
- **`.claude/rules/review.md` s'il existe** : points de review propres au projet (architecture à vérifier, identifiants sensibles pour la passe IDOR, commandes de couverture, stratégie de migrations). Ses consignes complètent et priment sur les étapes ci-dessous.

Toute commande citée ici (tests, couverture, analyse statique, style) se lance **comme le projet l'indique**, jamais avec une commande devinée. Si le projet ne dit pas comment lancer quelque chose, demande-le plutôt que d'improviser.

Détermine la branche de base : celle que donne le projet, sinon `git symbolic-ref refs/remotes/origin/HEAD`. Elle est notée `<base>` ci-dessous.

# Étape 0 — Le besoin

1. Si l'utilisateur ne t'a donné ni ticket ni énoncé du besoin, **demande-le** avant toute chose.
2. **Lis le ticket** là où le projet indique qu'ils vivent (mémoire locale, outil connecté, ou contenu collé par l'utilisateur). Comprends le besoin métier, les pages/routes/endpoints attendus, les règles de gestion et les critères d'acceptation.
3. **Reformule** brièvement ta compréhension du besoin. Si quoi que ce soit est ambigu, incomplet ou contradictoire, **pose des questions à l'utilisateur AVANT de continuer**.

# Étape 0 bis — Œil neuf sur le besoin (obligatoire)

Le piège principal d'une review faite dans la session qui a développé le ticket : tu relis le code en te souvenant de **ce qui a été implémenté**, pas de **ce que le ticket demande**. Les attendus oubliés restent invisibles, et ressortent en retour de review.

Tu dois donc repartir du ticket **sans le contexte du développement**, comme un dev qui connaît le projet mais découvre le ticket.

1. Délègue cette relecture à un **sous-agent à contexte neuf** : `Agent` avec `subagent_type: "general-purpose"` — **jamais** `fork`, qui hériterait de ton contexte de développement.
2. Son prompt ne contient **que** : l'identifiant du ticket, l'endroit où le lire ou, à défaut, son **contenu brut** tel que l'utilisateur l'a fourni (copie littérale, aucune reformulation), la branche à comparer, et la consigne ci-dessous. N'y recopie **aucune** décision, justification, arbitrage ni résumé issus du développement (« on a décidé de… », « X est hors scope », « c'est déjà géré par… », « le ticket voulait dire… »).
   Si le chantier n'a **pas** de ticket (objectif énoncé à l'oral, refacto décidée en session), passe l'**énoncé verbatim de l'utilisateur** et rien d'autre. Ne le résume pas : résumer, c'est déjà réintroduire tes arbitrages.
3. Consigne à donner au sous-agent :
   - Lire le ticket et se documenter sur le projet (`CLAUDE.md`, `.claude/rules/`, code et patterns existants).
   - Produire la **liste exhaustive et atomique des attendus** : routes et méthodes HTTP, écrans et blocs d'UI, règles de gestion, états et transitions autorisés, cas limites nommés, messages et clés de traduction, codes d'erreur, contraintes de sécurité/perf/SEO, impacts annoncés sur l'existant, critères d'acceptation.
   - Pour chaque attendu, dire ce qu'un dev du projet implémenterait (couche, fichier probable, pattern du projet) — **avant de regarder le diff**.
   - Ensuite seulement, confronter cette liste au diff (`git diff <base>...HEAD`) et statuer pour chaque attendu : ✅ implémenté / ⚠️ partiel / ❌ absent / ❓ introuvable, avec `fichier:ligne` quand c'est implémenté, et un test associé ou non.
4. La liste d'attendus rendue par le sous-agent devient la **référence** de l'étape 2. Un attendu absent ou partiel est un point de review, **même si tu « sais » pourquoi il a été écarté** : la seule justification recevable est écrite dans le ticket ou la MR/PR.
5. Si un attendu te semble hors scope, obsolète ou déjà couvert ailleurs, ne l'enterre pas : remonte-le comme point à confirmer avec l'utilisateur.
6. Symétriquement, un attendu que le sous-agent a **inventé** faute de contexte — une règle qui ne figure ni dans le ticket ou l'énoncé, ni dans les conventions du projet — s'écarte explicitement, en disant pourquoi. Il ne se traite pas comme un manque : sinon la review se remplit de faux positifs et l'exercice perd sa crédibilité.
7. **Cas d'une refacto** (pas de ticket, un existant à préserver) : ce que le sous-agent liste, ce sont les garanties du code actuel, et sa liste est une **hypothèse**, pas une référence — contrairement au texte d'un ticket, qui fait foi. Exige que chaque garantie soit **sourcée par `fichier:ligne`** dans le code existant ; une garantie non sourçable se dit « supposée, non vérifiée », jamais comme un fait. Le tri se fait alors en trois cas, pas deux :
   - **sourcée** → référence : la perdre est une régression ;
   - **inventée** (rien dans le code, rien dans les conventions — le réflexe « un client HTTP maison a sûrement des retries ») → écartée explicitement, avec la raison ;
   - **absente du code mais souhaitable** → remontée comme décision à prendre, ni comme manque ni comme faux positif.

# Étape 1 — Périmètre

Détermine les fichiers modifiés (`git diff <base>...HEAD --name-only`) et classe-les selon l'organisation du projet : code de production par couche, tests, migrations, configuration, traductions, assets. C'est ce périmètre qui sert de référence pour la couverture et les vérifications.

# Étape 2 — Conformité fonctionnelle au besoin

- Reprends **un par un** les attendus listés à l'étape 0 bis et vérifie leur statut sur le code réel. Tout ⚠️ / ❌ / ❓ est un point de review.
- Le code répond-il **exactement** au besoin ? Rien en trop, rien en moins.
- Le métier ajouté **s'insère-t-il correctement dans l'existant** (réutilisation, cohérence avec les patterns déjà en place) ?
- Cherche les **effets de bord non désirés ou non testés** : autres consommateurs des services, handlers ou entités modifiés, événements dispatchés, écrans d'administration, machines à états, groupes de sérialisation, impacts sur d'autres routes ou composants.

# Étape 3 — Architecture & coding style

- Vérifie le respect de l'architecture **telle que le projet la décrit** (`CLAUDE.md`, `.claude/rules/`). Ne réclame pas un pattern que le projet réserve à des cas précis, et signale un contournement d'un pattern que le projet impose.
- Conventions du projet : langue du code et des commentaires, nommage, règles d'analyse statique, structure des dossiers.
- Lisibilité, duplication, complexité inutile.

## Commentaires

Un commentaire décrit **ce que fait le code** quand les noms et la structure ne le disent pas, ou nomme une **contrainte invisible localement** : une bizarrerie d'une lib tierce, un invariant que l'appelant doit respecter, la raison d'être d'un contrôle là où un plus simple semblerait suffire. Il ne justifie **jamais un choix de conception** — pourquoi ce motif plutôt qu'un autre, pourquoi la validation est ici et pas là, pourquoi ce type. Cette justification relève d'une décision écrite (ADR, `docs/`, fichier de règles), pas d'un commentaire répété à chaque point d'appel, où il devient faux dès que la décision change et que personne ne met à jour les douze commentaires qui l'expliquaient.

Le test : si je supprime ce commentaire, un lecteur futur ignore-t-il un **fait** qu'il ne peut pas voir autrement — ou lui manque-t-il seulement un **argument** en faveur de ce qu'on a fait ? Dans le second cas, il doit partir.

Ne pas laisser passer un commentaire qui :

* **Redit ce que la clause de garde dit déjà** dans son message d'exception.
* **Explique pourquoi un motif a été choisi** : des contrôles champ par champ plutôt qu'un contrôle groupé, une structure reconstruite plutôt que réutilisée. Si toutes les classes d'une même famille suivent la convention, un seul commentaire sur la convention — ou une règle dans `.claude/rules/` — les couvre toutes.
* **Raconte la mécanique d'une fonction du langage** quand le code en dessous se lit tout seul pour qui connaît les conventions du dépôt.

En revanche, **garder** le commentaire qui signale quelque chose de vraiment non évident **à cet endroit précis** : un type d'exception d'une lib tierce, une valeur qui franchit une frontière sous une forme surprenante, le contournement d'un bug identifié. Une justification copiée-collée d'une classe sœur à l'autre est le signal qu'il faut la couper.

**En review** : un commentaire qui justifie un choix de conception est un point à remonter (🟡), avec la suggestion de le supprimer — ou de l'écrire une seule fois, dans `docs/` ou dans une règle.

# Étape 4 — Sécurité

Recherche les failles : autorisation/authentification manquante ou incohérente, injection (SQL, requêtes ORM, templates, shell), exposition de données sensibles (sérialisation, templates, logs), validation d'entrée insuffisante, secrets en dur, contournement de règles métier ou de prix.

## Étape 4 bis — IDOR : passe obligatoire, une ligne par identifiant

L'IDOR ne se trouve pas en relisant du code « avec la sécurité en tête » — il se trouve en **énumérant**. Le mentionner dans une liste de vigilance ne suffit pas.

Pour **chaque** route, contrôleur, endpoint API, composant interactif et action du diff, dresser un tableau : `identifiant accepté` | `d'où il vient` | `le contrôle qui le rattache à l'utilisateur courant` | `fichier:ligne de ce contrôle`. Une case vide dans la dernière colonne est un bloquant, et se dit à voix haute — « aucun contrôle » est une conclusion valide, pas un oubli à passer sous silence.

Ce qui **ne compte pas** comme contrôle d'appartenance :
- être authentifié — ça prouve qu'on est *un* utilisateur, pas *le* propriétaire ;
- une contrainte de format sur la route (`\d+`), qui n'empêche que les valeurs absurdes ;
- l'obscurité de l'identifiant : un code à 6 chiffres se force, un id séquentiel se parcourt ;
- le fait que le front n'envoie jamais une mauvaise valeur — l'URL et le corps de requête sont à l'utilisateur.

Points d'attention génériques :
- un identifiant venant d'un **système externe** n'est pas protégé par ce système : vérifier la chaîne d'appartenance de bout en bout ;
- un identifiant qui **n'a pas besoin** de voyager ne doit pas voyager : si l'étape précédente est dans la même session, le garder côté serveur plutôt que dans l'URL ; pour un lien envoyé par e-mail, un jeton opaque plutôt que l'id ;
- attention aussi à la **sortie** : une réponse construite depuis une entité renvoie tout ce que sa sérialisation expose. Un endpoint public qui répond à partir d'une donnée fournie par le visiteur (un e-mail…) peut renvoyer les données personnelles d'un tiers.

Les identifiants sensibles propres au projet sont listés dans `.claude/rules/review.md` quand il existe.

# Étape 5 — Tests unitaires & couverture

- Chaque fichier de production modifié doit être couvert par des tests unitaires, selon l'organisation des tests du projet.
- **Objectif : 100 % de coverage sur les fichiers modifiés.** Lance la couverture avec la commande du projet, puis croise le rapport avec la liste des fichiers modifiés de l'étape 1.
- Signale précisément toute condition, branche, ou exception non testée.
- Respecte les conventions de test du projet (fixtures, vraies entités vs mocks, nommage).

# Étape 6 — Tests des parcours

Pour **chaque parcours demandé par le besoin** (route, contrôleur, composant, endpoint API, commande) :

- **Happy paths** testés.
- **Edge cases** testés : bornes, valeurs nulles/optionnelles, collections vides, états interdits, etc.
- **Erreurs** testées : chaque cas d'erreur a un test qui vérifie **le résultat attendu** (code HTTP + corps pour une API ; rendu, redirection ou message pour une page).
- Vérifie que la sortie correspond **exactement** à l'attendu : status, payload/markup, sérialisation, format.

# Étape 7 — Qualité statique

Lance les outils que le projet définit (analyse statique, coding style, refactoring automatique en dry-run, détection de code de debug oublié) et attends **0 erreur / aucun changement**. Si le style casse, propose la commande de correction du projet plutôt que de corriger à la main.

# Étape 8 — Migrations & schéma

Si le projet a des migrations de base de données :

- Si une entité, un mapping ou un schéma a changé, la branche **DOIT** contenir la migration correspondante (ou être cohérente avec la stratégie de migrations décrite par le projet — vérifie le contexte avant de bloquer).
- Vérifie avec l'outil du projet qu'aucun changement de schéma n'est en attente et que le mapping est valide.
- Si une migration existe : vérifie `up`/`down`, qu'elle ne contient **que** les changements pertinents (pas de drift non lié), et qu'elle est cohérente avec le mapping.

# Format de sortie

- D'abord la **table de couverture du besoin** (issue de l'étape 0 bis) : `attendu` | `statut` | `fichier:ligne` | `testé ?`.
- Puis une **liste** d'observations groupées par sévérité : 🔴 Bloquant / 🟠 Majeur / 🟡 Mineur / 💡 Suggestion.
- Pour chaque point : `chemin/fichier:ligne` + description concise + action recommandée.
- Termine par un **verdict** + checklist :
  - [ ] Relecture du besoin à l'œil neuf faite (sous-agent à contexte vierge)
  - [ ] Tous les attendus couverts (aucun ⚠️ / ❌ / ❓ non justifié par écrit)
  - [ ] Besoin rempli
  - [ ] Architecture et conventions du projet respectées
  - [ ] Pas d'effet de bord non testé
  - [ ] Pas de faille de sécurité
  - [ ] Passe IDOR faite : un identifiant énuméré par ligne, chacun avec le `fichier:ligne` du contrôle qui le rattache à l'utilisateur courant
  - [ ] 100 % coverage sur les fichiers modifiés
  - [ ] Chaque parcours testé (happy + edge + erreurs avec résultat vérifié)
  - [ ] Qualité statique OK
  - [ ] Migrations présentes / cohérentes si nécessaire
- Verdict final : ✅ prêt à merger / ⚠️ corrections nécessaires.

# Agents

Pour la collecte (lecture de fichiers, grep, lancement des tests et des outils de qualité), parallélise via des sous-agents légers (`model: "haiku"`) puis synthétise toi-même.
