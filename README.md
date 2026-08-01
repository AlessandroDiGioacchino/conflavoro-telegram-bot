# 🤖 Conflavoro AI assistant
Assistente conversazionale Telegram per rispondere a domande su documenti
istituzionali, sviluppato con **n8n**, **Docker** e **LLM open-source**
via OpenRouter.

<!-- ![Bot Demo](screenshots/telegram-conversation.png) -->

## 🎯 Obiettivo
Dimostrare la capacità di costruire un'automazione AI end-to-end:
- integrazione API (Telegram Bot API + OpenRouter)
- containerizzazione e orchestrazione (Docker Compose)
- gestione webhook HTTPS (ngrok tunneling)
- prompt engineering per mitigare allucinazioni
- gestione dati sensibili e privacy

## 🏗️ Architettura
```
[Telegram user] 
    ↓ (message)
[n8n: Telegram trigger] 
    ↓
[n8n: Set node - knowledge base context]
    ↓
[n8n: HTTP request - OpenRouter API]
    ↓ (JSON payload con prompt di sistema)
[LLM: Llama 3.2 / Gemma 2 / Qwen 2.5]
    ↓
[n8n: Telegram send message]
    ↓
[Telegram user] ← (response)
```

### Componenti chiave
- **n8n** &nbsp; Piattaforma di automazione workflow (self-hosted)
- **Docker Compose** &nbsp; Orchestrazione container con volumi persistenti
- **ngrok** &nbsp; Tunnel HTTPS per esporre webhook locali
- **OpenRouter** &nbsp; API gateway per modelli LLM open-source (gratuiti)
- **Telegram bot API** &nbsp; Interfaccia utente conversazionale

## 🚀 Setup rapido
### Prerequisiti
- Docker e Docker Compose installati
- Account Telegram
- API key [OpenRouter](https://openrouter.ai)
- ngrok (per tunneling HTTPS)

### 1. Clona il repository
```bash
git clone https://github.com/AlessandroDiGioacchino/conflavoro-telegram-bot.git
cd conflavoro-telegram-bot
```

### 2. Configura le variabili d'ambiente
Modifica `docker-compose.yml` e sostituisci:
- `N8N_WEBHOOK_URL` &nbsp; Il tuo URL ngrok (es. https://abc123.ngrok-free.app)

### 3. Avvia n8n
```bash
docker compose up [-d]
```

### 4. Configura il workflow
1. Apri `http://localhost:5678`
2. Importa `workflow/n8n-workflow.json`
3. Configura le credenziali:
    - __Telegram bot__ &nbsp; Inserisci il token da [@BotFather](https://t.me/BotFather)
    - __OpenRouter__ &nbsp; Inserisci la tua API key

### 5. Attiva il workflow
Clicca su «Activate» in alto a destra.

# 🔧 Sfide e soluzioni
### Sfida 1: Webhook HTTPS obbligatorio
__Problema__ &nbsp; Telegram richiede URL HTTPS pubblici, ma n8n gira in locale su
`http://localhost`.  
__Soluzione__ &nbsp; Ho configurato un tunnel ngrok e iniettato l'URL pubblico
tramite variabile d'ambiente `N8N_WEBHOOK_URL` in Docker Compose. n8n registra
automaticamente il webhook corretto per le API di Telegram.

```yaml
environment:
  - N8N_WEBHOOK_URL=https://abc123.ngrok-free.app
```

### Sfida 2: Parsing JSON con caratteri di controllo
__Problema__ &nbsp; Il nodo HTTP Request falliva con errore
«Bad control character in string literal» perché il documento conteneva ritorni
accapo senza escape.  
__Soluzione__ &nbsp; Invece di costruire il JSON come stringa, ho usato un'espressione
JavaScript che crea un oggetto nativo, gestendo tutti i caratteri speciali.

```javascript
{{
  {
    "model": "meta-llama/llama-3.1-8b-instruct",
    "messages": [
      {
        "role": "system",
        "content": "Sei un assistente virtuale di Conflavoro PMI. Rispondi alle domande basandoti ESCLUSIVAMENTE su questo documento:\n\n" + $json.document + "\n\nSe non trovi la risposta, di' che non hai l'informazione."
      },
      {
        "role": "user",
        "content": $('Telegram Trigger').item.json.message.text
      }
    ]
  }
}}
```

### Sfida 3: Volumi Docker orfani
__Problema__ &nbsp; Docker Compose creava volumi con prefisso del progetto (es.
`conflavoro-agent_n8n_data`), ignorando il volume originale con i dati.  
__Soluzione__ &nbsp; Ho forzato il nome del volume nel `docker-compose.yml` con
`name: n8n_data` per evitare prefissi automatici e garantire persistenza dei
dati.

```yaml
volumes:
  n8n_data:
    name: n8n_data
```

### Sfida 4: Errori di parsing Markdown
__Problema__ &nbsp; Telegram falliva l'invio messaggi con errore «can't parse entities:
Character '.' is reserved».  
__Soluzione__ &nbsp; Ho cambiato «Parse Mode» da «MarkdownV2» a «HTML» per evitare problemi
con caratteri speciali generati dallo LLM. In produzione aggiungerei sanitizzazione
dell'output.

# 🔒 Sicurezza e privacy
- __API key__ &nbsp; Gestite tramite credenziali criptate di n8n.  
- __Dati sensibili__ &nbsp; Il prompt di sistema istruisce il modello a rispondere
solo basandosi sul documento fornito, evitando allucinazioni o leak di informazioni
(sarà sufficiente?)
- __Volumi Docker__ &nbsp; I dati dei workflow sono persistiti in volumi nominati,
non in bind mount esposti.  
- __HTTPS__ &nbsp; Tutti i webhook transitano su TLS tramite ngrok.

# 📊 Performance
- __Latenza di risposta__ &nbsp; ~2-3 secondi (dipende dal modello LLM)
- __Costo per messaggio__ &nbsp; ~$0.0001 (modelli gratuiti)
- __Uso memoria__ &nbsp; ~200MB (container n8n)
- __Uso CPU__ &nbsp; Trascurabile (nessuna inferenza locale)

# 🧪 Test e validazione
Il bot è stato testato con:
- domande fattuali sul documento (risposte corrette)
- domande fuori contesto (risposta: “Non ho questa informazione”)
- input malformati (gestiti “graziosamente”)

# 🚀 Possibili miglioramenti
- Implementare RAG con vector store (Pinecone/Qdrant) per documenti grandi
- Aggiungere memoria conversazionale (Redis/Memory node)
- Logging su database (PostgreSQL) per audit trail
- Rate limiting per prevenire abusi
- Deploy su VPS con dominio personalizzato (no ngrok)
- CI/CD con GitHub Actions per test automatici

# 👤 Autore
Alessandro Di Gioacchino  
[LinkedIn](https://www.linkedin.com/in/alessandrodigioacchino/) |
[GitHub](https://github.com/AlessandroDiGioacchino)

# 📄 Licenza
MIT License - vedi file `LICENSE` per dettagli.

---

_Questo progetto è stato sviluppato come dimostrazione pratica di competenze in
automazione AI, containerizzazione e integrazione API._
