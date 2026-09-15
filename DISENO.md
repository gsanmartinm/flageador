# Flageador — diseño de herramienta (spec v1)

> Estado: **diseño**, sin implementar todavía. Este documento define alcance, algoritmo y estructura antes de escribir código, siguiendo el mismo patrón que **SW_Geomet** y **SW_Composita**.

## 1. Qué es

Flageador es una herramienta web de un solo archivo (`index.html`, sin backend, sin `npm install`) que resuelve un problema recurrente en geometalurgia: **"¿qué variables del dataset A le corresponden a cada punto del dataset B, según cercanía espacial?"**

Dos casos de uso, dos modos dentro de la misma app:

- **Modo 1 — Bloque → Muestra:** dado un modelo de bloques y una base de muestras con posición (X,Y,Z), encuentra el bloque cuyo centroide está más cerca del centroide de cada muestra, y copia variables seleccionadas del bloque a la muestra (más la distancia).
- **Modo 2 — Muestra ↔ Muestra:** dadas dos bases de muestras con posición, encuentra para cada muestra de la base A la muestra más cercana de la base B (por distancia pura, no por DHID/ID), reporta la distancia y traslada variables seleccionadas de B hacia A.

En ambos casos el resultado es una tabla exportable a Excel (.xlsx) o CSV: filas de la base "objetivo" + columnas trasladadas + distancia + advertencias.

## 2. Dónde encaja en el ecosistema

```
SW_Geomet ──────► visualiza bloques / sondajes / DXF en 3D
SW_Composita ───► desurveying + compositación → genera centroide (X,Y,Z) por tramo de muestra
Flageador ──────► toma esos centroides (u otra base de muestras) y los cruza espacialmente
                   contra un modelo de bloques u otra base de muestras
```

La salida típica de Composita (columnas `centroide_x/y/z`) es exactamente el tipo de input que espera el Modo 1 de Flageador — no es necesario que el usuario tenga las muestras ya desurveyadas por otra vía, pero si las tiene, encajan directo.

## 3. Stack técnico (igual que Composita)

- HTML + CSS + JS puro, un solo archivo, tema oscuro, mismas variables CSS y branding M2P (reutilizar el `<style>` de `06_SW_Composita/index.html` como base).
- [SheetJS (xlsx.js)](https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js) desde CDN — se usa no solo para exportar sino también para **leer `.xlsx`/`.xls` directamente** además de `.csv`/`.txt`/`.tsv`. Esto es una mejora respecto a Geomet/Composita (que solo leen CSV/TXT): los datos reales de ejemplo en `05_SW_Geomet/data/test_GMLC/*.xlsx` confirman que el input real muchas veces viene en Excel.
- Sin build, sin dependencias de servidor. Publicable en GitHub Pages igual que los otros dos.
- Corre 100% en el navegador: los archivos no se suben a ningún servidor.

## 4. Flujo de UI

Sigue el mismo patrón de "wizard" de pasos numerados que Composita:

**Selector de modo** (arriba de todo): dos tarjetas/tabs — "Modelo de bloques → Muestras" / "Muestras ↔ Muestras" — cambia qué tarjetas de carga se muestran debajo.

### Modo 1 — Bloque → Muestra

1. **Cargar modelo de bloques** (dropzone, csv/txt/xlsx) → mapeo de columnas: X, Y, Z del centroide (heurística: `midx/midy/midz`, `east/north/elev`, `x/y/z`, `coordx/coordy/coordz`). Checklist de columnas del bloque a trasladar a la muestra.
2. **Cargar base de muestras** (dropzone) → mapeo de columnas: X, Y, Z del centroide de la muestra (mismas heurísticas). Checklist de columnas de la muestra a mantener tal cual en el resultado (por defecto todas).
3. **Parámetros de flageo:** distancia máxima de búsqueda (radio, en las unidades del proyecto — default sin límite, o un valor sugerido según el tamaño de bloque detectado); qué hacer si dos bloques empatan en distancia (tomar el primero encontrado, se reporta como advertencia).
4. **Ejecutar** → tabla de resultados (muestra + variables de bloque + distancia) y tabla de "sin match" (muestras fuera del radio o sin bloques cerca). Exportar Excel/CSV.

