---
name: analisis-cobertura-mediatica
description: "Analizar la cobertura de prensa y la conversación en redes sociales sobre una persona, empresa o caso para asesoría en PR y comunicación de crisis: extraer notas (Excel/Nexnews y búsqueda digital), recuperar cuerpos completos (HTML, JSON-LD, PDFs y agent-browser), clasificar relevancia y tono en escala auditable de -2 a +2 con evidencia textual y presencia de defensa, consolidar métricas cuantitativas y cualitativas, y redactar el informe Word (abriendo con Resumen Ejecutivo de Hallazgos y Conclusiones y notas al pie reales) más anexos Excel y de trazabilidad."
---

# Análisis de cobertura mediática y conversación en redes sociales

Convierte un conjunto de informes de monitoreo de prensa, búsquedas digitales y exportes de redes sociales en un análisis integral de asesoría estratégica: qué se dijo del sujeto, en qué medios y plataformas, con qué tono en escala de `-2` a `+2`, en qué proporción apareció su defensa, cómo evolucionó mes a mes y qué medios o cuentas lideraron la crítica o la cobertura informativa. El producto final es una base unificada auditable, un informe Word profesional respaldado por notas al pie reales y enlaces clicables, una línea de tiempo documentada y tres anexos de trazabilidad.

## 1. Reunir y validar la información

Identifica la carpeta de trabajo: suelen convivir informes mensuales de monitoreo de prensa (`.xlsx`), exportes de redes sociales (`.xlsx`) y un documento `.docx` que funciona como plantilla de diseño. Extrae con `openpyxl` (todas las hojas de cada libro, encabezados, número de filas y muestra de registros) y con `python-docx` (estilos, secciones, tablas y pie de página). Confirma el diccionario de datos real antes de avanzar; no diseñes sobre columnas supuestas ni descartes hojas secundarias sin inspeccionarlas.

Las noticias de prensa provienen de búsquedas web directas y de servicios de clipping de prensa: **Nexnews** (el servicio principal que usamos), **Simbiu** o **LitoralNews**. Los exportes de estos servicios suelen traer: `nombremedio`, `fecha`, `titulo`, `noticia` (soporte: Internet/Escrita/Radio/TV/Suplemento/Portada), `bajada`, `valorizacion`, `lectoria`, `tiraje`, `urlfuente` y la URL del propio servicio de clipping (`url_Nexnews` en Nexnews; homólogos en Simbiu y LitoralNews). Registra el calendario exacto cubierto por cada archivo (fecha mínima, máxima y días con presencia) para advertir cuando los exportes representan ventanas parciales del mes y no meses corridos completos.

Un riesgo constante es el homónimo: una búsqueda por nombre o apellido arrastra personas distintas (fútbol, política, gremios, alcance internacional). Verifica cada pieza con la mención literal y su contexto antes de contarla.

## 2. Recuperar los cuerpos completos (servicios de clipping, `urlfuente` y `agent-browser`)

1. **Intento primario (`url_Nexnews`) y fallback por `urlfuente`:**
   - Si el portal de Nexnews (`portal.nexnews.cl`) responde, extrae el cuerpo completo y la URL original desde el ancla `Link` según [scraping con agent-browser](references/scraping-agent-browser.md).
   - Si `portal.nexnews.cl` falla por certificado o protocolo obsoleto (`SEC_ERROR_UNKNOWN_ISSUER` / `Legacy TLS`), **no des las notas por perdidas**: documenta el fallo técnico con su código de error y ejecuta extracción masiva sobre la columna `urlfuente` combinando:
     - Descarga HTML directa + extracción de párrafos de artículo y `articleBody` en bloques `application/ld+json` (JSON-LD) o hidratación `__NEXT_DATA__` (Arc Publishing / Next.js).
     - Extracción de texto desde recortes PDF de prensa escrita o suplementos alojados en `link1.simbiu.es` mediante `pypdf` (filtrando por título/bajada cuando el PDF contiene una página completa de diario con varias noticias).
     - Sesiones de `agent-browser` para sitios con renderizado dinámico o contenido cargado por cliente (scrapeo de la página renderizada, nunca consultas a servicios internos).
