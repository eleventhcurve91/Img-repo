## **Inicialización del Enlace MCP con el Informe Activo**

Con el proyecto `EcoBici_Lima.pbip` guardado y abierto en Power BI Desktop (sección anterior) y los servidores `powerbi-modeling-mcp` y `archivos` en estado **Running** (sección *Reinicio del Servicio*), se establece la sesión de control con el agente de IA.

Todo el desarrollo del informe se realiza en **un único chat** de Claude Desktop: los prompts de esta guía se pegan uno tras otro, en orden, esperando a que Claude termine cada respuesta. Las reglas que se dan al inicio se mantienen para todo el chat, por eso los últimos prompts son más cortos.

1. En Claude Desktop, iniciar un **nuevo chat**.

2. En el icono de herramientas (**+**) junto al cuadro de texto, dejar activos solo los conectores `powerbi-modeling-mcp` y `archivos` (desactivar búsqueda web y otros conectores).

3. Pegar el prompt de conexión:

### **PROMPT 1 (Establecimiento de Conexión y Reglas de Trabajo):**
```text
Vamos a construir en este chat un informe de Power BI completo usando solo las herramientas MCP (powerbi-modeling-mcp y archivos). Reglas para todo el chat: trabaja directo, no uses la operación Help, agrupa en una sola llamada todo lo que la herramienta permita (listas), no muestres código ni JSON y responde siempre en pocas líneas.
Conéctate por MCP al informe de Power BI Desktop que tengo abierto (EcoBici_Lima) y responde solo con el nombre del modelo y el puerto.
```

4. Cuando Claude solicite permiso para usar la herramienta MCP, pulsar **Permitir siempre**.

5. Verificar que Claude responde con el nombre del modelo y el puerto local de Analysis Services.

> **Fundamento de Ingeniería:**
> Power BI Desktop levanta en segundo plano una instancia local de Analysis Services en un puerto dinámico; el servidor MCP la localiza y se conecta a la base de datos tabular activa. Trabajar en un solo chat conserva el contexto (reglas, nombres del modelo, formato PBIR) y agrupar operaciones reduce el número de llamadas a las herramientas.

<!--CELL-->

## **Auditoría Técnica y Perfilado de Datos de Origen**

Antes de articular transformaciones en el motor de Power Query, se ejecuta una auditoría sobre el directorio `Datos_Fuente` para identificar tipologías de datos, anomalías y requerimientos de estandarización.

1. En el mismo chat, pegar el prompt:

### **PROMPT 2 (Perfilado y Diagnóstico de Calidad):**
```text
Con la herramienta de archivos lee ÚNICAMENTE Datos_Fuente\Estaciones_Centro.csv, Estaciones_Sur.csv y Usuarios.csv (separador ";"). Lo que ya sé de los Excel (no los abras):
- Viajes.xlsx: hojas 2023, 2024, 2025. Columnas IdViaje, InicioViaje, FinViaje (texto "dd/mm/aaaa hh:mm"), DuracionMin, IdEstacionOrigen, IdEstacionDestino, IdBicicleta, IdUsuario, DistanciaKm, Notas.
- Bicicletas.xlsx (hoja Bicicletas): IdBicicleta, CodigoBici, Tipo (Eléctrica/Mecánica), Modelo, FechaAdquisicion, RegistroInterno.
- Mapa_Zonas.xlsx (hoja Zonas): Distrito, Zona.
Responde con un resumen de calidad de datos en máximo 10 viñetas (problema → transformación propuesta) y el esquema en estrella propuesto en 3 líneas.
```

2. Revisar que el diagnóstico identifique:
   * La estructura multi-hoja de `Viajes.xlsx` (una hoja por año) y las fechas en texto `dd/mm/aaaa hh:mm`.
   * El `Distrito` de las estaciones con espacios y mayúsculas distintas (`MIRAFLORES`, `  Cercado de Lima `), que debe cruzarse con `Mapa_Zonas`.
   * El `Genero` con códigos (`M`, `F`) y vacíos (90 de 500 usuarios), que requiere el valor *No especificado*.
   * Las columnas sobrantes `Notas` y `RegistroInterno`.

