# Rúbrica de tono de la cobertura y la conversación (-2 a +2)

Escala de cinco puntos (`-2`, `-1`, `0`, `+1`, `+2`) más estado no evaluable (`null` / `n.d.`), aplicada pieza por pieza tanto a **notas de prensa** como a **publicaciones y comentarios en redes sociales** (X, Instagram, etc.). Se califica **siempre sobre la evidencia literal del texto**, nunca por la reputación previa del medio o la cuenta, y se registra en columnas separadas la **justificación analítica** (`justificacion_tono`) y la **cita textual** (`evidencia_textual`) para que cualquier tabla o ranking sea 100 % auditable.

---

## 1. Escala operativa de cinco puntos (-2 a +2) y `null`

| Valor | Etiqueta | Cuándo aplica en Prensa | Cuándo aplica en Redes Sociales (Posts y Comentarios) |
|-------|----------|-------------------------|-------------------------------------------------------|
| **-2** | **Severamente crítico / negativo** | Presenta al sujeto u organización como red criminal o fraude probado; usa lenguaje incriminatorio directo ("estafa", "vaciamiento", "engaño deliberado"); exige su salida/destitución o publica columnas/editoriales de descalificación directa sin matices. | Ataque directo, acusación de delito/corrupción/robo, insulto, burla explícita, exigencia de salida ("fuera", "renuncia") o responsabilización directa por el daño o crisis. |
| **-1** | **Levemente crítico** | Cobertura mayormente informativa pero con encuadre adverso, énfasis en sanciones, multas, reveses judiciales/regulatorios, cuestionamientos de gestión o titulares cargados ("el negocio de…", "nuevo revés", "bajo la lupa", "cómo colapsó"). | Crítica moderada a la gestión, escepticismo ante sus explicaciones, reproche sin insulto o difusión de notas adversas destacando el ángulo negativo. |
| **0** | **Informativo / neutro** | Crónica procesal, regulatoria, financiera o societaria descriptiva (formalización, cautelares, plazos, fallos, hechos esenciales) que reporta los antecedentes o consigna las versiones de las partes sin cargar la balanza editorial. Incluye sindicación de agencias. | Publicación meramente informativa de titulares/hechos sin adjetivación, o comentario descriptivo/consulta sin carga valorativa hacia el sujeto. |
| **+1** | **Levemente favorable** | Destaca de forma predominante matices, contexto favorable, fallos que constituyen un revés para el acusador, rebaja de cautelares o descargos extensos de la defensa sin omitir el marco del caso. | Reconocimiento parcial de argumentos de la defensa, matización frente a ataques de terceros o valoración positiva de un hito puntual de gestión. |
| **+2** | **Favorable** | Presenta al sujeto positivamente, publica un perfil laudatorio, respalda abiertamente su inocencia/gestión o reproduce de forma íntegra y sin contrapeso su vocería o defensa corporativa. | Defensa cerrada del sujeto, elogio explícito a su gestión o descalificación frontal a sus acusadores/reguladores en apoyo directo al actor. |
| **`null` (`n.d.`)** | **Tono no evaluable / No aplica** | Registro clasificado como `caso_general` (trata la causa sin mencionar ni atribuir rol al sujeto), `off_topic` (homónimo) o `pendiente`/`inaccesible` (solo titular truncado sin cuerpo suficiente para juzgar el encuadre). | Post que solo comparte un enlace o titular sin opinión propia ni desarrollo, comentario ambiguo/irónico sin contexto suficiente, o comentario dirigido a otro tema/interlocutor ajeno al sujeto. |

---

## 2. Reglas de oro para no distorsionar los indicadores

1. **Prohibido inflar el neutro (`0`) con piezas sin lectura:** Si de una nota solo se conoce el titular corto o el enlace está caído/bajo muro de pago y el titular por sí solo no define el encuadre, el tono debe quedar en `null` (`n.d.`) y `nivel_lectura = pendiente`. Los porcentajes de distribución de tono (`-2` a `+2`) y el **tono promedio** se calculan **exclusivamente sobre las menciones directas con tono evaluable**.
2. **Separar el Tono (`-2` a `+2`) de la Presencia de Defensa (`con_defensa`):**
   - El campo booleano `con_defensa` (`true`/`false` o `1`/`0`) registra si la pieza incluye citas, comunicados, recursos o argumentos del sujeto o sus abogados.
   - Una nota puede tener tono `0` (informativa) o `-1` (levemente crítica por el hecho reportado) y simultáneamente `con_defensa = true`. Medir ambos ejes por separado permite responder qué proporción de las notas críticas o neutras recoge la versión de la defensa.
3. **Atribuir no es sentenciar:**
   - *"La Fiscalía / el regulador sostiene que…"* dentro de una crónica descriptiva es **`0` (Informativo)**, o **`-1`** si el titular y el primer párrafo enfatizan exclusivamente el ángulo incriminatorio.
   - *"El esquema con que vació la empresa"* o *"la trama de abusos de…"* asumido por la voz del periodista o columnista es **`-2` (Severamente crítico)**.
4. **Doble lectura en comentarios de redes sociales:**
   - En los hilos de comentarios bajo posts de prensa o de opinólogos, distingue siempre si la crítica (`-2` / `-1`) se dirige **al sujeto analizado** o **al autor del post / medio**. Solo los comentarios referidos al sujeto o a su caso entran al denominador de postura de la conversación (`incluir_conteo = 1`).
5. **Métricas estándar a reportar siempre con esta escala:**
   - **Distribución en 5 niveles (`-2`, `-1`, `0`, `+1`, `+2`)** en cantidad y porcentaje sobre menciones directas evaluables.
   - **Bloques agregados:** `% Crítico` (`-2` + `-1`), `% Neutro` (`0`) y `% Favorable` (`+1` + `+2`).
   - **Tono promedio ponderado ($\bar{T}$):** media aritmética de los puntajes entre `-2,00` y `+2,00` para el corpus general, por mes/ventana temporal y por cada medio o plataforma. Permite comparar de inmediato el clima del corpus general (ej. `-0,23`) frente a medios de nicho o columnistas específicos (ej. `-1,08` o `-0,64`).
   - **Tasa de presencia de defensa:** porcentaje de menciones directas evaluables que incluyen descargos (`con_defensa = true`), desglosado entre notas críticas, neutras y favorables.
