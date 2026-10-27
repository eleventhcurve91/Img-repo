# **Power BI + Claude Desktop (Plan Free) mediante MCP**
### **Versión para estudiantes: el informe EcoBici Lima completo, en un solo chat y sin pagar suscripción**

---

> **¿Por qué existe esta versión?**
> Con el plan gratuito, los prompts originales agotan el límite de uso antes de terminar y Claude pide actualizar a un plan de pago.
> Esta versión reúne todo en **7 prompts dentro de un mismo chat**. Cada prompt ya trae los nombres de columnas, el código DAX y las coordenadas de los visuales, así Claude **no gasta mensajes explorando ni corrigiendo errores**.
>
> **Todo el recorrido funciona con el plan Free de Claude.** No se usa Claude Code, ni plugins, ni ninguna función de pago.
>
> **Del manual original se mantienen:** Node.js, la extensión Power BI Modeling MCP de VS Code (solo alternativa B), `claude_desktop_config.json` con los servidores `powerbi-modeling-mcp` y `archivos`, la API local de Power BI y el guardado del proyecto como `EcoBici_Lima.pbip` dentro de la carpeta de trabajo.
>
> **Guía verificada:** todos los prompts se ejecutaron sobre `EcoBici_Lima.pbip` con Power BI abierto; las capturas de cada paso son de esa ejecución.
>
> **Del manual original se OMITEN (requieren Claude Code, que es de pago):**
> * *Instalación Global de Herramientas CLI de Microsoft* (`powerbi-desktop-bridge-cli` y `powerbi-report-authoring-cli`).
> * *Aprovisionamiento de Microsoft Fabric Skills* (plugins en la pestaña Code).
> * Cualquier instrucción que diga "pestaña Code" o "consola de Claude Code": aquí todo se hace en el **chat** normal.

<!--CELL-->

## **Paso 0. Reglas para trabajar en el plan Free**

### **¿Qué sí funciona en Free y qué no?**

| Componente | Plan Free | Qué usamos |
| :--- | :--- | :--- |
| Chat de Claude Desktop | Sí | Un **único chat** donde se pegan los 7 prompts, uno tras otro. |
| Servidores MCP locales (`powerbi-modeling-mcp`, `archivos`) | Sí | Modelo y medidas (MCP de Power BI) y páginas del informe (archivos PBIR). |
| Pestaña **Code** (Claude Code) y plugins *Fabric Skills / Power BI Authoring* | No | El prompt 4 le da a Claude las reglas del formato PBIR que antes daban los plugins. |

### **Reglas de oro**
1. **Todo en un mismo chat.** Pega el prompt 1, espera la respuesta, pega el 2, y así hasta el 7. Las reglas que se dan en un prompt sirven para los siguientes, por eso los últimos prompts son más cortos.
2. **Espera a que Claude termine** antes de pegar el siguiente prompt y **no escribas mensajes intermedios** ("¿ya?", "gracias"): cada mensaje reenvía todo el chat y consume límite.
3. **Desactiva lo que no se usa** (icono de herramientas / **+** junto al cuadro de texto): búsqueda web y cualquier conector que no sea `powerbi-modeling-mcp` o `archivos`.
4. Cuando Claude pida permiso para usar una herramienta, pulsa **Permitir siempre**.
5. **Respuestas cortas.** Todos los prompts terminan pidiendo una respuesta mínima. Si quieres entender algo, pregúntalo al final.
6. **Power BI siempre abierto.** En los prompts 4 a 7 (páginas), **guarda con `Ctrl + S` antes de pegar el prompt**. Cuando Claude termine, Power BI muestra un aviso de cambios en los archivos del informe: pulsa **Apply external changes** (*Aplicar cambios externos*). No guardes en Power BI mientras Claude está escribiendo.
7. **Si aparece el aviso de límite:** no cierres el chat. Guarda con `Ctrl + S`, espera a la hora de reinicio que indica Claude y escribe en **el mismo chat**: *"Continúa exactamente donde te quedaste"*.

<!--CELL-->

## **Prompt 1. Conexión MCP y Perfilado de Datos**

Manteniendo abierta la instancia de Power BI Desktop con `EcoBici_Lima.pbip`:
1. Abrir **Claude Desktop** en la vista de **chat** (no la pestaña Code).
2. Comprobar en **Ajustes → Desarrollador** que `powerbi-modeling-mcp` y `archivos` están en **Running**.
3. **Nuevo chat** → pegar el prompt 1. **Este chat se usa para toda la guía.**

