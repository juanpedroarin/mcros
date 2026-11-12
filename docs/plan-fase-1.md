# Fase 1 - Corte vertical: comidas con IA en el celular

## Objetivo

Una PWA instalable en Android e iOS donde se puede conectar OpenRouter, sacar una foto o describir una comida, revisar la estimación de macros y guardarla en el dispositivo. Desplegada en GitHub Pages, con CI y tests de los flujos clave.

## Criterios de cierre

1. La app está instalada desde la pantalla de inicio en un Android (Chrome) y en un iPhone (Safari).
2. La conexión con OpenRouter funciona en ambos teléfonos.
3. Una foto tomada con la cámara termina como comida guardada, después de la pantalla de confirmación.
4. Los datos siguen ahí después de cerrar la app y al día siguiente.
5. CI en verde (lint, unit, E2E en Chromium y WebKit, build) y deploy automático.

## Fuera de alcance

Peso, mediciones corporales, resúmenes para copiar, vista semanal, extras, presets, base nutricional y carga manual con buscador, backup e importación, multilenguaje, modelos pagos (se pueden elegir en OpenRouter, pero la fase no los optimiza).

## Tareas

### 1. Repo, CI y deploy

- Vite + React + TypeScript strict, ESLint y Prettier, Vitest, Playwright.
- Workflow de GitHub Actions: lint, unit, E2E (Chromium móvil y WebKit iPhone), build y deploy a Pages solo si todo pasa.
- `base` de Vite configurada para el subpath de Pages.

**Aceptación:** una página mínima publicada en `https://<usuario>.github.io/<repo>/` por HTTPS, desplegada por CI.

### 2. Base PWA

- Manifest (nombre, íconos, colores, `display: standalone`, `scope` y `start_url` con el subpath).
- Service worker con vite-plugin-pwa: la app abre sin conexión (las estimaciones requieren red y lo indican).
- CSP por `<meta>` (ver `CLAUDE.md`).
- Aviso de instalación: en Android, el prompt nativo; en iOS, instrucciones guiadas ("Compartir" y luego "Agregar a inicio"), mostradas solo en Safari y no en modo instalado.

**Aceptación:** instalable en ambos teléfonos; abre en modo standalone; funciona sin conexión salvo la estimación.

### 3. Datos locales

- Dexie con dos tablas:
  ```ts
  type Meal = {
    id: string;            // uuid
    day: string;           // YYYY-MM-DD local
    name: string;
    kcal: number; protein: number; fat: number; carbs: number; // gramos salvo kcal
    createdAt: number;
    source: 'ai' | 'manual';
    model?: string;        // id del modelo que estimó
  };
  type Settings = {
    id: 'settings';
    goals: { kcal: number; protein: number; fat: number; carbs: number };
    ai?: { provider: 'openrouter'; apiKey: string; model: string; connectedAt: number };
  };
  ```
- Índice por `day`. Versionado de esquema desde el inicio.
- `navigator.storage.persist()` al guardar la primera comida; registrar el resultado.

**Aceptación:** unit tests de repositorios; E2E que guarda, recarga y sigue viendo los datos.

### 4. Pantalla del día

- Navegación por día (anterior y siguiente, sin pasar de hoy).
- Totales del día contra objetivos: barras por kcal, proteína, grasa y carbohidratos, con porcentaje y alerta al superar el 105%.
- Lista de comidas: tocar para editar macros, borrar con confirmación y opción de deshacer.
- Ajustes mínimos: los 4 objetivos diarios. Valores por defecto genéricos (no los del prototipo).
- Estado vacío que explica cómo agregar la primera comida.

**Aceptación:** E2E de editar, borrar y deshacer; unit tests de totales y porcentajes.

### 5. Conexión con OpenRouter

- Modo con redirección: generar `code_verifier`, `code_challenge` (S256) y `state`; guardarlos; redirigir a
  `https://openrouter.ai/auth?callback_url=<url>&code_challenge=<c>&code_challenge_method=S256&state=<s>`.
  Al volver, verificar `state` y canjear con `POST https://openrouter.ai/api/v1/auth/keys` (`code`, `code_verifier`, `code_challenge_method`). El código vence a los 10 minutos y es de un solo uso.
