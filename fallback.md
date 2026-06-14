# Arquitetura de Fallback e Provedores do Hermes Agent

Este documento serve como um guia para entender como o Hermes Agent gerencia provedores de IA, autenticação e recuperação de erros. O objetivo é facilitar a criação, modificação e entendimento de rotinas de fallback para novos modelos ou provedores.

---

## 1. Como os Provedores são Registrados

O Hermes usa um sistema de múltiplas fontes para identificar provedores, consolidado em `hermes_cli/providers.py`.

### A Cadeia de Resolução (`resolve_provider_full`)
Quando o agente precisa resolver um nome de provedor (ex: `google-antigravity`, `openai`, `anthropic`), ele segue esta ordem de prioridade:

1.  **Configuração do Usuário (`config.yaml`)**: Provedores definidos manualmente pelo usuário no bloco `providers:` ou `custom_providers:`.
2.  **Catálogo `models.dev` + Overlays do Hermes**: O Hermes consulta um banco de dados interno de provedores (`agent.models_dev`). Como esse catálogo não tem detalhes de transporte ou fluxos OAuth específicos do Hermes, ele aplica **Overlays** (`HERMES_OVERLAYS` em `providers.py`).
3.  **Plugins (Registry Fallback)**: Se o provedor não estiver no `models.dev` (ex: plugins externos customizados como o `google-antigravity`), ele cai em um fallback que busca perfis injetados dinamicamente via plugins.

### Adicionando um Novo Provedor
Se o provedor for puramente API Key e compatível com OpenAI, muitas vezes basta configurá-lo no `config.yaml`.
Se o provedor precisar de um fluxo OAuth próprio ou endpoints especiais:
*   Registre a base no `HERMES_OVERLAYS` em `hermes_cli/providers.py` (informando o `transport` e `auth_type`).
*   Registre a configuração de autenticação (Client ID, Scope, etc.) no `PROVIDER_REGISTRY` em `hermes_cli/auth.py`.

---

## 2. Modos de Transporte (`api_mode`)

A forma como o Hermes se comunica com o provedor é definida pelo `api_mode` (ou `transport`). O `chat_completion_helpers.py` (na função `try_activate_fallback` e durante a inicialização) mapeia o provedor/modelo para o transporte correto:

*   **`openai_chat`**: O padrão da indústria (`/v1/chat/completions`). Usado pela maioria dos provedores (OpenRouter, DeepSeek, XAI, etc.).
*   **`anthropic_messages`**: Protocolo nativo da Anthropic. Se o fallback for para a Anthropic, o Hermes reconstrói o cliente usando o SDK nativo da Anthropic (`build_anthropic_client`).
*   **`codex_responses`**: API de respostas diretas da OpenAI (usada pelo ChatGPT backend e GitHub Copilot).
*   **`bedrock_converse`**: Protocolo específico da AWS.

**Como modificar:** Se você adicionar um provedor que diz ser compatível com OpenAI mas na verdade usa o formato da Anthropic, você deve garantir que o `HERMES_OVERLAYS` dele defina `transport="anthropic_messages"`.

---

## 3. As Três Camadas de Recuperação (Fallback & Rotation)

Quando uma chamada de API falha, o Hermes passa a requisição por um classificador de erros (`agent/error_classifier.py`) e tenta se recuperar usando três camadas diferentes:

### Camada A: Rotação de Pool de Credenciais (`agent_runtime_helpers.py`)
**Escopo**: *Mesmo Provedor, Mesma Conta/Outra Chave*.
Se um provedor falha com erros específicos, a função `recover_with_credential_pool` tenta salvar a chamada rodando a fila de credenciais disponíveis para *aquele provedor*.

*   **Rate Limit (429)**: Tenta de novo com a mesma chave. Se falhar de novo, rotaciona para a próxima chave do pool.
*   **Billing (402/Sem créditos)**: Descarta a chave imediatamente e rotaciona para a próxima.
*   **Auth (401/403)**: Tenta fazer o refresh do token (se for OAuth). Se o refresh falhar, marca como exaurido e rotaciona.

> **Importante**: Esta camada tem uma trava de segurança. Ela **não** atua se o provedor do erro for diferente do provedor primário do pool (para evitar vazar chaves entre provedores diferentes num fallback).