El chat de Claude no puede abrir los `.xlsx` (son binarios), así que el prompt le entrega su estructura y solo le pide leer los CSV, que son texto.

```text
Vamos a construir en este chat un informe de Power BI completo usando solo las herramientas MCP (powerbi-modeling-mcp y archivos). Reglas para todo el chat: trabaja directo, no uses la operación Help, agrupa en una sola llamada todo lo que la herramienta permita (listas), no muestres código ni JSON y responde siempre en pocas líneas.

1) Conéctate por MCP al informe de Power BI Desktop que tengo abierto (EcoBici_Lima).
2) Con la herramienta de archivos lee ÚNICAMENTE Datos_Fuente\Estaciones_Centro.csv, Estaciones_Sur.csv y Usuarios.csv (separador ";"). Lo que ya sé de los Excel (no los abras):
- Viajes.xlsx: hojas 2023, 2024, 2025. Columnas IdViaje, InicioViaje, FinViaje (texto "dd/mm/aaaa hh:mm"), DuracionMin, IdEstacionOrigen, IdEstacionDestino, IdBicicleta, IdUsuario, DistanciaKm, Notas.
- Bicicletas.xlsx (hoja Bicicletas): IdBicicleta, CodigoBici, Tipo (Eléctrica/Mecánica), Modelo, FechaAdquisicion, RegistroInterno.
- Mapa_Zonas.xlsx (hoja Zonas): Distrito, Zona.
Responde con: nombre del modelo conectado y un resumen de calidad de datos en máximo 10 viñetas (problema → transformación propuesta) más el esquema en estrella propuesto en 3 líneas.
```

### **Verificación:**
* Claude confirma el modelo conectado (puerto local de Analysis Services).
* El resumen menciona: hojas por año en `Viajes.xlsx` y fechas en texto `dd/mm/aaaa`; `Distrito` sucio en las estaciones (espacios y mayúsculas: `MIRAFLORES`, `  Cercado de Lima `) que se cruza con `Mapa_Zonas`; `Genero` con códigos y vacíos que requieren *No especificado*; `RegistroInterno` y `Notas` sobrantes.

> ![Confirmación de Conexión en Claude Desktop](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/17.png)
> * **Descripción:** Respuesta de Claude confirmando el acoplamiento con la instancia local de Power BI mediante MCP.

> ![Reporte de Perfilado de Calidad en Claude](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/18.png)
> * **Descripción:** Diagnóstico de calidad de datos y propuesta de arquitectura dimensional.

<!--CELL-->

## **Prompt 2. Modelo Dimensional Completo (Esquema en Estrella)**

En un solo prompt se crean el parámetro `RutaDatos`, las cinco tablas y las relaciones.

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

### **Validación de Arquitectura en Power BI:**
1. Cambiar a la **Vista de modelo**.
2. `fctViajes` en el centro con relaciones `1:*` hacia `dimEstaciones`, `dimUsuarios`, `dimBicicletas` y `dimFecha`.
3. La relación `IdEstacionDestino → dimEstaciones` aparece como **línea punteada** (inactiva).
4. Guardar con `Ctrl + S`.

> ![Diagrama de Modelo en Estrella en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-02-modelo-estrella.png)
> * **Descripción:** Vista de modelo con `fctViajes` en el núcleo y las dimensiones `dimEstaciones`, `dimUsuarios`, `dimBicicletas` y `dimFecha` conectadas.

<!--CELL-->

## **Prompt 3. Medidas DAX en Tabla Dedicada**

Las 15 medidas en la tabla `_Medidas` con sus carpetas (*Display Folders*). El DAX viene listo para que Claude no gaste mensajes inventando y corrigiendo fórmulas.

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

### **Protocolo de Sincronización:**
Guardar con `Ctrl + S`. En el panel **Datos** debe aparecer `_Medidas` con las carpetas Base, Avanzadas y Tendencias.

> ![Tabla de Medidas y Carpetas de Despliegue](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-03-medidas.png)
> * **Descripción:** Panel de datos con la tabla `_Medidas` y sus tres carpetas: Base, Avanzadas y Tendencias.

<!--CELL-->

## **Prompt 4. Diseño Institucional y Página Base "Resumen"**