### Modo 2 — Muestra ↔ Muestra

1. **Cargar base origen** (la que se "flagea") y **base destino** (la que aporta variables) — mismo dropzone + mapeo X/Y/Z para ambas.
2. Checklist de columnas de la base destino a trasladar; checklist de columnas de la base origen a mantener.
3. Parámetro de distancia máxima igual que Modo 1.
4. Ejecutar → misma estructura de salida: fila de la base origen + variables trasladadas de la base destino + distancia + ID de la muestra destino usada (trazabilidad) + advertencias.

Ambos modos comparten: tabla de resultados paginada/limitada a 1000 filas en pantalla (igual que Composita), tabs "Resultados" / "Sin match", cuadro resumen con estadísticas (total, flageados OK, flageados con advertencia, sin match), exportación a `.xlsx` (con hoja de omitidos) y `.csv`.

## 5. Algoritmo de vecino más cercano

Con modelos de bloques de cientos de miles de filas y bases de muestras de miles de filas, una búsqueda fuerza bruta (O(n·m)) es demasiado lenta para correr en el navegador de forma fluida. Se indexa el dataset "fuente" (bloques en Modo 1, base destino en Modo 2) en una **grilla espacial uniforme 3D**, adaptando `SpatialIndex3D` de `05_SW_Geomet/js/octree.js`:

1. Calcular el bounding box del dataset fuente y construir una grilla de N×N×N celdas (resolución ajustada según cantidad de puntos, para que cada celda tenga en promedio pocas decenas de puntos).
2. Insertar cada punto fuente en su celda.
3. Para cada punto de la base a flagear: partir en su propia celda y expandir en anillos concéntricos (radio de celdas creciente) hasta encontrar al menos un candidato; seguir expandiendo un anillo más para garantizar que no quede un vecino más cercano justo al otro lado de un borde de celda (patrón estándar de búsqueda de vecino más cercano sobre grilla uniforme).
4. Entre los candidatos recolectados, calcular distancia euclidiana 3D exacta y quedarse con el mínimo.
5. Si no aparece ningún candidato dentro del radio máximo configurado (o dentro de los límites del dataset), la fila queda marcada como "sin match".

Esto baja la complejidad práctica a ~O(n log n) para indexar + O(1) amortizado por consulta, en vez de O(n·m).

Distancia: euclidiana 3D por defecto (`sqrt(dx²+dy²+dz²)`); se deja como parámetro futuro (no v1) la opción de ponderar Z distinto a X/Y, útil cuando el espaciamiento vertical de bloques es muy distinto al horizontal.

## 6. Detección de columnas (heurística de nombres)

Extender el `NAME_HINTS` de Composita con las variantes vistas en los datos reales del proyecto:

| Campo | Hints adicionales a agregar |
|---|---|
| X | `midx`, `midx_vlcn`, `coordx`, `east`, `este` |
| Y | `midy`, `midy_vlcn`, `coordy`, `north`, `norte` |
| Z | `midz`, `midz_vlcn`, `coordz`, `elev`, `cota`, `rl` |

Igual que en Composita, la detección es solo una sugerencia editable — el usuario siempre puede corregir el mapeo manualmente por dropdown.

## 7. Estructura de salida

**Modo 1 (por muestra):**

| Columna | Contenido |
|---|---|
| (columnas de la muestra) | Las que el usuario eligió mantener tal cual |
| `bloque__<col>` | Cada variable de bloque seleccionada |
| `distancia_m` | Distancia euclidiana centroide muestra ↔ centroide bloque |
| `bloque_id` | Fila/índice del bloque usado (trazabilidad) |
| `advertencias` | Ej. "sin bloque dentro del radio máximo", "empate de distancia entre 2+ bloques" |

**Modo 2:** misma estructura, cambiando `bloque__` por `destino__` y agregando `id_muestra_destino` si existe una columna identificadora en la base destino.

## 8. Datos de prueba disponibles (en `05_SW_Geomet/data` y `06_SW_Composita/data`)

