# Stack tecnológico y decisiones de arquitectura

Registro de las decisiones tomadas antes de escribir código. Cada decisión indica el motivo para que quien llegue después pueda revisarla con criterio. Fecha de las decisiones: 18 de octubre de 2026.

## Resumen

| Capa | Elección |
|---|---|
| Lenguaje | TypeScript en todo el proyecto, Node.js 24 LTS, `pnpm` con workspaces |
| Módulo común | [`turbica/valencia-open-data-core`](https://github.com/turbica/valencia-open-data-core) `v0.2.0`: cliente CKAN, `SnapshotManager` (con compresión zstd) y utilidades geo |
| Colector SAIH | `fetch` nativo + `node-html-parser`, validación con Zod y tests contra HTML guardados |
| Planificador | Proceso `worker` con `croner` (cada 5 minutos y diario) |
| Base de datos | PostgreSQL 18 + PostGIS 3.6 |
| Acceso a datos | Drizzle ORM y migraciones versionadas con `drizzle-kit`; consultas espaciales en SQL de PostGIS |
| Histórico bruto | Todo el HTML del SAIH comprimido con zstd en `data/raw/`, sin sobrescribir nunca |
| Modelo de alerta | Umbrales oficiales del SAIH + grafo estático de cuenca |
| API | Hono |
| Frontend | Astro con islas interactivas, MapLibre GL JS |
| Idiomas | Castellano, valenciano e inglés, en la web y en las alertas |
| Avisos | Web y mapa público, bot de Telegram (grammY), RSS, CAP 1.2, webhooks y email por SMTP (Nodemailer) |
| Vigilancia | `/health`, aviso de datos desactualizados y aviso al administrador por Telegram |
| Despliegue | `docker compose` con Caddy (HTTPS automático), backups de Postgres e imágenes publicadas en GHCR |
| Calidad | ESLint, Prettier, Vitest y CI en GitHub Actions |

Todas las dependencias citadas tienen licencias compatibles con AGPL-3.0 (MIT, Apache-2.0, BSD o MIT-0). Cualquier dependencia nueva se consulta antes de añadirla.

## Decisiones

### Lenguaje: TypeScript en todo el proyecto

El módulo común `valencia-open-data-core`, compartido con `radar-vivienda` y `monitor-vmp`, está escrito en TypeScript para Node 24. Usar otro lenguaje en el backend obligaría a duplicar el cliente CKAN y la gestión de instantáneas, que es justo lo que el módulo común evita.

El argumento habitual a favor de Python (geopandas, shapely) aquí pesa poco: los cruces espaciales se hacen en SQL dentro de PostGIS, y las capas de PATRICOVA y de equipamientos se pueden descargar ya en GeoJSON (ver [Fuentes verificadas](#fuentes-verificadas)). Con un único lenguaje, frontend, backend y módulo común comparten tipos y herramientas.

El módulo común se instala desde GitHub **fijado a una etiqueta**, nunca a `#main`, para que los despliegues sean reproducibles:

```json
"@turbica/valencia-open-data-core": "github:turbica/valencia-open-data-core#v0.2.0"
```

### Colector del SAIH Júcar

Comprobado el 18 de octubre de 2026:

- `saih.chj.es/aforos`, `saih.chj.es/resumen-aforos` y las fichas `saih.chj.es/aforos/{id}` llegan **renderizadas en el servidor**. No hace falta navegador headless.
- Hay 80 puntos de aforo. Las filas llevan atributos como `data-supera-umbral`, que facilitan un parseo robusto.
- No hay API documentada, `robots.txt` (devuelve 404) ni aviso legal visible.

Diseño:

- El parser trabaja sobre los atributos `data-*` y las cabeceras de tabla, no sobre posiciones fijas. Cada fila se valida con Zod. Si la validación falla, se guarda el HTML igualmente, se marca la captura como no procesada y se avisa al administrador. **Nunca se publica un dato dudoso como bueno.**
- Los tests de regresión se ejecutan contra HTML reales guardados en `backend/test/fixtures/`. Cuando el SAIH cambie su HTML, fallará un test en vez de romperse el sistema en silencio.
- Como el HTML se conserva entero, si mejora el parser se puede reprocesar todo el histórico.

### Uso responsable del SAIH

Como el SAIH no publica condiciones de uso ni `robots.txt`, aplicamos estas normas:

- **Frecuencia:** dos peticiones cada 5 minutos (`/aforos` y `/resumen-aforos`), que es la frecuencia de actualización del propio dato. Las fichas de estación (QT500, mínimos ecológicos) se consultan una vez al día, con al menos 5 segundos entre peticiones. En total, unas 660 peticiones al día.
- **Identificación:** `User-Agent` con el nombre del proyecto, la URL del repositorio y un contacto configurable (`CONTACT_EMAIL`).
- **Cortesía:** desfase aleatorio en el horario, timeouts, reintentos con espera exponencial, y respeto de `429`/`503` y de `Retry-After`. Si el servidor falla de forma continuada, el colector reduce el ritmo en lugar de insistir.
- **Atribución:** todos los datos se muestran con la cita «Fuente: SAIH Júcar, Confederación Hidrográfica del Júcar».
- **Redistribución:** el HTML bruto se queda en el servidor de quien despliega. La API pública sirve solo datos procesados con su atribución.
- **Acción pendiente:** escribir a la CHJ para informar del proyecto, pedir que confirmen las condiciones de reutilización y preguntar si existe un acceso estructurado a los datos. La solicitud se anota en el registro público de solicitudes (ver `PRODUCT.md`, Gobernanza).

### Almacenamiento: PostgreSQL + PostGIS

- Unos 80 aforos × 288 lecturas al día ≈ 8,4 millones de filas al año. Postgres lo gestiona sin extensiones de series temporales. TimescaleDB queda descartado: añade una extensión que mantener y su licencia (TSL) tiene matices.
- Las geometrías se guardan en EPSG:4326. Los cálculos métricos (distancias, áreas) se hacen con `ST_Transform` a ETRS89 / UTM 30N (EPSG:25830).
- Drizzle define el esquema y genera las migraciones. Las consultas espaciales se escriben en SQL de PostGIS (`sql\`\``), legible para perfiles GIS.
- Las migraciones se ejecutan en un servicio `migrate` de un solo uso antes de arrancar `api` y `worker`, para que quien despliegue actualice sin perder datos.

### Histórico bruto

- Se guarda **todo** el HTML del SAIH en cada captura, comprimido con zstd. Ocupa unos 2 GB al año; sin comprimir serían unos 17 GB.
- Las capturas se guardan con el `SnapshotManager` del módulo común (`compression: "zstd"`, `includeTimeInFileName: true`). Este genera el nombre fechado (`…_aforos.html.zst`) y el `.meta.json` (URL, fecha, SHA-256 del contenido original, tamaño, licencia). `readSnapshot()` devuelve el HTML descomprimido y verifica su hash, lo que permite reprocesar el histórico.
- Para el SAIH se usa `skipIfUnchanged: false`: guardar también las capturas idénticas deja constancia de que el dato no cambió.
- La guía de despliegue documentará una rotación opcional para quien tenga poco disco.

### Modelo de alerta: umbrales + grafo de cuenca

- **Crecida:** cuando una estación supera su umbral oficial (niveles 1, 2 y 3 del SAIH), se activa la alerta en esa estación y en los tramos aguas abajo.
- **Sequía:** cuando el caudal cae por debajo del mínimo ecológico durante un periodo configurable, para evitar falsas alarmas por lecturas sueltas.
- **Grafo de cuenca** (`config/cuencas.json`): estación → tramos → municipios aguas abajo, con **tiempos de viaje orientativos** de la onda (p. ej. Requena → Forata → Macastre → Guadassuar). Es un fichero editable y revisable, sin modelo hidrológico, y cada alerta explica qué estación la ha provocado.
- **Equipamientos afectados:** son los situados en zonas PATRICOVA de los municipios aguas abajo, ordenados por nivel de peligrosidad y por tipo de colectivo (educativo, mayores, discapacidad…).
- **Histéresis:** una alerta no se desactiva hasta que el caudal baja de un margen bajo el umbral, para que no se active y desactive repetidamente.
- Los modelos predictivos (tasa de subida, aprendizaje automático sobre el histórico) se dejan para una fase 2, posiblemente con la UPV, cuando haya histórico propio.

### API

Hono sobre Node, con validación Zod compartida con el resto del código. Endpoints previstos:

- Estado actual por estación, cuenca, municipio y equipamiento.
- Series históricas.
- Alertas activas y su historial.
- Feeds RSS y CAP 1.2 (Common Alerting Protocol, estándar OASIS para que se integren ayuntamientos, medios o Home Assistant).
- Gestión de suscripciones.
- `/health`.

### Frontend

- **Astro** genera páginas estáticas para los protocolos ciudadanos, la metodología, el aviso legal y la privacidad. El mapa y los paneles en vivo son **islas** interactivas. Así la web pesa poco en el móvil durante una emergencia.
- **MapLibre GL JS** con un mapa base libre (OpenStreetMap y teselas vectoriales de un proveedor configurable).
- **i18n** con el sistema nativo de Astro: rutas `/es/`, `/va/` y `/en/` (`lang="ca-ES-valencia"` para valenciano). Las plantillas de alertas también existen en los tres idiomas.
- Accesibilidad AA: el nivel de alerta nunca se comunica solo con color.

### Canales de aviso

| Canal | Implementación | Nota |
|---|---|---|
| Web y mapa | Astro + API | Siempre activo |
| Telegram | grammY | Suscripción por municipio, cuenca o equipamiento. Se desactiva si no hay `TELEGRAM_BOT_TOKEN` |
| RSS / CAP 1.2 | Generados por la API | Sin registro |
| Webhooks | POST firmado con HMAC-SHA256 | Para integraciones de terceros |
| Email | Nodemailer + SMTP genérico (`SMTP_*`) | Doble opt-in y baja en un clic. Se desactiva si no se configura |

Los envíos pasan por una cola en Postgres con reintentos y deduplicación, para no repetir avisos al reiniciar el `worker`.

### Vigilancia del propio sistema

Un sistema de alerta que falla en silencio es peor que no tenerlo.

- `/health` informa de la antigüedad del último dato válido de cada fuente.
- Si el último dato del SAIH tiene más de 15 minutos, la web muestra el aviso «Datos desactualizados» y los feeds lo indican.
- Si se pierden datos, falla el parseo o no se entrega una alerta, el administrador recibe un aviso por Telegram (`ADMIN_TELEGRAM_CHAT_ID`).

### Despliegue

`docker compose up -d` con estos servicios:

| Servicio | Función |
|---|---|
| `db` | PostgreSQL + PostGIS |
| `migrate` | Aplica migraciones y termina |
| `worker` | Colectores, motor de alertas y envíos |
| `api` | Hono |
| `caddy` | Sirve la web estática compilada y actúa como proxy de `/api`. HTTPS automático con Let's Encrypt solo con configurar `DOMAIN` |
| `backup` | `pg_dump` programado con retención configurable |

- **Imágenes:** la CI publica en GHCR imágenes versionadas de `backend` y `web`. Se actualiza con `docker compose pull && docker compose up -d`, sin compilar nada.
- **Perfil de desarrollo:** `compose.dev.yaml` con recarga en caliente, Mailpit para probar el email y un modo *fixture* que lee HTML guardados del SAIH en lugar del servidor real.
- **Configuración:** un único `.env` (plantilla en `deploy/.env.example`). Las URLs de origen siguen en `config/fuentes.json`.
- **Sin servicios gestionados en la nube:** la documentación explica cómo desplegar en cualquier servidor con Docker, pero el proyecto no depende de ningún proveedor.

## Ámbito geográfico

**Comunitat Valenciana.** Las fuentes autonómicas de equipamientos y de PATRICOVA existen, se pueden descargar sin intervención manual y cubren todo el territorio (ver tabla siguiente). En València ciudad se añaden las capas municipales, que tienen más detalle.

El ámbito se configura en un solo lugar, así que se puede restringir a un municipio, una comarca o una cuenca si quien despliega lo prefiere.

## Fuentes verificadas

Comprobado el 18 de octubre de 2026 a través de las APIs CKAN de [Dades Obertes GVA](https://dadesobertes.gva.es) y del [portal del Ayuntamiento](https://opendata.vlci.valencia.es). Todas tienen licencia CC BY 4.0. Las URLs se incorporarán a `config/fuentes.json` al escribir cada colector.

| Uso | Dataset (id CKAN) | Publica | Acceso automatizable |
|---|---|---|---|
| Peligrosidad de inundación (7 niveles) | `patricova-peligrosidad-por-inundacion-…` | ICV | ArcGIS REST de `carto.icv.gva.es` (`ordenacion_territorial/MapServer`, capas 4 a 9: peligrosidad 1 a 6; capa 10: geomorfológica), `f=geojson` paginado. El SHP oficial se genera con un trabajo de geoproceso asíncrono (`GPServer/tarea_descarga_datos`) |
| Centros docentes CV (3.836) | `centros-docentes-de-la-comunitat-valenciana4` | ICV | WFS `terramapas.icv.gva.es/12_Centros_wfs`, en CSV o GPKG |
| Centros sociales CV (1.695) | `centros-inclusivos-de-la-comunitat-valenciana` | ICV | WFS `terramapas.icv.gva.es/19_RECESSO_wfs`. Incluye 302 residencias de mayores dependientes, 146 centros de día de mayores, y centros para personas con discapacidad o enfermedad mental |
| Parques empresariales CV | `ind-par-emp` | GVA | CSV. **Falta comprobar si trae geometría o solo dirección** |
| Equipamientos municipales València (n.º 37) | `equipamients-municipals-equipamientos-municipales` | Ajuntament | ArcGIS REST de `geoportal.valencia.es`, GeoJSON |
| Centros educativos València | `centros-educativos-en-valencia` | Ajuntament | GeoJSON |
| Recursos sociales para mayores y dependientes València | `recursos-socials-dirigits-a-majors`, `recursos-socials-per-a-persones-dependents` | Ajuntament | GeoJSON |
| Calidad del agua de consumo (n.º 16) | **No publicado** ni en el portal municipal ni en el autonómico | — | Se solicitará por los canales formales y se anotará en el registro público de solicitudes. La referencia nacional es SINAC (Ministerio de Sanidad), que ofrece consulta pero no descarga |
| Áreas industriales València (n.º 10) | Sin dataset específico. Lo más cercano es `edificaciones-industriales` | Ajuntament | Para la CV se usa `ind-par-emp` |

## Aviso legal

Debe aparecer en el pie de la web, en una página propia, en cada alerta (versión corta) y en los feeds RSS y CAP (campo `note`):

> **Escudo Hidrológico es un sistema ciudadano e informativo, no un sistema oficial de alerta.** No sustituye a los avisos de la Confederación Hidrográfica del Júcar, AEMET, Protección Civil, el 112 Comunitat Valenciana ni ES-Alert. Ante una emergencia, sigue siempre las instrucciones de las autoridades y llama al 112. Los datos proceden de fuentes públicas (SAIH Júcar, PATRICOVA, Generalitat Valenciana, Ajuntament de València) y pueden contener errores, retrasos o interrupciones. Este servicio se ofrece sin garantía de ningún tipo.

Versión corta para alertas:

> Aviso ciudadano no oficial. En emergencia, sigue a las autoridades y llama al 112.

## Protección de datos (RGPD)

Las suscripciones por email, Telegram y webhook implican tratar datos personales.

- **Responsable del tratamiento:** la persona o entidad que despliega la instancia, no el proyecto. La guía de despliegue lo deja claro y la plantilla de política de privacidad tiene campos para sus datos.
- **Base legal:** consentimiento explícito. En el email se confirma con doble opt-in.
- **Datos mínimos:** email o `chat_id` de Telegram, zonas suscritas, idioma y fecha del consentimiento. Nada más: ni nombre, ni ubicación precisa, ni analítica de terceros.
- **Baja:** en un clic desde cada email, con `/baja` en Telegram, y con borrado efectivo de los datos.
- **Conservación:** suscripciones sin confirmar borradas a las 48 horas; registros de envío borrados a los 90 días.
- **Web sin cookies de seguimiento.** Si se añade analítica, será de alojamiento propio y sin datos personales.
- Plantillas de política de privacidad y aviso legal en `web/` en los tres idiomas.

## Estructura del repositorio

```
backend/              Paquete Node: colectores, motor de alertas, envíos y API (una imagen, dos procesos: worker y api)
  src/collectors/     SAIH, PATRICOVA, equipamientos (vía valencia-open-data-core)
  src/alerts/         Umbrales, grafo de cuenca, cruce espacial
  src/notify/         Telegram, email, webhooks, RSS/CAP
  src/api/            Hono
  src/db/             Esquema Drizzle y migraciones
  test/fixtures/      HTML del SAIH guardados para los tests
web/                  Astro + MapLibre, i18n es/va/en
deploy/               compose.yaml, compose.dev.yaml, Caddyfile, .env.example
config/fuentes.json   URLs y datasets de origen (única fuente de verdad)
config/cuencas.json   Grafo de cuenca: estaciones, tramos, municipios y tiempos de viaje
data/raw/             Descargas tal cual llegan, con fecha en el nombre (no versionado)
data/processed/       Datos limpios y cruzados (no versionado)
doc/                  Informe de origen, este documento y guía de despliegue
```

## Pendientes

- [ ] Escribir a la CHJ sobre la reutilización de los datos del SAIH y anotarlo en el registro público de solicitudes.
- [ ] Solicitar los datos de calidad del agua de consumo (n.º 16) al Ayuntamiento y a la GVA.
- [ ] Comprobar si `ind-par-emp` tiene geometría.
- [x] Compresión zstd en el `SnapshotManager` del módulo común (`v0.2.0`).
- [x] Etiqueta publicada en el módulo común para fijar la versión (`v0.2.0`).
- [ ] Revisar los tiempos de viaje de `config/cuencas.json` con alguien con criterio hidrológico (UPV, CHJ).
