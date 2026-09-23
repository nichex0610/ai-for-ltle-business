# Fábrica UGC en n8n — Contexto del proyecto

## Cómo hablar con el usuario
- Siempre en español, sin tecnicismos, con pasos numerados y concretos.

## Objetivo
Fábrica automatizada en n8n (hosteado en Railway) para clonar y replicar vídeos virales de dropshipping orgánico.
Genera vídeos UGC estilo iPhone (hiperrealistas, no cinematográficos) en 9:16 para TikTok/Reels/Shorts.

## Acceso a n8n desde sesiones en la nube
- Variables de entorno: `N8N_BASE_URL` y `N8N_API_KEY` (cabecera `X-N8N-API-KEY`, API en `/api/v1`).
- Instancia actual (la buena): `https://primary-production-a550a.up.railway.app`. Conexión verificada.
- La instancia antigua `primary-production-863df` ya NO se usa. El usuario borró todo para empezar de cero.
- ⚠️ Los workflows, IDs y credenciales de abajo son del sistema ANTIGUO: sirven como plano de diseño, pero hay que reconstruirlos en la instancia nueva (hoy tiene 0 workflows).
- En la nube no hay n8n-mcp: se trabaja con la API REST de n8n. Hosts a permitir en la red del entorno: `primary-production-a550a.up.railway.app`, `n8n.io`, `api.n8n.io`.

## Arquitectura (secuencia) — diseño de referencia del sistema antiguo
[1] FILTRO → [2] RESEARCH → [3] GUION → [4] GANCHO (genera vídeo)

### 1 — FILTRO (ID `mpu45nja4mcWxGId`) ✅ en producción — NO modificar sin petición explícita
- Trigger: bot de Telegram recibe URL de producto.
- 7 checks Organic Ecom: WOW / grabable / precio / 4.5★ / proveedor / envío / precio final con envío.
- Output: HTML a Telegram con 🟢🔴🟡 por check + veredicto + score.

### 2 — RESEARCH (ID `WzYGgzmCBNv4lEom`) ✅ creado — 12 nodos
- Trigger manual. 5 llamadas Claude en cascada: Schwartz Awareness, RMBC, Psicografía, Documento Unificado, Winning Angles.
- Vision de la foto del producto → `descripcion_visual` se propaga a todos los prompts.
- Output: 3 winning angles rankeados → Telegram.

### 3 — GUION (ID `X0EJqQdvHCVEyEQQ`) ✅ creado
- Trigger manual. Descarga vídeo viral de Drive → Gemini 2.5 Flash (transcripción + escenas + timing) → Claude genera 4 guiones (1 réplica + 3 ángulos).
- Output: 4 guiones a Telegram + 4 items para GANCHO.
- Límite: vídeos ≤ 20 MB (inline Gemini). Instagram se baja a mano (snapinsta) y se sube a Drive.

### 4 — GANCHO / Replicar Video (ID `5W13ST0bMwD5eepz`) — 25 nodos
| Fase | Nodos | Estado |
|---|---|---|
| 0 Trigger + Config | Set - Configurar Gancho | ✅ |
| 1 Guion IA | Claude genera 4 escenas (array) | ✅ |
| 2 Imagen (APIMart) | Submit → Wait 3 min → Poll → URL pública (ImgBB) | ✅ |
| 3 Vídeo (Kling V3 Omni) | Submit → Wait 5 min → Poll → URL MP4 | ✅ |
| 4 TTS | Gemini Flash TTS → audio → tmpfiles.org | ✅ |
| 5 Ensamblado | Cloudinary (concat 4 vídeos + 4 audios, firma SHA1) | Pendiente probar |
| 6 Distribución | Telegram → el usuario publica a mano | ✅ |

## Credenciales del sistema antiguo (IDs obsoletos; hay que crearlas de nuevo)
- APIMart API Key `7FF7nxmKOz1QrqfN` (Bearer)
- ImgBB API Key `IG4YW9DHezltqzib` (query auth)
- Telegram Bot Ganchos `ak4k0D7rzTnLfDaD`
- Cloudinary vonh7eox `OwJykF6TxpWJCq3u`
- Anthropic `ZcYc53AsGqqtunFX` (todos los nodos Claude)

## Parámetros anti-gasto (obligatorios en nodos de vídeo/imagen)
`{ "aspect_ratio": "9:16", "motion_strength": "low", "seed": 42 }`
Prompts siempre con: `[1080p, iPhone 17 Pro camera texture, natural indoor light]`

## Filosofía visual
- Hiperrealista, iPhone, 720–1080p. Nunca cinematográfico (Veo 3 / Google Flow prohibidos).
- Nicho total: avatar con ropa del nicho, entorno decorado del nicho, props del nicho al fondo.
- 3 plantillas de gancho: El Robo / El Descubrimiento Mágico / Tutorial en Loop.
- Anti-detección entre variaciones: cambiar avatar (tono/género/edad) + entorno; mantener la estructura emocional.

## Testeo y escalado
- Testeo: 2 productos × 4 vídeos (4 emociones) = 8 vídeos. Stack $0: Cloudinary + Telegram manual.
- Escalado: 3 vídeos/día, mismo cuerpo ganador, cambiar solo los primeros 3 s (hook). Stack: APIMart (Kling V3 / Seedance 2.5) + ElevenLabs TTS.
- Publicación manual desde el móvil vía Telegram (TikTok penaliza subidas por API).

## Próximos pasos
0. Reconstruir los 4 workflows en la instancia nueva (empezando por FILTRO) y crear sus credenciales.
1. Probar GANCHO end-to-end con producto real (ensamblado Cloudinary).
2. Orquestador que dispare GANCHO ×4 en paralelo con los 4 guiones de GUION.
3. Producto nuevo: pedir foto mockup fondo blanco (URL pública) para Vision.
4. Verificar que el chat_id `541043415` de GANCHO es el correcto.

## Reglas de trabajo
1. Desplegar siempre en la instancia real (API REST en la nube / n8n-mcp en local), nunca solo JSON estático.
2. Validar el workflow antes de crear/actualizar.
3. Para editar un nodo, cambio parcial; no reemplazar el workflow entero.
4. Credenciales: usar las credenciales de n8n listadas arriba; nunca pegar claves en nodos ni en el chat.
