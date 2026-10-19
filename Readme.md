# Reporte automático de commits con n8n y MCP

Workflow de n8n que revisa los commits nuevos de un repositorio de GitHub, genera un reporte con un modelo de IA (Gemini) y lo envía por Telegram.

## ¿Qué es n8n?

n8n es una herramienta para automatizar tareas conectando aplicaciones entre sí. Funciona con bloques llamados **nodos**: cada nodo realiza una acción (leer datos, consultar un servicio, enviar un mensaje) y las líneas que los unen indican el orden en que se ejecutan. Un conjunto de nodos conectados es un **workflow**.

Este proyecto es un workflow ya armado, que se importa desde el archivo `ProyectoFinal_MCP.json`.

[1](1.png)

## Qué hace este workflow

```
Schedule Trigger → Get row(s) → AI Agent (Gemini + GitHub MCP) → Hay cambios? → Telegram → Upsert row(s)
```

1. Se ejecuta cada cierto tiempo (cada hora por defecto).
2. Lee de una tabla la fecha de la última revisión.
3. El agente de IA consulta en GitHub los commits posteriores a esa fecha y redacta el reporte.
4. Si hay commits nuevos, envía el reporte por Telegram y guarda la fecha de esta revisión. Si no los hay, no envía nada.

[2][2.png]

## Requisitos

- **n8n.** Puede usarse de dos formas: instalado en la propia computadora (gratuito, ver la sección siguiente) o en n8n Cloud, creando una cuenta en n8n.io, sin instalar nada.
- **API key de Gemini.** Se crea gratis en [Google AI Studio](https://aistudio.google.com) con la opción *Get API key*.
- **Token de GitHub de solo lectura.** En GitHub: Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token. En *Repository access* elegir solo el repositorio a revisar y en permisos asignar **Contents: Read-only**. Copiar el token al crearlo, porque no se vuelve a mostrar.
- **Bot de Telegram.** Hablar con **@BotFather** (cuenta oficial verificada de Telegram), enviar `/newbot` y copiar el token. Después escribirle cualquier mensaje al bot nuevo y abrir en el navegador `https://api.telegram.org/botTU_TOKEN/getUpdates` (reemplazando `TU_TOKEN`) para obtener el **chat id**, que es el número que aparece junto a `"chat":{"id":`.

## Instalar n8n en la computadora (opcional)

Si se usa n8n Cloud, omitir esta sección.

**Opción A: con Docker.** Requiere tener [Docker Desktop](https://www.docker.com/products/docker-desktop/) instalado y abierto. En una terminal ejecutar:

```
docker volume create n8n_data
docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

**Opción B: con Node.js.** Requiere tener [Node.js](https://nodejs.org) instalado. En una terminal ejecutar:

```
npx n8n
```

En ambos casos, esperar a que la terminal indique que n8n está listo, abrir `http://localhost:5678` en el navegador y crear la cuenta de usuario que se solicita la primera vez (es local, solo para esa instalación).

Importante: con una instalación local, el workflow solo se ejecuta mientras n8n esté encendido. Si se cierra la terminal o se apaga la computadora, las ejecuciones programadas se detienen. Para detener n8n, presionar `Ctrl + C` en la terminal.

## Configuración del workflow

1. **Importar.** En n8n, crear un workflow nuevo, abrir el menú `⋯` (arriba a la derecha) y elegir **Import from file**. Seleccionar `ProyectoFinal_MCP.json`.
2. **Crear la tabla de estado.** En la sección **Data tables** crear una tabla llamada `github_status` con dos columnas de tipo *string*: `repo` y `lastCommitDate`. Luego volver a seleccionarla en los nodos **Get row(s)** y **Upsert row(s)**.
3. **Crear las credenciales.** Abrir cada nodo y crear la credencial que pide:
   - **Google Gemini Chat Model:** la API key de Gemini.
   - **GitHub MCP:** credencial de tipo *Bearer* con el token de GitHub. Pegar solo el token, sin escribir la palabra "Bearer".
   - **Send a text message:** el token del bot de Telegram.
4. **Reemplazar los valores de ejemplo:**

   | Nodo | Qué cambiar |
   |---|---|
   | Get row(s), filtro `repo` | `USER/REPO` por `usuario/repositorio` |
   | AI Agent, prompt | `USER/REPO` por el mismo valor, y la rama `main` si se usa otra |
   | Upsert row(s), filtro y columna `repo` | `USER/REPO` por el mismo valor |
   | Send a text message, Chat ID | `YOUR_CHAT_ID` por el chat id obtenido |

   `usuario/repositorio` es la parte de la URL del repo que sigue a `github.com/`. Por ejemplo, en `https://github.com/ejemplo/mi-proyecto` el valor es `ejemplo/mi-proyecto`. Debe escribirse igual en los tres nodos, respetando mayúsculas y minúsculas.
5. **Probar.** Hacer un commit reciente en la rama a revisar y pulsar **Execute workflow**. La primera vez no hay fecha guardada, por lo que se revisa la última hora. El reporte debe llegar a Telegram y debe aparecer una fila en la tabla `github_status`.
6. **Activar.** Encender el interruptor **Active** (arriba a la derecha). El intervalo de revisión se cambia en el nodo **Schedule Trigger**.

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| Aparece "Node was not executed" | No había commits nuevos y el flujo se detuvo en el nodo IF. Es el comportamiento esperado. |
| Error 401 o 404 en GitHub MCP | Token mal pegado, vencido o sin acceso al repositorio. |
| Error 429 | Se alcanzó el límite gratuito de Gemini. Esperar o aumentar el intervalo. |
| Siempre se revisa solo la última hora | El valor de `repo` no coincide entre los nodos, o la tabla no está seleccionada. |
| No aparecen las herramientas MCP | En GitHub MCP, revisar que `list_commits` y `get_commit` estén en *Tools to Include*. |

## Notas

- En el nivel gratuito de Gemini, Google puede usar los datos enviados para mejorar sus productos. No usarlo con repositorios privados o con código sensible.
- Los tokens son contraseñas: no subirlos a GitHub ni compartirlos. Si alguno se expone, revocarlo y crear uno nuevo.