> **Fundamento de Ingeniería:**
> Los archivos `.xlsx` son binarios y el servidor `archivos` solo lee texto; por eso el prompt entrega su estructura y Claude perfila directamente los CSV. Detectar los problemas de calidad antes de modelar evita reprocesar tablas en Power Query.

<!--CELL-->

## **Construcción Programática del Modelo Dimensional (Esquema en Estrella)**

Se ejecuta la instrucción de modelado semántico que genera el parámetro de entorno `RutaDatos`, procesa las tablas de hechos y dimensiones, y articula la topología de relaciones tabulares.

1. En el mismo chat, pegar el prompt:

### **PROMPT 3 (Generación de Modelo, Parámetro y Relaciones):**
```text
Genera el modelo de datos en el Power BI abierto:
1) Parámetro de Power Query "RutaDatos" (texto, obligatorio) = ruta absoluta de la carpeta Datos_Fuente terminada en "\" (obtenla con list_allowed_directories de la herramienta de archivos). Todas las tablas leen desde RutaDatos.
2) fctViajes (M): Excel.Workbook(File.Contents(RutaDatos & "Viajes.xlsx"), true), filtra Kind = "Sheet", Table.Combine de las hojas; quita Notas (MissingField.Ignore); InicioViaje y FinViaje: si son texto DateTime.FromText(_, [Format="dd/MM/yyyy HH:mm"]), tipo datetime; enteros IdViaje, DuracionMin, IdEstacionOrigen, IdEstacionDestino, IdBicicleta, IdUsuario; DistanciaKm decimal con cultura "en-US"; añade FechaInicio = DateTime.Date([InicioViaje]) (date) y HoraInicio = Time.Hour([InicioViaje]) (entero).
3) dimEstaciones (M): Csv.Document de Estaciones_Centro.csv y Estaciones_Sur.csv (Delimiter=";", Encoding=65001), PromoteHeaders, Table.Combine. IdEstacion y Capacidad enteros, NombreEstacion con Text.Trim, Latitud y Longitud decimales con cultura "en-US". Cruza con Mapa_Zonas.xlsx (hoja Zonas) usando la clave Text.Lower(Text.Trim(Distrito)) en ambos lados (LeftOuter); deja Distrito con la escritura de Mapa_Zonas y añade Zona; quita auxiliares. Latitud dataCategory Latitude y Longitud dataCategory Longitude, resumen Average.
4) dimUsuarios (M) desde Usuarios.csv (";"): IdUsuario y AnioNacimiento enteros; Genero: Text.Upper(Text.Trim) "M"/"MASCULINO" = "Masculino", "F"/"FEMENINO" = "Femenino", resto o vacío = "No especificado"; RangoEdad con Edad = 2025 - AnioNacimiento: <=24 "18-24", <=34 "25-34", <=44 "35-44", resto "45+", null "No especificado".
5) dimBicicletas (M) desde Bicicletas.xlsx (hoja Bicicletas, encabezados en la primera fila): sin RegistroInterno; IdBicicleta entero; FechaAdquisicion date.
6) dimFecha (tabla calculada DAX):
VAR T = CALENDAR(DATE(YEAR(MIN(fctViajes[FechaInicio])),1,1), DATE(YEAR(MAX(fctViajes[FechaInicio])),12,31))
RETURN SELECTCOLUMNS(T, "Fecha", [Date], "Anio", YEAR([Date]), "MesNum", MONTH([Date]), "Mes", MID("EneFebMarAbrMayJunJulAgoSepOctNovDic", (MONTH([Date])-1)*3+1, 3), "Trimestre", "T" & QUARTER([Date]), "AnioMes", FORMAT([Date], "yyyy-mm"))
Fecha tipo date, Mes ordenado por MesNum, Anio y MesNum sin resumir. Márcala como tabla de fechas (columna Fecha).
7) Actualiza las tablas y crea las relaciones (muchos a uno, una dirección): fctViajes[IdEstacionOrigen] → dimEstaciones[IdEstacion] ACTIVA; fctViajes[IdEstacionDestino] → dimEstaciones[IdEstacion] INACTIVA; fctViajes[IdBicicleta] → dimBicicletas[IdBicicleta]; fctViajes[IdUsuario] → dimUsuarios[IdUsuario]; fctViajes[FechaInicio] → dimFecha[Fecha]. Oculta las columnas Id* de fctViajes.
Notas para las herramientas: crea RutaDatos con named_expression_operations Create (expresión M "...ruta..." meta [IsParameterQuery=true, Type="Text", IsParameterQueryRequired=true]); al crear tablas M con table_operations define explícitamente todas sus columnas (nombre y dataType) porque no se infieren; crea las 4 tablas M en una sola llamada y después dimFecha.
Si algo falla, corrígelo tú. Responde solo con: filas de cada tabla y relaciones (activa/inactiva).
```

