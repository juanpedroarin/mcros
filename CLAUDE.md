# McRos

PWA para registrar comidas y estimar macros con IA a partir de texto o fotos. Proyecto de portafolio: global, gratuito y sin infraestructura propia.

Nace de un prototipo en un artifact de claude.ai (registro personal). Este repo es una reimplementación desde cero: se reutilizan ideas, prompts y cálculos, no datos ni valores personales.

## Restricciones de producto (no negociables)

- **100% frontend.** Sin backend, sin servidor propio, sin login ni cuentas de McRos.
- **Los datos viven en el dispositivo** (IndexedDB). Si se borra la app, se borra el historial.
- **Costo cero para el proyecto.** Hosting estático gratuito (GitHub Pages). Nada que requiera facturación.
- **IA del usuario:** cada usuario conecta su propia cuenta de OpenRouter (OAuth PKCE). Por defecto, modelos gratuitos (`:free`) con visión.
- **Sin terceros en el cliente:** ni analytics, ni telemetría, ni scripts externos. Las únicas llamadas de red salientes van a `openrouter.ai`.

## Stack

| Área | Elección |
|---|---|
| Build | Vite + TypeScript (`strict: true`) |
| UI | React |
| PWA | vite-plugin-pwa (manifest + service worker con Workbox) |
| Datos | Dexie.js sobre IndexedDB |
| Validación | Zod (toda respuesta de la IA y todo dato importado) |
| Unit tests | Vitest |
| E2E | Playwright |
| CI/CD | GitHub Actions; deploy a GitHub Pages en cada push a `main` |

Usar las últimas versiones estables al momento de instalar. No agregar dependencias fuera de esta lista sin registrar la decisión (ver "Decisiones").

## Estructura

```
src/
  domain/     # Lógica pura, sin DOM ni red: tipos, fechas, totales, porcentajes
  db/         # Dexie: esquema, migraciones, repositorios
  ai/         # Interfaz Estimator, adaptador OpenRouter, prompt, esquema Zod, parser tolerante
  auth/       # OAuth PKCE con OpenRouter (redirect y modo sin redirección)
  ui/         # Componentes y pantallas
  i18n/       # Textos de la UI (es.ts). Nada de strings visibles sueltos en componentes
tests/
  unit/       # Vitest, sobre domain/, ai/ (parser, esquema) y auth/ (PKCE)
  e2e/        # Playwright, con page objects en tests/e2e/pages/
docs/
  decisiones.md
  plan-fase-1.md
```

## Convenciones

- Código, nombres e identificadores en inglés. Textos visibles para el usuario en español, centralizados en `src/i18n/es.ts` (la app será multilenguaje más adelante).
- `domain/` no importa nada de `ui/`, `db/` ni `ai/`. Todo lo calculable se testea con Vitest.
- La IA es una dependencia detrás de una interfaz:
  ```ts
  interface Estimator {
    estimate(input: { text?: string; image?: Blob }): Promise<EstimateResult>;
  }
  ```
  La UI nunca llama a OpenRouter directamente.
- Las respuestas de la IA son **entrada no confiable**: extraer el JSON de forma tolerante, validar con Zod, acotar rangos y mostrar siempre una pantalla de confirmación editable antes de guardar.
- Errores tipados y mensajes en `i18n`. Cada error indica qué pasó y qué puede hacer el usuario.
- Fechas: los días se identifican con clave local `YYYY-MM-DD`. Una sola función para el inicio de semana (lunes) cuando haga falta.
- Accesibilidad básica: controles con etiqueta, foco visible, contraste en tema claro y oscuro.
- Selectores de test: `data-testid` en elementos que los tests usan. Preferir roles y textos accesibles cuando sean estables.

## Seguridad

- La API key de OpenRouter del usuario se guarda solo en IndexedDB del dispositivo. Nunca en logs, URLs ni mensajes de error.
- CSP por `<meta http-equiv>` (GitHub Pages no permite headers): `default-src 'self'; connect-src 'self' https://openrouter.ai; img-src 'self' blob: data:`; sin `unsafe-eval`.
- PKCE con `S256` y `state` aleatorio verificado al regresar.

## Tests

- **Nunca** llamar a OpenRouter real en CI. Los E2E interceptan con `page.route`:
  - `https://openrouter.ai/auth*` y `POST https://openrouter.ai/api/v1/auth/keys` (conexión)
  - `POST https://openrouter.ai/api/v1/chat/completions` (estimación), incluyendo respuestas malformadas, 401, 402, 429 y errores de política de datos
- Proyectos de Playwright: Chromium con dispositivo móvil (Pixel) y WebKit con dispositivo iPhone.
- Los flujos clave tienen E2E: agregar comida (texto y foto), editar, borrar con deshacer, conectar y desconectar OpenRouter, persistencia tras recargar.
- La validación en dispositivos reales (PWA instalada en Android e iOS) es manual y se documenta en `docs/plan-fase-1.md`.

## Deploy

- GitHub Pages bajo `https://<usuario>.github.io/<repo>/`. Configurar `base` de Vite, `scope` y `start_url` del manifest y el `callback_url` del OAuth con ese prefijo.
- El workflow de CI corre lint, unit, E2E y build; el deploy solo ocurre si todo pasa.

## Comandos

```
npm run dev        # servidor local
npm run build      # build de producción
npm run lint
npm run test       # Vitest
npm run test:e2e   # Playwright
```

## Forma de trabajo

- Por etapas. Trabajar solo en lo que define el plan de la fase actual (`docs/plan-fase-1.md`). Si algo fuera de alcance parece necesario, proponerlo antes de implementarlo.
- Cada tarea cierra con sus criterios de aceptación cumplidos y tests en verde.
- Commits pequeños y descriptivos.

## Decisiones

Toda decisión de arquitectura o producto se registra en `docs/decisiones.md` con fecha, decisión y motivo, en el mismo PR que la introduce.
