# Configuración de New Relic para Genweb6

Este proyecto usa instrumentación WSGI directa de New Relic Python Agent para Plone 6.
La instrumentación captura transacciones HTTP via el wrapper WSGI.

## Resumen de archivos configurados

### Local (desarrollo)
- **Buildout**: `genwebupc.cfg` - contiene eggs, environment-vars e initialization
- **Customizeme**: `customizeme.cfg` - contiene license_key (placeholder), app_name, monitor_mode
- **Arranque**: `bin/instance fg`

### Producción
- **Buildout**: `zope-only.cfg` + `deploy/zope-default.cfg` - contiene eggs, environment-vars e initialization
- **Customizeme**: `customizeme.cfg` (en las máquinas PRO) - contiene license_key real, app_name, monitor_mode
- **Arranque**: `bin/supervisorctl restart all` o `bin/zc1 start`, etc.
- **Template**: `customizeme.cfg.production.example` - ejemplo para PRO

## Qué está configurado

✅ New Relic Python Agent instalado
✅ Instrumentación WSGI de ZPublisher
✅ Variables de entorno configuradas
✅ Archivo newrelic.ini con 4 entornos (development, test, staging, production)
✅ Captura automática de:
  - Transacciones HTTP
  - Llamadas externas (requests, urllib3)
  - Métricas de servidor (CPU, memoria, threads, GC)

## Configuración Local

### 1. Añadir la License Key

Edita `customizeme.cfg` y añade la license key de New Relic en la sección `[newrelic]`:

```ini
[newrelic]
license_key = TU_LICENSE_KEY_AQUI
app_name = Genweb6 Local Dev
monitor_mode = false
environment = development
```

### 2. Ejecutar buildout

```bash
./bootstrap.sh
bin/buildout -c genwebupc.cfg
```

Esto instalará el paquete `newrelic` y configurará la instancia para inicializarlo.

### 3. Arrancar la instancia

La instrumentación se activa automáticamente, solo arranca normalmente:

```bash
bin/instance fg
```

Al arrancar deberías ver en el terminal:
```
New Relic: ZPublisher WSGI instrumented
```

Y en `/tmp/newrelic-python-agent.log` verás la inicialización del agente cuando llegue la primera petición HTTP.

### 4. Ver datos en New Relic

**En local** con `monitor_mode = true` (pero sin license key válida):
- El agente captura datos pero no puede enviarlos
- Puedes ver la actividad en `/tmp/newrelic-python-agent.log`
- Útil para verificar que funciona antes de desplegar

**En producción** con `monitor_mode = true` y license key válida:
- Los datos se envían automáticamente a https://one.newrelic.com/
- Verás dashboards, transacciones, métricas, errores, etc.

## Configuración en Producción

### 1. Configurar customizeme.cfg

En las máquinas de producción, edita el `customizeme.cfg` y añade/modifica la sección:

```ini
[newrelic]
# Obtén la license key real de New Relic para Genweb-PRO
# https://one.newrelic.com/ -> API Keys
license_key = TU_LICENSE_KEY_REAL_AQUI
app_name = Genweb6 Production
monitor_mode = true
environment = production
```

Puedes usar `customizeme.cfg.production.example` como referencia.

### 2. Ejecutar buildout

```bash
cd /path/to/genweb6.buildout
bin/buildout -c zope-only.cfg
```

Esto:
- Instala el paquete `newrelic`
- Configura las variables de entorno
- Añade la inicialización en cada Zope (zc1, zc2, zc3, zc4)

### 3. Reiniciar los Zopes

```bash
# Si usas supervisor:
bin/supervisorctl restart all

# O manualmente cada uno:
bin/zc1 stop
bin/zc1 start
# (repetir para zc2, zc3, zc4)
```

Al arrancar cada Zope deberías ver en los logs:
```
New Relic: ZPublisher WSGI instrumented
```

### Notas sobre la configuración de producción

**La configuración ya está lista** en:
- ✅ `zope-only.cfg`: Incluye `newrelic` en los eggs
- ✅ `deploy/zope-default.cfg`: Tiene el `initialization` y `environment-vars` configurados en `[zope-common]`

Todos los Zopes (zc1, zc2, zc3, zc4, zcdebug) heredan de `[zope-common]`, por lo que **todos tendrán New Relic automáticamente**.

**NO necesitas**:
- ❌ Crear wrappers manuales
- ❌ Usar `newrelic-admin run-program`
- ❌ Modificar supervisor/circus
- ❌ Scripts de arranque especiales

