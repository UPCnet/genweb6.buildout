# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Qué es este repo

Buildout clásico de **Plone 6.0.6** (sin Volto ni frontend separado) para proyectos basados en Genweb de UPCnet. Solo existe un build: Genweb UPC (`genwebupc.cfg`). El código de los paquetes **no vive aquí**: se clona en `src/` (gitignored) con mr.developer.

## Comandos

```bash
# Setup inicial (venv con Python 3.11.x)
cp customizeme.cfg.in customizeme.cfg      # gitignored; contiene secretos (LDAP, New Relic, API keys): no mostrar ni commitear
cp bootstrap.sh.in bootstrap.sh
./bootstrap.sh <venv_dir>                  # pip install -r requirements.txt + buildout -N -c genwebupc.cfg (dos veces)

# Re-ejecutar buildout tras cambiar .cfg
<venv_dir>/bin/buildout -N -c genwebupc.cfg

# Arrancar (puerto 11001, admin:admin)
./bin/instance fg

# Tests (zope.testrunner generado por [test] en genwebupc.cfg)
./bin/test -s genweb6.core                 # un paquete
./bin/test -s genweb6.core -t test_nombre  # un test concreto

# Traducciones
./bin/i18ndude ...   # y ./bin/update_locale
```

No hay Makefile, CI, Docker ni linter. `bootstrap.sh` tiene un `venv` por defecto hardcodeado a la ruta local de un desarrollador y un shebang malformado (`#/bin/sh`); pasa siempre `<venv_dir>`.

## Arquitectura de los .cfg

- `genwebupc.cfg` (dev) extiende `https://dist.plone.org/release/6.0.6/versions.cfg`, `projects.cfg`, `sources.cfg`, `customizeme.cfg` y `versions.cfg`. Partes: filestorage, newrelic-config, instance, i18ndude, site-packages (omelette) y test.
- `projects.cfg` agrupa eggs por proyecto (genwebcore, genwebupc, genwebaddons, genwebmigration, genwebpatents). Añadir un egg nuevo = ponerlo aquí, en `[sources]` (`sources.cfg`) y pinear su versión en `versions.cfg`.
- `sources.cfg`: `[sources]` de mr.developer, repos git de github.com/UPCnet (master/main; `collective.newrelic` en `develop`). Los eggs de `developeggs` se auto-checkoutean (`auto-checkout`, `always-checkout = false`). `src/` está gitignored.
- `versions.cfg`: pins de versiones liberadas (genweb6.core, genweb6.theme, genweb6.upc, …). Índice propio: `find-links = http://pypi.upc.edu/links`.
- `[filestorage]` (collective.recipe.filestorage) crea mount points ZODB `var/filestorage/Data_N.fs` (partes 2–10).
- **Producción** (separada del dev): `zope-only.cfg` (extiende `deploy/zope-default.cfg`: zc1–zc4, supervisor, Varnish purge) y `zeo-only.cfg` (extiende `deploy/zeo-default.cfg`: zeo1–zeo6 + backups); `deploy/prefe-default.cfg` es el balanceador/pre-frontend. Cada uno tiene sus `bootstrap.sh.*-only.in` y `customizeme.*-only.cfg.in`. Control: `bin/supervisorctl restart all` o `bin/zcN start`. Plantillas en `config/templates/`.

## Gotchas

- `newrelic.ini` y `parts/instance/etc/wsgi.ini` se regeneran en cada buildout: edita `newrelic.ini.template` / el `.cfg`, no el generado.
- La línea `zcml =` de `[instance]` es necesaria para que se recojan las traducciones de workflows.
- `collective.recipe.backup` está pineado a 4.2.0 porque otras versiones rompen la plantilla de backup.
- `TMPDIR` y el venv por defecto están hardcodeados a rutas de un usuario concreto; ajústalos en local.
