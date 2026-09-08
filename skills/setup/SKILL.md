---
name: setup
description: Configurar dbmysql para conectarse a MySQL, definiendo y aprobando las variables de entorno correctas.
---

# Configurar dbmysql

Este Power maneja **un solo motor** (MySQL) con **un solo paquete MCP** (`@benborla29/mcp-server-mysql`).

## Paso 0 (obligatorio, antes de cualquier consulta): las tres validaciones

Antes de ejecutar cualquier consulta o explorar el esquema, el agente debe confirmar estas tres cosas. No asumir que ya están listas solo porque el Power está instalado.

1. **Node.js disponible.** Correr `node -v`. Este paquete no declara una versión mínima estricta, pero conviene una LTS moderna (18+). Si `npx` no resuelve el paquete, la versión de Node activa al arrancar Kiro es el primer sospechoso — el proceso fija la versión al iniciar, no la relee después.

2. **Las cinco variables de entorno creadas.** Verificar únicamente su **presencia**, nunca su valor. El agente no debe correr ni sugerir ningún comando que imprima el contenido de una variable — ni `echo %VAR%`, ni `$env:VAR` a secas, ni nada equivalente. En Windows, usar exclusivamente el patrón `if defined`:

```
if defined MYSQLPOWER_HOST (echo Definida) else (echo NO definida)
```

(repetir cambiando el nombre para las otras cuatro). En PowerShell: `if ($env:MYSQLPOWER_HOST) { "Definida" } else { "NO definida" }`. Ninguno revela el valor.

3. **Las cinco aprobadas en Kiro.** Preguntar al usuario si ya las aprobó en `Ctrl+,` → **Mcp Approved Env Vars**, o si vio la ventana emergente de aprobación. Una variable sin aprobar produce un error de autenticación que no menciona nada sobre esto.

**Cuándo repetir esta verificación:** al menos una vez al empezar una sesión de trabajo. Si una consulta falla con error de conexión o autenticación, volver a este Paso 0 antes de reintentar, en vez de repetir la misma consulta a ciegas.

## Regla absoluta sobre credenciales

**El agente NUNCA abre el contenido de un archivo que pueda contener credenciales.** Eso incluye `.env` y variantes, `application.properties`, `application.yml`, `dev.properties`, `local.properties`, `settings.py`, `appsettings.json`, `database.yml`, `my.cnf`, y cualquier archivo con `secret`, `credential`, `password` o `key` en el nombre. Puede listar sus nombres; no los abre ni para diagnosticar un fallo de conexión.

Las credenciales viven únicamente como **variables de entorno del sistema**. Cuando el usuario las tenga dentro de un archivo del proyecto, el agente **no las lee ni las migra**; le explica el mecanismo y lo deja actuar.

**El agente nunca sugiere ni ejecuta un comando que imprima el valor de una variable de entorno.** Toda verificación se hace con comandos que confirman presencia sin revelar contenido. Aplica incluso si el usuario pide explícitamente ver el valor para depurar.

**Cualquier comando de ejemplo usa placeholders inequívocamente falsos** (`tuhostaqui`, `tuusuarioaqui`, `tupasswordaqui`, `tubasedatosaqui`, `tupuertoaqui`). El agente jamás inventa ni sugiere un valor concreto en nombre del usuario — solo valida que la variable exista, nunca manipula ni propone su contenido.

## Paso 1: Definir las variables de entorno (credenciales)

| Variable del sistema | Variable interna del paquete | Descripción |
|---|---|---|
| `MYSQLPOWER_HOST` | `MYSQL_HOST` | Host o IP del servidor |
| `MYSQLPOWER_PORT` | `MYSQL_PORT` | Puerto (3306 por defecto) |
| `MYSQLPOWER_USER` | `MYSQL_USER` | Usuario |
| `MYSQLPOWER_PASS` | `MYSQL_PASS` | Contraseña |
| `MYSQLPOWER_DB` | `MYSQL_DB` | Base de datos |

En Windows:

```
setx MYSQLPOWER_HOST tuhostaqui
setx MYSQLPOWER_PORT tupuertoaqui
setx MYSQLPOWER_USER tuusuarioaqui
setx MYSQLPOWER_PASS tupasswordaqui
setx MYSQLPOWER_DB tubasedatosaqui
```

