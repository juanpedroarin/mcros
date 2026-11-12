# McRos - Registro de decisiones

Formato: fecha · decisión · motivo. Las entradas nuevas van al final.

## 2026-11-12 · Prototipo en un artifact separado

- **Decisión:** el prototipo original (artifact de claude.ai de uso personal) queda intacto. Se creó una copia como sandbox con base de datos propia y vacía.
- **Motivo:** separar el producto del registro personal y sus datos.

## 2026-11-12 · App 100% frontend, sin backend ni login

- **Decisión:** toda la app vive en el cliente, incluida la base de datos. No hay usuarios, cuentas ni servidor propio. Si se borra la app, se borra el historial.
- **Motivo:** proyecto de portafolio, global y gratuito, sin pagos ni mantenimiento de infraestructura.
- **Consecuencias:** los datos dependen del almacenamiento del dispositivo, así que exportar e importar un backup es una función necesaria. No hay sincronización entre dispositivos.

## 2026-11-12 · PWA como plataforma

- **Decisión:** aplicación web progresiva (PWA), instalable en Android e iOS. Una app nativa queda como opción futura (envolviendo el mismo código con Capacitor).
- **Motivo:** sin costos de tiendas (Google Play 25 USD, App Store 99 USD por año), distribución por URL, actualizaciones inmediatas y tests completos con Playwright.
- **Riesgo conocido:** Safari borra el almacenamiento de un sitio tras 7 días de uso del navegador sin visitarlo. No aplica a apps agregadas a la pantalla de inicio. Mitigación: invitar a instalar, `navigator.storage.persist()` y backup.

## 2026-11-12 · Estimación con IA vía OpenRouter (OAuth PKCE)

- **Decisión:** el usuario conecta su propia cuenta de OpenRouter desde la app con OAuth PKCE, sin backend. Por defecto se usan modelos gratuitos (`:free`) con visión; el usuario puede cargar créditos y elegir modelos pagos.
- **Motivo:** costo cero para el proyecto, cada usuario usa su propia cuota y la conexión es un flujo de un clic.
- **Alternativas descartadas:**
  - Suscripciones de consumo (Claude, ChatGPT, Gemini): no permiten uso por apps de terceros.
  - API key de Google AI Studio: demasiada fricción para usuarios no técnicos.
  - IA integrada de Chrome: solo desktop.
  - Proxy propio con clave del proyecto: implica backend y cuota compartida.
  - Automatizar productos de consumo (scraping): viola términos y es frágil.
  - Flujo "copiar y pegar" o enlace desde la IA del usuario: se prefiere que todo ocurra en la interfaz de la app.
- **No elegido por ahora:** Firebase AI Logic (Gemini gratis sin backend propio, cuota compartida). Queda como posible modo de prueba para visitantes.
- **A validar en fase 1:** regreso del OAuth en una PWA instalada en iOS, ajuste de privacidad de OpenRouter para modelos gratuitos y calidad de los modelos `:free` frente a Claude.

## 2026-11-12 · Base de datos local

- **Decisión:** IndexedDB con Dexie.js.
- **Motivo:** liviana, tipada, con índices y consultas reactivas. El volumen esperado es de unos miles de registros por año, así que SQLite WASM no se justifica.

## 2026-11-12 · Stack de la fase 1

- **Decisión:** Vite, TypeScript strict, React, vite-plugin-pwa, Dexie, Zod, Vitest, Playwright, GitHub Actions y GitHub Pages.
- **Motivo:** JavaScript/TypeScript como stack preferido; React por su relevancia para el portafolio; Playwright desde el inicio en los flujos clave; hosting y CI gratuitos.

## 2026-11-12 · Herramientas de trabajo

- **Decisión:** el código se construye con Claude Code sobre este repo. El proyecto de claude.ai queda para discusiones de producto e investigación. Este archivo es la fuente de verdad de las decisiones.
- **Motivo:** la PWA necesita un hosting real (OAuth con URL pública, service worker, llamadas a OpenRouter), algo que un artifact de claude.ai no permite por su sandbox.

## 2026-11-12 · Alcance de la fase 1

- **Decisión:** corte vertical: PWA instalable, comidas del día, conexión con OpenRouter, estimación con texto y foto, persistencia local, CI y deploy. Validación en un Android y un iPhone reales.
- **Motivo:** probar de punta a punta el riesgo principal (IA desde una PWA sin backend en ambos sistemas) antes de portar el resto de las funciones del prototipo.
