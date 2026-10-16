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
| `Sources/Win32Api.twin` | Declaraciones Win32 y utilidades |
| `Sources/PdfTests.twin` | Pruebas automáticas para Win32/Win64 |
| `samples/sample.pdf` | PDF de 2 páginas que usan las pruebas |

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
[abrir] [guardar] | [miniaturas] [buscar] | [|<] [<] [ 3 / 10 ] [>] [>|] | [-] [ 100 % ] [+] [|<->|] [ajustar página] | [imprimir] [organizar]
```

| Grupo | Elementos |
|---|---|
| Archivo | **Abrir** y **Guardar** con el cuadro de diálogo de Windows. Se ocultan desde código con `ShowOpenButton` / `ShowSaveButton` |
| Paneles | Miniaturas y búsqueda. Abren un panel a la izquierda; son excluyentes y el botón activo queda resaltado |
| Páginas | Primera, anterior, etiqueta «página / total», siguiente, última |
| Zoom | `−`, zoom actual y `+` en tramos de 25 % (110 % → 125 % / 100 %); **ajustar al ancho** y **ajustar página**, resaltados cuando su modo está activo |
| Impresión | Abre el diálogo de impresión |
| Organizar | Vista de miniaturas para reordenar páginas arrastrándolas (ver abajo) |

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
Organizar páginas  [↺][↻] | [🗑] | [+□][+📄][⇄] | [⇱][⌗]   2 seleccionada(s) — Ctrl/Mayús para varias   Tamaño ──●──
```

Todas las acciones se aplican **solo a la selección**, que puede tener una o varias páginas:
- **Selección:** clic selecciona una página, **Ctrl+clic** añade o quita, **Mayús+clic** selecciona un intervalo, **Ctrl+A** todas, y un clic en un hueco vacío la anula.
- **Información:** al dejar el ratón sobre cualquier botón aparece una descripción.
- **Si no caben:** los botones que no caben en la cabecera se ocultan, pero las acciones siguen disponibles por código.

| Botón | Método | Acción |
|---|---|---|
| Girar a la izquierda / derecha | `RotateSelectedPages(Clockwise)` | Gira 90° cada página seleccionada (`/Rotate`) |
| Eliminar (Supr) | `DeleteSelectedPages(Confirm)` | Pide confirmación. Nunca deja el documento sin páginas; las demás se deslizan a los huecos |
| Insertar página | `InsertBlankPage()` | Página en blanco, del tamaño de la última seleccionada, **justo después de la selección** |
| Insertar desde archivo | `InsertPagesFromFile([FileName])` | Elige un PDF e inserta todas sus páginas después de la selección |
| Reemplazar | `ReplaceSelectedPages([FileName])` | Sustituye cada página seleccionada por la página correspondiente del archivo elegido (1ª por 1ª, 2ª por 2ª…) |
| Extraer | `ExtractSelectedPages([FileName])` | Guarda la selección en un **PDF nuevo**. Propone `<nombre>_paginas.pdf` en la misma carpeta; el documento actual no cambia |
| Recortar | `CropSelectedPages([MarginPt])` | Recorta los márgenes en blanco: busca el contenido y ajusta la `CropBox` dejando 8 pt de margen |

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
| Ctrl + rueda (organizar) | Tamaño de las miniaturas |
| Ctrl + rueda | Zoom en el punto del cursor |
| Ctrl+A / Ctrl+C | Seleccionar todo / copiar |
| F3 / Mayús+F3 | Siguiente / anterior coincidencia |
