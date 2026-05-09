# Audit sécurité + tooling — paperasse

- **Fork audité**: https://github.com/gabrielstuff/paperasse
- **Upstream**: https://github.com/romainsimon/paperasse
- **Révision auditée**: `bb54a0386a7eea1cbef754a1457d38f292e184dd`
- **Date**: 2026-05-09
- **Mode**: audit statique + tests offline, sans identifiants réels, sans upload externe

## Verdict

**Utilisable avec restrictions.** Le repo est intéressant comme base de copilote administratif/comptable/fiscal, surtout pour préparer des dossiers, générer des checklists et cadrer les questions à poser à un notaire/fiscaliste/comptable/syndic.

Il ne faut pas l’utiliser comme autorité juridique/fiscale finale, ni l’exécuter avec des secrets ou accès bancaires/paiement sans durcissement préalable.

## Baseline vérifiée

Commandes exécutées localement dans `/tmp/paperasse-fork`:

```bash
npm ci --ignore-scripts
npm audit --package-lock-only --omit=dev --json
npm run test:calc
python3 fiscaliste/scripts/calc_ir.py --rni 45000 --parts 1
python3 -m unittest evals.tests.test_run_evals -v
python3 -m compileall -q evals scripts fiscaliste notaire comptable syndic cac
python3 scripts/update_data.py --check
python3 scripts/test_fetch_notaire_data.py
```

Résultats:

- `npm ci --ignore-scripts`: OK
- `npm run test:calc`: OK
- `python3 -m unittest evals.tests.test_run_evals -v`: OK, 5 tests
- `python3 scripts/update_data.py --check`: OK
- `python3 scripts/test_fetch_notaire_data.py`: OK, 8 réussis, 2 ignorés; MatchID bloqué Cloudflare 403 mais ignoré par le test
- `npm audit`: 2 vulnérabilités connues, 1 high + 1 moderate

## Synthèse priorisée

### P0 — À corriger avant usage avec secrets, CI agentique ou données sensibles

1. **Désactiver Bash dans les evals CI agentiques**
   - `evals/config.yaml` donne `Read,Bash` au skill `comptable`.
   - `.github/workflows/evals-smoke.yml` injecte `ANTHROPIC_API_KEY`.
   - `evals/run_evals.py` passe des `SKILL.md` comme `--system-prompt-file` à Claude.
   - Risque: un `SKILL.md` modifié est du code agentique capable de provoquer des commandes shell et une exfiltration de secret.

2. **Remplacer l’installation CI `curl | bash`**
   - `.github/workflows/evals-smoke.yml` installe Claude CLI via `curl -fsSL https://claude.ai/install.sh | bash`.
   - Risque supply-chain avec secret présent dans le job.
   - Fix: version pinée + checksum, ou image CI préconstruite.

3. **Sanitiser HTML/Markdown/PDF et bloquer le réseau Puppeteer**
   - `scripts/generate-pdfs.js` transforme Markdown en HTML via `marked.parse` puis `page.setContent(... networkidle0)`.
   - `scripts/generate-facturx.js` et `scripts/upload-qonto-attachments.js` interpolent des champs facture/client/société dans HTML.
   - Risques: injection HTML, SSRF/local network probing via `<img src>`, CSS `url()`, iframe, scripts dans PDF, DoS par réseau lent.
   - Fix: échappement systématique, sanitizer/renderer allowlist, CSP, request interception Puppeteer, blocage `http(s)://`, `file://`, `ftp://`, éviter `--no-sandbox` hors container durci.

4. **Ne jamais lancer les scripts Qonto/Stripe avec secrets réels sans confirmation humaine**
   - `scripts/upload-qonto-attachments.js` est dry-run pour l’upload, mais contacte Qonto et récupère des transactions dès que les variables existent.
   - Fix: mode offline par défaut, double confirmation `--upload --confirm-upload-qonto`, allowlist organisation/compte, résumé avant mutation.

### P1 — À corriger pour rendre le fork maintenable

5. **Ajouter une CI standard sans secret**
   - Actuellement, la CI couvre surtout les evals LLM.
   - Ajouter `ci.yml`: `npm ci --ignore-scripts`, tests Node/Python, validation JSON/YAML, lint/format, audit high.

