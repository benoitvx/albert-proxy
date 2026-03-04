# CLAUDE.md

## Projet

albert-proxy — Proxy de compatibilité entre clients OpenAI (Vibe CLI, OpenCode) et Albert API (OpenGateLLM, DINUM).

## Stack

- Python 3 / FastAPI / httpx / uvicorn
- Pas de base de données, pas de tests, pas de CI

## Structure

```
proxy.py          # Tout le code (proxy FastAPI, ~210 lignes)
requirements.txt  # fastapi, httpx, uvicorn
```

## Lancer le projet

```bash
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
ALBERT_API_KEY=xxx uvicorn proxy:app --port 4000
```

Mode debug : ajouter `PROXY_DEBUG=1`

## Variables d'environnement

- `ALBERT_API_KEY` (requis) — clé API Albert
- `ALBERT_BASE_URL` — défaut `https://albert.api.etalab.gouv.fr/v1`
- `PROXY_TIMEOUT` — défaut 300s
- `PROXY_DEBUG` — `1` pour activer les logs

## Ce que fait le proxy

1. Intercepte toutes les requêtes HTTP (catch-all `/{path:path}`)
2. Corrige le payload JSON via `fix_payload()` :
   - Force `strict: false` dans chaque tool definition (OpenGateLLM refuse `null`)
   - Supprime les champs non supportés : `parallel_tool_calls`, `stream_options`, `service_tier`, `store`
3. Forward vers Albert API avec le header `Authorization: Bearer`
4. Supporte le **streaming SSE** : si `stream: true` dans le body, les chunks sont forwardés en temps réel via `StreamingResponse` (`_proxy_stream`). Sinon, réponse bufferisée classique (`_proxy_buffered`).

## Conventions

- Langue du code : français (docstrings, commentaires, logs)
- Fichier unique `proxy.py` — pas de découpage en modules sauf si nécessaire
