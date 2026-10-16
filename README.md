# ucPDF — control visor de PDF para twinBASIC

`ucPDF` es un UserControl de twinBASIC que muestra documentos PDF usando **PDFium**.
Lo dibuja todo con GDI y doble búfer, sin controles hijos, y funciona en **Win32 y Win64**.

## Estructura

| Archivo | Contenido |
|---|---|
| `Sources/PdfiumApi.twin` | Carga dinámica de `pdfium.dll` (x86/x64), recuento de referencias y declaraciones de la API C |
| `Sources/PdfDocument.twin` | Documento abierto: páginas y texto con carga diferida, renderizado y extracción de texto |
| `Sources/PdfDib.twin` | DIB de 32 bpp sobre el que PDFium escribe directamente |
| `Sources/ucPDF.twin` | El control: visor, herramientas, eventos y propiedades |
| `Sources/ucPdfSupport.twin` | Enumeraciones públicas, subclasificación y diálogo de impresión |
| `Sources/GdiPlus.twin` | API plana de GDI+ (`gdiplus.dll` de Windows) para la barra de herramientas |
| `Sources/PdfFormData.twin` | Datos de formularios: exportar e importar en XFDF, FDF e INI |
| `Sources/PdfOps.twin` | Operaciones sin interfaz: texto, coordenadas, búsqueda, páginas, imágenes, unir, dividir, imprimir |
| `Sources/modCommandLine.twin` | `Sub Main` y comandos de línea de comandos |
| `Sources/Win32Api.twin` | Declaraciones Win32 y utilidades |
| `Sources/PdfTests.twin` | Pruebas automáticas para Win32/Win64 |
| `samples/sample.pdf` | PDF de 2 páginas que usan las pruebas |
| `samples/marcadores.pdf` | PDF de 3 páginas con marcadores anidados (uno plegado) |
| `samples/formulario.pdf` | Formulario de prueba: texto, casilla, radios, combo, lista múltiple, multilínea y solo lectura |

> Los archivos `.twin` usan `\r` (CR) como salto de línea. `.gitattributes` evita que git los convierta.

## Instalación

1. Descarga PDFium de <https://github.com/bblanchon/pdfium-binaries> (`pdfium-win-x86` y `pdfium-win-x64`).
2. Copia cada `bin/pdfium.dll` junto al ejecutable:
   ```
   <App.Path>\x86\pdfium.dll
   <App.Path>\x64\pdfium.dll
   ```
   También puedes indicar otra ruta, de archivo o de carpeta, en la propiedad `PdfiumPath`.
3. En twinBASIC, importa los módulos y clases de `Sources/`.
4. Crea un UserControl llamado **`ucPDF`** (*Project → Add → UserControl*). Conserva los atributos que genera el IDE (`[FormDesignerId]`, `[ClassId]`, …) y sustituye el resto por el contenido de `Sources/ucPDF.twin`.
5. Para que el DPI sea correcto, activa *DPI awareness* (Per-Monitor V2) en el manifiesto del proyecto.

> **Convención de llamada (Win32):** todas las funciones de PDFium se declaran `CDecl`. La cabecera `fpdfview.h` solo usa `__stdcall` si se compila con `FPDFSDK_EXPORTS`, y los binarios de bblanchon no lo definen. Declararlas `stdcall` descuadra la pila en 32 bits y provoca `ACCESS_VIOLATION` aleatorios. En 64 bits no hay diferencia.

## Los cuatro apartados

### 1. Base funcional
- `PdfiumAcquire` carga `pdfium.dll` desde `x86\` o `x64\` según `#If Win64`, llama a `FPDF_InitLibrary` y lleva un recuento de referencias. Si la DLL es de otra arquitectura (error 193), lo informa con un mensaje claro.
- `PdfDocument.OpenFile` / `OpenBytes` usan `FPDF_LoadMemDocument`, así que admiten rutas Unicode y documentos en memoria. También aceptan contraseña (en UTF-8).
- `PdfDocument.RenderTo` renderiza una página o solo una porción de ella sobre un DIB (`FPDFBitmap_CreateEx`), sin copias intermedias.

### 2. Visor
- Las páginas se muestran en vertical y en continuo. Se navega con `GoToPage`, `NextPage`, `PreviousPage`, `FirstPage`, `LastPage` y `CurrentPage`.
- Zoom de 5 % a 6400 %, con los modos `pdfZoomCustom`, `pdfZoomFitWidth` y `pdfZoomFitPage`. Ctrl + rueda amplía alrededor del cursor.
- Desplazamiento con barras de Windows, rueda (Mayús + rueda para el horizontal), teclado y arrastre con la mano. El botón central también arrastra.
- **Sin artefactos:**
  - se ignora `WM_ERASEBKGND` y se pinta todo con doble búfer;
  - cada página visible tiene un *mosaico* que cubre la zona visible más medio cliente de margen, así que desplazarse es solo un `BitBlt`;
  - la memoria no crece con el zoom, porque a 6400 % solo se renderiza lo visible;
  - al redimensionar y hacer zoom, un *ancla* mantiene el mismo punto del documento bajo el cursor o en el borde superior;
  - el cálculo de la disposición se repite hasta que el área cliente se estabiliza, ya que mostrar u ocultar barras cambia su tamaño.
- Rotación en pasos de 90° (`Rotation`, `RotateClockwise`, `RotateCounterClockwise`).

### 3. Herramientas
- **Selección de texto:**
  - arrastrar sobre el texto selecciona, incluso entre páginas y con desplazamiento automático;
  - doble clic selecciona una palabra y Ctrl+A todo el documento;
  - Ctrl+C copia en Unicode;
  - API: `SelectedText`, `HasSelection`, `SelectAll`, `ClearSelection`, `CopySelection`, `GetPageText`.