2. Verificar las filas que reporta Claude: `fctViajes` 6.200, `dimEstaciones` 24, `dimUsuarios` 500, `dimBicicletas` 120 y `dimFecha` 1.096.

3. Si Power BI muestra la barra *"One or more relationships have been modified and need to be refreshed manually"*, pulsar **Refresh now**.

### **Validación de Arquitectura en Power BI:**

4. Cambiar a la **Vista de modelo** (tercer icono de la barra lateral izquierda) y arrastrar las tablas para dejar `fctViajes` en el centro.

5. Verificar que `fctViajes` se conecta con cardinalidad de 1 a varios (`1:*`) a las cuatro dimensiones y que la relación `IdEstacionDestino → dimEstaciones` aparece como **línea punteada** (relación inactiva).
> ![Vista de modelo con el esquema en estrella](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-02-modelo-estrella.png)

6. Guardar con `Ctrl + S`.

> **Fundamento de Ingeniería:**
> El esquema en estrella separa los **hechos** (`fct`, cada viaje) de las **dimensiones** (`dim`: quién, qué, dónde y cuándo). Las relaciones `1:*` propagan el filtro desde la dimensión hacia los hechos. Entre dos tablas solo puede existir una relación activa; la de estación de destino queda inactiva y se activa dentro de una medida con `USERELATIONSHIP`. Marcar `dimFecha` como tabla de fechas habilita la inteligencia de tiempo (`TOTALYTD`, `DATEADD`, `DATESINPERIOD`), y el parámetro `RutaDatos` permite reubicar la carpeta de datos cambiando un solo valor.

<!--CELL-->

## **Implementación y Organización de Medidas DAX en Tabla Dedicada**

Siguiendo las directrices de diseño de Business Intelligence, se centralizan todas las medidas de cálculo en un contenedor independiente (`_Medidas`), clasificadas en carpetas de visualización (*Display Folders*). El prompt incluye el DAX completo para que Claude solo lo cree.

1. En el mismo chat, pegar el prompt:

### **PROMPT 4 (Creación y Jerarquización de Medidas DAX):**
```text
Crea una tabla calculada "_Medidas" = {BLANK()} con su columna oculta, y en ella estas medidas EXACTAS en UNA sola llamada de measure_operations (displayFolder entre corchetes; formato después de "|"):
[Base]
Total Viajes = COUNTROWS(fctViajes) | #,0
Distancia Total (km) = SUM(fctViajes[DistanciaKm]) | #,0
Distancia Promedio (km) = AVERAGE(fctViajes[DistanciaKm]) | #,0.00
Duración Promedio (min) = AVERAGE(fctViajes[DuracionMin]) | #,0.0
Usuarios Activos = DISTINCTCOUNT(fctViajes[IdUsuario]) | #,0
Bicicletas en Uso = DISTINCTCOUNT(fctViajes[IdBicicleta]) | #,0
Viajes por Usuario = DIVIDE([Total Viajes], [Usuarios Activos]) | #,0.0
[Avanzadas]
% Viajes Eléctricos = DIVIDE(CALCULATE([Total Viajes], dimBicicletas[Tipo] = "Eléctrica"), [Total Viajes]) | 0.0%
Viajes por Destino = CALCULATE([Total Viajes], USERELATIONSHIP(fctViajes[IdEstacionDestino], dimEstaciones[IdEstacion])) | #,0
Viajes Año Anterior = VAR A = YEAR(MAX(fctViajes[FechaInicio])) RETURN CALCULATE([Total Viajes], dimFecha[Anio] = A - 1) | #,0
Var. Viajes vs AA = VAR A = YEAR(MAX(fctViajes[FechaInicio])) VAR Act = CALCULATE([Total Viajes], dimFecha[Anio] = A) VAR Ant = [Viajes Año Anterior] RETURN IF(NOT ISBLANK(Ant), Act - Ant) | +#,0;-#,0;0
Var. % Viajes vs AA = VAR A = YEAR(MAX(fctViajes[FechaInicio])) VAR Ant = [Viajes Año Anterior] RETURN DIVIDE(CALCULATE([Total Viajes], dimFecha[Anio] = A) - Ant, Ant) | +0.0%;-0.0%;0.0%
[Tendencias]
Viajes Acumulados Año = TOTALYTD([Total Viajes], dimFecha[Fecha]) | #,0
Viajes Media Móvil 3M = CALCULATE(AVERAGEX(VALUES(dimFecha[AnioMes]), [Total Viajes]), DATESINPERIOD(dimFecha[Fecha], MAX(dimFecha[Fecha]), -3, MONTH)) | #,0
Viajes Mes Anterior = CALCULATE([Total Viajes], DATEADD(dimFecha[Fecha], -1, MONTH)) | #,0
Var. % Viajes vs Mes Anterior = VAR U = MAX(fctViajes[FechaInicio]) VAR Ini = DATE(YEAR(U), MONTH(U), 1) VAR Act = CALCULATE([Total Viajes], DATESBETWEEN(dimFecha[Fecha], Ini, EOMONTH(U, 0))) VAR Ant = CALCULATE([Total Viajes], DATESBETWEEN(dimFecha[Fecha], EDATE(Ini, -1), Ini - 1)) RETURN DIVIDE(Act - Ant, Ant) | +0.0%;-0.0%;0.0%
Si "Eléctrica" no coincide con los valores de dimBicicletas[Tipo], ajústalo. Responde solo con el valor sin filtros de Total Viajes, % Viajes Eléctricos y Var. % Viajes vs AA.
```

