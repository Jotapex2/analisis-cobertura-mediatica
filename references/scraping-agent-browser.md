# Scraping con agent-browser

Patrones probados para extraer notas de Nexnews y perfiles/post de Instagram con `agent-browser`, sin que el proceso se cuelgue.

## Antes de empezar

Verifica con `agent-browser --version` y lee la guía de la versión instalada con `agent-browser skills get core --full`. Usa siempre una sesión nombrada (`--session <nombre>`) para no pisar el navegador de otra tarea.

## Deadlock de pipes en Windows (crítico)

No captures la salida de `agent-browser open` con `subprocess` por pipe: el daemon lanzado hereda el handle y el proceso nunca devuelve EOF, con lo que `subprocess.run(capture_output=True)` se cuelga. En su lugar, **redirige a archivo**:

```python
def run_t(cmd, timeout):
    p = subprocess.Popen(cmd, shell=True, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
    try:
        p.wait(timeout=timeout); return True
    except subprocess.TimeoutExpired:
        subprocess.run("taskkill /F /T /PID %d > NUL 2>&1" % p.pid, shell=True); return False
```

- `open` y `close` redirigen a `NUL`.
- `get text body` y `eval` redirigen a un archivo de salida propio y luego se lee ese archivo.
- Usa `taskkill /T` para matar el árbol (el hijo `agent-browser` sobrevive si solo matas la shell).
- Escribe el progreso a un archivo y **reanuda** saltando los archivos ya guardados (tamaño > umbral), para no repetir todo ante un cuelgue.

## Nexnews: texto, URL original y fallback ante fallo TLS

Por cada `url_Nexnews`, abre el portal, extrae el cuerpo y recupera la URL de origen:

```bash
agent-browser --session notas open "<url_nexnews>" > NUL 2>&1
agent-browser --session notas get text body > "nota_%03d.txt" 2>&1
agent-browser --session notas eval "JSON.stringify(Array.from(document.querySelectorAll('a')).map(function(a){return {t:(a.textContent||'').trim(),h:a.href};}).filter(function(x){return x.h && x.h.indexOf('nexnews')===-1 && /^https?:/.test(x.h);}).slice(0,10))"
```

El ancla cuyo texto es `Link` es la URL de origen. Úsala para la cita; si no existe, cae a `urlfuente` o al propio `url_Nexnews`. La página de Nexnews incluye metadatos (País, Región, Fecha, Tipo, Sección) y el cuerpo completo, lo que basta para clasificar sin abrir el medio original.

**Importante sobre salida de `eval`:** las versiones recientes de `agent-browser` anteponen líneas informativas como `[agent-browser] Connecting to Existing Chrome Instance...` en `stdout`. Antes de hacer `json.loads()` sobre el archivo capturado, filtra cualquier línea que comience con `[agent-browser]`.

**Fallback cuando `portal.nexnews.cl` falla por TLS (`SEC_ERROR_UNKNOWN_ISSUER` / `Legacy TLS`):**
- Documenta el error técnico del portal y extrae en paralelo desde la columna `urlfuente`:
  1. Descarga HTML directa + parseo de `<article>`, párrafos `<p>` y bloques `application/ld+json` (`articleBody`) o `__NEXT_DATA__` (Arc Publishing / Next.js).
  2. Si `urlfuente` apunta a recortes PDF de prensa escrita en `link1.simbiu.es`, descarga el PDF y extrae el texto con `pypdf` (`PdfReader`), verificando que el título/bajada corresponda a la nota cuando el PDF contenga una página completa de diario con varias notas.
  3. Para notas sindicadas (ESPN, MSN, agencias) con contenido cargado por cliente, abre la página con `agent-browser`, espera el render y extrae el texto del DOM; si el cuerpo nunca aparece, registra la nota como `parcial` con el titular y la bajada disponibles.

## Instagram con la sesión del usuario

Para perfiles que exigen login, usa el navegador del usuario cuando esté autorizado:

```bash
agent-browser --profile Default --executable-path "C:\...\brave.exe" --headed open "https://www.instagram.com/accounts/login/"
```

Si el navegador está abierto, el perfil queda bloqueado y agent-browser abre un perfil vacío (0 cookies): pide al usuario que lo cierre, o que inicie sesión en la ventana abierta. Confirma la sesión comprobando la cookie `ds_user_id`.

Búsquedas por nombre en el buscador interno de Instagram (requiere sesión, se ejecuta en el contexto de la página):

```javascript
fetch('/web/search/topsearch/?context=blended&query='+encodeURIComponent(q)).then(r=>r.json())
```

Devuelve `users` y `hashtags`; extrae handles y, para las cuentas del caso, abre el perfil y recolecta los posteos. El grid carga con retardo: **haz el `eval` de links después de esperar (no intercales `get text body`)**, o el selector devolverá vacío:

```javascript
JSON.stringify(Array.from(document.querySelectorAll('a')).map(a=>a.getAttribute('href')).filter(h=>h&&(h.indexOf('/p/')>=0||h.indexOf('/reel/')>=0)).filter((v,i,arr)=>arr.indexOf(v)===i).slice(0,24))
```

Cada post abierto con `get text body` entrega caption, likes y comentarios visibles; cuando el actor limita los comentarios, aparece "Se han limitado los comentarios" y no hay conversación que scrapear.

## Límites

El eval secuencial de muchos `fetch` supera el timeout por defecto (25 s): divide en lotes o sube `AGENT_BROWSER_DEFAULT_TIMEOUT`. Registra lo que no se pudo abrir y no sustituyas por inferencia.