En el manual original el diseño se aplica a "todas las páginas" y luego cada página repite cabecera y KPIs. Aquí **se construye una sola vez** en "Resumen" (panel lateral, 5 segmentadores, cabecera y tarjeta de KPIs) y después **se duplica la página** con un clic para las otras dos.

### **Antes del prompt (manual, 0 mensajes):**
1. **Vista → Temas → Examinar temas** → seleccionar `Datos_Fuente\Tema_EcoBici.json`.
2. `Ctrl + S`.

### **Grilla de referencia (lienzo 1920 x 1080, márgenes de 24 px)**
| Zona | x | y | ancho | alto |
| :--- | :--- | :--- | :--- | :--- |
| Panel lateral | 0 | 0 | 300 | 1080 |
| Cabecera (título / subtítulo / línea) | 324 | 24 / 70 / 102 | 1572 / 1572 / 96 | 44 / 26 / 5 |
| Tarjeta de KPIs | 324 | 131 | 1572 | 150 |
| Fila central | 324 | 305 | 1572 | 360 |
| Fila inferior | 324 | 689 | 1572 | 367 |

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
1. En Power BI pulsar **Apply external changes** (*Aplicar cambios externos*) en la barra que aparece arriba del lienzo. Si un visual muestra error, pega el mensaje en el chat con: *"Corrige solo ese visual"*.
2. Comprobar que los 5 segmentadores aparecen en **Ver → Sincronizar segmentaciones**.
3. **Manual (0 mensajes):** clic derecho en la pestaña "Resumen" → **Duplicar página** (dos veces). Renombrar las copias como **Análisis de Uso** y **Tendencias**. Así el panel, los segmentadores sincronizados, la cabecera y la tarjeta de KPIs quedan en las 3 páginas.
4. `Ctrl + S`.

> ![Maquetación Base del Lienzo y Panel Lateral](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-04-diseno-base.png)
> * **Descripción:** Estructura base con el panel lateral azul oscuro de 300 px, los 5 segmentadores desplegables y la cabecera ejecutiva.

<!--CELL-->

## **Prompt 5. Página: Resumen Operacional**

La cabecera y los KPIs ya existen. Solo faltan los cuatro gráficos. Guardar con `Ctrl + S` antes de pegar.

```text
En la página "Resumen" crea (mismas reglas PBIR; no leas otros archivos salvo pages.json):
1) lineChart "lineasMes" x324 y305 w1032 h360 "Viajes por mes · 2024 vs 2025": Category dimFecha[Mes], Series dimFecha[Anio], Y Total Viajes; filtro del visual dimFecha[Anio] in (2024, 2025); líneas de 3 px con marcadores.
2) donutChart "donaMembresia" x1380 y305 w516 h360 "Viajes por tipo de membresía": Category dimUsuarios[TipoMembresia], Y Total Viajes; leyenda abajo; etiquetas en % del total; sin valor central ("centerValue" show false).
3) barChart "topEstaciones" x324 y689 w774 h367 "Top 10 estaciones de origen por viajes": Category dimEstaciones[NombreEstacion], Y Total Viajes; filtro TopN 10 por Total Viajes; barras #1B4965; etiquetas de datos; orden descendente.
4) azureMap "mapaEstaciones" x1122 y689 w774 h367 "Estaciones · tamaño según viajes de origen": Category dimEstaciones[NombreEstacion], Y dimEstaciones[Latitud] y X dimEstaciones[Longitud] con Aggregation Function 1 (promedio), Size Total Viajes; burbujas #F6AE2D.
Responde solo con "Listo".
```

> **Si el Top 10 no queda aplicado (manual):** seleccionar el gráfico de barras → panel **Filtros** → `NombreEstacion` → **N superior** → Mostrar **10** → arrastrar `Total Viajes` a *Por valor* → **Aplicar filtro**.

### **Protocolo de Sincronización:**
Pulsar **Apply external changes** (*Aplicar cambios externos*) en Power BI y luego `Ctrl + S`.
*(Azure Maps requiere tener la sesión iniciada en Power BI Desktop; ver la tabla de incidencias si no tienes cuenta institucional).*

> ![Página de Resumen Operacional en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-05-resumen.png)
> * **Descripción:** Página Resumen Operacional: tarjeta de KPIs, líneas 2024 vs 2025, dona de membresías, ranking de estaciones y mapa con Azure Maps.

<!--CELL-->