`setx` las deja permanentes; `set` solo dura la terminal actual. Verificar en una terminal **nueva** que existan, sin imprimir el valor:

```
if defined MYSQLPOWER_HOST (echo Definida) else (echo NO definida)
if defined MYSQLPOWER_PORT (echo Definida) else (echo NO definida)
if defined MYSQLPOWER_USER (echo Definida) else (echo NO definida)
if defined MYSQLPOWER_PASS (echo Definida) else (echo NO definida)
if defined MYSQLPOWER_DB (echo Definida) else (echo NO definida)
```

**Usar siempre estos nombres en MAYÚSCULAS exactas.** Windows no distingue mayúsculas para usar la variable, pero la lista de aprobación de Kiro compara el nombre como texto exacto.

**Importante para `MYSQLPOWER_PASS`:** usar `setx` sin la bandera `/M`. Sin esa bandera la variable queda a nivel de **usuario** (`HKCU`), visible solo para tu perfil. Con `/M`, o definiéndola en "Variables del sistema" desde el Panel de Control, queda a nivel de **máquina** (`HKLM`), visible para cualquier usuario de esa computadora. Nunca guardes la contraseña a nivel de máquina.

## Solo-lectura: activado de fábrica

`@benborla29/mcp-server-mysql` deshabilita las escrituras por defecto. Este Power lo hace explícito en su `mcp.json`, fijando los tres flags en `false`:

| Variable interna | Valor | Qué controla |
|---|---|---|
| `ALLOW_INSERT_OPERATION` | `false` | Permite `INSERT` |
| `ALLOW_UPDATE_OPERATION` | `false` | Permite `UPDATE` |
| `ALLOW_DELETE_OPERATION` | `false` | Permite `DELETE` |

Están fijos a propósito, no vienen de variables de entorno: habilitar escrituras debe ser una decisión deliberada de quien edita el Power, no algo que se active por accidente al definir una variable. Si alguien realmente necesita escrituras, edita esos valores directamente en el `mcp.json` — y conviene que sea contra una base no productiva.

Esta protección es **a nivel del servidor MCP**, más fuerte que una regla del agente: aunque el agente intente ejecutar un `INSERT`, el servidor lo rechaza.

## Paso 2 (obligatorio en el IDE): Aprobar las variables

Por seguridad, el IDE de Kiro solo expande variables de entorno explícitamente aprobadas. Al guardar el `mcp.json`, debería aparecer una ventana emergente de aprobación; si no aparece, `Ctrl + ,` → **Mcp Approved Env Vars** → agregar cada nombre.

`kiro-cli` tiene su propia lista aparte, en `~/.kiro/settings/cli.json`; aprobar en un lado no aprueba en el otro.

## Paso 3: Reiniciar Kiro si las variables se definieron con el IDE ya abierto

El proceso de Kiro fija su entorno al arrancar. Si corriste `setx` con Kiro ya abierto, no va a ver las variables nuevas hasta cerrarlo y volverlo a abrir por completo.

## Paso 4: Activar el Power

Para un Power tipo Agent Plugin, el servidor MCP **no tiene un interruptor propio** de encendido/apagado — se activa y desactiva junto con el Power.

Dos formas: **automática**, cuando le pedís al agente algo relacionado con MySQL en el chat; o **manual**, desde el panel de **Powers** → `dbmysql` → **Try power**.

## Diagnóstico

| Síntoma | Causa probable |
|---|---|
| Error de autenticación con credenciales correctas | Variable no aprobada en el IDE (Paso 2), o definida con Kiro ya abierto (Paso 3) |
| Aparece `${MYSQLPOWER_...}` literal en un error, y ya verificaste nombre, aprobación y reinicio completo | Bug conocido del runtime de Kiro (ver README, "Problemas conocidos") |
| El servidor no arranca | `npx` no resuelve el paquete con la versión de Node activa al arrancar Kiro |
| Cero herramientas aunque el Power está instalado | Probar **Try power** desde el panel de Powers |
| Un `INSERT`/`UPDATE`/`DELETE` es rechazado | Comportamiento esperado: solo-lectura activo a nivel del servidor MCP |
| Error de acceso denegado en una tabla puntual | Permisos del usuario de MySQL, no del Power; revisar los `GRANT` de ese usuario |
| El agente propone abrir `.env` o `my.cnf` | Comportamiento prohibido; las credenciales van solo en variables de entorno |