- **Modo 1:** `test_ANT/Block_Model_Mineralogy_PtXt_2021011.csv` (bloques, columnas `midx/midy/midz`) + `test_GMLC/MuestrasPFS.csv` (muestras, columnas `midx_vlcn/midy_vlcn/midz_vlcn`) — mismo estilo de datos que se va a flagear en producción.
- **Modo 1 alternativo:** `test_GMLC/FAS/BM_fas.csv` (bloques, columnas `EAST/NORTH/ELEV`, delimitador `;`) + `test_GMLC/FAS/MuestrasPFS_FAS.csv` (muestras).
- **Modo 2:** las variantes `MuestrasPFS.csv` vs `MuestrasPFS_FAS.csv` sirven como dos bases de muestras distintas para probar el cruce muestra↔muestra.
- Las bases `.xlsx` en `test_GMLC/BD_*.xlsx` son buen caso de prueba para la lectura directa de Excel (sin pasar por CSV primero).

## 9. Convenciones y límites (igual que Geomet/Composita)

- Convención geominera: X = Este, Y = Norte, Z = Elevación/RL.
- Corre enteramente en el navegador — el límite real depende de la RAM disponible en la pestaña.
- No hay persistencia entre sesiones ni sincronización entre usuarios.
- Requiere navegador moderno (Chrome/Edge/Firefox recientes).
- No hace matching por atributo/ID — es puramente espacial (centroide más cercano). Si el usuario necesita cruce por ID/DHID + traslape de intervalos, esa es la función de **Composita**, no de Flageador.

## 10. Fases de implementación

1. **Fase 1 (implementada):** Modo 1 (bloque → muestra) completo: carga CSV/TXT/Excel, mapeo de columnas con heurística de nombres, grilla espacial 3D con búsqueda de vecino más cercano, sugerencia automática de distancia máxima basada en el espaciamiento real de los bloques, tabla de resultados/sin-match, export a Excel/CSV.
2. **Fase 2 (implementada):** Modo 2 (muestra ↔ muestra) agregado como un segundo modo dentro del mismo `index.html`, con selector arriba de la página. Reutiliza el mismo motor `FLG.run` (base destino = dataset indexado, base origen = dataset flageado) y toda la UI genérica (dropzones, mapeo, checklists, tabla de resultados, export), sin duplicar el motor de cálculo.
3. **Fase 3 (implementada):** Modo 3 (bloque vs. topografía DXF) — ver sección 12 para el detalle completo. Agregado como tercer modo, con su propio motor (Web Worker) en vez de reutilizar `FLG.run`, porque el problema es geométricamente distinto (punto-en-triángulo + drape, no vecino-más-cercano).
4. **Fase 4 (mejoras futuras, no bloqueantes):** ponderación distinta de Z en la distancia de Modo 1/2, filtro de bloques por condición antes de indexar (ej. excluir bloques `categ = -99`), detección y reporte de empates, matching mutuo (nearest-neighbor bidireccional) como chequeo de calidad opcional en Modo 2. Para Modo 3: un segundo caso de flageo con DXF (mencionado por el usuario, aún no especificado en detalle) y soporte de múltiples topografías simultáneas (ej. techo/piso de una capa) en una sola pasada.

## 11. Estado actual

`index.html` en esta carpeta implementa Fase 1, Fase 2, Fase 3 y Modo 4. Probado con datos reales del proyecto: Modo 1/2 con `BM_fas.csv`/`MuestrasPFS_FAS.csv` (`test_GMLC/`) — motor de vecino más cercano validado contra fuerza bruta, integración end-to-end con las 198 muestras reales, wiring de UI del Modo 2 verificado con jsdom. Modo 3 probado con el motor de clasificación aislado en Node (ver sección 12.5): parser DXF, índice 2D de triángulos y clasificación aire/depósito, todos verificados contra fuerza bruta y contra la estructura real de `test_ANT/Ant_10_EOY_CD2025_2S2029.dxf`. **Modo 4 implementado y revisado manualmente línea por línea (ver sección 13.5); pendiente de pruebas unitarias automatizadas y de verificación end-to-end en navegador por un problema de infraestructura ajeno al código (ver 13.5).**

## 12. Modo 3 — Bloque vs. Topografía (DXF)

### 12.1 Problema y alcance

