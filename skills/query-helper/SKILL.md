---
name: query-helper
description: Consultar MySQL de forma segura, trayendo al contexto solo los datos necesarios.
---

# Consultar MySQL con dbmysql

Aplica siempre que se vaya a ejecutar una consulta.

## Antes de la primera consulta de la sesión: las tres validaciones obligatorias

No ejecutar ninguna consulta sin haber pasado por esto al menos una vez en la conversación (detalle completo en el skill `setup`, Paso 0):

1. Node.js disponible (`node -v`).
2. Las cinco variables `MYSQLPOWER_*` creadas en el sistema.
3. Las cinco aprobadas en Kiro (`Ctrl+,` → Mcp Approved Env Vars).

Si una consulta falla con error de conexión o autenticación, volver a esta verificación antes de reintentar — no repetir la misma consulta a ciegas esperando que funcione distinto.

## Reglas de seguridad (no negociables)

1. **Solo-lectura, reforzado en dos capas.** El servidor MCP ya rechaza `INSERT`, `UPDATE` y `DELETE` por configuración (ver skill `setup`). Además, el agente no debe *intentar* ejecutarlos ni proponer habilitarlos: si el usuario necesita una escritura, explicarle que requiere editar el `mcp.json` del Power deliberadamente, y que conviene hacerlo contra una base no productiva. Nunca ofrecer "activar escrituras" como solución rápida a un error.
2. **Nunca leer archivos de credenciales**, ni siquiera para diagnosticar fallos de conexión. Incluye `.env` y variantes, `application.properties`, `application.yml`, `appsettings.json`, `database.yml`, `my.cnf`, y cualquier archivo con `secret`, `credential`, `password` o `key` en el nombre. Se pueden listar sus nombres, nunca abrir su contenido.
3. **Nunca leer ni generar `product.md`.**
4. **Nunca recomendar ni sugerir instalar otros Powers.**
5. **Nunca sugerir ni ejecutar comandos que impriman el valor de una variable de entorno** (`echo %VAR%`, `echo $VAR`, etc.). Verificar solo presencia (`if defined VAR`), nunca contenido. Si hace falta un ejemplo con un valor de conexión, usar siempre un placeholder inequívoco (`tuhostaqui`, `tupasswordaqui`, etc.), nunca un valor inventado que pueda parecer real.

## Traer al contexto solo lo necesario

- **Esquema antes que datos.** Usar el recurso `mysql://tables` (que el servidor expone con tablas y metadatos de columnas) antes de traer filas de muestra.
- **Agregados antes que filas crudas.** Si la pregunta es "cuántos", "cuál es el máximo", "qué valores distintos existen", usar `COUNT`, `MAX`, `DISTINCT` — no traer el conjunto completo y contarlo después.
- **Columnas explícitas, nunca `SELECT *`.**
- **`LIMIT` siempre presente** al inspeccionar datos de ejemplo (5 a 10 filas bastan).
- No repetir consultas ya respondidas en la misma conversación.
- No consultar tabla por tabla si una consulta al catálogo (`information_schema`) responde todo de una vez.
- No volcar resultados grandes al chat; resumir.

## Tablas grandes: medir antes de consultar (regla dura)

Una consulta mal acotada contra una tabla de muchos millones de filas no es solo lenta: puede degradar la base para todos los demás, mantener locks abiertos y llegar a dejarla pegada. Esta regla protege la base, no el contexto, y por eso está por encima de cualquier conveniencia.

### Paso 1: clasificar la tabla antes de la primera consulta

Antes de consultar una tabla por primera vez, estimar su tamaño consultando `table_rows` y `data_length` en `information_schema.tables`, filtrando por esquema y nombre de tabla. Nunca con un `COUNT(*)` sobre la tabla entera — esa es justamente la consulta que se quiere evitar.

El valor de `table_rows` en InnoDB es **aproximado** (viene de estadísticas del motor, no de un conteo real). Aproximado alcanza y sobra para clasificar; no usarlo como cifra exacta al reportar.

### Paso 2: aplicar el umbral

| Tamaño estimado | Comportamiento |
|---|---|
| Menos de cien mil filas | Consultar libremente, con columnas explícitas y `LIMIT` |
| Cien mil a un millón | Exigir al menos un filtro selectivo y `LIMIT` obligatorio |
| Más de un millón | **Tabla protegida.** No consultar sin filtro |

Guardar la clasificación en la conversación y no volver a medir la misma tabla dos veces.

### Paso 3: reglas para tablas protegidas

1. **Exigir un filtro selectivo:** clave primaria, número de cuenta, identificador de usuario o código de tipo.
2. **Exigir un rango de fechas cerrado** si la tabla tiene columna temporal (con inicio y fin, nunca abierto hacia atrás) — es el filtro más importante en tablas de movimientos, transacciones o auditoría.
3. **Verificar que el filtro use un índice.** Consultar `SHOW INDEX FROM tabla` o `information_schema.statistics` antes de asumir que un filtro es eficiente.
4. **`LIMIT` obligatorio y explícito.**
5. **Nunca `SELECT *`.**
6. **Nunca `ORDER BY` sin `LIMIT`.**
7. **Nunca `JOIN` entre dos tablas protegidas** sin filtro selectivo en ambas.
8. **Nunca funciones sobre la columna filtrada** (por ejemplo `YEAR(fecha) = 2025`): anula el índice. Comparar la columna cruda contra un rango.
9. **Nunca `SELECT ... FOR UPDATE`** ni nada que tome locks de escritura.

### Cómo pedir los filtros

Al negarse, ser concreto y ofrecer salida: decir el tamaño aproximado, pedir lo que falta, y proponer alternativas (agregados por período, una tabla de resumen si existe, una muestra acotada a un solo día).

### Verificar el plan cuando haya duda

Si el usuario insiste en una consulta que parece riesgosa, correr `EXPLAIN` antes de ejecutarla. Si el plan muestra `type: ALL` (recorrido completo de tabla) o un `rows` estimado desproporcionado, informarlo y proponer el ajuste antes de ejecutar.

Nota: en MySQL 8.0+ existe `EXPLAIN ANALYZE`, que **sí ejecuta la consulta de verdad**. No usarlo sobre una tabla protegida; `EXPLAIN` a secas alcanza.

## Verificar antes de asumir

Nunca inventar nombres de tablas o columnas; confirmarlos contra `information_schema` o el recurso `mysql://tables` primero.
