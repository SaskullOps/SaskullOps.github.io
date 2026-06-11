---
title: Automatiza tu SOC personal con n8n + Telegram + IA
description: >-
  Combina n8n, Telegram y modelos de IA (OpenRouter) para crear un sistema de
  alertas, análisis y resúmenes de seguridad en tu homelab. Código y workflows incluidos.
date: 2026-06-11 18:00:00 -0600
categories: [Homelab, Automatización]
tags: [n8n, telegram, openrouter, ia, homelab, automatización, soc, wazuh, docker]
---

## Problema

Un SOC (Security Operations Center) real cuesta decenas de miles de euros al año.
Herramientas como Splunk, Sentinel o incluso Wazuh en la nube tienen costes de
infraestructura, licenciamiento o mantenimiento que no asumirías para un homelab.

Pero la necesidad es real: recibir alertas de seguridad, analizar IOC, correlacionar
eventos y tener visibilidad de lo que pasa en tu red.

La solución low-cost existe y corre en una Raspberry Pi.

---

## Contexto

Vamos a conectar tres piezas que ya tienes o puedes montar en minutos:

- **n8n** — Automatización low-code. Corre en Docker, conecta cualquier API.
- **Telegram Bot** — Canal de notificaciones gratis, rápido, con API sencilla.
- **OpenRouter** — Gateway a modelos de IA (GPT-4o, Claude, Llama 3) sin suscripción fija. Pagas por uso.

El flujo es:

```
Wazuh / Log → Webhook n8n → Procesamiento → OpenRouter (IA) → Telegram
```

> Todo lo que voy a mostrar está corriendo en mi Raspberry Pi 4 (4GB) en Docker,
> y me cuesta ~$0.50/mes en APIs de IA. El resto es gratis.

---

## Implementación

### Requisitos