2. **Estado de acceso y lectura por registro (`nivel_lectura`):**
   - `texto_recuperado`: cuerpo completo rescatado.
   - `parcial`: título y bajada/extracto suficientes para verificar relevancia y tono.
   - `pendiente`: enlace sin `urlfuente` pública o bajo muro de pago cuyo titular aislado no basta para determinar relevancia o tono.
   - `inaccesible`: HTTP 404/410/500 o timeout documentado tras reintentos.

## 3. Clasificar pieza por pieza (Escala de tono `-2` a `+2`)

Asigna a cada nota de prensa, post de redes o comentario tres dimensiones con **evidencia textual literal** (`evidencia_textual`) y nunca por impresión general ni por reputación del medio:

- **Relevancia**:
  - `directo`: menciona al sujeto y le atribuye rol, decisiones, responsabilidad o descargos.
  - `caso_general`: trata la causa, empresa o conflicto relacionado sin mencionar directamente al sujeto (útil para contexto y línea de tiempo, pero separado del denominador de tono personal).
  - `off_topic`: homónimo o nota ajena al caso (se conserva en la base con `incluir_conteo = 0` para trazabilidad).
  - `pendiente`: sin texto suficiente para confirmar si se trata del sujeto o de un homónimo.
- **Tono (`-2` a `+2` o `null` / `n.d.`)**: aplica rigurosamente la [rúbrica de tono de -2 a +2](references/rubrica-tono.md) tanto a la prensa como a la conversación en redes sociales:
  - **`-2` (Severamente crítico / negativo)**: imputación directa de delito/fraude, lenguaje incriminatorio ("estafa", "vaciamiento", "engaño deliberado"), exigencia de salida, burla explícita o columna/post de descalificación directa sin matices.
  - **`-1` (Levemente crítico)**: cobertura o post con encuadre adverso, énfasis en sanciones, multas, reveses judiciales/regulatorios, cuestionamientos de gestión o titulares cargados ("bajo la lupa", "nuevo revés", "cómo colapsó").
  - **`0` (Informativo / neutro)**: crónica procesal, regulatoria, financiera o societaria descriptiva que reporta hechos o consigna las versiones de las partes sin cargar la balanza editorial.
  - **`+1` (Levemente favorable)**: destaca de forma predominante matices, descargos de la defensa, fallos favorables (ej. rebaja de cautelares) o respaldos de terceros sin omitir el contexto del caso.
  - **`+2` (Favorable)**: perfil laudatorio, defensa cerrada o reproducción íntegra y sin contrapeso de la vocería del propio actor.
  - **`null` (`n.d.`)**: obligatorio cuando la pieza es `caso_general`, `off_topic`, `pendiente`/`inaccesible` o cuando un post en redes solo comparte un titular/link sin desarrollo suficiente. **Nunca asignes `0` por defecto a piezas sin lectura**, para no inflar artificialmente la neutralidad.
- **Presencia de defensa (`con_defensa: true/false`)**:
  - Registra en un campo booleano independiente si la pieza incluye argumentos, citas, recursos o descargos del sujeto o sus abogados. Permite medir qué proporción de las notas críticas (`-2`/`-1`) o informativas (`0`) recoge la versión de la defensa.
- **Ejes temáticos**: etiquetas cortas y consistentes (`sanciones_cmf`, `formalizacion_fiscalia`, `gobernanza_corporativa`, `defensa_descargos`, etc.).

## 4. Consolidar lo cuantitativo y los focos mediáticos

Calcula y reconcilia entre la base JSON, el informe Word y los anexos Excel:
- **Embudo documental**: registros brutos → excluidos `off_topic` y `pendiente` → duplicados técnicos → publicaciones pertinentes únicas (`directo` + `caso_general`) → menciones directas con tono evaluable.
- **Métricas de la escala `-2` a `+2`**:
  - Cantidad y porcentaje en cada nivel (`-2`, `-1`, `0`, `+1`, `+2`) calculados exclusivamente sobre menciones directas con tono evaluable.
  - Bloques agregados: **% Crítico (`-2` + `-1`)**, **% Neutro (`0`)** y **% Favorable (`+1` + `+2`)**.
  - **Tono promedio ponderado ($\bar{T}$)** general, por mes/ventana y por medio o plataforma, más **% con presencia de defensa**.
- **Deduplicación y grupos de réplica**: agrupa titulares idénticos/sindicados (ej. agencias replicadas en `Msn.com`, portales y diarios regionales) para distinguir el volumen bruto del número de historias únicas.
- **Focos mediáticos de opinión o nicho**: cuando el usuario pida analizar medios específicos (ej. portales de nicho, programas digitales o columnistas), extrae su archivo histórico completo mediante sus etiquetas (`/tag/...`) y buscadores internos, aplica la misma escala `-2` a `+2` y compara su **tono promedio** y su **% crítico** contra el promedio de la prensa general.

