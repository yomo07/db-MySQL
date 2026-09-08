# DBMYSQL

> **Sin variables de entorno aprobadas por Kiro, este Power no funciona.** Kiro solo expande `${VARIABLE}` en el `mcp.json` cuando esa variable está explícitamente aprobada — si no, la conexión falla con un error de autenticación que no menciona nada sobre variables. Ver "Puesta en marcha" más abajo.

Un Kiro Power dedicado exclusivamente a **MySQL**, usando el paquete MCP [`@benborla29/mcp-server-mysql`](https://www.npmjs.com/package/@benborla29/mcp-server-mysql).

Separado a propósito de otros motores: un Power por servidor MCP, para que el panel de Kiro quede claro y cada uno se active de forma independiente sin que un Power intente coordinar varios paquetes distintos.

## Soporte y privacidad

**Soporte:** yoel.moreno.ym@gmail.com

**Política de privacidad:** [PRIVACY.md](./PRIVACY.md)

## ⚠️ Tres validaciones obligatorias antes de usarlo

Estas tres cosas tienen que estar en orden **antes de intentar cualquier consulta**. Son la causa de la gran mayoría de los fallos de conexión. El agente debe verificarlas al arrancar una sesión de trabajo, no asumir que ya están listas.

### 1. Node.js disponible

Verificar en una terminal: `node -v`. Este paquete no declara una versión mínima estricta, pero conviene una LTS moderna (18+). Si trabajás con varios proyectos que usan versiones distintas de Node, asegurate de que la correcta esté activa **en el momento en que arrancás Kiro** — el proceso fija la versión al arrancar, no la vuelve a leer después.

### 2. Las cinco variables de entorno creadas

`MYSQLPOWER_HOST`, `MYSQLPOWER_PORT`, `MYSQLPOWER_USER`, `MYSQLPOWER_PASS`, `MYSQLPOWER_DB` — las cinco tienen que existir en el sistema y **en MAYÚSCULAS exactas**. Verificar **solo que existan, nunca su valor**, con `if defined MYSQLPOWER_HOST (echo Definida) else (echo NO definida)` en `cmd.exe` (repetir por variable) — nunca con `echo %VAR%`, que sí imprime el valor en pantalla.

### 3. Las cinco variables aprobadas en el IDE de Kiro

Tener la variable creada no alcanza: Kiro además tiene que tenerla **aprobada** para expandirla. Confirmar en `Ctrl + ,` → **Mcp Approved Env Vars** que las cinco figuren en la lista.

---

## Variables de entorno

| Variable del sistema | Variable interna del paquete |
|---|---|
| `MYSQLPOWER_HOST` | `MYSQL_HOST` |
| `MYSQLPOWER_PORT` | `MYSQL_PORT` |
| `MYSQLPOWER_USER` | `MYSQL_USER` |
| `MYSQLPOWER_PASS` | `MYSQL_PASS` |
| `MYSQLPOWER_DB` | `MYSQL_DB` |

## Solo-lectura de fábrica

`@benborla29/mcp-server-mysql` deshabilita las escrituras por defecto. Este Power lo hace explícito, fijando `ALLOW_INSERT_OPERATION`, `ALLOW_UPDATE_OPERATION` y `ALLOW_DELETE_OPERATION` en `false` dentro del `mcp.json`.

Están fijos a propósito, no vienen de variables de entorno: habilitar escrituras debe ser una decisión deliberada de quien edita el Power, no algo que se active por accidente al definir una variable.

Esta protección vive **en el servidor MCP**, no solo en las instrucciones del agente — así que es más fuerte que una regla de comportamiento: aunque el agente intente un `INSERT`, el servidor lo rechaza.

## Qué NO hace (por decisión de diseño)

- **No lee ningún archivo que pueda contener credenciales.**
- **No lee ni genera `product.md`.**
- **No recomienda ni sugiere instalar otros Powers.**
- **No imprime nunca el valor de una variable de entorno**, ni siquiera para depurar.

## Instalación (a nivel de usuario, global para todos los proyectos)

Panel de **Powers** en Kiro → **Add Custom Power** → **Import power from a folder** (con esta carpeta descomprimida) o **Import power from GitHub** (con la URL del repositorio).

Los Powers se instalan a nivel de usuario por defecto (`C:\Users\<tu usuario>\.kiro\powers\installed\dbmysql`, con el nombre en minúsculas porque el esquema de Agent Plugins solo admite minúsculas, dígitos, guiones y puntos en el campo `name`), y quedan disponibles en **todos los workspaces** sin reinstalar por repositorio.

## Puesta en marcha

1. **Definir las cinco variables** en una terminal (`cmd.exe` en Windows):

   ```
   setx MYSQLPOWER_HOST tuhostaqui
   setx MYSQLPOWER_PORT tupuertoaqui
   setx MYSQLPOWER_USER tuusuarioaqui
   setx MYSQLPOWER_PASS tupasswordaqui
   setx MYSQLPOWER_DB tubasedatosaqui
   ```

   Verificar en una terminal **nueva** que existan, sin imprimir el valor:

   ```
   if defined MYSQLPOWER_HOST (echo Definida) else (echo NO definida)
   if defined MYSQLPOWER_PORT (echo Definida) else (echo NO definida)
   if defined MYSQLPOWER_USER (echo Definida) else (echo NO definida)
   if defined MYSQLPOWER_PASS (echo Definida) else (echo NO definida)
   if defined MYSQLPOWER_DB (echo Definida) else (echo NO definida)
   ```

   **Usar siempre estos nombres en MAYÚSCULAS exactas.** Windows no distingue mayúsculas para usar la variable, pero la lista de aprobación de Kiro compara el nombre como texto exacto.

   **Importante para `MYSQLPOWER_PASS`:** usar `setx` sin la bandera `/M`, para que quede a nivel de **usuario** (`HKCU`), visible solo para tu perfil. Con `/M`, o definiéndola en "Variables del sistema" desde el Panel de Control, queda a nivel de **máquina** (`HKLM`) — visible para cualquier usuario de esa computadora. Nunca guardes la contraseña a nivel de máquina.

2. **Aprobarlas en el IDE.** Al guardar el `mcp.json` (o al activar el Power por primera vez), debería aparecer una ventana emergente de aprobación. Si no aparece: `Ctrl + ,` → **Mcp Approved Env Vars** → agregar los cinco nombres. Nota: `kiro-cli` tiene su propia lista aparte, en `~/.kiro/settings/cli.json`.

3. Si las variables se definieron con Kiro ya abierto, **cerrar y reabrir el IDE por completo**.

4. **Activar el Power** — no hay ningún servidor que "habilitar" por separado: para un Agent Plugin, el servidor MCP se activa junto con el Power. Se activa solo cuando le pedís al agente algo de MySQL, o manualmente desde **Powers** → `dbmysql` → **Try power**.

El skill `setup` incluye el detalle completo y una tabla de diagnóstico.

## Protección de tablas grandes

Antes de consultar una tabla por primera vez, el Power estima su tamaño con `information_schema.tables` (nunca con un `COUNT(*)` sobre la tabla entera). Por debajo de cien mil filas consulta libremente. Entre cien mil y un millón exige al menos un filtro selectivo. Por encima de un millón la trata como **tabla protegida**: exige filtro selectivo, rango de fechas cerrado si aplica, índice verificado, `LIMIT` explícito, y prohíbe `SELECT *`, ordenar sin límite, uniones sin filtro y funciones sobre la columna filtrada.

Si hay duda sobre una consulta, revisa el plan con `EXPLAIN` — nunca `EXPLAIN ANALYZE` sobre una tabla protegida, porque ese sí ejecuta la consulta de verdad.

## Múltiples bases MySQL

Este Power trae una sola conexión configurada. Si hace falta una segunda (por ejemplo, desarrollo y producción), se duplica el bloque `mysql` en el `mcp.json` del paquete con otro nombre y otro juego de variables — es un cambio del Power en sí, no algo que se alterne en tiempo de uso.

## Problemas conocidos

**Las variables `${VAR}` a veces no se expanden, incluso aprobadas y presentes en el entorno.** En un Power tipo Agent Plugin, las referencias `${VAR}` del bloque `env` pueden llegar literales al servidor MCP (el texto `${MYSQLPOWER_HOST}` tal cual, no el valor), aunque la variable esté aprobada y confirmada presente en el entorno. Es un bug del runtime de Kiro, reportado y con reproducciones en varias superficies (CLI/Docker, stdio, Kiro Web, Agent Plugins). Si te pasa, probá en este orden:

1. Cerrar Kiro **por completo** (no solo recargar la ventana) y volver a abrirlo.
2. Confirmar que la variable esté a nivel de **usuario** (`HKCU`), no de máquina (`HKLM`).
3. Revisar si hay una actualización de Kiro pendiente.
4. Reinstalar el Power desde cero.

**Si después de todo eso sigue sin expandir, el último recurso es hardcodear el valor — pero solo en tu copia ya instalada, nunca en el paquete que publicás o compartís:**

- Editá el `mcp.json` de la carpeta **instalada** (`~/.kiro/powers/installed/dbmysql/mcp.json` o el equivalente en Windows), reemplazando cada `"${MYSQLPOWER_...}"` por el valor real. El agente **nunca escribe el valor real por vos** — solo indica dónde y cómo hacerlo, con un placeholder de ejemplo:

  ```jsonc
  "env": {
    "MYSQL_HOST": "tuhostaqui",
    "MYSQL_PORT": "tupuertoaqui",
    "MYSQL_USER": "tuusuarioaqui",
    "MYSQL_PASS": "tupasswordaqui",
    "MYSQL_DB": "tubasedatosaqui"
  }
  ```

- **Nunca** hagas ese cambio en la carpeta fuente que vas a publicar o subir a GitHub.
- Tratá esa copia con hardcode como un archivo de credenciales: no la compartas, no la subas a ningún repositorio, no la pegues en un chat.
- **Vas a tener que repetir el hardcode después de cada actualización del Power** (`Check for updates` sobrescribe el `mcp.json` instalado).
- Si llegás a este punto, conviene rotar esa credencial después, ya que estuvo en texto plano en un archivo en disco.

**El validador de esquemas de VS Code/Kiro puede bloquear `agent-plugins.org` por defecto**, con el error "Location is untrusted". Se resuelve agregando ese dominio a `json.schemaDownload.trustedDomains` en el `settings.json` de usuario (`Ctrl+Shift+P` → "Preferences: Open User Settings (JSON)").

## Actualizar el Power

Panel de Powers → `dbmysql` → **Check for updates** → **Install updates**.

## Estructura

```
DBMYSQL/
├── plugin.json
├── mcp.json
├── README.md
├── PRIVACY.md
├── LICENSE
└── skills/
    ├── setup/
    │   └── SKILL.md
    └── query-helper/
        └── SKILL.md
```

## Servidor MCP usado

| Motor | Paquete |
|---|---|
| MySQL | `@benborla29/mcp-server-mysql` |

Paquete de terceros con sus propios términos y licencia. Revisalo antes de usarlo contra bases productivas.

## Licencia

MIT
