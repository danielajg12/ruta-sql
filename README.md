# Ruta SQL — analítica de growth en marketplaces

Evaluación técnica de SQL que corre entera en el navegador. Sin servidor, sin base de datos externa, sin registro: el motor es **SQLite compilado a WebAssembly** y los datos se generan en el momento con una semilla fija, así que la base es idéntica para todo el que la abra.

Pensado como preparación para entrevistas de analista en empresas de movilidad, delivery y marketplaces en general: adquisición, activación, retención, referidos, fraude y experimentación.

## Qué incluye

**33 retos en cuatro niveles**, de básico a experto: 22 consultas SQL y 11 preguntas de concepto.

Tu consulta se ejecuta de verdad y su resultado se compara contra una solución de referencia. Además se revisa el texto en busca de hábitos que dan el número correcto por casualidad: `COUNT(*)` después de un join, un `LEFT JOIN` roto por un filtro en el `WHERE`, `NOT IN` con nulos, atribución sin deduplicar.

Cada reto SQL trae qué preguntarías antes de escribir, dos niveles de pista, la explicación de por qué la solución está estructurada así, y cómo dirías el razonamiento en inglés.

**A/B testing con datos reales.** La base incluye cuatro experimentos sembrados, cada uno con una patología distinta que hay que descubrir consultando:

| Experimento | Qué esconde |
|---|---|
| `EXP-01` | Efecto real y reparto limpio. El caso sano contra el que comparar. |
| `EXP-02` | Diferencia grande sobre una muestra diminuta. No hay efecto real: es ruido. |
| `EXP-03` | Reparto desbalanceado y conductores expuestos a las dos ramas. |
| `EXP-04` | Agregado plano que oculta efectos opuestos según la ciudad. |

Los efectos están generados dentro de los datos, no escritos en la respuesta.

**Laboratorio.** Una calculadora de tamaño de muestra (95% de confianza, 80% de potencia) y un simulador de *peeking* que corre 30 días de un experimento sin efecto real y grafica el valor p día a día. Alrededor de una de cada cuatro corridas cruza el 0,05 por puro azar.

**Consola libre.** Un botón flotante abre una consola para lanzar cualquier consulta contra la base sin que cuente para la evaluación.

## Publicarlo en GitHub Pages

1. Crea un repositorio nuevo y **público** (Pages gratis requiere repo público).
2. Sube `index.html` a la raíz, junto con este `README.md`.
3. En el repositorio: **Settings → Pages**.
4. En *Source* elige `Deploy from a branch`, rama `main`, carpeta `/ (root)`. Guarda.
5. En uno o dos minutos queda en `https://<usuario>.github.io/<repositorio>/`.

Es un solo archivo. No hay build, ni dependencias que instalar, ni variables de entorno.

## Probarlo en local

```bash
python3 -m http.server 8000
# abre http://localhost:8000
```

Abrirlo con doble clic también funciona, pero un servidor local evita problemas de CORS con el WebAssembly.

Necesita conexión la primera vez: `sql.js` se descarga desde cdnjs (~1,5 MB). Para que funcione sin internet, baja `sql-wasm.js` y `sql-wasm.wasm`, ponlos junto al `index.html` y cambia las dos rutas del CDN por rutas relativas.

## Dialecto

El motor es SQLite con funciones puente para escribir casi igual que en BigQuery:

| Escribe | Nota |
|---|---|
| `SAFE_DIVIDE(a, b)` | devuelve `NULL` si el denominador es 0 |
| `DATE_DIFF(fin, inicio, 'DAY')` | la unidad va entre comillas |
| `DATE_TRUNC(fecha, 'MONTH')` | la unidad va entre comillas |
| `DATE_SUB(fecha, 30, 'DAY')` | firma distinta a la de BigQuery |

`QUALIFY` no existe en SQLite: envuelve la función de ventana en un subquery y filtra por `rn = 1`. Las fechas son texto `AAAA-MM-DD`, así que se ordenan y comparan bien. La fecha de análisis fija es `2026-08-20`.

## Modelo de datos

```
drivers               driver_id, registration_date, approval_date, city,
                      acquisition_channel, referrer_driver_id
referrals             referral_id, referrer_driver_id, referred_driver_id,
                      campaign_id, referral_date
trips                 trip_id, driver_id, trip_date, trip_status, fare
incentive_payments    payment_id, referral_id, payment_date,
                      recipient_type, incentive_amount
campaigns             campaign_id, campaign_name, campaign_start_date,
                      campaign_end_date, city
experiments           experiment_id, name, primary_metric,
                      start_date, end_date
experiment_exposures  exposure_id, experiment_id, driver_id,
                      variant, exposure_date
```

Los pagos cuelgan de `referral_id`, no de `driver_id`. Una invitación puede tener dos pagos: uno para quien invita y otro para el invitado.

## Trampas sembradas a propósito

Los datos no son limpios, porque los de producción tampoco lo son:

- Invitaciones con `referred_driver_id` nulo: el invitado nunca se registró. Un `JOIN` interno las borra y arruina el conteo.
- Conductores invitados por dos personas distintas. Sin deduplicar la atribución, el costo se cuenta dos veces contra un solo activado.
- Conductores registrados hace pocos días, que todavía no pueden haber llegado al día 30.
- Viajes cancelados mezclados con los completados.
- Referidores con invitaciones en ráfaga y algunos auto-referidos.

## Añadir o cambiar retos

Cada reto es un objeto en los arreglos `OLD`, `NEW` o `ABQ` dentro de `index.html`:

```js
{
  lvl: 2,                    // 0 a 3
  type: "sql",               // "sql" o "concept"
  title: "Título corto",
  body: "Enunciado en HTML.",
  cols: ["<code>columna</code> — qué significa"],
  ask:  "Qué preguntarías antes de escribir.",
  hint: "Primera pista.",
  hint2:"Segundo empujón, más concreto.",
  why:  "Por qué la solución está estructurada así.",
  en:   "Cómo lo dirías en inglés.",
  solution: `SELECT ...`,    // la referencia contra la que se compara
  rubric: [
    { test: s => /count\s*\(\s*\*/i.test(s),
      level: "w",            // "w" aviso, "e" error de criterio
      msg: "Explica qué falla y por qué importa." }
  ]
}
```

La comparación ignora el orden de las filas y los nombres de las columnas: mira cantidad de columnas, cantidad de filas y valores, con los decimales redondeados a cuatro posiciones. Si un reto depende del orden, ponle `ORDER BY` en el enunciado y en la solución.

## Licencia

MIT. Haz lo que quieras con él.
