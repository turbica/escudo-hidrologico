# Escudo Comunitario de Alerta Hidrológica, Sequía y Protección de Equipamientos: visión de producto

## Concepto

Alerta temprana hiperlocal de crecidas y sequía para equipamientos vulnerables. Observatorio vivo alimentado por flujos de datos automatizados, no un estudio estático.

## Problemática social

Traducir la telemetría hidrológica en tiempo real de las cuencas del Turia, Júcar, Magro, Palancia, Mijares y Vinalopó en alertas tempranas hiperlocales para centros educativos, residencias de mayores y zonas agrícolas ante crecidas súbitas o sequía extrema.

## Frecuencia de actualización

**Tiempo real (cada 5 minutos) / diaria**

## Funcionamiento

- Extracción cada 5 minutos de caudales circulantes (m³/s), umbrales de alerta (niveles 1, 2 y 3), QT500 y caudales mínimos ecológicos (normalidad / sequía) del SAIH Júcar.
- Cruce espacial con los 7 niveles de peligrosidad de inundación de PATRICOVA (6 probabilísticos + 1 geomorfológico).
- Cruce con Equipamientos municipales (n.º 37) para localizar colegios, centros de día y servicios sociales en zonas de riesgo aguas abajo.
- En estiaje, cruce con Calidad del agua de consumo (n.º 16) y Áreas empresariales o industriales (n.º 10).

## Impacto esperado

Sistema de aviso temprano para centros educativos, centros de mayores y colectivos agrarios cuando las cuencas altas superan umbrales de crecida (p. ej. Magro en Requena/Forata/Macastre o Turia en Zagra/Loriguilla, umbrales iniciales de 5 a 30 m³/s) antes de que la onda llegue a puntos de llanura como Guadassuar, o cuando los caudales caen bajo los mínimos ecológicos de sequía (Vinalopó en Sax 0,01 m³/s, Palancia en Jérica 0,03 m³/s, Magro en Requena 0,07 m³/s).

## Por qué es viable a largo plazo

Convierte métricas hidráulicas crudas en protocolos ciudadanos comprensibles y accionables. Nicho totalmente inexplorado por proyectos sociales.

## Fuentes de datos

Configuración programática en `config/fuentes.json`:

```json
{
  "saih_jucar_resumen_umbrales": "https://saih.chj.es/resumen-aforos",
  "saih_jucar_aforos_tiempo_real": "https://saih.chj.es/aforos",
  "patricova_inundabilidad": "https://datos.gob.es/es/catalogo/a10002983-patricova-peligrosidad-por-inundacion-plan-de-accion-territorial-de-caracter-sectorial-sobre-prevencion-del-riesgo-de-inundacion-en-la-comunitat-valenciana",
  "geoportal_valencia_capas": "https://geoportal.valencia.es/apps/GeoportalHome/es/inicio/geodatos-abiertos",
  "valencia_ckan_portal": "https://opendata.vlci.valencia.es/",
  "datasets_municipales_femp80": [
    "Nº 10 - Áreas empresariales o industriales",
    "Nº 16 - Calidad del agua para el consumo humano",
    "Nº 37 - Equipamientos municipales (centros públicos, educativos y sociales)"
  ],
  "estaciones_referencia_saih": [
    "Turia: EA 18 Zagra (Tuéjar), Embalse de Benagéber, EA 147 Salida Loriguilla (Chulilla)",
    "Magro: EA 60 Requena, Embalse de Forata (Yátova), EA 93 Macastre, MC Guadassuar",
    "Palancia y Mijares: EA 148 Jérica, EA 74 Fuente del Baño (Navajas), EA 145 Salida de Arenós, EA 5 Presa de Vila-real",
    "Vinalopó: MC Sax"
  ]
}
```

## Gobernanza

- Si un dataset clave llega con retraso o poca desagregación, solicitarlo por los canales formales (Ayuntamiento, Oficina de Estadística, mecanismo participativo de Dades Obertes GVA, Ley 1/2022) y llevar un registro público de esas solicitudes dentro de la plataforma.
- Alianzas en cuádruple hélice con Las Naves y el grupo de datos de la UPV; vincular el proyecto a la Misión Climática València 2030.
- Cuando haya prototipo, presentarlo a los Premios de Datos Abiertos y Periodismo de Datos del Ayuntamiento y a los encuentros de innovación de la Diputación y Red.es.
