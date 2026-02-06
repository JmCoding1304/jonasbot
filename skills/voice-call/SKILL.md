# Voice Call Skill

Enable phone calling for Jonasbot. Supports both **inbound** and **outbound** voice calls with real-time conversational AI.

---

## Quick Start

### Make an Outbound Call

```bash
openclaw voicecall call --to "+5521986870202" --message "Hey, checking in"
```

### Receive an Inbound Call

Configure in `openclaw.json` with `inboundPolicy: "allowlist"` and your phone number in `allowFrom` array.

---

## How It Works

- **Provider**: Currently using `mock` (local dev). Production uses Twilio, Telnyx, or Plivo.
- **TTS (Text-to-Speech)**: Powered by OpenAI or ElevenLabs for natural voice responses
- **Streaming**: Real-time audio streaming for responsive conversations
- **Agent Integration**: Calls trigger OpenClaw's agent system for intelligent responses

---

## CLI Commands

| Command    | Usage                           | Example                                                        |
| ---------- | ------------------------------- | -------------------------------------------------------------- |
| `call`     | Initiate outbound call          | `openclaw voicecall call --to "+5551234567" --message "Hello"` |
| `continue` | Send message during call & wait | `openclaw voicecall continue --call-id <id> --message "..."`   |
| `speak`    | Send message without waiting    | `openclaw voicecall speak --call-id <id> --message "..."`      |
| `end`      | Hang up                         | `openclaw voicecall end --call-id <id>`                        |
| `status`   | Check call state                | `openclaw voicecall status --call-id <id>`                     |
| `tail`     | Live call logs                  | `openclaw voicecall tail`                                      |

---

## Agent Tool Access

Inside any agent prompt, you can invoke voice calls:

```
Tool: voice_call
Actions:
  - initiate_call(message, to?, mode?)
  - continue_call(callId, message)
  - speak_to_user(callId, message)
  - end_call(callId)
  - get_status(callId)
```

---

## Config Reference

Located in `openclaw.json` under `plugins.entries.voice-call.config`:

```json
{
  "provider": "mock|twilio|telnyx|plivo",
  "fromNumber": "+15550001234",
  "toNumber": "+5521986870202",

  "inboundPolicy": "allowlist|disabled",
  "allowFrom": ["+5521986870202"],
  "inboundGreeting": "Hey! What do you need?",

  "outbound": {
    "defaultMode": "notify|conversation"
  },

  "streaming": {
    "enabled": true,
    "streamPath": "/voice/stream"
  },

  "tts": {
    "provider": "openai|elevenlabs",
    "openai": {
      "model": "gpt-4o-mini-tts",
      "voice": "nova|alloy|echo|fable|onyx|shimmer"
    }
  }
}
```

---

## Providers

### Mock (Local Dev)

- No API keys required
- Simulates calls locally for testing
- Perfect for development

### Twilio (Production)

- Most mature provider
- Requires: `accountSid`, `authToken`
- Supports SMS + voice
- Needs public webhook URL

### Telnyx / Plivo

- Similar setup to Twilio
- Different API credentials format

---

## Next Steps

1. **For Production**: Migrate from `mock` → `twilio`
2. **Phone Number**: Get a Twilio number or use your own
3. **Webhook**: Set `publicUrl` for inbound call routing
4. **TTS Selection**: Choose OpenAI or ElevenLabs voice based on preference

---

## Troubleshooting

### "No OpenAI API key found"

The TTS provider needs credentials. Check that OpenAI auth is configured in OpenClaw.

### Port 3334 already in use

Webhook server is still running. Kill the process: `pkill -f "voice/webhook"`

### Call ID not responding

Use `openclaw voicecall status --call-id <id>` to check if the call is still active.

---

**Last updated:** 2026-02-06  
**Status:** Working (mock provider ready, production setup pending)