Dado un modelo de bloques y una topografía (superficie triangulada en DXF, entidades `3DFACE`), clasificar cada bloque como **aire** (su centroide queda sobre la topografía — ya no es parte del depósito) o **depósito** (su centroide queda bajo la topografía), agregando esa clasificación como nueva(s) columna(s) al modelo de bloques original, exportable en CSV o Excel. Clasificación por centroide únicamente (mismo criterio que Modo 1), no considera el volumen del bloque ni si este "atraviesa" la superficie.

### 12.2 Por qué un Web Worker (y no el motor síncrono de Modo 1/2)

El DXF de prueba real (`test_ANT/Ant_10_EOY_CD2025_2S2029.dxf`) tiene **~1.34 millones de entidades `3DFACE`** en un único layer (`10_EOY_CD2025_2S2029_3D_10`), lo que implica un archivo de varios cientos de MB — de una escala comparable o mayor al máximo (~650MB) que `05_SW_Geomet` soporta hoy mediante parsing en Web Worker con lectura en trozos (chunks). Un parser síncrono en el hilo principal (como el resto de Flageador) congelaría la pestaña o directamente fallaría con un archivo de este tamaño. Se optó, de forma consciente y consultada con el usuario, por replicar la arquitectura de Web Worker de Geomet (`js/dxf-parser.js` + `js/worker-parser.js`) en vez de un parser simple con limitación de tamaño documentada.

Como Flageador debe seguir siendo un único archivo `.html` autocontenido (sin `js/worker-parser.js` externo), el código del worker vive como texto plano dentro de un `<script id="dxfWorkerSrc" type="text/plain">` (nunca se ejecuta ahí) y se instancia en tiempo de ejecución como `new Worker(URL.createObjectURL(new Blob([...])))`. Esto preserva la ejecución en un hilo aparte sin necesitar un segundo archivo.

### 12.3 Parseo del DXF (dentro del worker)

Adaptado de `05_SW_Geomet/js/dxf-parser.js`, simplificado a **solo `3DFACE`** (se omiten `LINE`/`LWPOLYLINE`/`POLYLINE`, que no definen una superficie triangulada útil para esta clasificación). Lectura en trozos de 32MB vía `FileReaderSync` (disponible solo dentro de Workers), alimentando un parser incremental par-a-par (código de grupo + valor) que nunca materializa el archivo completo ni un array con todas sus líneas en memoria — mismo patrón y misma razón que Geomet (evitar los límites de tamaño de string de V8 y el uso excesivo de memoria de un array de decenas de millones de líneas).

**Decisión de precisión numérica:** los triángulos se guardan como `Float64Array`, no `Float32Array` (a diferencia de Geomet, que los usa directo en buffers de Three.js/WebGL). Con coordenadas UTM del orden de 7,500,000 m, `Float32` solo tiene resolución de ~0.5-1m en esa magnitud — suficiente para graficar, pero podría desplazar el borde de un triángulo o la cota interpolada en una cantidad no despreciable para una clasificación aire/depósito. `Float64` evita el problema al costo de ~2x memoria (irrelevante a esta escala: ~1.34M triángulos ocupan ~100MB como `Float64Array`).

Si el DXF trae más de una capa (layer) con entidades `3DFACE`, la UI lista todas ordenadas por cantidad de triángulos (la más grande, preseleccionada, suele ser la topografía real) y el usuario puede elegir cuál usar; las demás se ignoran.

### 12.4 Índice espacial 2D + clasificación (dentro del worker)

Para responder "¿qué triángulo contiene este punto (x,y)?" en O(1) amortizado en vez de recorrer los ~1.34M triángulos por cada bloque, se construye una grilla 2D (`TriGrid2D`) sobre el bounding box XY de los triángulos, usando un layout CSR (Compressed Sparse Row: `offsets` + `cellTriangles`) en vez de un array dinámico por celda, para minimizar la sobrecarga de memoria/GC a esta escala:

1. Se calcula el bounding box XY de todos los triángulos y se dimensiona una grilla de `res × res` celdas (`res` ajustado según la cantidad de triángulos, ~6 triángulos por celda en promedio).
2. Cada triángulo se registra en todas las celdas que toca su bounding box 2D (normalmente 1 celda, dado que los triángulos de una malla de topografía son pequeños respecto al tamaño de celda).
3. Para clasificar un bloque en (x,y): se ubica su celda y se expanden anillos concéntricos de celdas (mismo patrón de "cáscara" que la grilla 3D de Modo 1/2) hasta encontrar un triángulo que contenga el punto (test de coordenadas baricéntricas en 2D) o hasta cubrir toda la grilla (sin cobertura: el punto cae fuera de la extensión de la topografía o en un hueco de la malla).
4. La cota de la topografía en ese punto se interpola baricéntricamente sobre el plano del triángulo encontrado, y se compara con la cota del centroide del bloque: `aire` si el bloque queda sobre la topografía, `deposito` si queda bajo, `en_superficie` si la diferencia es menor o igual a una tolerancia opcional configurable (default 0 = no usar esta categoría), `sin_cobertura` si no hay ningún triángulo en esa posición XY.

Tanto el parseo del DXF como la clasificación de bloques corren como acciones separadas del mismo worker (una instancia de worker por acción, terminada al completar — mismo patrón que Geomet), reportando progreso (`postMessage({type:'progress', percent, message})`) para la barra de progreso de la UI.

### 12.5 Pruebas realizadas

Motor extraído a `dxf_engine.js` (Node, con `module.exports`) para pruebas unitarias aisladas del DOM/Worker:
- Parseo de un DXF ASCII sintético con una entidad `3DFACE` triangular (3 puntos) y una "cuadrada" (4 puntos → 2 triángulos), más una entidad `LINE` en otra capa que debe ignorarse sin romper el parser — todo correcto.
- Test punto-en-triángulo + interpolación baricéntrica de Z contra un triángulo plano inclinado conocido (`z = x`), incluyendo un punto exactamente en un vértice y un punto claramente fuera.
- `TriGrid2D` vs. fuerza bruta sobre una malla regular de ~3,200 triángulos (techo de dos aguas por celda) y 500 consultas aleatorias: 0 discrepancias. Verificación adicional de que la Z interpolada coincide con la función analítica del plano en 200 puntos.
- `classifyBlocks` end-to-end: aire, depósito, en_superficie (con tolerancia) y sin_cobertura, todos correctos, incluyendo que `topoZ` sea `NaN` cuando no hay cobertura.
- Rendimiento: ~100,000 triángulos indexados en 17ms; 5,000 consultas de clasificación en 4ms (0.0008 ms/consulta) — a la escala real del DXF de prueba (~1.34M triángulos, ~171,764 bloques) esto proyecta un tiempo total del orden de unos pocos segundos, no minutos.
- Estructura real verificada directamente sobre `test_ANT/Ant_10_EOY_CD2025_2S2029.dxf`: formato de entidad `3DFACE` (handle, owner, capa, color, vértices) idéntico al asumido por el parser, incluyendo bloques de datos extendidos (XDATA, códigos 1000/1001: Material, Label, Element name, File Name, Object Name) que el parser ignora correctamente sin desincronizarse; capa única (`10_EOY_CD2025_2S2029_3D_10`) confirmada en tres puntos distintos del archivo; rango de coordenadas reales (X ~407,000–414,076; Y ~7,493,430–7,505,454; Z real por vértice ~1,670–1,700 en la muestra inspeccionada) consistente con el modelo de bloques de prueba (`Block_Model_Mineralogy_PtXt_2021011.csv`, 171,763 bloques, columnas `midx/midy/midz` reconocidas automáticamente por la misma heurística de nombres que Modo 1).
- Pendiente (no realizable en este entorno): una corrida end-to-end real dentro de un navegador con el archivo DXF completo (~1.34M triángulos) y el modelo de bloques completo, dado que este entorno de desarrollo no tiene acceso a un navegador real ni al archivo montado en el sandbox de pruebas. Se recomienda al usuario probar Modo 3 directamente en `index.html` con los archivos de `test_ANT/` como primera verificación real.

## 13. Modo 4 — Bloque vs. Volúmenes DXF

### 13.1 Problema y alcance