- n8n corriendo (http://192.168.1.59:5678 o tu IP)
- Un bot de Telegram ([@BotFather](https://t.me/BotFather) para crearlo)
- Cuenta en [OpenRouter](https://openrouter.ai) — $5 de crédito inicial te duran meses
- Wazuh o cualquier fuente de logs (opcional para empezar)

### 1. Crear el bot de Telegram

Habla con [@BotFather](https://t.me/botfather):

```
/start
/newbot
Nombre: SOC Alertas
Username: socalertas_bot
```

Guarda el token. Luego crea un grupo (o usa tu chat ID) y añade al bot como admin.

Para obtener tu chat ID: envía un mensaje al bot, luego visita
`https://api.telegram.org/bot<TOKEN>/getUpdates` y busca `chat.id`.

### 2. Automatización #1 — Alerta de log → Telegram

Esta es la base: cuando Wazuh (o cualquier sistema) detecte algo, n8n envía un
mensaje formateado a Telegram.

**Workflow n8n** (importa el JSON al final del artículo):

```
[Webhook] → [Set] → [Telegram: Send Message]
```

**Webhook**: configura un endpoint POST como `https://tu-n8n.com/webhook/soc-alert`

**Set**: estructura el mensaje:

```json
{
  "chat_id": "TU_CHAT_ID",
  "text": "🚨 ALERTA SOC\n\nFuente: {{ $json.source }}\nNivel: {{ $json.level }}\nMensaje: {{ $json.message }}\nTimestamp: {{ $json.timestamp }}",
  "parse_mode": "Markdown"
}
```

**Telegram**: configura el nodo con tu bot token.

En Wazuh, añade una regla que envíe webhook a n8n cuando se active `level > 10`:

```xml
<integration>
  <name>custom</name>
  <hook_url>http://tu-ip:5678/webhook/soc-alert</hook_url>
  <level>10</level>
  <rule_id>5710,5712,5715</rule_id>
  <format>json</format>
</integration>
```

### 3. Automatización #2 — Análisis de IP sospechosa con IA

Cuando recibes una alerta con IP externa, n8n puede preguntar a OpenRouter
si esa IP es maliciosa y por qué.

**Workflow**:

```
[Webhook] → [Extract IP] → [HTTP Request (OpenRouter)] → [Telegram]
```

**HTTP Request** a OpenRouter:

```json
{
  "url": "https://openrouter.ai/api/v1/chat/completions",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer TU_API_KEY",
    "Content-Type": "application/json"
  },
  "body": {
    "model": "openai/gpt-4o-mini",
    "messages": [
      {
        "role": "system",
        "content": "Eres un analista SOC. Analiza la IP {{ $json.ip }}. Responde en español con: reputación, posible actividad maliciosa, recomendación. Máximo 3 líneas."
      }
    ],
    "max_tokens": 200
  }
}
```

El resultado llega a Telegram formateado:

```
🤖 Análisis IA — IP 185.220.101.x

Resultado: IP asociada a nodo de salida Tor conocido.
Actividad: escaneo de puertos en los últimos 30 días.
Recomendación: bloquear en firewall si no usas Tor.

Modelo: GPT-4o-mini | Coste: ~$0.0002
```

### 4. Automatización #3 — Resumen diario de seguridad

Un cron en n8n que cada mañana recopila los logs del día anterior y pide a la IA
un resumen ejecutivo.

**Workflow**:

```
[Schedule (0 7 * * *)] → [Fetch logs (API o DB)] → [OpenRouter: summarise] → [Telegram]
```

El prompt para el resumen:

> Eres un analista SOC senior. Revisa estos eventos de seguridad de las últimas
> 24 horas. Clasifícalos en: críticos, medios, informativos. Da una puntuación
> de salud del sistema (1-10). Si hay algo urgente, dímelo en una línea.

Resultado en Telegram cada mañana:

```
📊 Resumen SOC — 11 Jun 2026

Salud del sistema: 8/10 🔵

🔴 Críticos: 1
  - Escaneo de puertos desde 10.0.0.33 (WinRM probe)
  
🟡 Medios: 3
  - Login fallido SSH × 47 intentos (misma IP, bloqueado por fail2ban)
  ...

🟢 Informativos: 12

💡 No requiere acción inmediata.
```

---

## Resultados

Con estas 3 automatizaciones tienes:

- Alertas en tiempo real en el móvil (Telegram)
- Análisis de IOC con IA sin abrir el navegador
- Un resumen diario que lees en 10 segundos
- Coste total: ~$0.50/mes en APIs de IA

El stack completo:

| Componente | Coste |
|-----------|-------|
| Raspberry Pi 4 | Ya la tienes |
| n8n | Gratis (Open Source) |
| Telegram | Gratis |
| OpenRouter (créditos) | ~$0.50/mes |
| Wazuh | Gratis |

---

## Riesgos o limitaciones

- **Latencia**: OpenRouter añade 1-3 segundos por llamada. No es tiempo real para
  alertas críticas — para eso tienes el webhook directo.
- **Privacidad**: los logs viajan a APIs de terceros. No envíes datos sensibles
  sin ofuscar. Uso un prompt del sistema que filtra antes de enviar.
- **Dependencia**: si cae OpenRouter, el análisis IA se detiene. Las alertas
  directas (automatización #1) siguen funcionando.
- **Coste**: aunque es mínimo, si procesas miles de alertas al día, sube.
  Recomiendo rate-limiting en n8n: máx 100 llamadas IA/día.

---

## Próximos pasos

- Añade fuentes adicionales: firewall logs, Pi-hole, Docker events.
- Crea un dashboard con Grafana + Prometheus para ver métricas de seguridad.
- Extiende el análisis IA: haz que la IA proponga reglas de Yara o Sigma
  basadas en patrones detectados.

> 💡 ¿Quieres recibir contenido como este cada semana?
> Apúntate a la newsletter — una reflexión, tres herramientas y una automatización,
> directo a tu correo los domingos.

---

## Apéndice: Código de workflows

### n8n workflow — Alerta básica (import JSON)

```json
{
  "name": "SOC - Alerta básica",
  "nodes": [
    {
      "name": "Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [250, 300],
      "parameters": {
        "path": "soc-alert",
        "responseMode": "onReceived"
      }
    },
    {
      "name": "Set",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [450, 300],
      "parameters": {
        "values": {
          "string": [
            {
              "name": "chat_id",
              "value": "TU_CHAT_ID"
            },
            {
              "name": "text",
              "value": "🚨 *ALERTA SOC*\n\nFuente: {{ $json.source }}\nNivel: {{ $json.level }}\nMensaje: {{ $json.message }}"
            },
            {
              "name": "parse_mode",
              "value": "Markdown"
            }
          ]
        }
      }
    },
    {
      "name": "Telegram",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [650, 300],
      "parameters": {
        "token": {
          "$credentials": {
            "id": "TU_CREDENTIAL_ID"
          }
        },
        "resource": "message",
        "operation": "sendMessage"
      }
    }
  ],
  "connections": {
    "Webhook": {
      "main": [[ { "node": "Set", "type": "main" } ]]
    },
    "Set": {
      "main": [[ { "node": "Telegram", "type": "main" } ]]
    }
  }
}
```

> Reemplaza `TU_CHAT_ID` y `TU_CREDENTIAL_ID` con tus valores.
