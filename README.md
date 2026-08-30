# calvo-post1-u2
Post-contenido — Exportación de reportes académicos con patrones creacionales justificados

--
## Decisiones de diseño

### Decision 1.
**Patrón elegido :** Abstract Factory  
El sistema no crea un único tipo de objeto, crea dos productos que tiene que coincidir entre si: el cuerpo del reporte y el encabezado/pie de pagina. El enunciado explica que no es valido combinar, un cuerpo de Excel con un encabezado de PDF. Eso ya descarta pensar el problema como "un solo producto que varía" y lo ubica como una familia de productos que debe mantenerse consistente.

El otro patron candidato que se descarto fue Factory Method. El patrón Factory Method funciona bien cuando hay un solo producto variando (por ejemplo, si solo existiera un ReportDocument ya combinado), pero acá el riesgo no es instanciar la clase equivocada, sino terminar mezclando piezas de formatos distintos en una misma exportación. Con Abstract Factory, la interfaz ReportFormatFactory obliga a que createBody() y createHeaderFooter() siempre entreguen productos del mismo formato, así que esa coherencia queda garantizada por el propio diseño y no depende de que el código cliente se acuerde de hacerlo bien. Además, cuando llegue el formato CSV que ya está planeado, lo que hay que agregar es una familia completa (CsvReportBody + CsvHeaderFooter + su fábrica), no una sola clase suelta, lo cual también encaja mejor con Abstract Factory que con Factory Method.

### Desicion 2.
**Opción elegida:** Registro dinámico con Map<String, Supplier<ReportFormatFactory>>
La opción más directa para resolver la fábrica según el formato ("pdf", "excel", "html") habría sido un switch o una cadena de if/else. Se descarto porque cada vez que se agregue un formato nuevo —como el CSV que el equipo de datos ya tiene en mente— habría que volver a tocar ese mismo método, y eso viola OCP: un componente que ya funciona no debería modificarse solo para soportar un caso nuevo.

En vez de eso, ReportFactoryRegistry guarda en un Map la relación entre cada identificador de formato y un Supplier<ReportFormatFactory> que sabe cómo construirlo. Para resolver una fábrica, el registro simplemente busca en el mapa e invoca el Supplier correspondiente, sin ningún condicional que dependa del string. Esto permite registrar un formato nuevo con una sola línea (REGISTRY.register("csv", CsvReportFactory::new)) sin tocar el método resolve().