2. Verificar los valores de control: **Total Viajes 6,200**, **% Viajes Eléctricos 34.1%** y **Var. % Viajes vs AA +9.5%** (2025 frente a 2024).

3. Volver a la **Vista de informe** (primer icono de la barra lateral izquierda) y, en el panel **Data**, expandir `_Medidas` y sus carpetas **Avanzadas**, **Base** y **Tendencias**.
> ![Panel Data con la tabla _Medidas y sus carpetas](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-panel-medidas.png)

### **Protocolo de Sincronización:**

4. Guardar con `Ctrl + S` (no hace falta cerrar Power BI).

> **Fundamento de Ingeniería:**
> Una tabla dedicada separa la lógica de cálculo de los datos, y las carpetas agrupan las medidas por propósito. Las medidas de año anterior toman el **último año con datos del contexto** (`YEAR(MAX(...))`): sin filtros comparan 2025 contra 2024 en lugar de sumar todos los años. `CALCULATE` modifica el contexto de filtro, `USERELATIONSHIP` activa la relación inactiva solo dentro de la medida y `DIVIDE` evita errores por división entre cero.

<!--CELL-->

## **Sistema de Diseño Institucional y Especificaciones de Maquetación**

Se aplican los tokens de diseño institucional (`Tema_EcoBici.json`) y la grilla base para una resolución Full HD (1920 x 1080 px). El diseño se construye **una sola vez** en la página "Resumen" (panel lateral, 5 segmentadores, cabecera y tarjeta de KPIs) y luego se duplica para las otras dos páginas, garantizando consistencia cromática, espaciado y navegación transversal.

### **Aplicación del Tema Institucional:**

1. En Power BI, abrir la pestaña **Design** (*Diseño*) y pulsar **Import theme** (*Importar tema*).
> ![Pestaña Design con la opción Import theme](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-importar-tema.png)

2. Navegar a la carpeta `Datos_Fuente`, seleccionar `Tema_EcoBici.json` y pulsar **Abrir**.
> ![Selección del archivo Tema_EcoBici.json](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-seleccionar-tema.png)

3. Confirmar el mensaje **File successfully added** con **Got it** y guardar con `Ctrl + S`.
> ![Confirmación de tema importado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-tema-aplicado.png)

### **Maquetación de la Página Base:**

4. En el mismo chat de Claude, pegar el prompt:

### **PROMPT 5 (Layout Base: Panel Lateral, Segmentadores, Cabecera y KPIs):**
```text
Ahora el informe. Usa solo la herramienta de archivos sobre la carpeta EcoBici_Lima.Report\definition del proyecto (búscala con la herramienta de archivos; lee solo pages\pages.json y el page.json existente). Reglas PBIR para este y los próximos prompts: cada visual va en pages\<pagina>\visuals\<nombre>\visual.json con "$schema" visualContainer 2.x, "name" igual a la carpeta, "position" {x,y,z,width,height}. Columnas {"Column":{"Expression":{"SourceRef":{"Entity":"<tabla>"}},"Property":"<col>"}} y medidas {"Measure":{"Expression":{"SourceRef":{"Entity":"_Medidas"}},"Property":"<medida>"}}, queryRef "<tabla>.<campo>". Gráficos: título visible en negrita, sin subtítulo. Textboxes y shapes: sin fondo, borde, sombra ni título.
1) La página existente: displayName "Resumen", 1920 x 1080, fondo #EEF2F6.
2) Panel: shape "panelFondo" x0 y0 300x1080 relleno #1B4965; shape "panelFranja" x0 y0 300x6 relleno #F6AE2D.
3) Textboxes: "txtMarca" "EcoBici Lima" blanco 26px negrita (x24 y30 w252 h40); "txtLema" "Red de bicicletas públicas" #BEE3F8 13px (x24 y72 w252 h24); "txtFiltros" "FILTROS" #F6AE2D 12px negrita (x24 y130 w252 h24).
4) Slicers Dropdown con título, x24 w252 h80: "slAnio" y170 "Año" dimFecha[Anio]; "slMembresia" y266 "Tipo de membresía" dimUsuarios[TipoMembresia]; "slZona" y362 "Zona de origen" dimEstaciones[Zona]; "slTipoBici" y458 "Tipo de bicicleta" dimBicicletas[Tipo]; "slGenero" y554 "Género" dimUsuarios[Genero]. Cada uno con "syncGroup": {"groupName": "<columna>", "fieldChanges": true, "filterChanges": true} dentro de "visual".
5) Cabecera: "txtTitulo" "EcoBici Lima · Resumen Operacional" #1B4965 30px Segoe UI Semibold (x324 y24 w1572 h44); "txtSubtitulo" "Desempeño de la red de bicicletas públicas 2023-2025" #616E7C 15px (x324 y70 w1572 h26); shape "lineaAcento" x324 y102 96x5 relleno #F6AE2D.
6) cardVisual "kpis" x324 y131 w1572 h150 sin título: Total Viajes, Usuarios Activos, Distancia Total (km), Duración Promedio (min), % Viajes Eléctricos; valores #1B4965 30pt, displayUnits 1 (6,200 y no 6K), barra de acento #F6AE2D a la izquierda de cada valor.
Responde solo con "Listo".
```

### **Protocolo de Sincronización:**

5. Cuando Claude responda "Listo", Power BI muestra la barra *"This project's files were changed externally"*. Pulsar **Apply external changes** (*Aplicar cambios externos*). Este mismo procedimiento se repite después de cada prompt de páginas.
> ![Barra Apply external changes en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-aplicar-cambios-externos.png)

6. Si aparece el aviso **Overwrite your unsaved edits**, pulsar **Apply external changes**. Indica que había cambios sin guardar en Power BI; para evitarlo, guardar con `Ctrl + S` antes de pegar cada prompt y no guardar mientras Claude está escribiendo.
> ![Aviso Overwrite your unsaved edits](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-sobrescribir-cambios.png)

7. Revisar la página: panel lateral azul con los 5 segmentadores, cabecera y tarjeta de KPIs con los valores completos (6,200 y no 6K).
> ![Página base Resumen con panel lateral, cabecera y KPIs](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-04-diseno-base.png)

### **Sincronización y Duplicado de Páginas:**

8. Seleccionar el segmentador **Año** y abrir **View → Sync slicers** (*Ver → Sincronizar segmentaciones*) para comprobar el grupo de sincronización.
> ![Panel Sync slicers con el segmentador Año sincronizado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-sincronizar-segmentadores.png)

9. Clic derecho sobre la pestaña **Resumen** → **Duplicate** (*Duplicar*). Repetir una segunda vez.
> ![Menú contextual de la pestaña con la opción Duplicate](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-duplicar-pagina.png)