6. **Corriger `npm run fetch`**
   - Actuel: `node integrations/stripe/fetch.js; node integrations/qonto/fetch.js`.
   - Le `;` peut masquer l’échec du premier script si le second réussit.
   - Fix minimal: `&&`. Mieux: orchestrateur `scripts/fetch-all.js` avec rapport agrégé.

7. **Sortir la logique des scripts vers des modules testables**
   - Plusieurs scripts mélangent parsing CLI, I/O, logique métier et `process.exit`.
   - Cibles: `generate-fec`, `generate-statements`, `generate-facturx`, `generate-pdfs`, `validate-facture`, `import-stripe-invoices`.
   - Fix: `lib/` pour fonctions pures, `scripts/` pour wrappers CLI.

8. **Ajouter schemas JSON + validation**
   - `company.json`, factures, journal entries, evals et marketplace ne sont pas validés formellement.
   - Fix: `schemas/*.schema.json` + `ajv` + `npm run validate:data`.

9. **Standardiser money math en centimes entiers**
   - `calc.js` utilise `BigInt` et centimes: bon point.
   - D’autres scripts manipulent `Number`/division par 100.
   - Fix: `lib/money.js` unique avec arrondis déterministes.

10. **Corriger les vulnérabilités npm**
    - `basic-ftp@5.2.0`: High, transitif via `get-uri`.
    - `ip-address@10.1.0`: Moderate, transitif via `socks`.
    - Fix: `npm audit fix` dans branche dédiée, vérifier lockfile et tests.

### P2 — DX, robustesse, nettoyage

11. **Enrichir `package.json`**
    - Ajouter `engines.node >=20`, `license`, `repository`, `bugs`, `homepage`.
    - Ajouter scripts globaux: `test`, `test:node`, `test:python`, `lint`, `format`, `validate:data`, `doctor`, `smoke`.

12. **Ajouter lint/format**
    - `.editorconfig`, Prettier pour JS/JSON/MD/YAML, Ruff pour Python.

13. **Séparer tests unitaires et tests réseau**
    - `scripts/test_fetch_notaire_data.py` dépend d’APIs réelles et subit déjà un 403 Cloudflare sur MatchID.
    - Fix: mocks/fixtures offline par défaut, tests réseau opt-in.

14. **Corriger metadata marketplace**
    - `marketplace.json` version `1.0.0` alors que `package.json` est `1.1.0`.
    - `fiscaliste` est absent de `marketplace.json` alors que le README annonce 6 skills et le dossier existe.

15. **Rendre Puppeteer optionnel**
    - `puppeteer` alourdit l’installation et tire Chromium.
    - Options: `optionalDependencies`, `PUPPETEER_SKIP_DOWNLOAD=true`, `puppeteer-core` + chemin Chrome configurable, ou package PDF séparé.

## Findings sécurité détaillés

### HIGH-1 — Injection agentique via `SKILL.md` + CI Claude + Bash + secret

**Fichiers:**

- `evals/config.yaml`
- `.github/workflows/evals-smoke.yml`
- `evals/run_evals.py`
- `comptable/SKILL.md`

**Pourquoi c’est grave:** les `SKILL.md` ne sont pas de simples docs. Dans ce repo, ils deviennent des prompts système pour un agent Claude. Si l’agent a Bash et un secret en environnement, un changement malveillant dans un skill peut tenter de lire l’environnement, exfiltrer des tokens, modifier des fichiers ou lancer des scripts externes.

**Fix recommandé:**

- CI PR: `tools: Read` uniquement, pas Bash.
- Aucun secret sur contenu prompté non revu.
- `CODEOWNERS` sur `*/SKILL.md`, `evals/config.yaml`, `.github/workflows/*`.
- Wrapper d’exécution avec environnement minimal.
- Scanner statique de prompt-injection sur les skills.

### HIGH-2 — Injection HTML/PDF + SSRF Puppeteer

**Fichiers:**

- `scripts/generate-pdfs.js`
- `scripts/generate-facturx.js`
- `scripts/upload-qonto-attachments.js`

**Pourquoi c’est grave:** Markdown/factures/champs client peuvent devenir du HTML rendu par Chromium. Sans sanitation et sans blocage réseau, un champ contrôlé peut déclencher des requêtes sortantes/locales ou injecter du contenu dans les PDF.

**Fix recommandé:**

