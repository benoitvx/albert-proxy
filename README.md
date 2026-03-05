# albert-proxy

Proxy de compatibilité pour connecter des outils de vibe coding ([Mistral Vibe CLI](https://github.com/mistralai/mistral-vibe), [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [OpenCode](https://opencode.ai), etc.) à [Albert API](https://albert.api.etalab.gouv.fr), l'API IA interministérielle opérée par la DINUM.

## Pourquoi ce proxy ?

Albert API est basée sur [OpenGateLLM](https://github.com/etalab-ia/OpenGateLLM), qui expose une API compatible OpenAI. En pratique, certains clients envoient des champs non supportés par OpenGateLLM :

- **`strict: null`** dans les tool definitions — OpenGateLLM attend un booléen, pas `null` → erreur 422
- **`parallel_tool_calls`**, **`stream_options`**, **`service_tier`**, **`store`** — champs non reconnus
- **`tool_call_id`** au mauvais format — Mistral exige exactement 9 caractères alphanumériques
- **Noms de modèles** courants (`gpt-4o`, `mistral-small`) qui ne correspondent pas aux IDs Albert

Le proxy intercepte les requêtes, corrige ces incompatibilités, et les transmet à Albert de façon transparente.

## Ce que corrige le proxy

| Problème | Source | Fix |
|----------|--------|-----|
| `strict: null` dans tools | SDK OpenAI | Force `strict: false` |
| `parallel_tool_calls` | SDK OpenAI | Supprime le champ |
| `stream_options` | SDK OpenAI | Supprime le champ |
| `service_tier`, `store` | SDK OpenAI | Supprime le champ |
| `tool_call_id` trop long | SDK OpenAI | Réécrit en 9 chars (MD5) |
| `tool_calls: []` vide | SDK OpenAI | Supprime la clé |
| Noms de modèles courants | Alias pratiques | Remap vers IDs Albert |
| Qwen3 `enable_thinking` | Reasoning vide | Désactivé automatiquement |

### Alias de modèles

| Alias | Modèle Albert |
|-------|---------------|
| `gpt-4o`, `gpt-4` | `openai/gpt-oss-120b` |
| `gpt-4o-mini` | `mistralai/Mistral-Small-3.2-24B-Instruct-2506` |
| `gpt-3.5-turbo` | `mistralai/Ministral-3-8B-Instruct-2512` |
| `mistral-small` | `mistralai/Mistral-Small-3.2-24B-Instruct-2506` |
| `mistral-medium` | `mistral-medium-2508` |
| `qwen-coder` | `Qwen/Qwen3-Coder-30B-A3B-Instruct` |

## Installation

### Avec Python

```bash
git clone https://github.com/benoitvx/albert-proxy.git
cd albert-proxy
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Avec Docker

```bash
git clone https://github.com/benoitvx/albert-proxy.git
cd albert-proxy
docker compose up -d
```

## Configuration

Copier le fichier d'exemple et renseigner votre clé Albert API :

```bash
cp .env.example .env
# Éditer .env avec votre clé
```

### Variables d'environnement

| Variable | Requis | Défaut | Description |
|----------|--------|--------|-------------|
| `ALBERT_API_KEY` | **oui** | — | Clé API Albert |
| `ALBERT_BASE_URL` | non | `https://albert.api.etalab.gouv.fr/v1` | URL de base Albert |
| `PROXY_TIMEOUT` | non | `300` | Timeout en secondes |
| `PROXY_DEBUG` | non | `0` | Mode debug (`1` pour activer) |

## Lancer le proxy

### Avec Python

```bash
source venv/bin/activate
uvicorn proxy:app --port 4000
```

### Avec Docker Compose

```bash
docker compose up -d
```

Le proxy écoute sur `http://localhost:4000`.

En mode debug (logs détaillés sur stderr) :

```bash
PROXY_DEBUG=1 uvicorn proxy:app --port 4000
```

## Configurer les clients

### Mistral Vibe CLI

Dans `~/.vibe/config.toml` :

```toml
[[providers]]
name = "albert"
api_base = "http://localhost:4000"
api_key_env_var = "ALBERT_API_KEY"
api_style = "openai"
backend = "generic"

[[models]]
name = "openai/gpt-oss-120b"
provider = "albert"
alias = "gpt-oss"
temperature = 0.2
input_price = 0.0
output_price = 0.0

[[models]]
name = "mistral-medium-2508"
provider = "albert"
alias = "mistral-medium"
temperature = 0.2
input_price = 0.0
output_price = 0.0

[[models]]
name = "Qwen/Qwen3-Coder-30B-A3B-Instruct"
provider = "albert"
alias = "qwen3-coder"
temperature = 0.2
input_price = 0.0
output_price = 0.0
```

Puis sélectionner un modèle :

```toml
active_model = "gpt-oss"
```

### OpenCode

Dans `~/.config/opencode/opencode.json` :

```json
{
  "provider": {
    "albert": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Albert API",
      "options": {
        "baseURL": "http://localhost:4000",
        "apiKey": "{env:ALBERT_API_KEY}"
      },
      "models": {
        "openai/gpt-oss-120b": {
          "name": "GPT OSS 120B",
          "limit": { "context": 128000, "output": 16384 }
        }
      }
    }
  },
  "model": "albert/openai/gpt-oss-120b"
}
```

### Claude Code

```bash
ANTHROPIC_BASE_URL=http://localhost:4000 \
ANTHROPIC_API_KEY=$ALBERT_API_KEY \
claude --model gpt-4o
```

## Modèles disponibles sur Albert

```bash
curl -s -H "Authorization: Bearer $ALBERT_API_KEY" \
  https://albert.api.etalab.gouv.fr/v1/models \
  | jq '.data[] | select(.type == "text-generation") | .id'
```

## Architecture

```
Client (Vibe / OpenCode / Claude Code)
        │
        ▼
  albert-proxy (:4000)
   ├ fix strict: null → false
   ├ strip unsupported fields
   ├ remap model aliases
   ├ rewrite tool_call_ids
   └ stream SSE ou buffered
        │
        ▼
  Albert API (OpenGateLLM)
  albert.api.etalab.gouv.fr
```

Le proxy est un simple pass-through : il intercepte toutes les requêtes (`/{path:path}`), corrige le payload JSON via `fix_payload()`, et forward vers Albert avec le header `Authorization: Bearer`. Les réponses streaming (SSE) sont relayées chunk par chunk.

## Contribuer

Les PR sont bienvenues. Si vous rencontrez de nouvelles incompatibilités, ouvrez une issue avec les logs du mode debug (`PROXY_DEBUG=1`).

## Contexte

Développé dans le cadre de la mission EIG (Entrepreneurs d'Intérêt Général) à la DINUM, département IAE (Intelligence Artificielle dans l'État), pour faciliter l'utilisation des outils de vibe coding avec l'infrastructure IA souveraine.

## Licence

MIT — voir [LICENSE](LICENSE)