10. Doble clic sobre cada copia para renombrarlas como **Análisis de Uso** y **Tendencias**, y guardar con `Ctrl + S`.
> ![Pestañas Resumen, Análisis de Uso y Tendencias](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-pestanas.png)

> **Fundamento de Ingeniería:**
> En formato PBIR cada página es una carpeta y cada visual un archivo `visual.json`: Claude escribe esos archivos con el servidor `archivos` y Power BI los incorpora con **Apply external changes**, sin cerrar el programa. El tema JSON centraliza colores, tipografías, bordes y sombras de todos los visuales. El `syncGroup` vincula los segmentadores de distintas páginas: al duplicar la página, las copias conservan el mismo grupo y el filtro se comparte en todo el informe.

<!--CELL-->

## **Construcción de la Página: Resumen Operacional**

Despliegue de la primera vista analítica del informe, enfocada en los macroindicadores de operación y distribución espacial de la demanda. La cabecera y los KPIs ya existen; se agregan los cuatro gráficos.

1. Ir a la página **Resumen** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:

### **PROMPT 6 (Gráficos de la Página "Resumen"):**
```text
En la página "Resumen" crea (mismas reglas PBIR; no leas otros archivos salvo pages.json):
1) lineChart "lineasMes" x324 y305 w1032 h360 "Viajes por mes · 2024 vs 2025": Category dimFecha[Mes], Series dimFecha[Anio], Y Total Viajes; filtro del visual dimFecha[Anio] in (2024, 2025); líneas de 3 px con marcadores; color de 2024 #1B4965 y de 2025 #F6AE2D.
2) donutChart "donaMembresia" x1380 y305 w516 h360 "Viajes por tipo de membresía": Category dimUsuarios[TipoMembresia], Y Total Viajes; leyenda abajo; etiquetas en % del total; sin valor central ("centerValue" show false).
3) barChart "topEstaciones" x324 y689 w774 h367 "Top 10 estaciones de origen por viajes": Category dimEstaciones[NombreEstacion], Y Total Viajes; filtro TopN 10 por Total Viajes; barras #1B4965; etiquetas de datos; orden descendente.
4) azureMap "mapaEstaciones" x1122 y689 w774 h367 "Estaciones · tamaño según viajes de origen": Category dimEstaciones[NombreEstacion], Y dimEstaciones[Latitud] y X dimEstaciones[Longitud] con Aggregation Function 1 (promedio), Size Total Viajes; burbujas #F6AE2D.
Responde solo con "Listo".
```

3. Pulsar **Apply external changes** en Power BI.

4. Revisar la página terminada y guardar con `Ctrl + S`.
> ![Página Resumen Operacional terminada](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-05-resumen.png)

5. Si el gráfico de barras muestra todas las estaciones, aplicar el filtro manualmente: seleccionarlo → panel **Filters** → `NombreEstacion` → **Top N** → Mostrar **10** → arrastrar `Total Viajes` a *By value* → **Apply filter**.

> **Fundamento de Ingeniería:**
> El filtro de nivel de visual (2024 y 2025) solo afecta a ese gráfico. El filtro **Top N** se evalúa contra una medida, por lo que el ranking se recalcula con cada segmentador. En el mapa, latitud y longitud se agregan como **promedio**: si se sumaran, una estación con varias filas quedaría fuera de su ubicación real. Azure Maps requiere tener la sesión iniciada en Power BI Desktop.

<!--CELL-->

## **Construcción de la Página: Análisis de Uso**

Despliegue de la segunda página de analítica granular, orientada a estudiar el comportamiento por franjas horarias, segmentación demográfica y flujo de estaciones. La página ya existe como copia de "Resumen": el prompt cambia textos y KPIs, retira los gráficos copiados y crea los nuevos.

1. Ir a la página **Análisis de Uso** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:

### **PROMPT 7 (Generación de la Página "Análisis de Uso"):**
```text
En la página "Análisis de Uso" (copia de Resumen; búscala en pages.json, lista sus visuales y lee solo los que modifiques; no toques panel ni slicers):
1) Título: "EcoBici Lima · Análisis de Uso". Subtítulo: "Patrones de demanda por hora, mes, zona, usuarios y estaciones".
2) cardVisual: cambia las medidas por Viajes por Usuario, Distancia Promedio (km), Bicicletas en Uso, Viajes Año Anterior y Var. % Viajes vs AA (mismo formato).
3) Borra las carpetas de los visuales lineChart, donutChart, barChart y azureMap copiados.
4) Crea:
- columnChart "colHora" x324 y305 w774 h360 "Viajes por hora del día": Category fctViajes[HoraInicio] (orden ascendente), Y Total Viajes, columnas #1B4965.
- lineClusteredColumnComboChart "comboMes" x1122 y305 w774 h360 "Viajes y duración promedio por mes": Category dimFecha[Mes], Y Total Viajes (columnas), Y2 Duración Promedio (min) (línea).
- columnChart apilado "zonaTipo" x324 y689 w508 h367 "Viajes por zona de origen y tipo de bici": Category dimEstaciones[Zona], Series dimBicicletas[Tipo], Y Total Viajes.
- clusteredBarChart "edadGenero" x856 y689 w508 h367 "Viajes por rango de edad y género": Category dimUsuarios[RangoEdad], Series dimUsuarios[Genero], Y Total Viajes.
- tableEx "tablaEstaciones" x1388 y689 w508 h367 "Estaciones: salidas vs llegadas": NombreEstacion como "Estación", Zona como "Zona", Total Viajes como "Salidas", Viajes por Destino como "Llegadas"; encabezado fondo #1B4965 y texto blanco; orden descendente por Salidas.
Responde solo con "Listo".
```

3. Pulsar **Apply external changes** en Power BI.

4. Revisar la página y guardar con `Ctrl + S`.
> ![Página Análisis de Uso terminada](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-06-analisis-uso.png)

> **Fundamento de Ingeniería:**
> Al partir de una copia, el panel lateral y los segmentadores sincronizados se conservan sin reescribirse. El gráfico combinado usa **dos ejes**: columnas para el volumen de viajes y línea para la duración en minutos, magnitudes con escalas distintas. La tabla contrasta la relación activa (salidas por estación de origen) con la inactiva activada por `USERELATIONSHIP` (llegadas por estación de destino).

<!--CELL-->

## **Construcción de la Página: Tendencias**

Generación de la tercera vista, especializada en series de tiempo, tasas de variación y evolución de la mezcla de flotas. Se sigue el mismo procedimiento que en *Análisis de Uso*.

1. Ir a la página **Tendencias** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:

### **PROMPT 8 (Generación de la Página "Tendencias"):**
```text
En la página "Tendencias" (copia de Resumen; mismo procedimiento que en Análisis de Uso):
1) Título: "EcoBici Lima · Tendencias". Subtítulo: "Evolución mensual 2023-2025: volumen, ritmo de crecimiento y composición".
2) cardVisual: Viajes Acumulados Año, Viajes Media Móvil 3M, Var. % Viajes vs Mes Anterior y Var. % Viajes vs AA.
3) Borra los visuales lineChart, donutChart, barChart y azureMap copiados.
4) Crea:
- lineChart "lineasMediaMovil" x324 y305 w1572 h360 "Viajes mensuales y media móvil de 3 meses": Category dimFecha[AnioMes] (orden ascendente), Y Total Viajes y Viajes Media Móvil 3M.
- lineChart "acumuladoAnio" x324 y689 w508 h367 "Viajes acumulados en el año": Category dimFecha[Mes], Series dimFecha[Anio], Y Viajes Acumulados Año.
- hundredPercentStackedColumnChart "mezclaFlota" x856 y689 w508 h367 "Mezcla anual: eléctricas vs mecánicas": Category dimFecha[Anio], Series dimBicicletas[Tipo], Y Total Viajes, etiquetas de datos.
- clusteredColumnChart "viajesTrimestre" x1388 y689 w508 h367 "Viajes por trimestre y año": Category dimFecha[Trimestre], Series dimFecha[Anio], Y Total Viajes.
Responde solo con "Listo".
```

3. Pulsar **Apply external changes** en Power BI.

4. Revisar la página y guardar con `Ctrl + S`.
> ![Página Tendencias terminada](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-07-tendencias.png)

> **Fundamento de Ingeniería:**
> La **media móvil de 3 meses** suaviza la variación mensual para mostrar la tendencia de fondo. El **acumulado del año** (`TOTALYTD`) se reinicia en enero, por eso cada año es una curva creciente. Las columnas **100 % apiladas** muestran proporciones (eléctricas frente a mecánicas) y no volumen, lo que permite comparar años con distinta cantidad de viajes.

