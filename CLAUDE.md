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

### 1 — FILTRO (ID nuevo `ck3GRVqwVUd8PKMv`, instancia a550a) ✅ ACTIVO en producción
- **Basado en plantilla real de la comunidad n8n** ("Telegram AI bot with LangChain nodes", repo `enescingoz/awesome-n8n-templates`), adaptada — no un workflow inventado nodo a nodo. Patrón estándar "AI Agent": Trigger → Chat Model + AI Agent → Send, con reintento de error igual que la plantilla original.
- 6 nodos: `Listen for incoming events` (Telegram Trigger, con `download: true` + `imageSize: large` para bajar fotos) → `AI Agent` (nodo LangChain, alimentado por `Anthropic Chat Model` como Chat Model) → `Reparar Texto` (Code, arregla el bug de n8n de abajo) → `Telegram` (envía el HTML) → si falla el envío, reintenta por la rama de error con `Correct errors` (escapa `& < > "`).
- **Analiza imágenes**: si el usuario manda una foto/captura de la ficha del producto (con o sin texto/caption), el `AI Agent` la ve automáticamente — n8n pasa la imagen descargada al modelo como visión (`passthroughBinaryImages`, viene activado por defecto en el nodo Agent). El `systemMessage` le pide leer precio, valoraciones, envío y aspecto directamente de la captura en vez de marcar todo en 🟡 por falta de datos. Si no manda foto, sigue funcionando solo con texto/URL.
- El `AI Agent` recibe el texto del producto (o un prompt por defecto si solo manda foto) y devuelve el HTML final (con 🟢🔴🟡, veredicto y score) vía su `systemMessage`.
- **Prompt basado en el framework oficial** del PDF "Filtros para los productos" de Organic Ecom (Marc Verdú, subido por el usuario). Los 7 checks y sus umbrales exactos:
  1. Efecto WOW (reacción emocional instantánea, clave para contenido sin pagar publicidad)
  2. Fácil de grabar (contenido visual sin gastos extra, varios ángulos/usos)
  3. Precio razonable (no es el número: es el valor percibido vs precio)
  4. Valoración del producto ≥4,5★ (si no tiene reseñas aún, no se descarta → 🟡)
  5. Proveedor fiable: ≥93% valoración **Y** +1 año vendiendo (las dos condiciones a la vez)
  6. Envío ≤25-30 días (si solo se cumple "pagando más", sigue en 🟡 pero ese sobrecoste se arrastra al check 7)
  7. Precio final con envío: se vuelve a aplicar el criterio de valor percibido sumando el coste de envío
  - Es un **embudo de descarte secuencial** (no un promedio): un solo 🔴 = producto descartado en ese check. Si todo es 🟢/🟡, el veredicto dice "APTO — enviar a revisión final" (así es como termina el flujo real: revisión de un mentor antes de producir contenido).
  - Cada check lleva un comentario de 1-2 frases con el PORQUÉ aplicado al producto concreto (no una definición genérica), tal y como pidió el usuario.
- Output: HTML a Telegram con 🟢🔴🟡 por check + comentario razonado + veredicto (APTO/DESCARTADO en el check X/REVISAR MANUALMENTE) + score orientativo /100.
- **Bug conocido de n8n** ([n8n-io/n8n#33406](https://github.com/n8n-io/n8n/issues/33406)): el nodo AI Agent con streaming + Claude Sonnet 5 mete saltos de línea sueltos a mitad de palabra en la respuesta (se nota mucho en español por el tokenizer). Workaround aplicado: el nodo `Reparar Texto` quita todos los `\n` (reconstruye las palabras) y vuelve a insertar los saltos de línea correctos antes de cada check/veredicto/score. Si n8n arregla el bug en el futuro, este nodo se puede quitar.
- Credencial Telegram `Telegram Bot FILTRO` (id `uBmcPl6gZZxlWZjg`) creada con `TELEGRAM_BOT_TOKEN` del entorno. ✅
- Credencial Anthropic (nodo `Anthropic Chat Model`): creada por el usuario directamente en n8n como `Anthropic account 2` (id `gOgpOMeVH3GoNfho`). ✅
  - Queda en n8n una credencial `Anthropic account` (id `qAU7R77fsr5SnSwc`) del primer intento fallido, sin usar — se puede borrar cuando el usuario quiera, no afecta al funcionamiento.
- (ID viejo `mpu45nja4mcWxGId` = sistema antiguo, ya no existe.)
- Nota de red: `n8n.io`, `api.n8n.io` y `docs.n8n.io` siguen bloqueados por la política de red del entorno pese a estar en la lista de hosts a permitir — hay que añadirlos en Settings → Network del entorno si se quiere que Claude navegue la librería de templates directamente en n8n.io (por ahora se busca vía WebSearch + mirrors en GitHub, que sí son accesibles).

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
   - FILTRO: ✅ activo (`ck3GRVqwVUd8PKMv`). Falta reconstruir RESEARCH, GUION y GANCHO en la instancia nueva.
1. Probar GANCHO end-to-end con producto real (ensamblado Cloudinary).
2. Orquestador que dispare GANCHO ×4 en paralelo con los 4 guiones de GUION.
3. Producto nuevo: pedir foto mockup fondo blanco (URL pública) para Vision.
4. Verificar que el chat_id `541043415` de GANCHO es el correcto.

## Reglas de trabajo
1. Desplegar siempre en la instancia real (API REST en la nube / n8n-mcp en local), nunca solo JSON estático.
2. Validar el workflow antes de crear/actualizar.
3. Para editar un nodo, cambio parcial; no reemplazar el workflow entero.
4. Credenciales: usar las credenciales de n8n listadas arriba; nunca pegar claves en nodos ni en el chat.
5. **Antes de construir cualquier workflow nuevo, buscar primero en la librería de n8n** (n8n.io/api.n8n.io están bloqueados por red en el entorno cloud → usar WebSearch + mirrors accesibles en GitHub como `enescingoz/awesome-n8n-templates`) plantillas reales que hagan algo parecido, y adaptarlas — nunca inventar un workflow nodo a nodo desde cero si ya existe una plantilla funcional que se le parezca.
