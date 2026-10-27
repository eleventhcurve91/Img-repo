# **Power BI + Claude Desktop mediante MCP**
### **Construcción del informe EcoBici Lima en un solo chat**

---

> Esta guía construye el informe completo con **7 prompts dentro de un mismo chat** de Claude Desktop. Cada prompt trae los nombres de columnas, el código DAX y las coordenadas de los visuales, de modo que Claude trabaja directo sobre el modelo y los archivos del informe.
>
> **Requisitos previos (secciones anteriores del manual):** Node.js, `claude_desktop_config.json` con los servidores `powerbi-modeling-mcp` y `archivos`, la API local de Power BI y el proyecto guardado como `EcoBici_Lima.pbip` dentro de la carpeta de trabajo.

<!--CELL-->

## **Paso 0. Reglas de trabajo**

1. **Todo en un mismo chat.** Pegar el prompt 1, esperar la respuesta, pegar el 2, y así hasta el 7. Las reglas que se dan en un prompt sirven para los siguientes, por eso los últimos prompts son más cortos.
2. **Esperar a que Claude termine** antes de pegar el siguiente prompt.
3. **Desactivar lo que no se usa** (icono de herramientas / **+** junto al cuadro de texto): búsqueda web y cualquier conector que no sea `powerbi-modeling-mcp` o `archivos`.
4. Cuando Claude pida permiso para usar una herramienta, pulsar **Permitir siempre**.
5. **Power BI siempre abierto.** En los prompts 4 a 7 (páginas), **guardar con `Ctrl + S` antes de pegar el prompt** y no guardar mientras Claude está escribiendo.

> **Fundamento de Ingeniería:**
> Un único chat conserva el contexto: las reglas PBIR y los nombres del modelo se explican una sola vez y los prompts siguientes las reutilizan. Pedir operaciones agrupadas (listas) reduce el número de llamadas a las herramientas MCP y, con ello, el tiempo de respuesta.

<!--CELL-->

## **Prompt 1. Conexión MCP y Perfilado de Datos**

1. Abrir `EcoBici_Lima.pbip` en Power BI Desktop y mantenerlo abierto.

2. En Claude Desktop, ir a **Ajustes → Desarrollador** y comprobar que `powerbi-modeling-mcp` y `archivos` están en **Running**.
> ![Estado Running de los servidores MCP](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-powerbi-mcp/09-mcp-running.png)

3. Abrir un **chat nuevo** (este chat se usa para toda la guía) y pegar el prompt:
```text
Vamos a construir en este chat un informe de Power BI completo usando solo las herramientas MCP (powerbi-modeling-mcp y archivos). Reglas para todo el chat: trabaja directo, no uses la operación Help, agrupa en una sola llamada todo lo que la herramienta permita (listas), no muestres código ni JSON y responde siempre en pocas líneas.

1) Conéctate por MCP al informe de Power BI Desktop que tengo abierto (EcoBici_Lima).
2) Con la herramienta de archivos lee ÚNICAMENTE Datos_Fuente\Estaciones_Centro.csv, Estaciones_Sur.csv y Usuarios.csv (separador ";"). Lo que ya sé de los Excel (no los abras):
- Viajes.xlsx: hojas 2023, 2024, 2025. Columnas IdViaje, InicioViaje, FinViaje (texto "dd/mm/aaaa hh:mm"), DuracionMin, IdEstacionOrigen, IdEstacionDestino, IdBicicleta, IdUsuario, DistanciaKm, Notas.
- Bicicletas.xlsx (hoja Bicicletas): IdBicicleta, CodigoBici, Tipo (Eléctrica/Mecánica), Modelo, FechaAdquisicion, RegistroInterno.
- Mapa_Zonas.xlsx (hoja Zonas): Distrito, Zona.
Responde con: nombre del modelo conectado y un resumen de calidad de datos en máximo 10 viñetas (problema → transformación propuesta) más el esquema en estrella propuesto en 3 líneas.
```

4. Verificar que Claude confirma la conexión con el modelo de Power BI (puerto local de Analysis Services).
> ![Confirmación de conexión en Claude Desktop](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/17.png)

5. Revisar el resumen de calidad: debe mencionar las hojas por año de `Viajes.xlsx`, las fechas en texto `dd/mm/aaaa`, el `Distrito` con espacios y mayúsculas distintas (`MIRAFLORES`, `  Cercado de Lima `), el `Genero` con códigos y vacíos, y las columnas sobrantes `Notas` y `RegistroInterno`.
> ![Resumen de calidad de datos en Claude](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/18.png)

> **Fundamento de Ingeniería:**
> El servidor MCP localiza el puerto dinámico de la instancia de Analysis Services que Power BI Desktop levanta en segundo plano. Los archivos `.xlsx` son binarios y el servidor `archivos` solo lee texto; por eso el prompt entrega su estructura y Claude solo perfila los CSV. Detectar los problemas antes de modelar evita reprocesar tablas después.

<!--CELL-->

## **Prompt 2. Modelo Dimensional Completo (Esquema en Estrella)**