## **Prompt 6. Página: Análisis de Uso**

La página ya existe como copia de "Resumen". El prompt cambia textos y KPIs, quita los gráficos copiados y crea los cinco nuevos. Guardar con `Ctrl + S` antes de pegar.

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

### **Protocolo de Sincronización:**
Pulsar **Apply external changes** (*Aplicar cambios externos*) en Power BI y luego `Ctrl + S`.

> ![Página de Análisis de Uso en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-06-analisis-uso.png)
> * **Descripción:** Página Análisis de Uso con distribución horaria, combinado viajes vs duración, apertura demográfica y tabla de salidas vs llegadas por estación.

<!--CELL-->

## **Prompt 7. Página: Tendencias**

Mismo procedimiento sobre la copia "Tendencias". Guardar con `Ctrl + S` antes de pegar.

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

### **Protocolo de Sincronización:**
Pulsar **Apply external changes** (*Aplicar cambios externos*) en Power BI y luego `Ctrl + S`.

> ![Página de Tendencias en Power BI](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-07-tendencias.png)
> * **Descripción:** Página Tendencias con la serie mensual y su media móvil de 3 meses, curvas acumuladas por año, mezcla de flota y viajes por trimestre.

<!--CELL-->

## **Validación Técnica Integral y Control de Calidad**

Se hace **a mano** (0 mensajes):

1. **Persistencia de filtros:** elegir un valor en cada uno de los 5 segmentadores y cambiar de página; la selección debe mantenerse en las tres.
2. **Relación inactiva:** en *Análisis de Uso*, la columna `Llegadas` debe dar valores distintos de `Salidas` (usa `USERELATIONSHIP` con la estación de destino).
3. **Persistencia del proyecto:** `Ctrl + S`. La carpeta `informe` contiene las tres páginas en formato PBIR.

> ![Reporte Multi-página Completo en Funcionamiento](https://raw.githubusercontent.com/eleventhcurve91/Img-repo/main/claude-free/free-08-validacion-filtros.png)
> * **Descripción:** Informe multipágina final con filtrado cruzado y diseño consistente.

<!--CELL-->

## **Mensajes de Rescate (en el mismo chat)**

| Situación | Qué escribir en el chat |
| :--- | :--- |
| Apareció el aviso de límite | Esperar la hora de reinicio y escribir: *"Continúa exactamente donde te quedaste"*. |
| Un visual muestra error en Power BI | *"El visual <nombre> de la página <página> da este error: <pega el error>. Corrige solo ese visual."* |
| Una medida da un valor raro | *"La medida <nombre> devuelve <qué ves> y debería <qué esperas>. Corrígela sin tocar las demás."* |
| Claude sugiere usar Claude Code o instalar un plugin | *"No tengo Claude Code; hazlo solo con las herramientas MCP de este chat."* |
| Claude empieza a leer todos los archivos del informe | Pulsar **Stop** y escribir: *"No leas más archivos; escribe directamente los visuales."* |
| El chat dice que la conversación es demasiado larga | Abrir un chat nuevo, pegar el **primer párrafo del prompt 1** (las reglas) más: *"Conéctate al Power BI abierto"* y luego el prompt del paso que falta. Si es un prompt de páginas (5 a 7), pegar también el párrafo de reglas PBIR del prompt 4. |

---

| Incidencia | Solución |
| :--- | :--- |
| Claude responde que no tiene herramientas de Power BI | Revisar **Ajustes → Desarrollador** (*Running*) y que no estén desactivadas en el icono de herramientas del chat. |
| No aparece el aviso de actualizar en Power BI | Cerrar Power BI **sin guardar** y volver a abrir `EcoBici_Lima.pbip`. |
| Los cambios de Claude desaparecieron | Se guardó en Power BI mientras Claude escribía. Repetir el prompt y pulsar **Apply external changes** antes de guardar. |
| Aparece "Overwrite your unsaved edits" al aplicar | Había cambios sin guardar en Power BI. Pulsar **Apply external changes**; para evitarlo, guardar con `Ctrl + S` antes de pegar cada prompt. |
| Azure Maps en blanco | Iniciar sesión en Power BI Desktop con cuenta institucional o de trabajo (Power BI Desktop es gratuito, pero no acepta Gmail/Outlook personales). Sin cuenta: cambiar el visual a *Gráfico de dispersión* con `Longitud` en X y `Latitud` en Y. |
