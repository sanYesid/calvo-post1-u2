# Post-contenido — Unidad 2: Patrones Creacionales

## Descripción
Repositorio del post-contenido de la Unidad 2 de Patrones de Diseño
de Software — Sexto Semestre. Un único proyecto Maven
(exportador-reportes/) que resuelve la exportación de reportes
académicos en múltiples formatos (Parte 1) y se extiende con
configuración compleja y evaluación de Singleton (Parte 2).

## Cómo ejecutar
cd exportador-reportes    
mvn compile     
mvn exec:java -Dexec.mainClass="com.patrones.u2.Main"     

---

## Decisiones de diseño

### Decisión 1.
**Patrón elegido :** Abstract Factory     
El sistema no crea un único tipo de objeto, crea dos productos que tiene que coincidir entre si: el cuerpo del reporte y el encabezado/pie de pagina. El enunciado explica que no es valido combinar, un cuerpo de Excel con un encabezado de PDF. Eso ya descarta pensar el problema como "un solo producto que varía" y lo ubica como una familia de productos que debe mantenerse consistente.

El otro patron candidato que se descarto fue Factory Method. El patrón Factory Method funciona bien cuando hay un solo producto variando, pero acá el riesgo no es instanciar la clase equivocada, sino terminar mezclando piezas de formatos distintos en una misma exportación. Con Abstract Factory, la interfaz ReportFormatFactory obliga a que createBody() y createHeaderFooter() siempre entreguen productos del mismo formato, así que esa coherencia queda garantizada por el propio diseño y no depende de que el código cliente se acuerde de hacerlo bien. Además, cuando llegue el formato CSV que ya está planeado, lo que hay que agregar es una familia completa (CsvReportBody + CsvHeaderFooter + su fábrica), no una sola clase suelta, lo cual también encaja mejor con Abstract Factory que con Factory Method.

### Decisión 2.
**Opción elegida:** Registro dinámico con Map<String, Supplier<ReportFormatFactory>>    
La opción más directa para resolver la fábrica según el formato ("pdf", "excel", "html") habría sido un switch o una cadena de if/else. Se descarto porque cada vez que se agregue un formato nuevo —como el CSV que el equipo de datos ya tiene en mente— habría que volver a tocar ese mismo método, y eso viola OCP: un componente que ya funciona no debería modificarse solo para soportar un caso nuevo.

En vez de eso, ReportFactoryRegistry guarda en un Map la relación entre cada identificador de formato y un Supplier<ReportFormatFactory> que sabe cómo construirlo. Para resolver una fábrica, el registro simplemente busca en el mapa e invoca el Supplier correspondiente, sin ningún condicional que dependa del string. Esto permite registrar un formato nuevo con una sola línea (REGISTRY.register("csv", CsvReportFactory::new)) sin tocar el método resolve()

### Decisión 3.    
**Opción elegida:** Builder.     
ExportConfig tiene 1 parámetro obligatorio y 8 opcionales, varios del mismo tipo (String, boolean). Con un constructor de 9 parámetros, invertir dos strings seguidos es un bug que el compilador no detecta. Los constructores sobrecargados tampoco funcionan: con 8 opcionales las combinaciones razonables se disparan, y hubiéramos terminado con un montón de constructores casi idénticos solo para casos como "con watermark pero sin compresión".

Los setters sueltos parecían la salida fácil, pero dejan el objeto a medio configurar sin un punto donde validar que la combinación tenga sentido. Y acá eso importa: pedir compress=true sin outputPath no debería ser válido, y con setters esa regla queda regada por el código o depende de que alguien se acuerde de chequearla.

El Builder centraliza eso en build(). Si compress es true y outputPath es null, lanza IllegalStateException antes de que el objeto exista — se ve en el checkpoint: new ExportConfig.Builder("pdf").compress(true).build() falla con un mensaje claro. Y cada método (pageSize(), orientation(), watermarkText()) deja claro qué se está configurando, sin tener que memorizar el orden de nada.

### Decisión 4.
**Conclusión:** No.      
**Identidad de objeto:** en ningún punto del proyecto se necesita pasar el registro como objeto. Nada lo inyecta por constructor ni lo reemplaza con un mock en pruebas, y no implementa ninguna interfaz que dependa de una instancia. ReportExportService lo usa siempre así: ReportFactoryRegistry.resolve(...), como método estático directo.

**Inicialización costosa:** el bloque static solo llena un Map con tres entradas (pdf, excel, html). No hay lectura de archivos ni conexiones que justifiquen una inicialización perezosa; el classloading ya lo resuelve sin costo real.

**Fuente única de verdad:** REGISTRY ya es static final, así que hay un solo mapa compartido en toda la JVM. Agregar la maquinaria de Singleton (constructor con guardas, getInstance(), sincronización) no sumaría nada que el static no esté garantizando ya.

**Escenarios futuros razonables:** no hay ningún caso concreto en el proyecto donde se necesite más de un registro. Si esto llegara a ser una plataforma multi-institución con un conjunto de formatos distinto por institución, Singleton sería justo lo que habría que evitar, porque ahí sí haría falta más de una instancia.

---
## Herramientas utilizadas
- Java 17, Apache Maven, VS Code, Git, GitHub

---

## Conclusiones
Elegir un patrón resultó ser más una cuestión de analizar el problema con calma que de recordar una definición de manual. En la Parte 1, la decisión entre Factory Method y Abstract Factory terminó dependiendo de algo muy puntual: que el cuerpo y el encabezado del reporte no podían quedar en formatos distintos entre sí. Construir el registro con Map y Supplier en lugar de un switch tampoco fue solo por seguir la pista del enunciado, sino porque evita tener que tocar código que ya funciona cada vez que se agregue un formato nuevo. Con el Builder de la Parte 2 quedó claro que no basta con que el objeto se construya: lo que importa es que la validación ocurra antes de que exista, no después. La evaluación del Singleton fue quizás el ejercicio más útil, porque exigió dejar de lado la costumbre de pensar que todo registro central debe ser Singleton y preguntarse, con criterios objetivos, si eso realmente resolvía algo en este caso.