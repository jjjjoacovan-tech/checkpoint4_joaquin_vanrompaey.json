# Checkpoint 4: Integraciones avanzadas (HubSpot + Gmail + Slack)

**Autor:** Joaquín Van Rompaey · AI Automation Avanzado (Coderhouse)
**Archivo:** `checkpoint4_joaquin_vanrompaey.json` (se importa en n8n desde *Workflow > Import from File*)

Este checkpoint evoluciona el proyecto integrador. Parte del workflow del **Módulo 3** (Manager multi-agente del M2 con memoria de largo plazo en Airtable) y le suma las integraciones reales vía **OAuth2** con estas herramientas:

| Rol en el caso e-commerce | Herramienta | Operación de lectura (pasado) | Operación de escritura (futuro) |
|---|---|---|---|
| Casilla de soporte | **Gmail** | Trigger: emails no leídos de INBOX | **Create Draft** (nunca envía) |
| CRM / fuente única de verdad | **HubSpot** | Search contact por email | Create / Update contact |
| Canal del equipo de operaciones | **Slack** | — | Post message en `#checkpoint1-agente-leads` |

## Flujo

```
Gmail Trigger (INBOX, no leídos, sin adjuntos)
  → ① IF ¿es auto-reply?  ── Sí → Stop (corta el bucle infinito)
  → ④ Set Limpiar Payload (from, subject, body; sin HTML ni binarios) → IF ¿payload válido? ── No → Stop (evita 400)
  → Memoria M3 (Airtable, Session_ID = email del remitente)
  → Router de Triaje M2 → Worker 1 / Worker 2 / Escalamiento humano
  → Agente Manager (Tools Agent, memoria inyectada entre [INICIO/FIN DE CONTEXTO COMPARTIDO])
  → ② HubSpot Look up por email → IF existe → Update | No → Create   (evita 409)
  → ③ Gmail Create Draft en el hilo original (Human-in-the-loop)
  → Set Payload Slack Limpio (5 campos) → Slack aviso a Operaciones
  → Persistencia + Summarization M3 (Airtable, upsert idempotente)
```

## Los 4 nodos de la rúbrica

1. **① IF: ¿Es auto-reply?** Descarta asuntos con `auto-reply`, `automatic reply`, `respuesta automática`, `out of office`, `fuera de la oficina`, `undeliverable` o `delivery status notification`, y remitentes `no-reply`, `noreply` o `mailer-daemon`.
2. **② HubSpot: Look up Contacto.** Busca por `email` (EQ) antes de escribir. Si el contacto existe, lo actualiza; si no, lo crea. Así no quedan contactos duplicados (Error 409).
3. **③ Gmail: Create Draft (HITL).** Solo crea el borrador de respuesta en el hilo original. Un humano lo revisa y lo envía desde Gmail.
4. **④ Set: Limpiar Payload.** Deja solo `from_email`, `from_name`, `subject`, `body_text` (sin HTML, máximo 3000 caracteres), `thread_id` y `message_id`, y valida el email con una regex (evita el Error 400). Antes de Slack hay un segundo Set que reduce el mensaje a 5 campos.

## Mínimo privilegio

- **Gmail:** el trigger lee solo `INBOX` y no leídos, sin descargar adjuntos. La única escritura es `draft.create`: no hay ningún nodo `send`.
- **HubSpot:** solo lee `email`, `firstname` y `lifecyclestage`, y escribe `email`, `firstname`, `lifecyclestage` y `message`.
- **Slack:** solo `chat:write` en un canal fijo. No lee historial.
- **Airtable:** trabaja sobre una sola tabla (`Memoria_Sesiones`) y siempre filtra por `Session_ID`.

## Cómo probarlo

1. Importá el `.json` y conectá las credenciales OAuth2 de Gmail, HubSpot y Slack hasta que queden en verde. Airtable y Anthropic usan las del M3.
2. El trigger trae un **email de prueba fijado** (pin data), así que con **Execute Workflow** se puede testear el flujo completo sin esperar un email real.
3. Revisá **Borradores** en Gmail, el contacto en HubSpot y el aviso en Slack.

## Test de regresión realizado

| Prueba | Resultado |
|---|---|
| Email nuevo (contacto inexistente) | Contacto **creado** en HubSpot, borrador en Gmail, aviso en Slack, fila en Airtable ✅ |
| Mismo email por segunda vez | Look up encontró el contacto → **Update**, sin duplicado en HubSpot ✅ |
| Asunto `Out of office: vuelvo el lunes` | Cortado en *Stop - Auto-reply ignorado*, sin borrador ni Slack ✅ |
