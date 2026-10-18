# escudo-hidrologico

[![Licencia: AGPL v3](https://img.shields.io/badge/licencia-AGPL--3.0-blue.svg)](LICENSE)
[![Open Source](https://img.shields.io/badge/open%20source-s%C3%AD-brightgreen.svg)](LICENSE)

> **Proyecto open source** publicado bajo la licencia **GNU Affero General Public License v3.0 (AGPL-3.0)**.
> Puedes usar, estudiar, modificar y redistribuir el código, siempre que las versiones modificadas,
> incluidas las que se ofrezcan como servicio en red, se publiquen con la misma licencia. Texto completo en [`LICENSE`](LICENSE).

**Escudo Comunitario de Alerta Hidrológica, Sequía y Protección de Equipamientos**

Alerta temprana hiperlocal de crecidas y sequía para equipamientos vulnerables.

Proyecto social de datos abiertos de València y la Comunitat Valenciana. Forma parte de una iniciativa de tres proyectos paralelos (`radar-vivienda`, `monitor-vmp` y este), nacidos del informe [`doc/informe-datos-abiertos-valencia.md`](doc/informe-datos-abiertos-valencia.md).

## Stack

TypeScript (Node.js 24) en todo el proyecto, PostgreSQL + PostGIS, API con Hono, web con Astro y MapLibre, y despliegue con `docker compose`. Usa el módulo común [`valencia-open-data-core`](https://github.com/turbica/valencia-open-data-core) compartido con los proyectos hermanos. El detalle y el motivo de cada decisión están en [`doc/stack.md`](doc/stack.md).

## Estructura

```
backend/              Colectores, motor de alertas, envíos y API
web/                  Web pública con mapa (castellano, valenciano e inglés)
deploy/               docker compose, Caddy y plantilla de configuración
config/fuentes.json   Endpoints, datasets y URLs de origen
config/cuencas.json   Grafo de cuenca: estaciones, tramos y municipios aguas abajo
data/raw/             Descargas periódicas tal cual llegan (no versionado)
data/processed/       Datos limpios y cruzados (no versionado)
doc/                  Informe de origen, stack y documentación
```

## Aviso

Escudo Hidrológico es un sistema ciudadano e informativo, **no un sistema oficial de alerta**. No sustituye a los avisos de la Confederación Hidrográfica del Júcar, AEMET, Protección Civil, el 112 Comunitat Valenciana ni ES-Alert. Ante una emergencia, sigue siempre las instrucciones de las autoridades y llama al 112.

## Canales oficiales

- Solicitud de datos abiertos (Ayuntamiento de València): https://www.valencia.es/web/gobierno-abierto/cas/datos-abiertos/contacta
- Casos de uso oficiales: https://www.valencia.es/web/gobierno-abierto/cas/datos-abiertos/reutilizacion/reutilizacion-datos-abiertos/casos-de-uso
- Premios de Datos Abiertos y Periodismo de Datos: https://www.valencia.es/web/gobierno-abierto/cas/datos-abiertos/reutilizacion/reutilizacion-datos-abiertos/premios-de-datos-abiertos-y-periodismo-de-datos
- Oficina de Estadística municipal: infoestadistica@valencia.es / estadistica@valencia.es
- Fundación Las Naves: datoseguro@lasnaves.com
- API CKAN (documentación): http://docs.ckan.org/en/latest/api/index.html

## Archivos de IA y agentes de código

Puedes usar asistentes de IA y agentes de código (Claude Code, Codex, Gemini, Cursor, Copilot, Windsurf, Cline, Aider…) para trabajar en este proyecto, pero **sus archivos de configuración, instrucciones, memoria o contexto no se pueden publicar en este repositorio**. Cada desarrollador los mantiene solo en su copia local.

Esto incluye, entre otros: `CLAUDE.md`, `.claude/`, `AGENTS.md`, `.agents/`, `GEMINI.md`, `.gemini/`, `.codex/`, `.cursor/`, `.cursorrules`, `.github/copilot-instructions.md`, `.windsurf/`, `.clinerules`, `.aider*`, `.mcp.json` y `skills-lock.json`.

- Ya están en [`.gitignore`](.gitignore). No los fuerces con `git add -f`.
- La comprobación automática [`.github/workflows/politica-archivos-ia.yml`](.github/workflows/politica-archivos-ia.yml) rechaza cualquier push o pull request que los incluya.
- Si usas una herramienta que no está en la lista, añade sus rutas a `.gitignore` y al workflow en el mismo pull request.

## Licencia

Este proyecto es **software libre y open source** bajo la licencia [GNU AGPL-3.0](LICENSE).

- El código puede usarse, modificarse y redistribuirse libremente.
- Cualquier versión modificada que se distribuya o se ofrezca como servicio web debe publicar su código fuente bajo la misma licencia.
- Los datos de origen mantienen la licencia de cada organismo que los publica (consultar la ficha de cada dataset).
