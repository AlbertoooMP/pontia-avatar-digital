# PontIA — Módulo 9: Gemelo digital audiovisual

Este repositorio contiene el **prototipo del gemelo digital** desarrollado para el módulo de Herramientas de IA Generativa de PontIA. Reutiliza el ejemplo oficial de [Anam Cookbook](https://github.com/anam-org/anam-cookbook/tree/main/examples/elevenlabs-server-side-agent-nextjs): la aplicación web conecta un **ElevenAgent configurado en ElevenLabs** (conversación, LLM y voz clonada) con un **avatar visual de Anam** en tiempo real.

**Estado actual:** integración probada en local con un avatar y un agente propios. La conexión entre ambos servicios y la interfaz inicial **ya venían implementadas por Anam**; nuestro trabajo hasta ahora ha consistido en configurarlas, resolver los errores de autenticación e identificación del avatar y probar la conversación audiovisual. La incorporación de otros avatares y la personalización de la aplicación quedan pendientes.

**Configuración de esta adaptación:**

- Se utiliza `cara-4.1-express` en `src/app/api/anam-session/route.ts` en lugar del `cara-4` del ejemplo original. El parámetro `directorNotes.expressivity` puede ajustarse en esa misma ruta para controlar la intensidad de los movimientos.
- Cada integrante debe crear su propio `.env.local` a partir de `.env.example` y configurar las claves `ANAM_API_KEY`, `ELEVENLABS_API_KEY` y los identificadores `ANAM_AVATAR_ID` y `ELEVENLABS_AGENT_ID`. **`ANAM_AVATAR_ID` es el ID del avatar, no el ID de la persona (Persona ID).** La clave de ElevenLabs debe tener permiso **ElevenAgents: Escribir (`convai_write`)** para iniciar conversaciones.
- **No subir `.env.local` ni compartir claves API.** El agente, la voz y el avatar son recursos configurados en sus respectivas plataformas; no se incluyen dentro del repositorio.
- Requisitos: **Node.js 20.9 o superior**, **pnpm** y conexión a Internet. Las bibliotecas se instalan con `pnpm install` desde `package.json`/`pnpm-lock.yaml`; consulta también `requirements.txt` para el resumen de requisitos. Para arrancar: `pnpm dev` y abrir `http://localhost:3000`.

A continuación se conserva la documentación técnica original de Anam, con las referencias al modelo de avatar adaptadas a la versión utilizada aquí.

---

# Server-side ElevenLabs agent with an Anam avatar

This Next.js app pairs an ElevenLabs conversational agent with an Anam avatar. ElevenLabs handles speech recognition, LLM reasoning, and text-to-speech. Anam connects to the agent server-side, renders a real-time lip-synced avatar, and maps supported ElevenLabs v3 expressive tags to Director Notes.

> Read the [server-side ElevenLabs agents cookbook](https://anam.ai/cookbook/elevenlabs-server-side-agents) for a walkthrough.

This example was migrated from [anam-org/elevenlabs-agent](https://github.com/anam-org/elevenlabs-agent).

## Architecture

```
Client (Anam JS SDK) ──WebRTC──▶ Anam ◀──WebSocket──▶ ElevenLabs (STT → LLM → TTS)
       ▲                          │                           │
       └──── client tools ───────┴───────────────────────────┘
                                  │
                          Face generation
                                  │
                          WebRTC video/audio ──▶ Client
```

## How it works

1. The Next.js API route (`/api/anam-session`) fetches an ElevenLabs **signed URL** using your API key, then requests an Anam **session token** with `elevenLabsAgentSettings` attached.
2. The session token selects the Cara 4.1 Express avatar model and gives Director Notes a warm baseline style.
3. Anam uses the signed URL to open a WebSocket to ElevenLabs and manages the full voice pipeline.
4. Supported v3 tags in the agent response are mapped to Director Notes cues, so the face follows the voice's delivery.
5. The client creates an `AnamClient` with the session token and calls `streamToVideoElement()`. Microphone audio goes to Anam over WebRTC; the avatar video and speech audio come back over the same connection.
6. ElevenLabs client-tool calls travel through the same server-side connection. The Anam client runs the matching browser handler and sends its result back through Anam.
7. No ElevenLabs SDK is needed on the client. The only client dependency is `@anam-ai/js-sdk`.

## Setup

You need Node.js 20.9 or newer, an Anam API key, and an ElevenLabs account with a configured conversational agent.

### Configure an ElevenLabs agent

1. Go to [elevenlabs.io](https://elevenlabs.io) → **Agents** → **Create Agent**
2. Configure your agent's system prompt and personality
3. Under **Agent Voice**, select **V3 Conversational** as the TTS model (enables expressive mode)
4. Copy the **Agent ID**

### Enable Director Notes

> Director Notes are documented for **Cara 4** avatars. This adaptation uses `cara-4.1-express`; check that the selected Avatar ID is compatible with the configured model.

The API route sets `avatarModel` to `cara-4.1-express` and configures a warm baseline style at `0.5` expressivity:

```typescript
personaConfig: {
  avatarId,
  avatarModel: "cara-4.1-express",
  directorNotes: {
    presetStyle: "warm",
    expressivity: 0.5,
  },
},
```

Tell the ElevenLabs agent to use expressive tags in its system prompt. For example:

```text
Use ElevenLabs v3 expressive tags sparingly when the delivery should change.
For testing, use tags such as [excited], [laughs], [whispers], [sighs],
[gasps], [angrily], [sarcastic], or [nervous].
Place each tag immediately before the words it should affect.
Do not explain or read the tags aloud.
```

The integration maps these ElevenLabs tags:

| Director Note cue | Recognized ElevenLabs v3 tags |
|---|---|
| `happy` | `[excited]`, `[happily]`, `[cheerfully]`, `[elated]` |
| `laughter` | `[laugh]`, `[laughs]`, `[laughing]`, `[laughs harder]`, `[starts laughing]`, `[chuckles]`, `[chuckling]`, `[giggles]`, `[giggling]` |
| `warm` | `[whispers]`, `[whispering]`, `[quietly]` |
| `playful` | `[sarcastic]`, `[playfully]` |
| `curious` | `[curiously]`, `[quizzical]`, `[quizzically]` |
| `concerned` | `[sigh]`, `[sighs]`, `[exhales]` |
| `sad` | `[sadly]`, `[sorrowful]`, `[tearful]`, `[crying]` |
| `surprised` | `[gasp]`, `[gasps]`, `[startled]` |
| `angry` | `[angrily]`, `[furious]`, `[intensely]` |
| `distressed` | `[nervous]`, `[fearful]`, `[scared]` |

This table is the complete alias list that Anam recognizes. ElevenLabs supports other audio tags, but Anam does not map them to Director Notes. Recognized tags are removed from the visible transcript. Other bracketed tags remain ordinary text.

The page listens for `DIRECTOR_NOTE_CUE_APPLIED` and shows the normalized cue, such as `laughter`, when the avatar applies it. See the [Anam Director Notes docs](https://anam.ai/docs/personas/director-notes) for cue configuration and the [ElevenLabs Expressive mode docs](https://elevenlabs.io/docs/eleven-agents/customization/voice/expressive-mode) for v3 voice behavior.

### Add the client tool

In the ElevenLabs agent dashboard, open **Tools**, add a tool with **Tool Type** set to **Client**, and enable **Wait for response**. ElevenLabs has a [client tools guide](https://elevenlabs.io/docs/eleven-agents/customization/tools/client-tools) with screenshots of this setup.

Tool and parameter names are case-sensitive. Use these exact values:

| Tool | Description | Required parameter |
|---|---|---|
| `show_notification` | Show a notification in the demo page. | `message` (String): the notification text |

It also helps to add these instructions to the agent's system prompt:

```text
Use show_notification when the user asks to display a message on screen.
```

The handlers are registered in `src/components/ConversationView.tsx` before the Anam WebRTC session starts. Each handler validates the forwarded arguments, updates the browser UI, and returns a string to the ElevenLabs conversation. See the [Anam client tools docs](https://anam.ai/docs/personas/tools/client-tools) for the handler lifecycle.

### Choose an Anam avatar

1. Go to [lab.anam.ai](https://lab.anam.ai) → create an account
2. Copy your **API Key** from the API Keys page
3. Pick an **Avatar ID** compatible with the configured `cara-4.1-express` model from the Avatars page

### Add environment variables

```bash
cp .env.example .env.local
```

Fill in the keys for one avatar-agent pair:

| Variable | Source |
|---|---|
| `ANAM_API_KEY` | lab.anam.ai → API Keys |
| `ANAM_AVATAR_ID` | lab.anam.ai → Avatars (Avatar ID; compatible with `cara-4.1-express`, not Persona ID) |
| `ELEVENLABS_API_KEY` | elevenlabs.io → API Keys |
| `ELEVENLABS_AGENT_ID` | elevenlabs.io → Agents dashboard |
| `PERSONA_NAME` | Optional label shown before the session starts |

### Run locally

```bash
pnpm install
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000), click **Start**, grant mic permission, and speak.

To test the forwarded tool, try:

- `Show a notification that says the client tool worked.`

To test Director Notes, ask for a response that starts excited, becomes quiet, and ends with a laugh. The page displays the latest normalized Director Note cue alongside the client tool's lifecycle state.
