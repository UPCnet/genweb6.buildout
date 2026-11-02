# Plan: actualizar genweb6 (Plone 6.0.6 Classic) y unificar versiones en las 24 máquinas

> **Entregable inmediato al aprobar este plan:** generar un **PDF** con este contenido (contexto, Plan A, Plan B, comparativa de horas) para compartirlo con el equipo y tomar la decisión. Se hará con la skill `anthropic-skills:pdf`: HTML maquetado → PDF, guardado en `genweb6.buildout/docs/plan-actualizacion-plone.pdf` o en la ruta que se indique. Ese PDF es lo único que se crea ahora. El resto del plan se ejecuta cuando el equipo decida.

## Contexto

- **Situación actual:**
  - Producción corre Plone **6.0.6**: más de 500 webs en 24 frontends gw6fe y 6 backends ZEO gw6be. Cada celda es 1 backend + 4 frontends (por ejemplo gw6be1 con gw6fe1-4).
  - Cada BE tiene 100 instancias. Cada FE tiene 25 mountpoints, y su zcdebug solo ve esos 25.
  - El advisory del 23-06-2026 trae fixes críticos: RCE por TALES (9.9), DoS por iCalendar y por el portlet RSS (9.1), y DoS por longitud de title/description/filename (6.5). Además está el advisory del 05-06-2026 sobre `plone.app.textfield` y `plone.restapi`.
  - Cada máquina tiene paquetes distintos según cuándo se instaló, porque muchos paquetes no están pineados en `versions.cfg`.
- **Objetivo:**
  1. Aplicar los parches de seguridad.
  2. Llegar a la versión más reciente posible sin reescribir los paquetes Genweb.
  3. Tener **un `versions.cfg` con todos los paquetes pineados**, generado en local a partir de la instancia que funciona. Así, con `$venv/bin/buildout -N -c zope-only.cfg`, todas las máquinas bajan y quedan en las mismas versiones.

**Datos que condicionan el plan:**
- Últimas versiones publicadas: **6.0.15** (marzo de 2025, la 6.0 está en modo solo seguridad) y **6.1.5** (25-06-2026, ya incluye los fixes). El soporte de seguridad de **6.0 y 6.1 termina el mismo día, 31-12-2027**. La 6.2 tiene soporte hasta Plone 7, previsto para mediados de 2028.
- Ya existe la rama **`plone_6_0_15`** en genweb6.buildout (septiembre de 2025, sin mergear), con el bump de `requirements.txt` (zc.buildout 4.1.4, setuptools 75.8.2, pip 25.0.1). **El Plan A se hace sobre esta rama.**
- La URL `dist.plone.org/release/6.0.6/versions.cfg` aparece en 3 sitios: `genwebupc.cfg:3`, `zope-only.cfg:19` y `zeo-only.cfg:12`.
- `collective.exportimport` y `contentimport` se quedan como están, en `src/` desde git también en producción. `collective.newrelic` queda fuera del plan: solo está en local, en evaluación.
- Ningún paquete Genweb limita la versión de Plone. El riesgo está en otras tres cosas:
  - 22 overrides jbot en `genweb6.core/templates/`.
  - Unos 45 monkeypatches.
  - El parche de TinyMCE (`patches.py:1301`).
- **Ansible:** se usa tal como está.
  - `genweb_actuacio` hace git pull + `bootstrap.sh` + reinicio de los FE.
  - `genweb_upgrade_plone` ejecuta `@@plone-upgrade` en cada FE sobre sus 25 mountpoints. Como lanza todos los FE del inventario, cubre los 100 mountpoints de cada celda sin cambios.
  - `genweb_actuacio` hace pull de `master`, así que **la rama elegida se mergea a `master` justo antes de la ventana**.
- **Backups y rollback:** antes de empezar, Ops hace snapshot de las máquinas (FE y BE). La actuación se hace en **sábado**; como último recurso queda el backup del viernes.