1. En el mismo chat, pegar el prompt:
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

2. Claude responde con las filas de cada tabla: `fctViajes` 6.200, `dimEstaciones` 24, `dimUsuarios` 500, `dimBicicletas` 120 y `dimFecha` 1.096.

3. Si en Power BI aparece la barra *"One or more relationships have been modified and need to be refreshed manually"*, pulsar **Refresh now**.

4. Cambiar a la **Vista de modelo** (tercer icono de la barra izquierda) y arrastrar las tablas para dejar `fctViajes` en el centro. La relación `IdEstacionDestino → dimEstaciones` aparece como **línea punteada** (inactiva).
> ![Vista de modelo con el esquema en estrella](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-02-modelo-estrella.png)

5. Guardar con `Ctrl + S`.

> **Fundamento de Ingeniería:**
> El esquema en estrella separa los **hechos** (`fct`, cada viaje) de las **dimensiones** (`dim`, quién, qué, dónde y cuándo). Las relaciones `1:*` filtran desde la dimensión hacia los hechos. Entre dos tablas solo puede haber una relación activa; la de estación de destino queda inactiva y se activa dentro de una medida con `USERELATIONSHIP`. Marcar `dimFecha` como tabla de fechas habilita las funciones de inteligencia de tiempo (`TOTALYTD`, `DATEADD`, `DATESINPERIOD`). El parámetro `RutaDatos` permite mover la carpeta cambiando un solo valor.

<!--CELL-->

## **Prompt 3. Medidas DAX en Tabla Dedicada**

1. En el mismo chat, pegar el prompt:
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

2. Claude responde con los valores de control: **Total Viajes 6,200**, **% Viajes Eléctricos 34.1%** y **Var. % Viajes vs AA +9.5%** (2025 frente a 2024).

3. Volver a la **Vista de informe** (primer icono de la barra izquierda) y, en el panel **Data**, expandir `_Medidas` y sus carpetas **Avanzadas**, **Base** y **Tendencias**.
> ![Panel Data con la tabla _Medidas y sus carpetas](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-panel-medidas.png)

4. Guardar con `Ctrl + S`.

> **Fundamento de Ingeniería:**
> Una tabla dedicada a medidas separa la lógica de cálculo de los datos, y las carpetas de visualización (*Display Folders*) agrupan las medidas por propósito. Las medidas de año anterior toman el **último año con datos del contexto** (`YEAR(MAX(...))`): sin filtros comparan 2025 contra 2024 en lugar de sumar todos los años. `CALCULATE` cambia el contexto de filtro, `USERELATIONSHIP` activa la relación inactiva solo dentro de la medida y `DIVIDE` evita errores por división entre cero.

<!--CELL-->

## **Prompt 4. Diseño Institucional y Página Base "Resumen"**

El diseño se construye una sola vez en "Resumen" (panel lateral, 5 segmentadores, cabecera y tarjeta de KPIs) y después se duplica la página para las otras dos.

1. En Power BI, abrir la pestaña **Design** (*Diseño*) y pulsar **Import theme** (*Importar tema*).
> ![Pestaña Design con la opción Import theme](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-importar-tema.png)

2. Navegar a la carpeta `Datos_Fuente`, seleccionar `Tema_EcoBici.json` y pulsar **Abrir**.
> ![Selección del archivo Tema_EcoBici.json](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-seleccionar-tema.png)

3. Confirmar el mensaje **File successfully added** con **Got it** y guardar con `Ctrl + S`.
> ![Confirmación de tema importado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-tema-aplicado.png)

4. En el mismo chat de Claude, pegar el prompt:
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

5. Cuando Claude responda "Listo", Power BI muestra la barra *"This project's files were changed externally"*. Pulsar **Apply external changes** (*Aplicar cambios externos*).
> ![Barra Apply external changes en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-aplicar-cambios-externos.png)

6. Si aparece el aviso **Overwrite your unsaved edits**, pulsar **Apply external changes** (había algún cambio sin guardar en Power BI).
> ![Aviso Overwrite your unsaved edits](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-sobrescribir-cambios.png)

7. Revisar la página: panel lateral azul con los 5 segmentadores, cabecera y tarjeta de KPIs con los valores completos (6,200 y no 6K).
> ![Página base Resumen con panel lateral, cabecera y KPIs](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-04-diseno-base.png)

8. Seleccionar el segmentador **Año** y abrir **View → Sync slicers** (*Ver → Sincronizar segmentaciones*). Las tres páginas deben aparecer marcadas.
> ![Panel Sync slicers con el segmentador Año sincronizado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-sincronizar-segmentadores.png)

9. Clic derecho sobre la pestaña **Resumen** → **Duplicate** (*Duplicar*). Repetir una segunda vez.
> ![Menú contextual de la pestaña con la opción Duplicate](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-duplicar-pagina.png)

10. Doble clic sobre cada copia para renombrarlas como **Análisis de Uso** y **Tendencias**, y guardar con `Ctrl + S`.
> ![Pestañas Resumen, Análisis de Uso y Tendencias](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-pestanas.png)