- **Búsqueda:**
  - `FindText(texto, MatchCase, WholeWord)`, `FindNext` (F3), `FindPrevious` (Mayús+F3) y `ClearSearch`;
  - `SearchResultCount` y `SearchResultIndex` dan el total y la posición actual;
  - recorre el documento en círculo, resalta la coincidencia y la desplaza hasta hacerla visible.
- **Miniaturas:**
  - panel lateral (`ShowThumbnails`, `ThumbnailWidth`) con desplazamiento propio por rueda;
  - la página actual aparece resaltada y un clic navega a la página;
  - las miniaturas se renderizan bajo demanda y se liberan las que quedan lejos.
- **Impresión:**
  - `PrintDocument(ShowDialog, FromPage, ToPage, DocumentName)` muestra el diálogo estándar o usa la impresora predeterminada;
  - cada página se ajusta al área imprimible, se centra y se gira sola si es apaisada;
  - el renderizado está limitado a 300 ppp.

### 4. Integración
- **Eventos:** `DocumentLoaded`, `DocumentClosed`, `LoadFailed`, `PageChanged`, `ZoomChanged`, `SelectionChanged`, `SearchCompleted`, `PageClick` (coordenadas PDF en puntos), `PrintProgress` (cancelable), `ToolbarClick` (cancelable).
- **Propiedades de diseño**, guardadas en el PropertyBag: `BackColor`, `ThumbnailBackColor`, `SelectionColor`, `SearchHighlightColor`, `BorderStyle`, `PageGap`, `Zoom`, `ZoomMode`, `Rotation`, `ShowThumbnails`, `ThumbnailWidth`, `CursorMode`, `PdfiumPath`, `FileName` (se abre automáticamente al ejecutar), `EmptyText`, `ShowToolbar`, `ShowOpenButton` y `ShowSaveButton`. En ejecución, también `ShowSearchPanel`. En diseño se ve una vista previa sin cargar PDFium.
- **DPI:**
  - usa `GetDpiForWindow`, con `LOGPIXELSX` como alternativa;
  - atiende `WM_DPICHANGED_AFTERPARENT`;
  - zoom 100 % = tamaño real (puntos × DPI / 72), y márgenes y fuentes escalan con el DPI.
- **Win32/Win64:**
  - todo usa `LongPtr` y `PtrSafe`;
  - `PRINTDLGW` se construye con desplazamientos distintos para cada arquitectura (66 bytes empaquetada en Win32, 120 en Win64);
  - `LoWordS`/`HiWordS` funcionan en ambas.
- **Pruebas:** `RunPdfTests()` en `PdfTests.twin`. Compila y ejecuta como Win32 y después como Win64. Comprueba:
  - tamaño de puntero;
  - carga de la DLL;
  - páginas, tamaños y renderizado (color en la posición esperada, mosaico al 400 % y rotación);
  - texto, búsqueda y conversión de coordenadas;
  - códigos de error y recuento de referencias.

## Barra de herramientas

La barra se dibuja con **GDI+**: iconos vectoriales con antialias, fondos redondeados y texto. Solo usa `gdiplus.dll`, que viene con Windows, sin dependencias de terceros. Se muestra u oculta con la propiedad `ShowToolbar`.

```
[abrir] [guardar] | [miniaturas] [marcadores] [buscar] | [|<] [<] [ 3 / 10 ] [>] [>|] | [-] [ 100 % ] [+] [|<->|] [ajustar página] | [resaltar] [tachar] [comentario] | [imprimir] [organizar] [zonas] [formulario]
```

| Grupo | Elementos |
|---|---|
| Archivo | **Abrir** y **Guardar** con el cuadro de diálogo de Windows. Se ocultan desde código con `ShowOpenButton` / `ShowSaveButton` |
| Paneles | Miniaturas, marcadores y búsqueda. Abren un panel a la izquierda; son excluyentes y el botón activo queda resaltado |
| Páginas | Primera, anterior, etiqueta «página / total», siguiente, última |
| Zoom | `−`, zoom actual y `+` en tramos de 25 % (110 % → 125 % / 100 %); **ajustar al ancho** y **ajustar página**, resaltados cuando su modo está activo |
| Anotaciones | Resaltar, tachar y comentar (ver abajo) |
| Impresión | Abre el diálogo de impresión |
| Organizar | Vista de miniaturas para reordenar páginas arrastrándolas (ver abajo) |
| Zonas | Herramienta rectángulo para marcar zonas con nombre y guardarlas en un INI (ver abajo) |
| Formulario | Panel con los campos del formulario y los botones Exportar / Importar. Solo se activa si el PDF tiene formulario |

### Panel de búsqueda

```
┌───────────────────────────────────────┐
│ [ Buscar en el documento...         ] │  ← EDIT con fondo blanco
│ Resultados de la búsqueda 2/7    ▲ ▼  │
│ ┌───────────────────────────────────┐ │
│ │ Página 1                          │ │
│ │ Hola uc[PDF] - pagina 1 Documen…  │ │  ← resultado actual resaltado
│ ├───────────────────────────────────┤ │
│ │ Página 2                          │ │
│ │ …Busqueda: uc[PDF] aparece aqui…  │ │
│ └───────────────────────────────────┘ │
└───────────────────────────────────────┘
```

- Debajo del cuadro de texto aparece la **lista de resultados**. Cada elemento muestra la página y un extracto: unos 24 caracteres antes y 70 después, empezando en una palabra completa. El texto encontrado va en negrita sobre el color `SearchHighlightColor`.
- Un clic en un resultado salta a esa coincidencia. El resultado actual queda resaltado y la lista lo mantiene a la vista al usar ▲ / ▼, Intro o F3.
- La lista se desplaza con la rueda del ratón y muestra un indicador de posición. Solo se dibujan los elementos visibles, así que admite miles de resultados.
- Método `SelectSearchResult(Index)` (base 0) para ir a un resultado desde código.