Dado un modelo de bloques y uno o más volúmenes cerrados (sólidos triangulados en DXF, entidades `3DFACE`), determinar para cada bloque si su centroide queda **dentro** o **fuera** de cada volumen, agregando una columna `dentro__<nombre_volumen>` por cada volumen cargado más una columna resumen (nombre configurable, default `volumenes_dxf`) que lista los nombres de los volúmenes que contienen a ese bloque (o una etiqueta configurable, default `ninguno`, si no está dentro de ninguno). A diferencia del Modo 3 (que clasifica contra una superficie abierta tipo "manto"), aquí el DXF representa un sólido cerrado (una veta, un cuerpo mineralizado, un pit de diseño, etc.) y la pregunta es de contención 3D, no de "arriba/abajo" de una superficie.

Se decidió, consultado con el usuario, soportar **varios volúmenes a la vez** en una sola corrida: el usuario carga N archivos DXF (uno por volumen) y el resultado incluye la clasificación contra cada uno más el resumen combinado, en vez de tener que repetir el proceso volumen por volumen.

### 13.2 Por qué no basta con el índice 2D del Modo 3

El Modo 3 resuelve "¿qué altura tiene la topografía en (x,y)?" — una consulta 2D con una única respuesta por columna XY, adecuada para una superficie (cada XY tiene una sola Z de techo). La pregunta de Modo 4 es distinta: "¿este punto 3D está dentro de un sólido cerrado?", que no se puede reducir a una sola consulta 2D porque el sólido puede tener múltiples caras superpuestas en la misma columna XY (paredes verticales, cavidades, geometría no convexa).

La técnica estándar para este problema es **ray casting / point-in-polyhedron**: desde el punto de consulta se lanza un rayo en una dirección arbitraria y se cuenta cuántas veces cruza la superficie del sólido; si el conteo es impar, el punto está dentro (regla de paridad). Esto requiere intersección rayo-triángulo (no punto-en-triángulo-2D como en Modo 3) y es válido para cualquier sólido cerrado y watertight, sin importar su forma.

**Riesgo de un solo rayo:** si el rayo pasa exactamente rasante a una arista o cara del mesh (coplanar u "grazing"), el conteo de intersecciones puede quedar mal definido (ni claramente par ni impar) por errores de redondeo de punto flotante — un riesgo real en mallas mineras que suelen tener bancos horizontales y paredes verticales, es decir, caras alineadas con los ejes en ambas orientaciones. Un rayo puramente `+Z` es especialmente vulnerable contra bancos horizontales; un rayo puramente `+X` o `+Y` lo es contra paredes verticales alineadas a esos ejes.

**Solución adoptada — votación por mayoría de 3 rayos:** se lanzan 3 rayos por punto, cada uno mayormente alineado a un eje distinto pero con una pequeña inclinación no-axis-aligned (`RAY_Z ≈ (0.057, -0.081, 1)` normalizado, y análogos para `RAY_X`/`RAY_Y`), de forma que ningún rayo sea exactamente paralelo a las caras típicas de una malla minera. El punto se considera "dentro" si al menos 2 de los 3 rayos reportan paridad impar. Esto tolera que un rayo individual falle en un caso degenerado puntual sin afectar el resultado final.

### 13.3 Intersección rayo-triángulo (Möller–Trumbore)

Implementada directamente (`rayTriangleIntersect`) siguiendo el algoritmo estándar de Möller–Trumbore: expresa el punto de intersección en coordenadas baricéntricas (u, v) del triángulo más la distancia `t` a lo largo del rayo, descartando la intersección si el rayo es casi paralelo al plano del triángulo (`|det| < epsilon`), si (u, v) caen fuera del triángulo, o si `t` es negativo o ~0 (intersección detrás del origen del rayo o exactamente en él — se usa `t > 1e-7` como umbral para evitar contar dos veces una intersección que cae justo en un vértice/arista compartido por dos triángulos).

### 13.4 Índice espacial generalizado (`AxisGrid2D`) + `buildVolumeIndex`

Para no probar los ~5,000+ triángulos de cada volumen contra cada rayo de cada bloque, se generalizó la grilla CSR 2D del Modo 3 (`TriGrid2D`, fija a los ejes X/Y) a `AxisGrid2D(triangles, axisA, axisB)`, parametrizable sobre cualquier par de ejes. Como los 3 rayos de la votación viajan mayormente en direcciones distintas, cada uno necesita proyectar candidatos sobre un plano distinto:

- Rayo mayormente `+Z` → candidatos por proyección XY (`gridXY`, igual que Modo 3).
- Rayo mayormente `+X` → candidatos por proyección YZ (`gridYZ`).
- Rayo mayormente `+Y` → candidatos por proyección XZ (`gridXZ`).

`buildVolumeIndex(triangles)` calcula el bounding box real del volumen (nunca los `$EXTMIN`/`$EXTMAX` del header DXF, que en los archivos de prueba `test_MINSUR` son valores centinela `±1e20` — confirmando que la extensión siempre debe derivarse de los vértices reales, mismo criterio ya aplicado en Modo 3) y construye las 3 grillas. Como cada rayo tiene una leve inclinación respecto a su eje dominante, un punto de consulta puede desviarse lateralmente varias celdas a lo largo del recorrido del rayo dentro del volumen; `padFor` calcula dinámicamente cuántas celdas de radio hay que buscar alrededor de la celda inicial (proporcional a la inclinación del rayo y a la extensión del volumen en su eje dominante, acotado entre 1 y 40) para no perder triángulos candidatos por los que el rayo efectivamente pasa.

**Deduplicación sin asignar memoria por consulta:** un mismo triángulo puede quedar registrado en varias celdas superpuestas del vecindario buscado. En vez de crear un `Set` nuevo por cada consulta (costoso a la escala de cientos de miles de bloques), se usa un buffer compartido `visited` (`Int32Array` del tamaño del total de triángulos) junto con un contador `epoch` que se incrementa en cada consulta; un candidato ya visto en la consulta actual tiene `visited[idx] === epoch`.

### 13.5 Pruebas realizadas y pendientes

Se extendió `dxf_engine.js` (motor Node del Modo 3) con `AxisGrid2D`, `rayTriangleIntersect`, `buildVolumeIndex`, `countParity`, `pointInMesh` y `classifyBlocksInVolumes`, y se verificó su sintaxis (`node --check`) exitosamente.

**Estructura real verificada** directamente sobre `test_MINSUR/ms-cu40n-I.dxf` y `ms-cu40n-II.dxf`: versión DXF `AC1012` (AutoCAD R13), entidades `3DFACE` en capa única `"0"` (sin capas nombradas ni XDATA, a diferencia de `test_ANT`), confirmando que el parser existente (ya simplificado a solo `3DFACE`) funciona sin cambios. Volúmenes de tamaño moderado (miles de triángulos cada uno), muy por debajo de la escala de `test_ANT`, sin riesgo de performance.

**Pendiente por un problema de infraestructura, no de diseño:** el sandbox de shell (`bash`) de este entorno dejó de funcionar durante esta sesión (falla de montaje de archivos, reportada como una regresión conocida y rastreada, causada por una actualización de Windows del 8 de septiembre — no relacionada con el código de Flageador) y no se recuperó pese a reintentos espaciados en distintos momentos de la sesión. Como consecuencia, no fue posible ejecutar en este entorno:

- Las pruebas unitarias planeadas para el motor de punto-en-malla (cubo y tetraedro sintéticos con puntos dentro/fuera conocidos, comparación contra fuerza bruta, prueba de robustez de rayos degenerados/rasantes, prueba de rendimiento a la escala real de `test_MINSUR`).
- Una verificación funcional real (ej. que el centroide del bounding box de un volumen clasifique como "dentro" de sí mismo) contra los archivos reales de `test_MINSUR`.

Todo el código nuevo (worker: `AxisGrid2D`, `rayTriangleIntersect`, `buildVolumeIndex`, `countParity`, `pointInMesh`, `doClassifyVolumes`; UI: `state4`, carga multi-archivo de volúmenes, corrida, resultados, exportación) fue revisado manualmente línea por línea como sustituto parcial de las pruebas automatizadas, sin encontrar errores — pero esta revisión no reemplaza una corrida real. **Se recomienda al usuario probar Modo 4 directamente en `index.html` con los archivos de `test_MINSUR/` como primera verificación real**, antes de usarlo con datos de producción.