Solo ejecuta buildout y reinicia normalmente.

## Configuración del newrelic.ini

El archivo `newrelic.ini` contiene la configuración del agente. Los parámetros más importantes:

- `license_key`: La API key de New Relic (se puede dejar vacía, usa el del customizeme.cfg)
- `app_name`: Nombre de la aplicación en New Relic
- `monitor_mode`: true para enviar datos, false para solo logging
- `log_file`: Donde se guardan los logs del agente
- `log_level`: info, debug, warning, error
- `transaction_tracer.*`: Configuración del trazado de transacciones
- `error_collector.*`: Configuración de captura de errores

### Entornos

El archivo tiene 4 entornos configurados:
- `development`: Para desarrollo local (monitor_mode = false)
- `test`: Para testing (monitor_mode = false)
- `staging`: Para staging (monitor_mode = true)
- `production`: Para producción (monitor_mode = true)

El entorno se selecciona con el parámetro `environment` en `customizeme.cfg`.

## Verificación

Para verificar que New Relic está funcionando:

1. **Logs**: Revisa `/tmp/newrelic-python-agent.log`
2. **Arranque**: Al iniciar Zope verás mensajes de New Relic
3. **UI**: Si `monitor_mode = true`, verás datos en https://one.newrelic.com/

## Notas adicionales

### ¿Necesito usar bin/instance-newrelic?

**NO**. El script `bin/instance-newrelic` existe pero no es necesario porque la instrumentación ya está configurada en el `initialization` del buildout.

Simplemente usa `bin/instance fg` normalmente.

## Troubleshooting

### No aparecen datos en New Relic

- Verifica que `monitor_mode = true`
- Verifica que la `license_key` es correcta
- Revisa los logs en `/tmp/newrelic-python-agent.log`

### El agente no se inicializa

- Verifica que `newrelic` está en los eggs
- Ejecuta `bin/buildout` de nuevo
- Verifica que el archivo `newrelic.ini` existe

### Demasiado overhead

Ajusta estos parámetros en `newrelic.ini`:
- `transaction_tracer.transaction_threshold = 1.0` (aumenta el threshold)
- `transaction_tracer.record_sql = off` (desactiva SQL recording)

## Checklist para despliegue en PRO

- [ ] Obtener license key de New Relic para Genweb-PRO (https://one.newrelic.com/ -> API Keys)
  - Debe ser una key de tipo **"Ingest - License"**
- [ ] Editar `customizeme.cfg` en las máquinas PRO:
  - [ ] Añadir sección `[newrelic]`
  - [ ] Poner `license_key` real (sin comillas, sin espacios)
  - [ ] Configurar `app_name = Genweb-PRO`
  - [ ] Configurar `monitor_mode = true`
  - [ ] Configurar `environment = production`
- [ ] Verificar que `newrelic.ini` está en el directorio raíz del buildout
- [ ] Ejecutar `bin/buildout -c zope-only.cfg`
- [ ] Reiniciar todos los Zopes: `bin/supervisorctl restart all`
- [ ] Verificar en logs de Zope que aparece: "New Relic: ZPublisher WSGI instrumented"
- [ ] Verificar en `/tmp/newrelic-python-agent.log` que:
  - La license key se lee correctamente (no debe ser 'local-dev-placeholder')
  - El app_name es correcto
  - El agente conecta con New Relic
- [ ] Esperar 5-10 minutos
- [ ] Verificar en https://one.newrelic.com/ que aparece la aplicación
- [ ] Verificar que se reciben transacciones HTTP

## Importante: Variables de entorno

Las variables `NEW_RELIC_LICENSE_KEY` y `NEW_RELIC_APP_NAME` se configuran automáticamente desde el `customizeme.cfg` mediante las siguientes líneas en los archivos de buildout:

```ini
environment-vars =
  NEW_RELIC_LICENSE_KEY ${newrelic:license_key}
  NEW_RELIC_APP_NAME ${newrelic:app_name}
```

Esto hace que la license_key del `customizeme.cfg` sobrescriba el placeholder del `newrelic.ini`.

## Referencias

- [New Relic Python Agent Docs](https://docs.newrelic.com/docs/agents/python-agent/)
- [Agent Configuration](https://docs.newrelic.com/docs/agents/python-agent/configuration/python-agent-configuration)
- [WSGI Application API](https://docs.newrelic.com/docs/apm/agents/python-agent/python-agent-api/wsgiapplication-python-agent-api/)