- **Intro** busca. Si el texto no ha cambiado, va a la siguiente coincidencia; **Mayús+Intro** va a la anterior.
- **▲ / ▼** recorren los resultados en círculo y **Esc** devuelve el foco al visor.
- **Ctrl+F** abre el panel y pone el foco en el cuadro de búsqueda.
- La búsqueda recorre todo el documento, por eso muestra el total (`x/x`).
- El cuadro de texto es un `EDIT` de Win32 creado como ventana hija. El resto del panel se dibuja con GDI+.
- Propiedad `ShowSearchPanel`. El panel mide como mínimo 260 px lógicos de ancho.

El evento `ToolbarClick(Command, Cancel)` permite interceptar o sustituir cualquier botón, incluidos los del panel.

Al abrir un PDF **no se muestra ningún panel**: ni miniaturas ni búsqueda. Se abren con sus botones.

## Organizar páginas

El botón **Organizar** (después de Imprimir) sustituye la vista por una cuadrícula de miniaturas:

```
┌ Organizar páginas — arrastra una miniatura…   ● Cambios sin guardar   Tamaño ──●──── ┐
│  ┌───┐   ┌───┐  ┊┌┄┄┄┐  ┌───┐                                                         │
│  │ 1 │   │ 2 │  ┊┊   ┊  │ 4 │      ← hueco punteado y barra: dónde quedará la página   │
│  └───┘   └───┘  ┊└┄┄┄┘  └───┘                                                         │
│                        ╔═══╗                                                          │
│                        ║ 3 ║  ← página arrastrada: ampliada, translúcida, con sombra   │
│                        ╚═══╝                                                          │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

**Arrastrar páginas:**
- Se arrastra una miniatura para cambiar su posición; por ejemplo, poner la segunda delante de la primera.
- **Efecto óptico:**
  - la página arrastrada se ve un 6 % más grande, translúcida, con sombra y borde de color;
  - el resto de miniaturas se desliza con animación suavizada para dejar libre la casilla de destino;
  - esa casilla se marca con un recuadro punteado y una barra de inserción.
- Al soltar, la página vuela a su sitio y destella brevemente. Cerca de los bordes, la vista se desplaza sola.
- La vista previa coincide siempre con el orden final; lo he comprobado para todas las combinaciones de hasta 6 páginas.

**Controles:**
- **Tamaño:** control deslizante en la cabecera o Ctrl+rueda, de 80 a 360 px lógicos (`OrganizeThumbSize`). Al cambiar el ancho, la cuadrícula se recoloca con animación.
- Clic para seleccionar una página; doble clic para volver a la vista normal en esa página.
- Esc cancela el arrastre o sale del modo organizar.

**Cambios en el documento:**
- Al soltar se reordena el PDF real con `FPDF_MovePages`. El documento queda **con cambios sin guardar** (`Modified = True`) y se lanza `PagesReordered(FromPage, ToPage)`.
- Al guardar, PDFium genera un PDF nuevo (`FPDF_SaveAsCopy`) con el nuevo orden.

### Botones de edición (cabecera de Organizar)

```
Organizar páginas  [↶][↷] | [↺][↻] | [🗑] | [+□][+📄][⇄] | [⇱][⌗]   2 seleccionada(s) — Ctrl/Mayús para varias   Tamaño ──●──
```

Todas las acciones se aplican **solo a la selección**, que puede tener una o varias páginas:
- **Selección:** clic selecciona una página, **Ctrl+clic** añade o quita, **Mayús+clic** selecciona un intervalo, **Ctrl+A** todas, y un clic en un hueco vacío la anula.
- **Información:** al dejar el ratón sobre cualquier botón aparece una descripción.
- **Si no caben:** los botones que no caben en la cabecera se ocultan, pero las acciones siguen disponibles por código.

| Botón | Método | Acción |
|---|---|---|
| Deshacer / Rehacer (Ctrl+Z / Ctrl+Y) | `Undo()`, `Redo()` | **Sin límite de pasos.** Cubre mover, girar, eliminar, insertar, reemplazar y recortar |
| Girar a la izquierda / derecha | `RotateSelectedPages(Clockwise)` | Gira 90° cada página seleccionada (`/Rotate`) |
| Eliminar (Supr) | `DeleteSelectedPages(Confirm)` | Pide confirmación. Nunca deja el documento sin páginas; las demás se deslizan a los huecos |
| Insertar página | `InsertBlankPage()` | Página en blanco, del tamaño de la última seleccionada, **justo después de la selección** |
| Insertar desde archivo | `InsertPagesFromFile([FileName])` | Elige un PDF e inserta todas sus páginas después de la selección |
| Reemplazar | `ReplaceSelectedPages([FileName])` | Sustituye cada página seleccionada por la página correspondiente del archivo elegido (1ª por 1ª, 2ª por 2ª…) |
| Extraer | `ExtractSelectedPages([FileName])` | Guarda la selección en un **PDF nuevo**. Propone `<nombre>_paginas.pdf` en la misma carpeta; el documento actual no cambia |
| Recortar | `CropSelectedPages()` | Abre el **editor de recorte**: se dibuja un rectángulo sobre la primera página seleccionada y se aplica a toda la selección (ver abajo) |

#### Deshacer y rehacer

- **Cómo funciona:** tras cada cambio se guarda una instantánea completa del documento en `%TEMP%`. Como no se guarda en memoria, no hay límite de pasos. Las instantáneas se borran al cerrar el documento o el control.
- **Animación:** cada página lleva un identificador interno. Al deshacer o rehacer, las páginas que ya existían se deslizan a su posición anterior y conservan la miniatura, y las que reaparecen lo hacen con un destello.
- **Cambios sin guardar:** el aviso tiene en cuenta el historial. Si deshaces hasta el estado guardado, el documento deja de estar «con cambios sin guardar».
- **Nuevo cambio tras deshacer:** se descartan los pasos de rehacer, como en cualquier editor.
- **Por código:** `CanUndo`, `CanRedo`, `UndoCount`, `RedoCount`. El evento `PagesEdited` llega con `pdfOpUndo` / `pdfOpRedo`.

#### Editor de recorte

```
┌──────────────────────────────────────────────────────────┐
│ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │  ░ = zona que se descarta (oscurecida)
│ ░░░░░ ■━━━━━━━━━━━━■━━━━━━━━━━━━■ ░░░░░░░░░░░░░░░░░░░░ │
│ ░░░░░ ┃     ┊           ┊        ┃ ░░░░░░░░░░░░░░░░░░░ │  ■ = tiradores (esquinas y lados)
│ ░░░░░ ■ ┄┄┄┄┼┄┄┄┄┄┄┄┄┄┄┄┼┄┄┄┄┄┄┄ ■ ░░░░░░░░░░░░░░░░░░░ │  ┊ = guías de tercios
│ ░░░░░ ┃     ┊           ┊        ┃ ░░░░░░░░░░░░░░░░░░░ │
│ ░░░░░ ■━━━━━━━━━━━━■━━━━━━━━━━━━■ ░░░░░░░░░░░░░░░░░░░░ │
├──────────────────────────────────────────────────────────┤
│ Dibuja o ajusta el rectángulo — 70 x 53 mm — se aplicará a 3 página(s)   [Ajustar al contenido] [Cancelar] [Recortar] │
└──────────────────────────────────────────────────────────┘
```

**Manejo:**
- Se abre con el rectángulo ya ajustado al contenido de la página, con 8 pt de margen.
- Arrastrar fuera del rectángulo dibuja uno nuevo, arrastrar dentro lo mueve, y los ocho tiradores cambian su tamaño. El cursor cambia según la zona.
- La barra inferior muestra el tamaño resultante en milímetros y a cuántas páginas se aplicará.
- **Recortar** (o Intro) aplica el rectángulo y **Cancelar** (o Esc) cierra sin cambios.

**Cómo se aplica:**
- El rectángulo se guarda **en proporción a la página**, así que en páginas de distinto tamaño recorta la misma zona relativa.
- Se tiene en cuenta el giro y cualquier recorte anterior de cada página.

**Por código:**
- `CropSelectedPagesMargins(LeftPt, TopPt, RightPt, BottomPt)` recorta por márgenes en puntos.
- `CropSelectedPagesAuto(MarginPt)` hace el recorte automático al contenido.
- `ApplyCrop`, `CancelCrop` y `CropEditorActive` controlan el editor.

**Comportamiento común:**
- Tras cada cambio, las miniaturas se recolocan con animación y las páginas nuevas aparecen con un destello.
- El documento queda con cambios sin guardar y se lanza `PagesEdited(Operation, FirstPage, Count)`. Extraer lanza `PagesExtracted(FileName, Count)`.
- `SelectedPages` devuelve la selección como texto (`"1,3,4"`). Fuera del modo organizar, los métodos actúan sobre la página actual.

### Aviso de cambios sin guardar

Si hay cambios sin guardar, el control pregunta **Sí / No / Cancelar** antes de perderlos:
- al **cerrar la ventana principal**: el control intercepta `WM_CLOSE` y `WM_QUERYENDSESSION`, y Cancelar impide el cierre. Se desactiva con `ConfirmSaveOnClose = False`;
- al abrir otro PDF con el botón Abrir.

«Sí» abre el diálogo de guardar; si se cancela ese diálogo, tampoco se continúa. Desde código se puede usar `ConfirmSaveChanges()`, que devuelve `True` si se puede continuar. Por ejemplo, si el formulario se descarga con `Unload` en vez de cerrarse:

```vb
Private Sub Form_QueryUnload(Cancel As Integer, UnloadMode As Integer)
    If UnloadMode <> vbFormControlMenu Then Cancel = Not ucPDF1.ConfirmSaveChanges()
