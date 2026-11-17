# Prompt: construir un MCP propio para Navisworks (sesión local, Windows)

Copia el bloque siguiente en una sesión de Claude Code que corra en la PC con
Navisworks. Primero lee `docs/ANALISIS_NAVISWORKS_MCP.txt` (análisis del
proyecto de referencia Aitology/Navisworks_MCP).

---

Quiero que construyas desde cero un servidor MCP que me conecte contigo a
Navisworks, y que lo dejes instalado y probado en esta máquina.

## Contexto
- Lee primero `docs/ANALISIS_NAVISWORKS_MCP.txt`. Es el análisis de un proyecto
  de referencia (Aitology/Navisworks_MCP). Úsalo como inspiración de diseño, no
  copies su código sin revisarlo. Ese proyecto tiene dos piezas: un addin .NET
  dentro de Navisworks con un HttpListener en localhost:8765 y un servidor MCP
  en Python que le hace POST.
- Problemas conocidos que NO debes repetir: (1) el servidor Python falla con
  `mcp` 2.x porque usa decoradores de la serie 1.x; (2) el addin no autentica ni
  valida Host/Origin y expone rutas que borran datos; (3) manifiesto `.addin`
  con un nombre de DLL distinto al compilado; (4) `/health` con conteo de rutas
  incorrecto; (5) `.pdb` distribuido en el release.

## Paso 0: reconocimiento (no escribas código todavía)
1. Detecta la versión de Navisworks instalada (Manage o Simulate; Freedom no
   sirve), su carpeta de instalación y las rutas de `Autodesk.Navisworks.Api.dll`
   y `Autodesk.Navisworks.Clash.dll`.
2. Averigua el framework .NET que exige esa versión (no lo asumas: el
   proyecto de referencia usa net48 con la API 24.0.0.0) y la carpeta de plugins
   efectiva (`%APPDATA%\Autodesk\Navisworks <Manage|Simulate> <año>\Plugins\`).
3. Comprueba herramientas: .NET SDK / targeting pack necesario, Python, y la
   versión de `mcp` que se instalaría. Dime qué falta antes de seguir.
4. Dame un plan corto con la arquitectura elegida y espera mi aprobación.

## Arquitectura a implementar (dos piezas)
A. **Plugin dentro de Navisworks** (C#, `AddInPlugin` con atributo `[Plugin]`).
   - Ejecuta la API de Navisworks solo en el hilo de UI (captura el
     `SynchronizationContext` al inicializar y haz marshalling de cada llamada).
   - Canal local autenticado. Preferido: named pipe restringido al usuario
     actual. Alternativa: HTTP en 127.0.0.1 con token aleatorio en cabecera
     `Authorization`, validación de `Host`/`Origin`, `Content-Type:
     application/json` obligatorio y límite de tamaño de cuerpo.
   - Puerto/pipe configurable, y archivo de descubrimiento en %LOCALAPPDATA%.
   - Registro de eventos con rotación y sin volcar datos del modelo.
   - Nombre del ensamblado, carpeta del plugin, atributo `[Plugin]` y `.addin`
     deben coincidir exactamente. Compila sin distribuir `.pdb`.
B. **Servidor MCP** (Python, stdio).
   - Fija la dependencia: `mcp>=1.0,<2` en `pyproject.toml` o
     `requirements.txt`, o usa la API de 2.x si primero lees su documentación.
   - Genera la lista de herramientas desde UNA sola definición compartida
     (por ejemplo un JSON/YAML) para que plugin, servidor y README no se
     desincronicen y `/health` reporte el conteo real.
   - Marca como destructivas las herramientas `delete_*` y exige un parámetro
     de confirmación explícito.
   - Errores estructurados y un mensaje claro cuando el plugin no responde.

## Herramientas mínimas (fase 1, solo lectura, luego fase 2 escritura)
Fase 1: health, info del documento, estadísticas del modelo, selección actual,
buscar elementos por nombre/propiedad, propiedades de un elemento, listar
sets, listar viewpoints, listar clash tests y resultados.
Fase 2: crear selection/search sets, colorear/transparencia/ocultar/aislar,
guardar y navegar viewpoints, crear/ejecutar clash tests y grupos, y los
`delete_*` con confirmación.

## Cómo conectarte conmigo
1. Registra el servidor en mi cliente MCP (Claude Code:
   `claude mcp add navisworks -- python <ruta>\server.py`; Claude Desktop:
   `%APPDATA%\Claude\claude_desktop_config.json`). Dime qué hiciste.
2. Guíame para reiniciar el cliente y comprobar que aparecen las herramientas.
3. Pruébalo de extremo a extremo con Navisworks abierto y un modelo cargado:
   `health_check` → `get_document_info` → una búsqueda → una operación de
   escritura reversible. Muéstrame la salida real de cada prueba.

## Pruebas y entrega
- Un addin simulado (servidor falso del mismo protocolo) para probar el lado
  MCP sin Navisworks; pruebas automáticas del servidor.
- Un script `Install.ps1` idempotente que compile, despliegue en la carpeta de
  plugins y desbloquee los archivos, sin borrar nada fuera de la carpeta del
  propio plugin.
- README con requisitos, instalación, configuración del cliente, lista de
  herramientas y solución de problemas.
- Cuando termines, dime explícitamente qué probaste en Navisworks de verdad y
  qué no pudiste probar. No declares algo funcionando sin haberlo ejecutado.

## Reglas
- No modifiques ni borres nada fuera de la carpeta del proyecto y la carpeta
  del plugin. Pídeme confirmación antes de cerrar o reiniciar Navisworks.
- No guardes tokens ni secretos en el repositorio.
- Haz commits pequeños y descriptivos.
