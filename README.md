# # AI Gateway — pacote

Dois arquivos independentes com o mesmo núcleo (provedores, roteamento, debate, barramento de mensagens). Nenhum precisa de Omniroute ou de `npm install`.

| Arquivo | Para quê | Onde roda |
|---|---|---|
| `ai-gateway.mcpb` | Extensão para o **Claude Desktop** (transporte stdio) | Local, no seu computador |
| `ai-gateway-http.js` | **Servidor HTTP** MCP (um arquivo, só Node ≥ 18) | Sua máquina, VPS ou PaaS; serve para o **claude.ai web** e outros clientes MCP |

## 1. Claude Desktop (`ai-gateway.mcpb`)

Dê dois cliques no arquivo (ou *Settings → Extensions → Advanced → Install Extension*) e preencha ao menos uma chave. As chaves ficam no cofre do sistema. Depois peça: "use ai_providers".

## 2. Servidor HTTP (`ai-gateway-http.js`)

```bash
export AIGW_HTTP_TOKEN=$(node -e "console.log(require('crypto').randomBytes(24).toString('hex'))")
export OPENAI_API_KEY=sk-...        # e/ou ANTHROPIC_API_KEY, GEMINI_API_KEY, OPENROUTER_API_KEY, ...
node ai-gateway-http.js             # http://127.0.0.1:8787/mcp
```

- Endpoint: `POST /mcp` com `Authorization: Bearer <token>`, ou `POST /mcp/<token>` (para clientes que só aceitam URL, como o conector personalizado do claude.ai). `GET /health` não exige token.
- **Claude web:** hospede com **HTTPS público** (VPS atrás de Caddy/nginx, Fly.io, Railway, Render…) e cadastre `https://seu-host/mcp/<token>` como conector personalizado. Em PaaS a variável `PORT` é respeitada e o servidor escuta em `0.0.0.0`.
- Sem `AIGW_HTTP_TOKEN` o servidor só sobe em localhost. Fora dele, recusa iniciar. O token precisa ter ≥ 16 caracteres. Quem tiver o token gasta suas chaves; guarde-o como uma senha e, no formato `/mcp/<token>`, lembre que a URL pode aparecer em logs de proxy.
- Não há limite de requisições por cliente; em exposição pública, limite no proxy.
- Testado com cliente HTTP local, **não** contra o claude.ai.

Variáveis: `AIGW_HTTP_TOKEN`, `PORT` / `AIGW_HTTP_PORT` (padrão 8787), `AIGW_HTTP_HOST`, `AIGW_TIMEOUT_SECONDS` (5–600, padrão 60), `AIGW_DATA_DIR` (barramento; padrão `~/.ai-gateway`), `MODEL_OVERRIDES` (JSON, ex.: `{"openai":"gpt-4.1-mini"}`), chaves: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `OPENROUTER_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `MISTRAL_API_KEY`, `OLLAMA_BASE_URL`, `CUSTOM_BASE_URL` + `CUSTOM_API_KEY` (qualquer API compatível com OpenAI).

## Ferramentas (as mesmas nos dois)

`ai_providers`, `ai_models`, `ai_chat` (`strategy`: fallback · round_robin · race), `ai_broadcast`, `ai_debate` (1–3 rodadas, juiz opcional), `bus_post`, `bus_read`, `bus_channels`, `bus_ask`, `bus_clear`.

Alvos: `"provider"` ou `"provider/model"` (ex.: `anthropic`, `openai/gpt-4o-mini`, `openrouter/meta-llama/llama-3.3-70b-instruct`). Os modelos padrão são sugestões e envelhecem; use `ai_models` ou `MODEL_OVERRIDES`.

## Segurança

Chaves são removidas das mensagens de erro. Respostas de outros modelos e mensagens do barramento são dados não confiáveis (risco de prompt injection). O barramento grava em `AIGW_DATA_DIR/bus.json` (permissão 600; no servidor HTTP, todos os clientes compartilham os mesmos canais). `ai_debate` custa rodadas × modelos chamadas.