### ¿Hay que actualizar el ZEO?
**No es necesario para aplicar los parches.**
- El servidor ZEO (gw6be) solo ejecuta ZODB/ZEO y no carga código de Plone. Todos los fixes del advisory están en paquetes que cargan los clientes Zope (FE).
- En PRE hay que comprobar que los clientes con el ZODB/ZEO de la versión nueva conectan bien al servidor ZEO actual.
- Como `zeo-only.cfg` extiende el mismo `versions.cfg`, el BE se puede reconstruir más adelante, fuera de la ventana, para que quede unificado.

---

## Base común: generar el `versions.cfg` completo en local
1. En el buildout local (`genwebupc.cfg`), con la versión de Plone elegida, ejecutar `bin/buildout -N -c genwebupc.cfg` y comprobar que `./bin/instance fg` arranca y funciona con datos reales.
2. Sacar las versiones de todos los eggs que carga `bin/instance`: parsear las rutas `eggs/<nombre>-<versión>-py3.11.egg` del `sys.path` del script, o usar la salida de `show-picked-versions = true` durante ese buildout.
3. Volcarlas a `versions.cfg`, en este orden y con un comentario por bloque:
   1. Pins UPCnet.
   2. Hotfixes de seguridad (cada pin con su advisory).
   3. El resto de paquetes resueltos en local.
4. Excluir los develop eggs de `src/`: `collective.exportimport` y `contentimport`.
5. Limpieza:
   - Quitar el `prompt-toolkit` duplicado y el `genweb6.patents` duplicado.
   - En los 3 cfg, cambiar la URL `6.0.6` por la de la versión elegida.
   - Crear un tag en el repo del buildout para la versión desplegada.

## Despliegue (los dos planes)
PRE (copia real de un BE) → celda canaria un sábado → resto de celdas en los sábados siguientes. En cada celda:
1. Ops hace el snapshot de los 4 FE y del BE.
2. Merge de la rama a `master`.
3. `genweb_actuacio` en los 4 FE.
4. `playbook-check-upgrade-plone.yml`, que es un dry run.
5. `playbook-upgrade-plone.yml`.
6. `bulk_action` de Genweb, si hace falta.
7. Smoke tests.

---

## PLAN A (recomendado): 6.0.15 + hotfixes ahora, salto de versión más adelante

**Por qué:** la mínima superficie de cambio para 500 webs, los paquetes funcionan casi sin tocar y el parche se puede desplegar pronto.

1. **Rama `plone_6_0_15`** (la que ya existe), actualizada con `master`:
   - Usar los pins genweb6 de `master` (core 1.92, theme 1.58, upc 1.36…) en lugar de los antiguos (core 1.71…).
   - Quitar `branch=plone_6_0_15` de core y theme en `sources.cfg`, salvo que esas ramas de los paquetes tengan cambios que hagan falta.
   - Revisar los pins que trae la rama (`z3c.jbot`, `plone.app.mosaic`, `plone.app.multilingual`, `plone.app.z3cform`, `pyOpenSSL`, `cryptography`).
2. **URL de extends:** `dist.plone.org/release/6.0.15/versions.cfg`.
3. **Hotfixes en `versions.cfg`**, sacados del advisory:
   ```
   plone.app.contenttypes = 3.0.12
   plone.app.dexterity    = 3.2.3
   plone.app.event        = 5.2.4
   plone.app.portlets     = 5.0.8
   RestrictedPython       = 8.3
   plone.app.textfield    = 2.0.2
   plone.restapi          = 9.15.6
   icalendar              = <versión corregida de la línea que usa 6.0.15>
   ```
   Además, revisar los anuncios de plone.org/security entre marzo de 2025 y hoy por si falta algún otro hotfix.
4. **Pins UPCnet que hay que revalidar:**
   - `plone.session=4.0.3`: se pineó para arreglar el SSO; probar el SSO/CAS antes de quitarlo.
   - `plone.api=2.3.0`.
   - `Products.PloneLDAP=2.0.0a1`.
   - `eea.facetednavigation=16.0a1`.
   - `collective.easyform=4.1.4`.
   - `collective.z3cform.datagridfield=3.0.0`.