### Camada B: Fallback de Conta (Específico do Codex / `codex_runtime.py`)
**Escopo**: *Contas Múltiplas do mesmo tipo*.
Exclusivo para `openai-codex` (e potencialmente outros usando fluxo similar). Se o stream falha no meio com 401, o `run_codex_stream` itera sobre todas as contas salvas no `codex_oauth.json` invisivelmente. Se uma funcionar, ele **promove** o token vencedor para o runtime do agente (`agent.api_key`), para que as próximas chamadas não usem o token velho.

### Camada C: Fallback de Provedor/Modelo (`chat_completion_helpers.py`)
**Escopo**: *Troca completa de modelo/serviço*.
Se o erro for um 5xx, ou se as credenciais do provedor atual acabarem (ou se for um erro de "Modelo Indisponível"), o agente engatilha o `agent._try_activate_fallback()`.

1.  Ele lê a `fallback_chain` (definida no `config.yaml` ou via flag).
2.  Desconecta o cliente atual (e destroi o pool de credenciais se o provedor for mudar).
3.  Usa `resolve_provider_client()` para criar um novo cliente (ex: Troca o cliente nativo da Anthropic por um cliente HTTPX padrão apontando pro OpenRouter).
4.  Ajusta o `agent.api_mode` para o novo provedor.
5.  O loop principal (`conversation_loop.py`) tenta a chamada novamente do zero.

---

## 4. Guia Prático: Como criar um novo Fallback para outro Modelo

Se você quiser criar um modelo de fallback, você tem duas opções:

### Opção 1: Via Configuração do Usuário (Recomendado)
A forma mais segura é definir no `~/.hermes/config.yaml`:

```yaml
# Define o comportamento padrão
model: "claude-3-opus-20240229"
provider: "anthropic"

# Define a cadeia de fallback
fallback_chain:
  - provider: "openrouter"
    model: "anthropic/claude-3-opus"
  - provider: "google-antigravity"
    model: "gemini-1.5-pro-latest"
```
Neste cenário, se a API nativa da Anthropic cair, o Hermes:
1. Chama o `try_activate_fallback`.
2. Vê que o próximo é OpenRouter. Destrói o cliente Anthropic e cria um cliente OpenAI apontando para a base URL do OpenRouter.
3. Se o OpenRouter também falhar, ele vai para o `google-antigravity`.

### Opção 2: Adicionando nativamente ao Código (Hardcoded)
Se você está criando um provedor do zero no Hermes (como foi feito com o Antigravity):

1. **Defina o Overlay em `providers.py`**:
   ```python
   HERMES_OVERLAYS["meu-novo-provedor"] = HermesOverlay(
       transport="openai_chat", # ou anthropic_messages
       auth_type="oauth_external", 
       base_url_override="https://api.meuprovedor.com/v1"
   )
   ```

2. **Garanta a Resolução de Autenticação (`auth.py`)**:
   Certifique-se de que se o usuário rodar `hermes auth add meu-novo-provedor`, o sistema sabe se vai pedir um Token por prompt (API Key) ou se vai abrir o navegador (OAuth).
   Se for OAuth de Plugin, o bloco que escrevemos no `auth.py` (linhas 478+) faz o registro automático lendo do manifesto do plugin.

3. **Injetando Multi-Contas (Opcional)**:
   Se o seu provedor suporta logar com 5 contas diferentes do Google ao mesmo tempo, faça um script no `providers.py` que lê as contas e cria slugs como `meu-novo-provedor-conta1`, `meu-novo-provedor-conta2`, exatamente como foi feito pro Antigravity (`google-antigravity-user-at-gmail-com`).

---

## Dicas de Debugging

Se o fallback não estiver disparando quando deveria:
1. Verifique `agent/error_classifier.py`: O erro HTTP retornado pelo provedor não mapeou para uma categoria "retryable" ou "fallback_eligible".
2. Verifique o log com `-vvv`: Procure pela frase `"Fallback to [provedor] failed: provider not configured"`. Isso significa que o `resolve_provider_client()` não achou uma API Key ou um Token válido para o provedor destino.
3. Verifique a Trava de Sessão do Anthropic: Se o fallback estiver indo para a Anthropic, mas falhando, pode ser porque faltam os cabeçalhos (`default_headers`), que o `try_activate_fallback` tenta preservar.