## 5. Conversación en redes sociales (X e Instagram)

- **Publicaciones (Posts)**: depura homónimos igual que en prensa, consolida métricas verificables (likes, comentarios, republicaciones, vistas, advirtiendo cuando las vistas vienen redondeadas o corresponden a capturas de distinta fecha) y clasifica el tono de `-2` a `+2` (dejando `n.d.` en titulares sin desarrollo).
- **Comentarios de usuarios**: limpia el ruido de interfaz y clasifica cada comentario pertinente según su postura hacia el sujeto o caso en la misma lógica gradual (`-2` ataque/insulto/exigencia de salida, `-1` crítica/reproche, `0` neutro/consulta, `+1`/`+2` apoyo/defensa, y `ambiguo/ironía` cuando no hay contexto suficiente), separando siempre los comentarios que atacan al medio o autor del post (`incluir_conteo = 0`).
- En Instagram, documenta si el actor o la institución **limita o cierra los comentarios** durante la crisis como hallazgo comunicacional relevante.

## 6. Ampliación digital e hitos históricos

No te limites a los Excel aportados: realiza búsqueda web en todo el rango histórico del caso (desde el primer hito público o regulatorio hasta la fecha de corte) para cubrir los vacíos entre ventanas de monitoreo. Concilia el solapamiento marcando la procedencia de cada registro (`excel_nexnews`, `busqueda_digital` o `ambas`) para no duplicar notas.

## 7. Estructura del Informe Word y Entregables

1. **Orden del Informe Word (`.docx`)**:
   - **Sección 1 — Resumen Ejecutivo: Hallazgos y Conclusiones**: el informe debe abrir siempre con los hallazgos centrales y conclusiones estratégicas (balance de cobertura, distribución `-2` a `+2`, tono promedio general vs. medios críticos, asimetría entre prensa y redes sociales, hitos que disparan el riesgo y conclusiones ejecutivas).
   - **Sección 2 — Metodología, Cobertura Real y Trazabilidad**: embudo de datos, alcance de las ventanas y rúbrica `-2` a `+2`.
   - **Sección 3 — Análisis Cuantitativo de Prensa**: evolución temporal, distribución por tono `-2` a `+2`, presencia de defensa y ranking de medios.
   - **Sección 4 — Análisis Cualitativo y Ejes Narrativos**: encuadres dominantes y citas textuales.
   - **Sección 5 — Focos Mediáticos Especiales** (cuando aplique): análisis profundo de medios o columnistas específicos solicitados por el usuario.
   - **Sección 6 — Conversación y Reacciones en Redes Sociales (X e Instagram)**: métricas, tono de posts (`-2` a `+2`) y postura de los comentarios.
   - **Sección 7 — Línea de Tiempo Documentada del Caso**: hitos cronológicos verificados desde el origen del caso hasta el cierre.
   - **Sección 8 — Límites Metodológicos y Riesgos Residuales**.
2. **Citas y notas al pie reales**:
   - Toda afirmación factual, cifra o cita debe llevar enlace clicable a la fuente primaria y **notas al pie nativas de Word** (`w:footnoteReference` en `word/footnotes.xml`), nunca superíndices planos sin vínculo.
3. **Entregables obligatorios (conservando versiones previas si existen)**:
   - Base de datos JSON unificada (`db/*.json`) en UTF-8 con textos completos sin truncar, relevancia, tono (`-2` a `+2` o `null`), `con_defensa`, evidencia textual y procedencia.
   - Informe Word principal (`.docx`) reutilizando la identidad visual de la plantilla.
   - Línea de tiempo en Word (`.docx`) y Markdown (`.md`).
   - **Anexo A — Prensa (`.xlsx`)**: hojas de `Resumen` (con fórmulas dinámicas para la escala `-2` a `+2`), `Prensa` (inventario completo), `Cobertura`, `Complemento` y `Digital`.
   - **Anexo B — Redes Sociales (`.xlsx`)**: hojas de `Resumen`, `Publicaciones`, `Comentarios` y `Revision`.
   - **Anexo C — Trazabilidad Documental (`.docx`)**.