- Modo sin redirección: omitir `callback_url`; OpenRouter muestra un código que el usuario pega en la app.
- En iOS instalado, el regreso puede abrirse en Safari (almacenamiento distinto al de la PWA). Implementar ambos modos y elegir en la tarea 7 cuál usar por plataforma.
- Pantalla de ajustes: estado de la conexión, modelo en uso, botón para desconectar (borra la clave local).
- Enviar headers de atribución de app que OpenRouter documente (verificar nombres al implementar).

**Aceptación:** unit tests de PKCE; E2E de conectar (ambos modos) y desconectar con OpenRouter simulado.

### 6. Estimación con IA

- Entrada: texto, foto con cámara (`capture="environment"`) o galería. Redimensionar a 1600 px máximo y comprimir a JPEG 0,85 antes de enviar (reutilizar la lógica del prototipo).
- Modelo por defecto: un modelo `:free` con entrada de imagen. No fijarlo sin más: consultar `GET https://openrouter.ai/api/v1/models` y filtrar por id terminado en `:free` y entrada de imagen (verificar los nombres de campos al implementar). Guardar el elegido en ajustes.
- Llamada: `POST https://openrouter.ai/api/v1/chat/completions` en formato compatible con OpenAI; la imagen como parte de contenido con data URL.
- Prompt base (generalizado del prototipo, sin datos personales):
  ```
  Eres un estimador de macronutrientes para un registro de comidas.
  Si hay foto, estima las porciones por lo que se ve. Si hay una etiqueta nutricional visible, usa sus valores.
  Si algo es ambiguo, asume la porción más probable.
  Responde SOLO con JSON, sin texto adicional:
  {"items":[{"name":"texto corto en español","kcal":0,"protein":0,"fat":0,"carbs":0}],"note":"una línea con lo que asumiste, o vacío"}
  protein, fat y carbs en gramos.
  Comida: <descripción del usuario o "(ver foto)">
  ```
- Parser tolerante: extraer el primer objeto JSON aunque venga con texto o bloques de código alrededor; validar con Zod; descartar ítems inválidos; acotar valores a rangos razonables.
- Pantalla de confirmación: ítems editables, nota del modelo visible, guardar o descartar.
- Errores con mensaje y acción:
  - Sin conexión.
  - 401: clave inválida, reconectar.
  - 402: saldo negativo en OpenRouter.
  - 429: límite alcanzado (los modelos gratuitos tienen 20 por minuto y 50 por día sin créditos comprados).
  - Sin proveedores por política de datos: explicar el ajuste de privacidad para modelos gratuitos en OpenRouter.
  - Respuesta ilegible: reintentar.

**Aceptación:** unit tests del parser y del esquema (respuestas válidas, con texto alrededor, malformadas, con valores fuera de rango); E2E del flujo completo y de cada error, con OpenRouter simulado.

### 7. Validación en dispositivos

Checklist manual, con resultados anotados en este archivo:

| Prueba | Android (Chrome) | iOS (Safari) |
|---|---|---|
| Instalar desde la pantalla de inicio | | |
| Conectar OpenRouter (modo con redirección) | | |
| Conectar OpenRouter (modo sin redirección) | | |
| Foto con cámara hasta comida guardada | | |
| Texto hasta comida guardada | | |
| Datos presentes tras cerrar y reabrir | | |
| Datos presentes al día siguiente | | |
| `storage.persist()` concedido | | |
| Abre sin conexión | | |

Prueba de calidad: 10 fotos reales, comparando kcal y proteína del modelo `:free` contra una estimación de referencia (Claude). Registrar la diferencia media y los casos con más error.

**Aceptación:** checklist completo; decisión registrada sobre el modo de conexión por plataforma y sobre si la calidad del modelo gratuito alcanza.