End Sub
```

## Abrir y guardar

| Miembro | Comportamiento |
|---|---|
| `ShowOpenDialog()` | Diálogo **Abrir** (filtro `*.pdf`). Empieza en la carpeta del documento actual o en la última usada. Devuelve `True` si se cargó el PDF (Ctrl+O) |
| `ShowSaveDialog()` | Diálogo **Guardar como**. Propone **la carpeta de donde se abrió y el mismo nombre de archivo**, y pide confirmación antes de sobrescribir (Ctrl+S) |
| `SaveAs(FileName)` | Guarda sin diálogo una copia exacta del PDF abierto. A partir de ahí, `FileName` apunta a la nueva ruta |
| `ShowOpenButton`, `ShowSaveButton` | Muestran u ocultan los botones de la barra. El separador desaparece si se ocultan los dos |
| Evento `DocumentSaved(FileName)` | Se lanza después de guardar |

**Detalles:**
- Se guardan los bytes originales que el control mantiene en memoria. Por eso se puede sobrescribir el mismo archivo del que se abrió y se admiten rutas Unicode.
- Los diálogos son los estándar de Windows (`GetOpenFileNameW` / `GetSaveFileNameW` de `comdlg32`, sin dependencias externas).

```vb
ucPDF1.ShowOpenButton = False     ' ocultar Abrir
ucPDF1.ShowSaveButton = False     ' ocultar Guardar
```

## Impresión

| Método | Comportamiento |
|---|---|
| `PrintDocument(ShowDialog, FromPage, ToPage, DocumentName)` | Muestra el diálogo de impresión estándar (o usa la predeterminada si `ShowDialog = False`) |
| `PrintToDefaultPrinter(FromPage, ToPage, Copies, DocumentName)` | Envía el documento **directamente** a la impresora predeterminada de Windows, sin diálogos (`GetDefaultPrinterW` + `CreateDCW`) |
| `DefaultPrinterName()` | Nombre de la impresora predeterminada (`""` si no hay) |

Los dos métodos de impresión ajustan cada página al área imprimible, la centran y giran las apaisadas. El evento `PrintProgress` permite cancelar.

## Zonas (rectángulos con nombre)

El botón **Zonas** de la barra (o `AreaMode = True`) activa una herramienta rectángulo y abre el panel **Zonas** a la izquierda.

- **Crear:** arrastra sobre una página. Al soltar se pide el nombre (propone `Zona1`, `Zona2`…). La zona se ajusta a esa página y mide al menos 4 × 4 pt.
- **Editar:** clic en una zona para seleccionarla y moverla. Se cambia el tamaño con los 8 tiradores, se renombra con doble clic o F2, y se borra con Supr. Las flechas la mueven 1 pt (10 pt con Mayús).
- **Panel:** muestra el nombre, la página, las coordenadas y el texto que captura cada zona. Clic en una zona de la lista para ir a ella. Botones **Guardar INI**, **Abrir INI** y **Borrar todo**.
- **Salir:** Esc quita la selección y, si no hay ninguna, sale de la herramienta.
- **Persistencia:** las zonas se conservan al abrir otro PDF, para aplicar la misma plantilla a varios documentos del mismo tipo.

El INI guarda **una clave por zona: nombre = página, izquierda, arriba, derecha, abajo**.
- Las medidas van en puntos desde la esquina superior izquierda de la página, con punto decimal.
- Son exactamente los argumentos de `GetTextInRect`.
- El archivo se escribe en UTF-16 para admitir cualquier nombre, de modo que también lo lee `GetPrivateProfileStringW`.

```ini
; Zonas de ucPDF
[Info]
Unidades=pt
Origen=superior-izquierda
Documento=factura.pdf
Total=2

