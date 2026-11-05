# 🌤️ Chatbot de Clima no Telegram com N8N

Chatbot do Telegram, feito com [N8N](https://n8n.io), que informa a **temperatura atual de qualquer cidade do Brasil**.
O usuário envia o nome da cidade (e opcionalmente o estado). O workflow consulta a API gratuita da
[OpenWeather](https://openweathermap.org/current) e responde com uma mensagem curta e amigável.

```
Usuário:  Belo Horizonte, MG
Bot:      🌤️ A temperatura em Belo Horizonte é de 25°C.
```

---

## 🧩 Como o workflow funciona

Arquivo: `workflow-chatbot-telegram.json`

| # | Nó | Tipo | O que faz |
|---|----|------|-----------|
| 1 | **Telegram Trigger** | Telegram Trigger | Recebe as mensagens de texto enviadas ao bot. |
| 2 | **Formatar entrada** | Set | Guarda o texto na variável `queue` já normalizado: remove espaços extras, acentos e maiúsculas, e garante o formato `cidade,uf,br`. Também guarda o `chatId`. |
| 3 | **Consultar OpenWeather** | HTTP Request | `GET https://api.openweathermap.org/data/2.5/weather` com *Send Query Parameters*: `q` = `{{ $json.queue }}`, `units=metric`, `lang=pt_br`, `appid` = `{{ $env.OPENWEATHER_API_KEY }}`. |
| 4 | **Resposta válida?** | IF | Verifica se o status HTTP é `200` e se a resposta contém `main.temp` e `name`. |
| 5 | **Formatar resposta** | Code | Extrai a temperatura, arredonda (`Math.round`) e monta a mensagem final. |
| 6 | **Enviar temperatura** | Telegram (Send Message) | Envia `🌤️ A temperatura em <Cidade> é de <N>°C.` |
| 7 | **Enviar erro** | Telegram (Send Message) | Se a cidade não for encontrada ou a resposta for inválida, envia `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

```
Telegram Trigger → Formatar entrada → Consultar OpenWeather → Resposta válida?
                                                               ├─ true  → Formatar resposta → Enviar temperatura
                                                               └─ false → Enviar erro
```

**Exemplos de normalização da entrada (`queue`):**

| Mensagem recebida | `queue` enviado à API |
|-------------------|-----------------------|
| `São Paulo,SP,BR` | `sao paulo,sp,br` |
| `  Belo   Horizonte , MG ` | `belo horizonte,mg,br` |
| `Teresina - PI` | `teresina,pi,br` |
| `Parnaíba` | `parnaiba,br` |

> **Sobre `queue` × `q`:** o texto formatado fica na variável `queue`, como pede o desafio. Na chamada HTTP ele é
> enviado no parâmetro **`q`**, que é o nome que a OpenWeather reconhece para a cidade. Um parâmetro chamado `queue`
> seria ignorado pela API.

> **Tratamento de erro:** o nó HTTP Request está com as opções **Full Response** e **Never Error** ativadas. Assim, um
> `404` (cidade inexistente) não interrompe o workflow e chega ao IF, que direciona para a mensagem de erro.

---

## ✅ Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) e Docker Compose
- Um bot no Telegram e o token gerado pelo **@BotFather** (`/newbot`)
- Uma API Key da OpenWeather: crie uma conta em <https://home.openweathermap.org/users/sign_up> e copie a chave em
  <https://home.openweathermap.org/api_keys>. Chaves novas podem levar alguns minutos (até ~2 h) para ativar.
- Uma URL pública **HTTPS** apontando para o N8N local. O Telegram só entrega mensagens a webhooks HTTPS públicos.
  Este guia usa o [ngrok](https://ngrok.com/), que oferece um domínio estático gratuito.

---

## 🔐 Variáveis esperadas

| Variável | Onde é usada |
|----------|--------------|
| `OPENWEATHER_API_KEY` | Variável de ambiente do container do N8N, lida no nó HTTP Request via `{{ $env.OPENWEATHER_API_KEY }}`. |
| `TELEGRAM_BOT_TOKEN` | Token do @BotFather. É cadastrado na credencial **Telegram API** dentro do N8N (passo 4). |
| `WEBHOOK_URL` | URL pública HTTPS do N8N (ex.: domínio do ngrok), usada para registrar o webhook do Telegram. |

O arquivo `.env.example` serve de modelo. **Nenhum token real está neste repositório**: o `.env` está no `.gitignore`
e o JSON do workflow não tem credenciais embutidas.

---

## 🚀 Passo a passo

### 1. Clonar o repositório

```bash
git clone https://github.com/<seu-usuario>/chatbot-telegram.git
cd chatbot-telegram
```

### 2. Expor a porta 5678 com o ngrok

```bash
ngrok http 5678 --url=seu-subdominio.ngrok-free.app
```

Mantenha esse terminal aberto. Sem domínio estático, use apenas `ngrok http 5678` e copie a URL `https://...` exibida.
Nesse caso, ela muda a cada execução.

### 3. Configurar o `.env` e subir o N8N

```bash
cp .env.example .env
# edite o .env e preencha OPENWEATHER_API_KEY, TELEGRAM_BOT_TOKEN e WEBHOOK_URL
docker compose up -d
```

O `docker-compose.yml` já define `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. As versões recentes do N8N bloqueiam o
`$env` em expressões por padrão, e sem essa variável o `appid` chegaria vazio.

Acesse o N8N em <http://localhost:5678> (ou pela URL do ngrok) e crie a conta de administrador.

### 4. Importar o workflow e cadastrar a credencial do Telegram

1. No N8N, clique em **Create Workflow** → menu **⋯** (canto superior direito) → **Import from File…**
2. Selecione `workflow-chatbot-telegram.json`.
3. Abra o nó **Telegram Trigger** → em *Credential to connect with*, clique em **Create new credential**:
   - Tipo: **Telegram API**
   - **Access Token**: cole o valor de `TELEGRAM_BOT_TOKEN`
   - Salve.
4. Selecione essa mesma credencial nos nós **Enviar temperatura** e **Enviar erro**.
5. A OpenWeather não precisa de credencial no N8N, porque a chave vem da variável de ambiente `OPENWEATHER_API_KEY`.

### 5. Ativar o workflow

Salve o workflow e ligue o botão **Active**. O N8N registra o webhook no Telegram usando a `WEBHOOK_URL`.

> Para testar sem ativar, use **Test workflow**. O N8N escuta uma única mensagem pela URL de teste.

---

## 💬 Como testar o chatbot

Abra o seu bot no Telegram e envie:

| Mensagem | Retorno esperado (a temperatura varia) |
|----------|----------------------------------------|
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `Belo Horizonte,MG,BR` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `Teresina, PI` | `🌤️ A temperatura em Teresina é de 34°C.` |
| `Cidadeinexistente,XX,BR` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

O formato recomendado é **`Cidade,UF,BR`**. O bot também aceita só o nome da cidade e completa o `,br` sozinho.

---

## 🛠️ Solução de problemas

| Sintoma | Causa provável |
|---------|----------------|
| Toda cidade retorna a mensagem de erro | `OPENWEATHER_API_KEY` ausente ou ainda não ativada (a API devolve `401`). Confira o `.env` e reinicie com `docker compose up -d`. |
| O bot não responde | Workflow inativo, ngrok parado ou `WEBHOOK_URL` diferente da URL atual do ngrok. |
| Erro ao ativar o Telegram Trigger | A `WEBHOOK_URL` precisa ser HTTPS e pública. |

---

## 📁 Estrutura do repositório

```
chatbot-telegram/
├── workflow-chatbot-telegram.json   # workflow exportado do N8N (sem credenciais)
├── README.md
├── docker-compose.yml               # N8N local via Docker
├── .env.example                     # modelo das variáveis (sem valores reais)
└── .gitignore                       # ignora o .env
```
