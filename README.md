# Clase 8 · El Juez: evaluación automática de agentes (n8n)

**Coderhouse · AI Automation Avanzado · Módulo 8**: AI-as-a-Judge, rúbricas, umbrales, autocorrección y human in the loop.

> **El Juez pone la nota. El código decide la ruta.** Y lo riesgoso lo aprueba una persona.

Partimos del Agente de Churn de la [clase 7](https://github.com/jeyzee23/clase-7-agentes-por-industria). El agente ya no entrega directo: cada respuesta pasa por un **Juez**, un sub-workflow aparte que la califica contra una rúbrica. Según la nota, la respuesta sigue, vuelve al agente para corregirse o la ve una persona.

## Qué hay acá

| Archivo | Qué es |
|---|---|
| [`flujos/1-supervisor-judge.json`](flujos/1-supervisor-judge.json) | **El Juez** (SUPERVISOR_Judge): sub-workflow de 5 nodos que solo evalúa. |
| [`flujos/2-agente-de-churn-con-juez.json`](flujos/2-agente-de-churn-con-juez.json) | **El Agente de Churn con el Juez implementado**: el flujo de la clase 7 más 4 nodos nuevos y 3 rutas. |
| [`guia/armado-paso-a-paso.html`](https://jeyzee23.github.io/clase-8-evaluacion-agentes/guia/armado-paso-a-paso.html) | **Receta para armarlo a mano** desde el Agente de Churn de la clase 7, con botón Copiar. |

## Cómo importarlo (el orden importa)

1. **n8n** → *Import from File* → **`1-supervisor-judge.json`** → guardalo (**Ctrl+S**).
2. Otro workflow nuevo → *Import from File* → **`2-agente-de-churn-con-juez.json`**.
3. En el nodo **Llamar al Juez** → *Workflow* → **From list** → elegí **Clase 8 · SUPERVISOR_Judge**. No viene elegido porque el ID cambia en cada cuenta.
4. Credencial **Groq** en los dos nodos *Groq*: el del Juez y el del agente. La API key es gratis en [console.groq.com](https://console.groq.com). No hay keys en este repo.
5. **Execute workflow** → se abre el formulario.

## Cómo funciona

```
Señal de cuenta → Health score → Agente de Churn → Preparar evaluación → Llamar al Juez → Veredicto del Juez
                                      ↑                                                     ├─ Aprobado → Semáforo de riesgo → ¿humano? → Resultado
                                      └──────────── Preparar reintento ←── Corregir ────────┤
                                                                                            └─ Escalar → Aprobación humana → Resultado
```

**Dentro del Juez:** Recibe la respuesta → Juez (AI Agent con la rúbrica y salida JSON) → Umbrales (código).

| Nota del Juez | Ruta |
|---|---|
| 9 o 10 | **Aprobado**: pasa al semáforo de riesgo de la clase 7. |
| 6 a 8 | **Corregir**: vuelve al agente con las críticas. Máximo 2 reintentos; después, Escalar. |
| menos de 6 | **Escalar**: decide una persona. |
| el borrador promete plata (reembolso, descuento, bonificación) | **Escalar siempre**, sin importar la nota. Es una **regla dura** en código. |

Son dos controles distintos: el **Juez** mira la **calidad** de la respuesta y el **semáforo** mira el **riesgo de negocio** (confianza, acción sensible, prioridad Alta).

## Casos de prueba

Siempre con **Logística Pampa** · ARR 80000 · Logins 3 · Tickets 4 · "Tercer error de API esta semana, si no lo resuelven evaluamos cancelar." Cambiá solo el **Modo de prueba**:

| Modo de prueba | Qué pasa |
|---|---|
| Normal | El Juez aprueba (10/10). El semáforo igual frena por prioridad Alta y pide aprobación. |
| Inventar datos | El agente usa un NPS y un vencimiento de contrato que no están en las señales. El Juez le pone 7 → **Corregir** → el agente rehace sin esos datos → 10/10. En *Executions* vas a ver el agente ejecutado 2 veces. |
| Prometer reembolso | El borrador promete un reembolso y 50% de descuento → **Escalar** por la regla dura → directo a la aprobación humana. |

> El **Modo de prueba** es solo para la clase: le agrega al agente datos que el Juez no ve. Sin eso, un buen agente casi siempre saca 10 y no se ven las otras rutas.

## Lo que aprendimos probándolo en vivo

- **Lo riesgoso no puede depender de la nota de un LLM.** En una corrida, el Juez le puso 6 a una promesa de reembolso y la mandó a *Corregir* en vez de a una persona. Por eso la promesa de plata pasó a ser una regla dura en el código.
- **Una rúbrica se calibra.** Al principio el Juez descontaba por cada dato inventado y mandaba a *Escalar* algo que se arreglaba reescribiendo. Ahora los datos inventados descuentan 3 una sola vez y van a *Corregir*.
- **Retry On Fail** en los dos agentes. A veces Groq devuelve un JSON mal armado (*Failed to parse tool call arguments*) y el reintento lo resuelve solo.
- **El formulario de prueba escucha unos 2 minutos** después de tocar *Execute workflow*. Si tarda más, volvé a ejecutarlo.

## Para la pre-entrega (Checkpoint Módulo 8)

Este flujo cubre el bloque **Supervisor + rúbrica** (60%). Te falta sumar:

1. **Registrar cada veredicto** en una tabla externa (Data Table de n8n, Google Sheets, Airtable o Postgres). El checkpoint lo pide.
2. El **test A/B**: 5 corridas con A y 5 con B (por ejemplo, otro modelo para el Juez), con precisión y costo.
3. El **dashboard**: precisión media, errores más frecuentes, costo proyectado a 1.000 ejecuciones y una recomendación.

**Ojo con los nombres:** los criterios del checkpoint piden `exactitud_factual` de 1 a 5 y los estados `ACEPTADO / CORREGIR / RECHAZADO`. Es el mismo patrón que este flujo: respetá esos nombres en tu entrega o explicá la equivalencia.