[Zonas]
NumFactura=1,410.50,72.00,560.00,90.25
Total=1,420.00,680.00,560.00,700.00
```

```vb
' Leer una zona del INI y extraer su texto
Dim v() As String
v = Split("1,410.50,72.00,560.00,90.25", ",")
Debug.Print ucPDF1.GetTextInRect(CLng(v(0)), Val(v(1)), Val(v(2)), Val(v(3)), Val(v(4)), "pt")

' O directamente
ucPDF1.LoadAreasIni "C:\plantillasactura_zonas.ini"
Debug.Print ucPDF1.GetAreaText("NumFactura")
Debug.Print ucPDF1.ExtractAreasText()          ' NumFactura=F-2026/0153 ...
```

> Usa `Val` (no `CDbl`) para leer los números: el INI siempre usa punto decimal, sea cual sea la configuración regional.

| Miembro | Descripción |
|---|---|
| `AreaMode` | Activa o desactiva la herramienta |
| `ShowAreasPanel` | Muestra el panel de zonas sin activar la herramienta |
| `AreaCount`, `AreaName(i)`, `AreaPage(i)`, `GetAreaRect(i, L, T, R, B)`, `SelectedArea`, `FindArea(nombre)` | Consulta (índice base 0, página base 1) |
| `AddArea(nombre, página, L, T, R, B, [Units])`, `RemoveArea(nombre)`, `ClearAreas`, `GoToArea(i)` | Edición desde código |
| `SaveAreasIni(archivo)`, `LoadAreasIni(archivo, [AppendAreas])` | Guardar o cargar el INI |
| `ShowSaveAreasDialog`, `ShowLoadAreasDialog` | Lo mismo con cuadro de diálogo. Por defecto propone `<pdf>_zonas.ini` en la carpeta del PDF |
| `GetAreaText(nombre)`, `ExtractAreasText([«ini»\|«tab»])` | Texto de una zona o de todas |
| Eventos `AreaCreated(ZoneName, Page, L, T, R, B)` y `AreasChanged` | Avisan de los cambios |

Sin interfaz, `PdfOps` ofrece `WriteAreasIni`, `ReadAreasIni` y `AreasText(doc, ini)`. Desde la línea de comandos se usa `areas` (ver más abajo).

## Resaltado, tachado y comentarios

Hay tres botones en la barra, entre el zoom y la impresión:

| Botón | Uso |
|---|---|
| **Resaltar** | Con texto seleccionado, lo resalta al momento. Sin selección, activa la herramienta: cada texto que selecciones después queda resaltado, hasta que vuelvas a pulsar el botón o Esc |
| **Tachar** | Igual, pero tacha el texto |
| **Comentario** | Haz clic en la página, escribe el texto y queda una nota (el icono de globo). La herramienta se desactiva tras colocar la nota |

- **Ver el texto:** al pasar el ratón sobre una anotación con texto aparece un globo con el autor y el comentario.
- **Seleccionar:** clic en una nota, o un clic sin arrastrar sobre un resaltado o un tachado. La selección se muestra con un marco discontinuo.
- **Editar y borrar:** doble clic edita el texto de cualquier anotación; Supr borra la seleccionada.
- **Historial:** todo se puede deshacer con Ctrl+Z.
- **Guardar:** las anotaciones se guardan en el PDF como anotaciones estándar (`/Highlight`, `/StrikeOut`, `/Text`) con autor y fecha. Acrobat y los demás visores las muestran y las editan.
- **Dibujo:** PDFium genera su apariencia.
- **Selección de varias líneas o páginas:** se crea una anotación por página, con un cuadrilátero por línea de texto.

| Miembro | Descripción |
|---|---|
| `AnnotTool` | Herramienta activa: `pdfAnnotNone`, `pdfAnnotHighlight`, `pdfAnnotStrikeout`, `pdfAnnotComment` |
| `HighlightSelection([Comentario])`, `StrikeoutSelection([Comentario])` | Marcan la selección actual |
| `AddComment(Página, Izquierda, Arriba, Texto)` | Nota en puntos desde la esquina superior izquierda (página base 1) |
| `AnnotationCount(Página)`, `AnnotationType(Página, i)`, `GetAnnotationText`, `SetAnnotationText`, `DeleteAnnotation`, `DeleteSelectedAnnotation` | Consulta y edición (índice base 0 dentro de la página) |
| `AnnotationsText()` | Lista: `Página⇥Tipo⇥Autor⇥Texto` |
| `HighlightColor`, `StrikeoutColor`, `CommentColor`, `AnnotationAuthor` | Colores y autor de las anotaciones nuevas (también en diseño; autor por defecto: el usuario de Windows) |
| Evento `AnnotationsChanged(Página)` | Se ha añadido, cambiado o borrado una anotación |

Sin interfaz, `PdfDocument` ofrece `AddTextMarkup`, `AddNote`, `AddNoteTopLeft`, `AnnotCount`, `AnnotSubtype`, `AnnotString`, `SetAnnotContents`, `AnnotAtPoint` y `RemoveAnnot`. `PdfOps` ofrece `AnnotationsReport` y `MarkupSearch`, que resalta todas las apariciones de un texto.

## Marcadores

El botón **Marcadores**, junto al de miniaturas (o `ShowBookmarksPanel = True`), abre el esquema del documento en el panel izquierdo:
- **Árbol:** los marcadores aparecen sangrados por nivel, con su página a la derecha. Los de nivel superior van en negrita.
- **Desplegar:** el triángulo despliega o pliega un nivel; el doble clic también. **Expandir todo** y **Contraer todo** actúan sobre el árbol entero. Al abrir el PDF se respeta el estado desplegado o plegado que guarda el documento.
- **Ir:** clic en un título para ir a su destino. Si el destino indica una posición, esa línea queda arriba de la vista; si no, se va al principio de la página.
- **Enlaces web:** los marcadores con enlace web abren el navegador (solo `http`, `https` y `mailto`).
- **Sección actual:** una barra de color marca la sección que se está leyendo, es decir, el último marcador cuya página es la actual o anterior.

| Miembro | Descripción |
|---|---|
| `ShowBookmarksPanel` | Muestra u oculta el panel |
| `BookmarkCount`, `BookmarkTitle(i)`, `BookmarkLevel(i)`, `BookmarkPage(i)`, `BookmarkUri(i)` | Consulta (índice base 0 en orden de lectura; nivel 0 = superior; página base 1, 0 si no tiene) |
| `GoToBookmark(i)`, `ExpandAllBookmarks`, `CollapseAllBookmarks` | Navegación |
| `BookmarksText()` | Esquema como texto: una línea por marcador, sangrada por nivel, con tabulador y página |
| Evento `BookmarkClick(Index, Title, Page, Cancel)` | Antes de navegar. `Cancel = True` lo impide |

Sin interfaz, `PdfDocument` ofrece `BookmarkCount`, `BookmarkTitle`, `BookmarkLevel`, `BookmarkPage`, `BookmarkTopPt` (posición desde arriba, en puntos) y `BookmarkUri`. `PdfOps.BookmarksReport` devuelve el esquema como texto.

## Formularios

Si el PDF tiene un formulario (AcroForm), sus campos se dibujan y se rellenan directamente sobre la página, como en Acrobat.

> PDFium solo dibuja los campos con su entorno de formularios (`FPDFDOC_InitFormFillEnvironment` + `FPDF_FFLDraw`). `PdfDocument` lo crea al abrir cualquier PDF con formulario. Antes de este cambio los campos no se veían.

**Rellenar con el ratón y el teclado:**
- Clic en un campo para escribir. Los campos se resaltan en azul claro (`FormHighlight`, `FormHighlightColor`); el resaltado no se imprime.
- Tipos admitidos: texto, multilínea, casillas, radios, combos y listas, incluida la selección múltiple.
- Teclas:
  - Tab y Mayús+Tab pasan al campo siguiente o al anterior.
  - Intro confirma un campo de una línea. En uno multilínea inserta un salto.
  - Esc confirma el campo y quita el foco.
  - Ctrl+A, Ctrl+C, Ctrl+X y Ctrl+V seleccionan, copian, cortan y pegan dentro del campo.
  - Ctrl+Z y Ctrl+Y deshacen o rehacen lo escrito en el campo.
- **Historial:** cada campo confirmado es un paso del historial. Fuera de un campo, Ctrl+Z y Ctrl+Y deshacen y rehacen los cambios del formulario, sin límite de pasos.
- **Guardar:** los valores se guardan con el botón **Guardar** (o `SaveAs`). El aviso de cambios sin guardar también los tiene en cuenta.
- **Desactivar la edición:** `FormFilling = False` deja los campos visibles pero no editables.

**Panel Formulario** (botón de la barra o `ShowFormPanel = True`):
- Lista cada campo con su nombre (`*` si es obligatorio), tipo, página y valor actual. Al hacer clic en uno, la vista va al campo y le da el foco.
- **Exportar...** guarda los valores. **Importar...** los carga e informa de los campos que no existen y de los valores no válidos.

**Formatos de datos** (se eligen por la extensión):

| Formato | Para qué |
|---|---|
| `.xfdf` | XML de Adobe. Acrobat lo importa y lo exporta. Los nombres `a.b` se escriben como `<field>` anidados |
| `.fdf` | Forms Data Format de Adobe, con cadenas UTF-16 y `/Kids` para los nombres jerárquicos. Se leen cadenas literales, hexadecimales y nombres |
| `.ini` | `[Campos]` y una línea `Nombre=Valor` por campo. Es fácil de generar desde otro programa. Los saltos de línea se escriben `\n` y la barra `\\` |

Los valores siguen estas reglas:
- **Texto, combo y lista simple:** el valor tal cual.
- **Lista múltiple:** las opciones separadas por `|`, por ejemplo `en|de`.
- **Casilla:** su valor de exportación (normalmente `Yes`) u `Off`. Al importar también se acepta `Si`, `1`, `true` o `X`.
- **Radio:** el valor de exportación de la opción elegida.
- **Al importar:** los campos de solo lectura y los que ya tienen ese valor no se tocan.

```ini
[Campos]
nombre=José Pérez
acepto=Yes
sexo=M
pais=Portugal
idiomas=en|de
comentarios=línea 1\nlínea 2
```

```vb
Debug.Print ucPDF1.HasForm, ucPDF1.FormFieldCount
ucPDF1.SetFieldValue "nombre", "José Pérez"
ucPDF1.SetFieldValue "acepto", "Yes"
Debug.Print ucPDF1.GetFieldValue("pais")
ucPDF1.ExportFormData "C:\datos\cliente.xfdf"
ucPDF1.ImportFormData "C:\datos\cliente.ini"
ucPDF1.SaveAs "C:\datos\cliente_relleno.pdf"
```

| Miembro | Descripción |
|---|---|
| `HasForm`, `FormFieldCount` | ¿Hay formulario? ¿Cuántos campos? |
| `FormFieldName(i)`, `FormFieldType(i)`, `FormFieldTypeName(i)`, `FormFieldPage(i)`, `FormFieldOptions(i)`, `FormFieldReadOnly(i)`, `FormFieldRequired(i)`, `FindFormField(nombre)` | Consulta (índice base 0, página base 1; tipos: 1 botón, 2 casilla, 3 radio, 4 combo, 5 lista, 6 texto, 7 firma) |
| `GetFieldValue(nombre)`, `SetFieldValue(nombre, valor)` | Leer y escribir valores |
| `ExportFormData(archivo, [formato])`, `ImportFormData(archivo, [formato], [informe])` | Exportar e importar. Devuelven cuántos campos (-1 si falla) |
| `ShowExportFormDialog`, `ShowImportFormDialog` | Lo mismo con cuadro de diálogo. Propone `<pdf>.xfdf` en la carpeta del PDF |
| `GoToFormField(i)` | Lleva la vista al campo y le da el foco |
| `FormFieldsText()` | Lista de campos: `Nombre⇥Tipo⇥Página⇥Valor⇥Opciones⇥Indicadores` |
| `FlattenForm()` | Aplana: los valores pasan a ser contenido fijo de la página (se puede deshacer). También aplana las demás anotaciones |
| `FormFilling`, `FormHighlight`, `FormHighlightColor`, `ShowFormPanel` | Propiedades (las tres primeras, también en diseño) |
| Evento `FormChanged` | Un valor ha cambiado: al confirmar un campo, marcar una casilla, importar… |

Sin interfaz, `PdfDocument` ofrece `FieldCount`, `FieldName`, `FieldValue`, `SetFieldValue`, `FieldOptions`, `FindField` y `FlattenForms`. `PdfFormData` ofrece `ExportFormData`, `ImportFormData`, `ReadFormData`, `ApplyFormValues` y `FormFieldsReport`.

## API completa y línea de comandos

Todas las funciones están en **tres capas**:

| Capa | Archivo | Para qué |
|---|---|---|
| Motor sin interfaz | `PdfOps.twin` + `PdfDocument.twin` | Operaciones sobre un `PdfDocument` o sobre rutas, sin formulario |
| Control | `ucPDF` (métodos públicos) | Lo mismo sobre el documento abierto, con deshacer/rehacer y la vista actualizada |
| Línea de comandos | `modCommandLine.twin` (`Sub Main`) | Ejecutar el programa anfitrión desde CMD, scripts o tareas programadas |

**Rangos de páginas** (base 1): `all`, `1-3,5`, `8-`, `-4`, `last`, `3-last`, `odd`, `even`. También en español: `todas`, `ultima`, `impares`, `pares`.

**Coordenadas:** se miden desde la esquina superior izquierda de la página tal como se ve, ya girada y recortada. Unidades: `pt` (1/72"), `mm`, `cm`, `in` o `%` del ancho o alto.

### Métodos del control

| Texto y consulta | Páginas | Salida |
|---|---|---|
| `GetDocumentText([Pages], [Separator])` | `ExtractPagesTo(Pages, File)` | `ExportPageImage(Page, File, [Dpi], [Quality])` |
| `ExportText(File, [Pages], [Encoding], [Separator])` | `DeletePages(Pages)` | `ExportPagesToImages(Pages, Pattern, [Dpi])` |
| `GetTextInRect(Page, L, T, R, B, [Units])` | `RotatePages(Pages, Angle)` | `PrintPagesTo([Pages], [Printer], [Copies])` |
| `ExportTextInRect(File, Page, L, T, R, B, …)` | `MovePage(From, To)` | `SplitTo(Folder, [Every], [BaseName])` |
| `SearchAll(Text, [MatchCase], [WholeWord], [Count])` | `InsertBlankPageAfter(After, [W], [H])` | `MergePdfFiles(Files(), OutFile)` |
| `GetDocumentInfo()`, `GetMetadata(Tag)` | `InsertFileAfter(File, After)` | `SaveAs(File)` |
| | `ReplacePagesFromFile(Pages, File)` | `RunCommand(CommandLine, [Output])` |
| | `CropPages(Pages, L, T, R, B, [Units])`, `CropPagesAuto(Pages)` | |

**Comportamiento:**
- Las ediciones pasan por el historial (deshacer/rehacer) y no muestran mensajes; si algo falla, el motivo queda en `LastOperationError`.
- `Encoding` del texto: `utf8` (con BOM), `utf8nobom`, `utf16` o `ansi`.
- `Separator`: `ff` (salto de página, como `pdftotext`), `header` (`===== Página N =====`) o `none`.

```vb
ucPDF1.ExportText "C:\salida\todo.txt"
Debug.Print ucPDF1.GetTextInRect(1, 20, 30, 100, 45, "mm")   ' zona de la página 1
ucPDF1.DeletePages "2,4-5"
ucPDF1.ExtractPagesTo "1-3", "C:\salida\primeras.pdf"
ucPDF1.ExportPagesToImages "all", "C:\salida\pag_{n}.png", 200
```

### Línea de comandos

Pon **«Startup object = Sub Main»** en el proyecto.
- **Sin argumentos:** se abre `Form1`. Cambia ese nombre en `modCommandLine.Main` si tu formulario se llama de otra forma.
- **Con argumentos:** se ejecuta el comando y el programa termina con un código de salida.

```
programa.exe info      factura.pdf
programa.exe text      factura.pdf factura.txt --pages 1-2 --encoding utf8
programa.exe textrect  factura.pdf total.txt --page 1 --rect 120,240,190,252 --units mm
programa.exe areas     factura.pdf factura_zonas.ini datos.txt --format ini
programa.exe bookmarks  manual.pdf indice.txt
programa.exe highlight  contrato.pdf revisado.pdf "importe total" --comment "Revisar"
programa.exe highlight  contrato.pdf revisado.pdf "cláusula 7" --strike --color FF0000
programa.exe annots     revisado.pdf anotaciones.txt
programa.exe fields     solicitud.pdf campos.txt
programa.exe formexport solicitud_rellena.pdf datos.xfdf
programa.exe formimport solicitud.pdf datos.ini rellena.pdf --flatten
programa.exe fill      solicitud.pdf rellena.pdf nombre="Ana Ruiz" acepto=Yes idiomas="en|de"
programa.exe flatten   rellena.pdf definitiva.pdf
programa.exe search    contrato.pdf "importe total" --out resultados.txt
programa.exe extract   in.pdf out.pdf --pages 1-3,7
programa.exe delete    in.pdf out.pdf --pages 2,4
programa.exe rotate    in.pdf out.pdf --pages odd --angle 90
programa.exe move      in.pdf out.pdf --page 5 --to 1
programa.exe insertblank in.pdf out.pdf --after 2 --size 210x297 --units mm
programa.exe insert    in.pdf out.pdf --from anexo.pdf --after last
programa.exe replace   in.pdf out.pdf --from nuevas.pdf --pages 3-4
programa.exe crop      in.pdf out.pdf --pages all --rect 10,10,200,287 --units mm
programa.exe crop      in.pdf out.pdf --auto --margin 12
programa.exe merge     total.pdf a.pdf b.pdf c.pdf
programa.exe split     in.pdf C:\partes --every 5
programa.exe image     in.pdf C:\img\pag_{n}.png --pages all --dpi 200
programa.exe print     in.pdf --printer "HP LaserJet" --pages 1-2 --copies 2
programa.exe help
```

**Opciones generales:** `--password`, `--pdfium <ruta>` y `--quiet` (o `-q`, que solo deja salir errores y resultados).
- **Formatos:** las opciones admiten `--nombre valor` y `--nombre=valor`, y las rutas con espacios van entre comillas.
- **Archivo de salida:** puede ser el mismo de entrada, porque el PDF se lee entero a memoria antes de escribir.
- **Sin archivo de salida:** en `text`, `textrect`, `areas`, `bookmarks`, `annots`, `fields` y `search` el resultado sale por la salida estándar; si se redirige (`> archivo`), se escribe en UTF-8.

**Códigos de salida:**

| Código | Significado |
|---|---|
| 0 | Correcto |
| 1 | Uso incorrecto |
| 2 | No se pudo abrir un archivo |
| 3 | La operación falló |
| 4 | PDFium no disponible |
| 5 | Sin resultados (`search`), INI sin zonas (`areas`), PDF sin marcadores (`bookmarks`) o sin anotaciones (`annots`), texto no encontrado (`highlight`), PDF sin formulario o ningún campo asignado (`fields`, `formexport`, `formimport`, `fill`) |

> Si el EXE es de subsistema Windows (lo normal en twinBASIC), CMD no espera a que termine. Para ver la salida y el código usa `start /wait programa.exe …` o compílalo como aplicación de consola. Desde un `.bat`, PowerShell o un programa que lo lance y espere, funciona directamente.

## Uso rápido

```vb
Private Sub Form_Load()
    ucPDF1.PdfiumPath = App.Path            ' busca x86\ o x64\ automáticamente
    If Not ucPDF1.LoadFile(App.Path & "\samples\sample.pdf") Then
        MsgBox ucPDF1.LastErrorText
    End If