<!--CELL-->

## **Validación Técnica Integral y Control de Calidad**

Concluida la construcción programática, se ejecutan las pruebas de integración del artefacto analítico:

### **Persistencia de contexto de filtrado:**

1. En cualquier página, abrir el segmentador **Año** y seleccionar **2024**.
> ![Segmentador Año desplegado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-validar-filtro.png)

2. Cambiar a otra página: el segmentador debe mostrar **2024** y los KPIs y gráficos deben reflejar solo ese año (por ejemplo, Total Viajes 2,100).
> ![Filtro 2024 sincronizado en la página Resumen](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-08-validacion-filtros.png)

3. Pasar el cursor sobre el segmentador y pulsar el icono de borrador (**Clear selections**) para quitar el filtro.

### **Evaluación de relaciones no activas:**

4. En **Análisis de Uso**, comprobar que la columna **Llegadas** de la tabla de estaciones da valores distintos de **Salidas**.

### **Persistencia final del repositorio PBIP:**

5. Guardar con `Ctrl + S`. La carpeta `EcoBici_Lima.Report` contiene las tres páginas en formato PBIR, preparadas para control de versiones o despliegue en Fabric / Power BI Service.

> **Fundamento de Ingeniería:**
> Validar el filtrado cruzado confirma que el `syncGroup` funciona en las tres páginas y que todas las medidas responden al contexto de filtro. La diferencia entre salidas y llegadas confirma que la relación inactiva se activa correctamente con `USERELATIONSHIP`.

<!--CELL-->

## **Resolución de Incidencias Técnicas (Troubleshooting)**

Guía de diagnóstico y resolución de eventos comunes, en el orden en que pueden aparecer:

### **Configuración de servidores MCP:**
* **`powerbi-modeling-mcp` en estado "Error" o "Stopped":** comprobar `node -v` y `npx -v`; en la alternativa B, ubicar el `.exe` con el comando de PowerShell indicado y duplicar cada barra invertida; validar `claude_desktop_config.json` con un validador JSON.
* **`archivos` en estado "Error" o "Stopped":** verificar que Node.js esté en el PATH y que la carpeta del último argumento exista y tenga las barras duplicadas.
* **Tiempo de espera agotado en comandos `npx`:** suele deberse a proxies o políticas de red; usar la alternativa B para el servidor de Power BI.
* **Alternativa B: el servidor dejó de funcionar tras actualizar VS Code:** la carpeta de la extensión cambia de versión; volver a ubicar el ejecutable y actualizar la ruta en `command`.

### **Conexión con el informe:**
* **Claude responde que no tiene herramientas de Power BI:** revisar **Ajustes → Desarrollador** (*Running*) y que los conectores estén activos en el icono de herramientas del chat.
* **Fallo de enlace MCP con el informe:** abrir Power BI con el proyecto `.pbip` y verificar que el acceso de herramientas externas mediante API locales seguras esté activado.
* **Error en la ruta de `Datos_Fuente`:** verificar que el parámetro `RutaDatos` apunte a la ubicación real de la carpeta.

### **Construcción de páginas:**
* **Los visuales no aparecen tras el prompt:** pulsar **Apply external changes** en la barra superior de Power BI.
* **Los cambios de Claude desaparecieron:** se guardó en Power BI mientras Claude escribía. Repetir el prompt y pulsar **Apply external changes** antes de guardar.
* **Un visual muestra error:** escribir en el chat *"El visual <nombre> de la página <página> da este error: <pega el error>. Corrige solo ese visual."*
* **Una medida da un valor inesperado:** escribir *"La medida <nombre> devuelve <qué ves> y debería <qué esperas>. Corrígela sin tocar las demás."*
* **Claude empieza a leer todos los archivos del informe:** pulsar **Stop** y escribir *"No leas más archivos; escribe directamente los visuales."*
* **Mapa de Azure Maps en blanco:** iniciar sesión en Power BI Desktop con cuenta institucional o de trabajo (esquina superior derecha) y verificar la habilitación de mapas en las opciones. Sin cuenta: cambiar el visual a *Gráfico de dispersión* con `Longitud` en X y `Latitud` en Y.