5. **Revisión de seguridad del código propio (crítico):**
   - Comparar con diff los overrides jbot y los monkeypatches que tocan código afectado por los fixes, porque pueden **reintroducir la vulnerabilidad**:
     - `plone.app.portlets.portlets.rss` y el `Renderer` del portlet de calendario.
     - `plone.app.event.browser.event_listing` y el patch de `EventListing`.
     - Los listados de `plone.app.contenttypes`, más los patches de `CollectionView` y `FolderView`.
     - `Products.CMFPlone.browser.syndication.templates.RSS`.
   - Aplicar los cambios del upstream fijado y publicar nuevas versiones de genweb6.* si hace falta.
6. **Generar el `versions.cfg` completo** (base común).
7. **Despliegue:**
   - Reinicio escalonado dentro de la celda (6.0.6→6.0.15 no cambia los pickles).
   - Lanzar el upgrade cuando los 4 FE ya estén en la versión nueva.
   - El ZEO no se toca.

**Fase 2 (más adelante):** como 6.0 y 6.1 tienen el mismo fin de soporte, el salto que de verdad alarga la vida útil es a **6.2.x Classic**. El trabajo de código es el mismo que en el Plan B.

---

## PLAN B: directo a 6.1.5

**Por qué:** una sola migración. Requiere trabajo de adaptación focalizado (no una reescritura) y retrasa el parche de seguridad varias semanas. **Si se elige, lo razonable es aplicar antes el Plan A, pasos 1 a 5, como parche urgente.**

1. **Ramas `plone61`** en el buildout y en cada paquete afectado. Publicar versiones mayores (p. ej. genweb6.core 2.x) para que la línea 1.x siga pudiendo recibir parches en 6.0.
2. **URL de extends:** `dist.plone.org/release/6.1.5/versions.cfg`. Ya trae contenttypes 4.0.10, dexterity 4.1.3, event 5.2.4, portlets 6.0.4 y RestrictedPython 8.3. Comprobar que trae `plone.app.textfield ≥ 3.0.2` y `plone.restapi ≥ 9.15.6`.
3. **Trabajo de código:**
   - **22 overrides jbot:** diff contra los originales de 6.1. Los de más riesgo son `search`, `login`, `listing*`, los macros y el widget de z3cform y `column`. En serveistic, las 5 plantillas de faceted.
   - **Parche de TinyMCE:** en 6.1 llega TinyMCE 7 y cambia el mockup, así que hay que reescribir esta parte.
   - **Unos 45 monkeypatches:** revisarlos uno a uno.
   - **Dependencias legacy:** quitar `plone.app.form` y `plone.formwidget.contenttree`, y migrar los portlets `zope.formlib` de serveistic.
   - **Imports:** `RelatedItemsFieldWidget` y `safe_unicode`, que pasa a `plone.base.utils.safe_text`.
   - **Subclases:** el portlet de login, los renderers de news y navigation y los viewlets de title y dublincore.
   - **Add-ons compatibles con 6.1:** mosaic, easyform, datagridfield, faceted, PloneLDAP, createdx, ftw.casauth y exportimport.
   - **Tests:** ejecutar `./bin/test -s <paquete>` en todos los paquetes.
4. **Python:** 3.11 sirve.
5. **Generar el `versions.cfg` completo** (base común) con 6.1.5.
6. **Migración de datos:** los upgrade steps de 6.0→6.1 en los 500 sites. Medir el tiempo por site en PRE con un BE completo.
7. **Despliegue:**
   - **Ventana completa por celda**: parar los 4 FE, actualizarlos, ejecutar el upgrade y arrancar.
   - No mezclar FE en 6.0 y en 6.1 contra el mismo ZEO.
   - El ZEO se puede actualizar después.
8. **Nota:** con el mismo esfuerzo se podría apuntar a **6.2.x**, que tiene más vida útil.

