# Lab 30 — Notes CI

## Exécutions de référence

| Exécution | Commit | Résultat | Lien |
|---|---|---|---|
| Baseline | `e87cdfd` Add full CI pipeline | ✅ succès | https://github.com/NasaEnder/res507-2025-2026-work/actions/runs/38217900415 |
| Échec volontaire | `a007062` Introduce intentional failure | ❌ échec | https://github.com/NasaEnder/res507-2025-2026-work/actions/runs/38218409783 |
| Réparation | `515ede7` Fix intentional failure | ✅ succès | https://github.com/NasaEnder/res507-2025-2026-work/actions/runs/38220524155 |

## Diagnostic de l'échec volontaire

**Étape en échec :** `Run container smoke test`.
Les étapes précédentes (installation, vérification syntaxe, build Docker) passent.
L'étape `Show container diagnostics` s'exécute ensuite (`if: failure()`) et affiche
l'état et les logs du conteneur, puis `Stop container` nettoie (`if: always()`).

**Cause :** dans `.github/workflows/ci.yml`, l'URL du health check a été changée de
`/health` en `/healthz`. L'application ne définit pas cette route : Fastify répond
`404`, `curl --fail` renvoie une erreur à chaque tentative, et après 30 essais le
script affiche `Health check failed` puis `exit 1`.
Le conteneur, lui, fonctionnait : les logs du conteneur montrent
`Route GET:/healthz not found` avec `statusCode: 404`.

**Correction :** remettre l'URL `http://localhost:3000/health`. Cette route existe
(`app.get("/health", ...)` dans `app/app.js`) et renvoie `{"ok": true}`, ce que
vérifie l'assertion `node -e ...`. La vérification n'a pas été supprimée ni
contournée : c'est le chemin testé qui était faux.

## Ce que le CI vert prouve (et ne prouve pas)

Un run vert prouve, pour ce commit et sur ce runner :

- les dépendances s'installent depuis `package-lock.json` (`npm ci`) ;
- `app.js` et `server.js` sont syntaxiquement valides (`node --check`) ;
- l'image Docker se construit ;
- le conteneur démarre et répond à `/health` avec `{"ok": true}`.

Il ne prouve **pas** que la base de données, la lecture/insertion des citations,
la persistance, la sécurité, les performances ou le déploiement en production fonctionnent.

**Exemple de panne non détectée :** une faute dans la requête SQL de `GET /`
(par ex. `SELECT * FROM quote` au lieu de `quotes`). `node --check` ne vérifie que
la syntaxe JavaScript, pas le SQL, et le smoke test tourne **sans PostgreSQL** :
`/health` n'interroge pas la base et répond toujours `{"ok": true}`. Le CI reste vert
alors que la page d'accueil renvoie une erreur 500 en conditions réelles.

Enfin, un CI vert est un signal, pas un blocage : sans règle de protection de
branche, rien n'empêche de fusionner un commit dont le CI est rouge.
