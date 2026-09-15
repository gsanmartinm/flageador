# Flageador

Herramienta web de un solo archivo (`index.html`), sin backend y sin instalación, para "flagear" datos espacialmente cercanos en minería/geología: cruzar un modelo de bloques con muestras, cruzar dos bases de muestras entre sí, o clasificar un modelo de bloques contra una topografía.

Desarrollado por **Mine to Port Consulting (M2P)**. Parte del mismo ecosistema que [SW_Geomet](../05_SW_Geomet) y [SW_Composita](../06_SW_Composita).

100% en el navegador — ningún archivo se sube a un servidor. No requiere `npm install` ni build: se abre el `.html` directamente o se sirve estático (ej. GitHub Pages).

## Modos

**1. Modelo de bloques → Muestras**
Dado un modelo de bloques y una base de muestras con posición (X, Y, Z), encuentra para cada muestra el bloque cuyo centroide está más cerca y traslada las variables seleccionadas del bloque a la muestra, junto con la distancia.

**2. Muestras ↔ Muestras**
Dadas dos bases de muestras (origen y destino), encuentra para cada fila de la base origen la fila más cercana de la base destino (por distancia espacial, no por ID) y traslada variables seleccionadas, junto con la distancia.

**3. Bloques vs. Topografía (DXF)**
Dado un modelo de bloques y una topografía triangulada (DXF, entidades `3DFACE`), clasifica cada bloque como "aire" (sobre la topografía) o "depósito" (bajo la topografía), agregando la clasificación y la cota interpolada como nuevas columnas al modelo de bloques. El parseo del DXF y la clasificación corren en un Web Worker embebido, para no bloquear el navegador con topografías de cientos de MB / millones de triángulos.

**4. Bloques vs. Volúmenes DXF**
Dado un modelo de bloques y uno o más sólidos cerrados en DXF (entidades `3DFACE`, ej. una veta, un cuerpo mineralizado o un pit de diseño), determina para cada bloque si su centroide queda dentro o fuera de cada volumen (por ray casting / point-in-polyhedron), agregando una columna `dentro__<volumen>` por cada volumen cargado más una columna resumen con los nombres de los volúmenes que lo contienen. Soporta cargar varios volúmenes a la vez en una sola corrida.

Los cuatro modos aceptan CSV, TXT o Excel (.xlsx/.xls) para las bases de datos tabulares, y exportan el resultado a Excel (.xlsx) o CSV.

## Uso

- **Local:** descarga `index.html` y ábrelo con doble clic en cualquier navegador moderno (Chrome, Edge, Firefox).
- **En línea:** publicado en GitHub Pages en `https://gsanmartinm.github.io/flageador/`.

## Detección de columnas

El mapeo de columnas X/Y/Z (y otras) se sugiere automáticamente por nombre (`midx`, `east`, `coordx`, `x`, etc.) pero siempre es editable manualmente antes de procesar.

## Límites

- El procesamiento corre íntegro en la pestaña del navegador; el límite práctico depende de la RAM disponible.
- El cruce es puramente espacial (centroide o cota más cercana), no por ID/DHID.
- No hay persistencia entre sesiones: cada carga de archivo es independiente.

## Documentación

Ver [`DISENO.md`](./DISENO.md) para el detalle de diseño, algoritmos (grilla espacial 3D para vecino más cercano, grilla 2D + baricéntricas para el drape de topografía) y estado de pruebas.