> **Fundamento de Ingeniería:**
> En formato PBIR cada página es una carpeta y cada visual un archivo `visual.json`; Claude escribe esos archivos y Power BI los recarga con **Apply external changes**. El tema JSON centraliza colores, tipografías, bordes y sombras para todos los visuales. El `syncGroup` vincula los segmentadores de distintas páginas: al duplicar la página, las copias conservan el mismo grupo y el filtro se comparte en todo el informe.

<!--CELL-->

## **Prompt 5. Página: Resumen Operacional**

1. Ir a la página **Resumen** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:
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

5. Si el gráfico de barras muestra todas las estaciones: seleccionarlo → panel **Filters** → `NombreEstacion` → **Top N** → Mostrar **10** → arrastrar `Total Viajes` a *By value* → **Apply filter**.

> **Fundamento de Ingeniería:**
> El filtro de nivel de visual (2024 y 2025) solo afecta a ese gráfico, sin tocar el resto de la página. El filtro **Top N** se evalúa contra una medida, por lo que el ranking se recalcula con cada segmentador. En el mapa, latitud y longitud se agregan como **promedio**: si se sumaran, una estación con varias filas quedaría fuera de su ubicación real. Azure Maps requiere tener la sesión iniciada en Power BI Desktop.

<!--CELL-->

## **Prompt 6. Página: Análisis de Uso**

1. Ir a la página **Análisis de Uso** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:
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

4. Revisar la página y guardar con `Ctrl + S`. En la tabla, la columna **Llegadas** debe tener valores distintos de **Salidas**.
> ![Página Análisis de Uso terminada](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-06-analisis-uso.png)

> **Fundamento de Ingeniería:**
> Como la página es una copia de Resumen, el panel lateral y los segmentadores sincronizados se conservan; solo se reemplazan los textos, los KPIs y los gráficos. El gráfico combinado usa **dos ejes**: columnas para el volumen de viajes y línea para la duración en minutos, magnitudes con escalas distintas. La tabla compara la relación activa (salidas por estación de origen) con la inactiva activada por `USERELATIONSHIP` (llegadas por estación de destino).

<!--CELL-->

## **Prompt 7. Página: Tendencias**

1. Ir a la página **Tendencias** y guardar con `Ctrl + S`.

2. En el mismo chat, pegar el prompt:
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
> La **media móvil de 3 meses** suaviza la variación mensual para mostrar la tendencia de fondo. El **acumulado del año** (`TOTALYTD`) reinicia en enero, por eso cada año es una curva que crece. Las columnas **100 % apiladas** muestran la proporción (eléctricas frente a mecánicas) y no el volumen, lo que permite comparar años con distinta cantidad de viajes.

<!--CELL-->

## **Validación Técnica Integral y Control de Calidad**

1. En cualquier página, abrir el segmentador **Año** y seleccionar **2024**.
> ![Segmentador Año desplegado](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/paso-validar-filtro.png)

2. Cambiar a otra página: el segmentador debe mostrar **2024** y los KPIs y gráficos deben reflejar solo ese año (por ejemplo, Total Viajes 2,100).
> ![Filtro 2024 sincronizado en la página Resumen](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-08-validacion-filtros.png)

3. Pasar el cursor sobre el segmentador y pulsar el icono de borrador (**Clear selections**) para quitar el filtro.

4. En **Análisis de Uso**, comprobar que la columna **Llegadas** de la tabla da valores distintos de **Salidas**.

5. Guardar con `Ctrl + S`. La carpeta `EcoBici_Lima.Report` contiene las tres páginas en formato PBIR.

> **Fundamento de Ingeniería:**
> Validar el filtrado cruzado confirma que el `syncGroup` funciona en las tres páginas y que todas las medidas responden al contexto de filtro. La diferencia entre salidas y llegadas confirma que la relación inactiva se activa correctamente con `USERELATIONSHIP`.

<!--CELL-->

## **Mensajes de Rescate e Incidencias**

* **Un visual muestra error en Power BI:** escribir en el chat *"El visual <nombre> de la página <página> da este error: <pega el error>. Corrige solo ese visual."*
* **Una medida da un valor raro:** escribir *"La medida <nombre> devuelve <qué ves> y debería <qué esperas>. Corrígela sin tocar las demás."*
* **Claude empieza a leer todos los archivos del informe:** pulsar **Stop** y escribir *"No leas más archivos; escribe directamente los visuales."*
* **Claude responde que no tiene herramientas de Power BI:** revisar **Ajustes → Desarrollador** (*Running*) y que no estén desactivadas en el icono de herramientas del chat.
* **Los cambios de Claude desaparecieron:** se guardó en Power BI mientras Claude escribía. Repetir el prompt y pulsar **Apply external changes** antes de guardar.
* **Azure Maps en blanco:** iniciar sesión en Power BI Desktop con cuenta institucional o de trabajo. Sin cuenta: cambiar el visual a *Gráfico de dispersión* con `Longitud` en X y `Latitud` en Y.
