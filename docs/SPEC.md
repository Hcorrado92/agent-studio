# Agent Studio — Product Specification

## 0. Factory execution constraints (read first)

This project is built by autonomous coding agents, one backlog item at a time, in a git repository that is the agent's current directory. These constraints are fixed by the environment:

- **Layout:** backend in `backend/` (Python package `app/`, tests in `backend/tests/`, pytest config in `backend/pyproject.toml` with `pythonpath = ["."]`). Frontend in `frontend/` (Vite + React + TypeScript + Tailwind; tests next to the code or in `frontend/src/**/*.test.tsx`).
- **Agents cannot run commands, install packages or access the network.** Dependencies are pre-installed and FROZEN:
  - Python: fastapi, uvicorn, pydantic (v2), pydantic-settings, sqlalchemy (2.x), python-multipart, pyjwt, argon2-cffi, httpx, pytest, anthropic.
  - Node (`frontend/package.json` and `node_modules` already exist; never add or change dependencies): react, react-dom, react-router-dom, typescript, vite, @vitejs/plugin-react, tailwindcss, @tailwindcss/vite, vitest, jsdom, @testing-library/react, @testing-library/jest-dom, @testing-library/user-event, @types/react, @types/react-dom, @types/node.
- Frontend config: a single `frontend/tsconfig.json` (no project references), `frontend/vite.config.ts` including the Vitest config (`environment: "jsdom"`) and an `/api` proxy to the backend, Tailwind v4 via `@tailwindcss/vite`.
- **Verification** after every item (must exit 0), run by the factory: backend `python -m pytest -q` in `backend/`; frontend, once the files exist: `tsc --noEmit -p tsconfig.json`, `vitest run` (when test files exist) and `vite build` (when `index.html` exists) in `frontend/`, plus a check that no undeclared/uninstalled dependency was added. Keep every previously passing test passing.
- Every item adds automated tests for its acceptance criteria. End-to-end flows are tested through the API with FastAPI `TestClient` (no browser automation is available).
- Never commit secrets, `.env` files or API keys; configuration comes from environment variables with safe defaults. The default LLM provider is a clearly labelled **mock**; real providers are used only when credentials are configured on the server.
- Use only fictitious data in tests and fixtures. No deployment, infrastructure or CI files.

### Milestones (priority order; deliver a working vertical slice before secondary features)

1. Project structure, persistence and backend.
2. Frontend, authentication and dashboard.
3. Conversational builder and agent specifications.
4. Agent runtime and agent chat.
5. Accounting tools and CSV analysis.
6. Integration, tests and fixes.
7. Documentation and functional delivery (README with how to run backend and frontend).

---

## 1. Product vision

Agent Studio es una plataforma web donde cualquier persona puede crear agentes de inteligencia artificial mediante conversaciones en lenguaje natural.

El usuario no necesita saber programar, escribir prompts técnicos ni conocer MCP, APIs o modelos de lenguaje.

Ejemplo:
- Usuario: "Quiero un asistente de contabilidad para mi empresa".
- Constructor IA: "Perfecto. ¿Qué tareas necesitás que realice?"
- Usuario: "Controlar facturas, clasificar gastos, analizar ingresos y avisarme de vencimientos".

El sistema comprende los requisitos, identifica las capacidades necesarias y construye una configuración funcional. Cuando el usuario presiona "CREAR AGENTE", la plataforma instancia el agente y lo incorpora a su panel. El usuario debe poder conversar con ese agente, importar documentos compatibles, utilizar sus herramientas autorizadas y administrar su configuración.

No es una maqueta de chatbot ni un CRUD de prompts.

## 2. MVP features

### A. Dashboard principal
Interfaz web profesional con: listado de agentes creados; estado de cada agente; cantidad de ejecuciones; historial de actividad; acceso a conversaciones; botón "Crear nuevo agente"; opciones de editar, duplicar, pausar y eliminar agentes.

### B. Constructor conversacional
Interfaz donde el usuario describe qué agente necesita. El constructor debe:
1. Interpretar la solicitud.
2. Hacer preguntas relevantes cuando falte información.
3. Identificar las funcionalidades necesarias.
4. Generar una especificación estructurada.
5. Actualizar una vista previa del agente durante la conversación.
6. Permitir modificar requisitos.
7. Mostrar herramientas y permisos requeridos.
8. Permitir probar el agente antes de publicarlo.
9. Crear el agente al presionar el botón correspondiente.

La conversación debe producir configuraciones estructuradas y validadas (Agent Specification validada con Pydantic), no únicamente texto libre. En modo mock, el constructor funciona con reglas deterministas (detección de intención y capacidades por palabras clave) y lo indica claramente en la interfaz.

