# Karen — Chief of Staff Personal con OpenClaw

Agente autónomo que revisa Gmail, clasifica correos no leídos, los prioriza según su impacto real y entrega un brief ejecutivo accionable por Telegram. Sin intervención manual. Solo, cada mañana.

---

## Índice

- [Qué hace](#qué-hace)
- [Arquitectura](#arquitectura)
- [Prerrequisitos](#prerrequisitos)
- [Instalación paso a paso](#instalación-paso-a-paso)
- [Archivos del workspace de OpenClaw](#archivos-del-workspace-de-openclaw)
- [Configuración del cron job](#configuración-del-cron-job)
- [Cómo replicarlo](#cómo-replicarlo)
- [Stack técnico](#stack-técnico)

---

## Qué hace

1. Cada mañana a las 7:00 AM (hora Bogotá) un cron job dispara una sesión aislada del agente.
2. El agente usa `gogcli` para leer los hilos no leídos del inbox de Gmail.
3. Clasifica cada hilo según una taxonomía real de labels personalizadas.
4. Aplica las labels en Gmail y marca los correos como leídos.
5. Entrega un brief ejecutivo estructurado por Telegram con secciones, emojis y acción sugerida.

Ejemplo de salida real en Telegram:

```
📋 Brief ejecutivo de inbox

🔴 Top prioridades:
- 🔒 [sin label | alta] Google: Alerta de seguridad sobre acceso reciente. Acción: revisar si fue esperado.
- 💰 [Financiero | alta] RappiCard: transacciones recientes. Acción: validar montos y comercios.
- 💰 [Financiero | alta] PSE + Bancoomeva: movimientos por $250,000. Acción: verificar operaciones.

🟡 Seguimientos relevantes:
- 📚 [Educación | media] Aulas Virtuales: link para la clase de hoy. Acción: asistir si aplica.

🔵 Promociones o ruido:
- 📢 [Promociones | baja] Ofertas de Shark Ninja, Falabella, Lulo. Acción: ignorar.

⚡ Acción sugerida ahora:
- Revisa primero seguridad y transacciones financieras del día.
```

---

## Arquitectura

```
Gmail (OAuth)
     │
     ▼
 gogcli (gog)
     │  lee hilos no leídos, aplica labels, marca como leído
     ▼
OpenClaw Gateway (systemd, headless, Ubuntu WSL)
     │  cron job 0 7 * * * America/Bogota
     │  sesión isolated + workspace (SKILL.md, HEARTBEAT.md, IDENTITY.md...)
     │  modelo: Gemini 2.5 Flash
     ▼
Telegram Bot (DM)
     │  brief ejecutivo accionable
     ▼
Carlos (el humano que ya no pierde tiempo en el inbox)
```

---

## Prerrequisitos

- Ubuntu en WSL2 (o Linux nativo)
- OpenClaw instalado: https://openclaw.ai
- gogcli instalado: https://gogcli.sh
- Cuenta de Google con OAuth Desktop App configurada en Google Cloud Console
- Bot de Telegram creado en @BotFather
- API key de Gemini (Google AI Studio): https://ai.google.dev

---

## Instalación paso a paso

### 1. Instalar OpenClaw en Ubuntu WSL

```bash
curl -fsSL https://install.openclaw.ai | bash
```

Iniciar onboarding:

```bash
openclaw onboard
```

### 2. Instalar gogcli (Linuxbrew)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install gogcli
```

Como el servicio headless de OpenClaw no lee el PATH de Homebrew, crear un wrapper global:

```bash
sudo ln -s $(which gog) /usr/local/bin/gog
```

### 3. Autenticar Gmail con OAuth

Crear credenciales OAuth tipo Desktop App en Google Cloud Console y descargar el JSON. Luego:

```bash
export GOG_KEYRING_BACKEND=file
export GOG_ACCOUNT=tu@gmail.com
gog auth credentials /ruta/al/client_secret.json
```

Validar:

```bash
gog --no-input --json gmail search 'label:INBOX is:unread' --max 5
```

### 4. Configurar el gateway de OpenClaw como servicio systemd

```bash
openclaw gateway install
systemctl --user enable openclaw-gateway.service
```

Inyectar variables de entorno permanentemente en el servicio:

```bash
systemctl --user edit openclaw-gateway.service
```

Agregar en la sección `[Service]`:

```ini
[Service]
Environment="GOG_KEYRING_BACKEND=file"
Environment="GOG_ACCOUNT=tu@gmail.com"
Environment="GOG_KEYRING_PASSWORD=TU_PASSWORD_KEYRING"
```

Recargar y arrancar:

```bash
systemctl --user daemon-reload
systemctl --user start openclaw-gateway.service
```

### 5. Configurar Telegram

En `~/.openclaw/openclaw.json`, asegurar que exista:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "botToken": "TU_BOT_TOKEN",
    "dmPolicy": "pairing",
    "streaming": "partial"
  }
}
```

Arrancar el gateway y enviar un DM al bot para aprobar el pairing:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

### 6. Configurar el modelo Gemini

Agregar la API key en `~/.openclaw/.env`:

```bash
echo 'GOOGLE_API_KEY=TU_API_KEY' >> ~/.openclaw/.env
```

En `openclaw.json`, actualizar el modelo:

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "google/gemini-2.5-flash"
    }
  }
}
```

### 7. Copiar los archivos del workspace

Clonar este repositorio y copiar los archivos del workspace a `~/.openclaw/workspace/`:

```bash
cp workspace/IDENTITY.md ~/.openclaw/workspace/
cp workspace/SOUL.md ~/.openclaw/workspace/
cp workspace/USER.md ~/.openclaw/workspace/
cp workspace/TOOLS.md ~/.openclaw/workspace/
cp workspace/HEARTBEAT.md ~/.openclaw/workspace/
cp workspace/AGENTS.md ~/.openclaw/workspace/
mkdir -p ~/.openclaw/workspace/skills/chief-of-staff-gmail
cp workspace/skills/chief-of-staff-gmail/SKILL.md ~/.openclaw/workspace/skills/chief-of-staff-gmail/
```

Adaptar `USER.md` con tu nombre, cuenta y contexto personal.
Adaptar `TOOLS.md` con tu cuenta de Gmail y tus labels reales.

---

## Archivos del workspace de OpenClaw

Estos archivos definen la identidad, contexto y comportamiento del agente. Se cargan en cada sesión como parte del bootstrap.

### IDENTITY.md

```markdown
# IDENTITY.md - Who Am I?

- **Name:** Karen
- **Creature:** Chief of Staff personal enfocada en triage de Gmail, priorizacion operativa y briefings ejecutivos
- **Vibe:** Eficiente, proactiva, estrategica y perspicaz. Habla siempre en espanol, a no ser que el usuario se lo diga
- **Emoji:** 🚀
```

### USER.md

Define quién es el humano al que sirve el agente, su contexto, sus reglas de triage y las expectativas de demo.

Ver archivo completo: [`workspace/USER.md`](workspace/USER.md)

Puntos clave:
- Nombre, timezone, rol profesional del usuario
- Formato esperado del brief con emojis de sección
- Heurísticas de prioridad: seguridad y finanzas primero
- Labels personalizadas permitidas

### TOOLS.md

Define el entorno técnico del agente: runtime, modelo, cuenta de Gmail, labels, patrones de alto valor en inbox y política de marcado como leído.

Ver archivo completo: [`workspace/TOOLS.md`](workspace/TOOLS.md)

### HEARTBEAT.md

Checklist que guía al agente en cada ciclo de heartbeat o cron. Define qué buscar, cómo clasificar, cuándo notificar y cuándo quedarse en silencio (`HEARTBEAT_OK`).

Ver archivo completo: [`workspace/HEARTBEAT.md`](workspace/HEARTBEAT.md)

### skills/chief-of-staff-gmail/SKILL.md

Skill principal del agente. Define:
- Labels canónicas y política zero-or-one
- Reglas de prioridad: `alta`, `media`, `baja`
- Ranking de correos cuando compiten por atención
- High-Signal Detection: patrones de remitentes y asuntos de alto valor
- Patrones de comandos `gog`
- Formato exacto del brief ejecutivo con emojis
- Guardrails: nunca inventar labels, nunca usar `Trabajo` como comodín

Ver archivo completo: [`workspace/skills/chief-of-staff-gmail/SKILL.md`](workspace/skills/chief-of-staff-gmail/SKILL.md)

---

## Configuración del cron job

Crear el cron de brief matutino diario:

```bash
openclaw cron add \
  --name "Morning Executive Brief" \
  --cron "0 7 * * *" \
  --tz "America/Bogota" \
  --session isolated \
  --model google/gemini-2.5-flash \
  --message "Eres el Chief of Staff personal de Carlos. Tu tarea ahora es revisar su Gmail no leido.

PASO 1 - Buscar no leidos:
gog --no-input --json gmail search 'label:INBOX is:unread' --max 15

PASO 2 - Clasificar con alta prioridad obligatoria:
- Google con asunto 'alerta de seguridad' -> sin label, alta, Top prioridades
- RappiCard, PSE, serviciopse@achcolombia.com.co, BANCOOMEVA, asuntos con 'transaccion', 'movimiento', 'pago', 'CUS' -> Financiero, alta, Top prioridades
- Aulas Virtuales o clases de hoy -> Educacion, media, Seguimientos relevantes
- Promociones y redes sociales -> Promociones, baja, comprimir en 1-2 bullets

PASO 3 - Agregar label si falta:
gog --no-input gmail labels modify <threadId> --add '<Label>'

PASO 4 - Marcar como leido:
gog gmail message modify <messageId> --remove-label UNREAD

PASO 5 - Entregar brief ejecutivo con este formato:

📋 Brief ejecutivo de inbox
🔴 Top prioridades: (bullets con 🔒💰)
🟡 Seguimientos relevantes: (bullets con 📚💼)
🔵 Promociones o ruido: (1-2 bullets con 📢)
⚡ Accion sugerida ahora:

Termina exactamente despues de la ultima linea. Sin preguntas ni frases de cierre." \
  --announce \
  --channel telegram \
  --to TU_TELEGRAM_USER_ID
```

Verificar que quedó registrado:

```bash
openclaw cron list
```

Disparar manualmente para probar:

```bash
openclaw cron run <jobId>
```

---

## Comandos útiles de operación

```bash
# Ver estado del gateway
systemctl --user status openclaw-gateway.service

# Reiniciar gateway
openclaw gateway restart

# Ver logs en tiempo real
openclaw logs --follow

# Disparar heartbeat manual
openclaw system event --text "Revisa Gmail no leido y entrega el brief ejecutivo segun HEARTBEAT.md" --mode now

# Listar correos no leídos
gog --no-input --json gmail search 'label:INBOX is:unread' --max 15

# Ver historial de ejecuciones del cron
openclaw cron runs --id <jobId>
```

---

## Stack técnico

| Componente | Versión / Detalle |
|---|---|
| OpenClaw | 2026.3.13 |
| gogcli | Linuxbrew, wrapper en `/usr/local/bin/gog` |
| Modelo LLM | `google/gemini-2.5-flash` |
| Gmail OAuth | Desktop App, scopes: read + labels + modify |
| Telegram | Bot via @BotFather, long polling, DM policy: pairing |
| Runtime | Ubuntu WSL2 + systemd user service |
| Scheduler | OpenClaw cron, `0 7 * * *`, `America/Bogota`, isolated session |

---

## Taxonomía de labels de Gmail

Las labels personalizadas que el agente reconoce y puede aplicar:

- `Coomeva MP`
- `Educación`
- `Financiero`
- `Personal`
- `Promociones`
- `Trabajo`
- `Viajes`

Política: zero-or-one label por hilo. Solo agregar, nunca quitar en v1.

---

## Notas de seguridad antes de publicar

- Rotar el bot token de Telegram si aparece en logs o archivos de configuración.
- No subir `client_secret_*.json` (credenciales OAuth de Google Cloud).
- No subir `~/.openclaw/openclaw.json` si contiene tokens en texto plano.
- No subir `~/.openclaw/.env` con la API key de Gemini.
- Añadir al `.gitignore`: `*.json`, `.env`, `openclaw.json`.

---

## Créditos

Construido para el Reto 3 de OpenClaw en Platzi — "Automatiza una tarea real de tu día a día".

Autor: Carlos Hernández — Director de Educación en Platzi.