End Sub

Private Sub ucPDF1_PageChanged(ByVal Page As Long)
    lblPagina.Caption = Page & " / " & ucPDF1.PageCount
End Sub

Private Sub cmdBuscar_Click()
    ucPDF1.FindText txtBuscar.Text
End Sub

Private Sub cmdImprimir_Click()
    ucPDF1.PrintDocument True
End Sub
```

## Atajos de teclado

| Tecla | Acción |
|---|---|
| ↑ ↓ ← → | Desplazar |
| RePág / AvPág | Pantalla anterior / siguiente (con Ctrl: página) |
| Inicio / Fin | Primera / última página |
| Ctrl + `+` / `-` / `0` | Ampliar / reducir (tramos de 25 %) / 100 % |
| Ctrl+F | Abrir el panel de búsqueda |
| Ctrl+O / Ctrl+S | Abrir / guardar con diálogo (si el botón está visible) |
| Esc (organizar) | Cancelar el arrastre o salir de organizar |
| Ctrl+Z / Ctrl+Y (organizar) | Deshacer / rehacer (también Ctrl+Mayús+Z) |
| Ctrl + rueda (organizar) | Tamaño de las miniaturas |
| Ctrl + rueda | Zoom en el punto del cursor |
| Ctrl+A / Ctrl+C | Seleccionar todo / copiar |
| F3 / Mayús+F3 | Siguiente / anterior coincidencia |
| Supr / Esc (anotación seleccionada) | Borrar / quitar la selección; Esc también desactiva Resaltar, Tachar o Comentario |
| Tab / Mayús+Tab (formulario) | Campo siguiente / anterior |
| Intro / Esc (formulario) | Confirmar el campo (Intro en multilínea: salto de línea) |
| Ctrl+Z / Ctrl+Y (vista normal) | Deshacer / rehacer cambios del formulario |
| Supr / F2 / Esc / flechas (zonas) | Borrar, renombrar, deseleccionar o salir, mover 1 pt (10 con Mayús) |