### C. Motor de agentes (Agent Runtime)
Runtime desacoplado del constructor. Cada agente tiene: identificador único; nombre y descripción; objetivos; instrucciones del sistema; modelo y proveedor configurables; herramientas permitidas; configuración persistente; estado; historial de conversaciones; registro de ejecución de herramientas; versionado de configuración.

El runtime carga una configuración desde la base de datos e instancia un agente funcional sin generar código Python nuevo por cada usuario. Interfaz extensible para proveedores LLM: proveedor mock para tests y modo sin credenciales, y un adaptador real (Anthropic) que solo se activa si existen credenciales en el servidor. Nunca colocar claves de API en el frontend.

### D. Herramientas e integraciones
Registro de herramientas con contratos tipados, validación de argumentos y permisos explícitos por agente. Diseño preparado para incorporar servidores MCP más adelante (interfaz de proveedor de herramientas), sin implementarlos en el MVP.

Herramientas locales funcionales:
- Leer archivos CSV autorizados (solo los importados por el propio usuario para ese agente).
- Analizar datos tabulares.
- Clasificar transacciones.
- Calcular totales y agrupaciones.
- Identificar vencimientos.
- Generar reportes descargables (CSV o Markdown).

Prohibido: ejecución arbitraria de código y acceso irrestricto al sistema de archivos por parte de los agentes creados por usuarios.

### E. Gestión y conversaciones
Cada agente tiene su propia pantalla donde el usuario puede: conversar con él; visualizar las herramientas que utiliza; consultar resultados; revisar el historial; modificar la configuración; activarlo o pausarlo.

### F. Autenticación y seguridad
Autenticación local adecuada al MVP (registro, login, contraseñas con hash, token de sesión) y separación de datos por usuario. Validar autorización en los endpoints; no confiar únicamente en filtros del frontend. Registrar las operaciones importantes (audit log). Un agente nunca accede a archivos, conversaciones o información de otro usuario.

## 3. First functional agent: accounting assistant

Demostración de extremo a extremo. El usuario entra al constructor y dice: "Quiero un asistente contable que controle ingresos, gastos, facturas y vencimientos de mi negocio". El sistema permite crearlo y luego utilizarlo.

Capacidades iniciales:
1. Importar un CSV de movimientos financieros.
2. Detectar sus columnas y validar su estructura.
3. Clasificar ingresos y gastos.
4. Calcular resúmenes por categorías y fechas.
5. Identificar vencimientos próximos.
6. Responder consultas sobre los datos importados.
7. Generar un reporte descargable.

Solo información ficticia en las pruebas. No implementar transferencias, declaraciones fiscales automáticas ni modificaciones irreversibles de información financiera.

## 4. Visual experience

SaaS moderno, no una aplicación de prueba. React + TypeScript + Vite + Tailwind CSS, responsive. Estética profesional, moderna y minimalista: sidebar de navegación; dashboard con tarjetas y estadísticas; constructor conversacional con preview lateral; pantallas de agentes independientes; estados de carga, errores y confirmaciones; buen diseño en desktop y celular.

Nada de botones que no hacen nada: cada componente visible tiene una función implementada o se identifica claramente como no disponible.

## 5. Architecture

- Backend: Python, FastAPI, Pydantic, SQLAlchemy; SQLite para desarrollo, preparado para PostgreSQL (URL de base de datos configurable).
- Frontend: React, TypeScript, Vite, Tailwind CSS. Llama al backend vía `/api` (proxy de Vite en desarrollo).
- IA: Agent Builder, Agent Specification Validator, Agent Runtime, Model Provider Abstraction, Tool Registry, preparación para MCP.
- Testing: pytest (endpoints, constructor, runtime, autorización y aislamiento entre usuarios), tests de frontend con Vitest + Testing Library, y pruebas de extremo a extremo del flujo principal a nivel API.
- Arquitectura modular, sin microservicios.

## 6. Acceptance criteria (MVP)

1. Abrir la aplicación en un navegador.
2. Registrar un usuario e iniciar sesión.
3. Acceder al dashboard.
4. Iniciar el constructor conversacional.
5. Describir un agente contable.
6. Obtener una especificación válida.
7. Crear y guardar el agente.
8. Encontrarlo en "Mis agentes".
9. Abrir una conversación con él.
10. Importar un archivo CSV de prueba.
11. Ejecutar herramientas autorizadas.
12. Consultar un resumen de movimientos.
13. Ver el historial.
14. Reiniciar la aplicación sin perder los datos persistidos.

Los tests verifican el comportamiento real de los componentes. Sin credenciales LLM, la aplicación funciona en modo mock claramente identificado; nunca presenta una respuesta mock como si proviniera de una IA real.