- Échapper toutes les variables HTML.
- Désactiver/sanitiser HTML brut Markdown.
- Intercepter et bloquer requêtes Puppeteer.
- CSP stricte.
- Tests avec payloads SSRF/XSS.

### HIGH-3 — Supply-chain CI via `curl | bash`

**Fichier:** `.github/workflows/evals-smoke.yml`

**Fix recommandé:** version pinée + checksum, pas d’installation distante non vérifiée dans un job avec secret.

### MEDIUM-1 — Données Stripe/Qonto `raw` en clair

**Fichiers:**

- `integrations/qonto/fetch.js`
- `integrations/stripe/fetch.js`

**Risque:** stockage local de payloads bancaires/paiement complets, PII, metadata, références. `.gitignore` protège certains dossiers, mais ça reste lisible par tout agent local avec accès fichier.

**Fix:** pas de `raw` par défaut; `--include-raw` opt-in; redaction; permissions `0600`; scanner PII/secrets.

### MEDIUM-2 — XML Factur-X construit par concaténation

**Fichier:** `scripts/generate-facturx.js`

**Risque:** échappement partiel et attributs non validés, notamment unit codes, TVA intracom, SIREN/SIRET.

**Fix:** builder XML, `escapeXmlText`/`escapeXmlAttr`, validation formats, validation XSD/Factur-X en CI.

### LOW-1 — Chemins `--input`/`--output` libres

**Fichiers:** plusieurs scripts sous `scripts/`.

**Risque:** écriture hors workspace en contexte agentique.

**Fix:** `--safe-root`, refuser chemins absolus par défaut, `path.resolve` + vérification de préfixe, `flag: wx` quand applicable.

### LOW-2 — `.env.example` montre `sk_live_...`

**Risque:** incite à démarrer avec clés Stripe production.

**Fix:** utiliser `sk_test_...` ou `<stripe_secret_key>` + avertissement explicite.

## Points positifs

- `.gitignore` exclut `.env`, `company.json`, `data/transactions/`, `output/`, `evals-workspace/`.
- `upload-qonto-attachments.js` n’upload pas par défaut.
- `evals/run_evals.py` limite `load_dotenv` à `ANTHROPIC_API_KEY`.
- `evals/run_evals.py` contient des garde-fous path traversal pour les chemins partagés.
- Les tests déterministes existants passent.
- Les skills métier sont utiles comme checklists et cadres de préparation.

## Operating mode recommandé pour Gabriel

1. Importer sélectivement uniquement ce qui sert aux démarches:
   - `notaire`
   - `fiscaliste`
   - `syndic`
   - éventuellement templates/références lus manuellement

2. Éviter pour l’instant:
   - installation globale de tout le repo comme skills actifs;
   - `comptable` avec Bash;
   - connecteurs Qonto/Stripe;
   - génération PDF/Factur-X avec données non maîtrisées;
   - CI agentique avec secrets.

3. Mode sûr:
   - clone local sandbox;
   - pas de `.env` réel;
   - pas de secrets Cloudflare/Google/GitHub/Qonto/Stripe;
   - lecture et synthèse seulement;
   - validation finale par professionnel pour fiscal/notaire/compta.

## Plan d’action proposé

### Branche 1 — `security-hardening`

- CI evals sans Bash et sans secrets sur contenu non revu.
- Remplacer `curl | bash`.
- Sanitizer HTML/Markdown/PDF.
- Bloquer réseau Puppeteer.
- Double confirmation Qonto upload.
- Supprimer `raw` par défaut pour Qonto/Stripe.

### Branche 2 — `tooling-baseline`

- `ci.yml` standard sans secret.
- `npm run test` global.
- `npm run fetch` corrigé.
- `.editorconfig`, Prettier, Ruff.
- Metadata package/marketplace corrigées.

### Branche 3 — `schemas-and-tests`

- `schemas/company.schema.json`, `invoice.schema.json`, `journal-entry.schema.json`.
- `validate:data`.
- Mocks offline Qonto/Stripe/notaire.
- Tests Factur-X/FEC/PDF basés sur fixtures.

## Conclusion

Le repo peut nous aider, mais surtout comme **référentiel de préparation** et **base à durcir**. La priorité n’est pas d’ajouter des features: c’est de sécuriser les surfaces agentiques, PDF/Puppeteer et connecteurs financiers, puis d’ajouter une CI non-LLM fiable.