---

## Estimación de horas

Supuestos: 1 persona que conoce el código de Genweb. Las estimaciones son orientativas (con rango mínimo–máximo) y no incluyen el tiempo de Ops para los snapshots.

| Tarea | Plan A (6.0.15) | Plan B (6.1.5) |
|---|---|---|
| Rama del buildout, `requirements.txt`, extends | 4–6 h | 6–8 h |
| Buildout local, arrancar la instancia y generar el `versions.cfg` completo | 8–12 h | 10–14 h |
| Hotfixes y revalidar los pins UPCnet (SSO/CAS, LDAP, faceted, easyform) | 8–12 h | incluido en add-ons |
| Revisión de seguridad de overrides y patches afectados, y releases genweb6.* | 12–16 h | incluido abajo |
| Diff y adaptación de los 22 overrides jbot (+5 de serveistic) | – | 30–45 h |
| Reescribir el parche de TinyMCE 7 | – | 16–24 h |
| Revisar unos 45 monkeypatches | – | 30–40 h |
| Dependencias legacy (formlib, form, contenttree), imports, subclases | – | 20–30 h |
| Add-ons compatibles (mosaic, easyform, datagrid, faceted, LDAP, casauth…) | – | 24–40 h |
| Ramas y releases de unos 11 paquetes | 2–4 h | 8–12 h |
| Tests unitarios y corrección | 4–8 h | 24–40 h |
| Upgrade sobre una copia real de un BE y medición del tiempo | 6–8 h | 12–16 h |
| PRE: despliegue, smoke tests funcionales y correcciones | 12–16 h | 40–60 h |
| Producción: 6 celdas en sábados (≈2–3 h por celda en A, ≈4–6 h en B) | 12–18 h | 24–36 h |
| Documentación y coordinación con Ops y el equipo | 4–6 h | 8–12 h |
| **Total** | **≈ 70–105 h (2–3 semanas)** | **≈ 270–380 h (7–10 semanas)** |
| Hotfix previo urgente (Plan A, pasos 1–5) si se elige B | – | +35–50 h |

**Lectura:**
- El Plan A cierra las vulnerabilidades críticas en unas 2–3 semanas y unifica versiones.
- El Plan B cuesta unas 4 veces más y lleva al mismo fin de soporte (31-12-2027).
- Si se invierte en el salto, conviene que sea a 6.2, con un coste similar al del Plan B.

---

## Ficheros clave
- genweb6.buildout:
  - Rama `plone_6_0_15`.
  - `genwebupc.cfg`, `zope-only.cfg`, `zeo-only.cfg`.
  - `versions.cfg`, `sources.cfg`, `requirements.txt`.
- `src/genweb6.core`: `templates/` (jbot), `patches.py` y `patches.zcml`, `setup.py`.
- genweb.ansible: `genweb_actuacio/` y `genweb_upgrade_plone/`, sin cambios.

## Verificación
1. **Local:**
   - El buildout termina sin errores y `./bin/instance fg` funciona.
   - `./bin/test -s <paquete>` en cada paquete.
   - `@@plone-upgrade` sobre una copia real de Data.fs.
   - Un buildout limpio en otro directorio da exactamente los mismos eggs.
2. **Seguridad:** comprobar en `bin/zc1` las versiones de los paquetes del advisory. Probar a mano el portlet RSS con un feed enorme, un .ics malformado y un título de varios MB.
3. **PRE:**
   - Upgrade (dry run y real) en los 4 FE de una celda.
   - Smoke tests: login CAS/LDAP, búsqueda, listados, eventos, TinyMCE, easyform, faceted, multilingual y exportimport.
   - Conexión de los clientes nuevos al ZEO actual.
4. **Producción, en cada celda:**
   - `/ping` de cada zc y URL públicas de cada mountpoint a través de varnish y nginx.
   - Comparar el `sys.path` de `bin/zc1` en los 4 FE.
   - Revisar New Relic antes de pasar a la celda siguiente.
