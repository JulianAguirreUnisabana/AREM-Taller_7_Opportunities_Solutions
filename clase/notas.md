# Registro de Trabajo en Clase - Taller 7: Opportunities & Solutions

## Fecha de la sesión
28 de septiembre de 2026

## Integrantes presentes
- Jorge Alarcon
- Julian Aguirre
- Brayan Presiga

## Actividades realizadas en clase

En esta sesión se trabajó el ejercicio guiado del taller, que parte del AS-IS de RedExpress (ya modelado en los Talleres 3 y 4) para proponer por primera vez su arquitectura objetivo (TO-BE) y el análisis de brechas. La idea central es que el TO-BE no se diseña "de cero": cada cambio propuesto debe poder rastrearse hasta un problema ya diagnosticado en un entregable anterior. El ejercicio se desarrolló en cuatro partes.

### Parte 1. Diagnóstico inicial

Se respondieron las tres preguntas orientadoras usando lo que ya se sabía del AS-IS:

| Pregunta | Hallazgo en RedExpress | Origen |
|---|---|---|
| ¿Dónde hay más fricción en la operación? | Un único balanceador de carga y una base de datos que solo escribe en Bogotá: en picos de demanda (campañas de fin de año) la plataforma se vuelve lenta o puede caerse por completo. | Taller 4 |
| ¿Qué problemas repiten los usuarios? | En Medellín se demora la asignación de rutas, porque cada solicitud depende del motor de rutas de Bogotá. | Taller 4 |
| ¿Qué riesgos quedaron evidenciados? | Punto único de falla en el balanceador, cuello de botella de escritura en la base de datos y poca capacidad de crecer geográficamente en Medellín. | Taller 4 |

De ahí salen tres brechas, todas de tipo técnico:

| Brecha | Tipo |
|---|---|
| Balanceador de carga con una sola instancia | Técnica |
| Escritura de la base de datos centralizada en Bogotá | Técnica |
| Medellín sin módulo de rutas propio | Técnica / Funcional |

Para este ejercicio solo se usaron brechas técnicas, porque son las únicas totalmente diagnosticadas en el curso; en un caso real también se sumarían las de seguridad (Taller 5) y las de cumplimiento normativo (Taller 6).

### Parte 2. Propuesta de mejoras

**Lluvia de ideas (sin descartar nada al inicio).** Se listaron 8 ideas, mezclando cambios técnicos con ajustes de proceso y de comunicación:

| # | Idea | Tipo |
|---|---|---|
| 1 | Segundo balanceador en modo activo-pasivo | Técnica |
| 2 | Base de datos particionada por región | Técnica |
| 3 | Módulo de rutas propio en Medellín | Técnica / Funcional |
| 4 | Avisos al cliente cuando se prevé un retraso en su entrega | Proceso / Comunicación |
| 5 | Lista de verificación digital del paquete al entregarlo, firmada por el mensajero | Proceso |
| 6 | Encuesta breve de satisfacción en la app justo después de la entrega | Proceso / Comunicación |
| 7 | Chatbot o canal de WhatsApp para consultar el estado del envío sin llamar a soporte | Proceso / Comunicación |
| 8 | Panel de monitoreo unificado para operadores, con alertas por región | Técnica (mediano plazo) |

**Priorización.** Se escogieron las tres ideas técnicas (1, 2 y 3) porque son las únicas respaldadas por una brecha ya evidenciada en el Taller 4, lo que permite rastrear directamente del AS-IS al TO-BE. Las ideas de proceso (4 a 7) son buenas y baratas de hacer, pero quedan como pendientes para una siguiente iteración porque todavía no tienen un hallazgo formal que las sustente. Si alguna idea incluyera un agente de IA (como el chatbot de la #7), habría que evaluar además sus riesgos propios, como respuestas inventadas o exceso de autonomía.

| Solución | Esfuerzo | Impacto | Tipo | Prioridad |
|---|---|---|---|---|
| Balanceador redundante | Medio | Alto | Quick win | 1 |
| BD particionada por región | Alto | Alto | Largo plazo | 2 |
| Módulo de rutas en Medellín | Alto | Medio | Largo plazo | 3 |

**Cómo elegir entre varias formas de cerrar una misma brecha (resumen de la matriz de decisión).** Se tomó la mejora #1 y se pregunta *cómo* eliminar el punto único de falla del balanceador. Las cifras de costo, plazo y volumen son supuestos del ejemplo.

- **Problema en términos de impacto:** si el balanceador cae en plena campaña de fin de año, toda la plataforma queda inaccesible; con un pico de unos 1.200 envíos por hora, cada hora de caída deja esos envíos sin asignar ni seguimiento.
- **Último momento responsable:** la campaña empieza el 1 de noviembre y la opción más probable tarda unas 8 semanas (6 de implementación y 2 de pruebas), así que la decisión debe tomarse a más tardar hacia el 6 de septiembre.
- **Criterios y pesos (fijados desde el negocio):** disponibilidad 35 %, costo 25 %, complejidad operativa 20 % y tiempo de implementación 20 %, con escala de 1 a 5 (5 es lo más favorable).
- **Opciones comparadas:** A) balanceador activo-pasivo con conmutación automática; B) activo-activo con almacén de sesión compartido; C) dejar el balanceador único y solo reforzar el monitoreo (la opción "radicalmente distinta").

| Opción | Disponibilidad | Costo | Complejidad | Tiempo | Total |
|---|---|---|---|---|---|
| A · Activo-pasivo | 4 | 4 | 4 | 3 | **3,80** |
| B · Activo-activo | 5 | 2 | 2 | 2 | 3,05 |
| C · Único + monitoreo | 1 | 5 | 5 | 5 | 3,60 |

Puntos a tener en cuenta al leer el resultado:

- C queda en segundo lugar solo porque compensa con criterios "baratos" un 1 en disponibilidad; por eso se define un criterio eliminatorio: toda opción con disponibilidad menor a 3 se descarta, y C sale del juego.
- Si los pesos cambian mucho (por ejemplo, disponibilidad solo 15 % y costo 35 %), el resultado se invierte y C ganaría. El desacuerdo real sería entonces sobre lo que valora el negocio, no sobre las opciones.
- Una diferencia menor a 0,3 puntos se trata como empate.

**Decisión:** implementar el balanceador activo-pasivo (opción A). **Trade-off aceptado:** se renuncia a una conmutación instantánea (queda una ventana de 30 a 60 segundos) a cambio de menor costo y menor complejidad. **Descartadas:** B por costo y por no alcanzar a llegar a la campaña; C porque no elimina el punto único de falla. **Reevaluación:** revisar la decisión al terminar la campaña, o antes si el tráfico supera en 30 % el pico registrado; en ese caso se reconsidera B. Esta decisión es el borrador de la ADR que se formaliza en el Taller 9.


### Parte 3. Beneficios y riesgos

**Brechas cerradas y beneficio esperado:**

| AS-IS | TO-BE | Brecha que cierra | Beneficio |
|---|---|---|---|
| Balanceador único | Balanceador redundante (activo-pasivo) | Punto único de falla | Plataforma con alta disponibilidad |
| BD con escritura única en Bogotá | BD particionada por región | Cuello de botella de latencia | Mejor rendimiento del rastreo en tiempo real fuera de Bogotá |
| Medellín sin módulo de rutas | Módulo de rutas replicado en Medellín | Límite de escalabilidad geográfica | La región puede crecer sin saturar Bogotá |

**Riesgos, limitaciones y dependencias:**

| Solución | Lo que podría impedir o retrasar su implementación |
|---|---|
| Balanceador redundante | Necesita que el proveedor cloud apruebe presupuesto adicional para la segunda instancia; sin eso no arranca a tiempo. |
| BD particionada por región | Exige una migración con ventana de mantenimiento; hay riesgo de caída parcial y de inconsistencias al sincronizar las particiones por primera vez. |
| Módulo de rutas en Medellín | Requiere contratar o reasignar personal técnico en la región; sin equipo local nadie lo opera ni lo mantiene. |

**Agrupación por capacidad de negocio.** Para explicarle el beneficio al negocio en su propio lenguaje, las brechas se agrupan por lo que la empresa necesita saber hacer (capacidades, con verbo + objeto y sin nombrar tecnologías), evaluadas con madurez de 1 a 5:

| Capacidad | AS-IS | TO-BE | Qué lo explica |
|---|---|---|---|
| Recepción y registro de envíos | 4 | 4 | Sin brechas; no se toca en esta iteración |
| Planeación y asignación de rutas | 2 | 4 | Medellín depende del motor de Bogotá y se demora |
| Seguimiento en tiempo real | 3 | 4 | Funciona, pero la escritura centralizada lo hace lento fuera de Bogotá |
| Notificación y atención al cliente | 3 | 3 | Sin brecha formal; las ideas 4 y 7 quedaron pendientes |
| Continuidad operativa de la plataforma | 2 | 4 | Punto único de falla en el balanceador |

Con eso se armaron dos paquetes de trabajo:

| Paquete | Brechas que incluye | Capacidad que mejora | Tipo |
|---|---|---|---|
| WP1 · Continuidad de la plataforma | Balanceador redundante activo-pasivo | Continuidad operativa (2 → 4) | Quick win, 4 a 6 semanas |
| WP2 · Rutas y datos regionales | Módulo de rutas en Medellín + BD particionada por región | Planeación de rutas (2 → 4) y seguimiento en tiempo real (3 → 4) | Largo plazo |


## Boceto inicial del modelo

### Diagrama 1. TO-BE de Aplicaciones

Extiende el C2 del Taller 3 añadiendo un Motor de Rutas propio para Medellín, réplica del de Bogotá, para que la región deje de depender de un único punto de procesamiento.

<img width="1602" height="763" alt="to-be-aplicaciones-borrador drawio" src="https://github.com/user-attachments/assets/ba5782bf-e897-40f1-910c-4355b3f2ef10" />


### Diagrama 2. TO-BE de Tecnología

Extiende el mapa de infraestructura del Taller 4: el balanceador pasa a ser redundante (activo-pasivo), la base de datos se particiona por región y Medellín gana su propio módulo de rutas.

<img width="669" height="792" alt="to-be-tecnologia-borrador drawio" src="https://github.com/user-attachments/assets/7061563e-2706-4e19-b429-80422987da3b" />


## Tareas definidas para complementar el taller

Anote las responsabilidades acordadas entre los miembros del equipo para completar la entrega final:

| Tarea asignada | Responsable | Fecha estimada |
|----------------|-------------|----------------|
| Modelado TO-BE de Aplicaciones | Brayan Presiga | 10/08 |
| Modelado TO-BE de Tecnología | Jorge Alarcon | 11/08 |
| Redacción de notas.md | Julián Aguirre | 12/08 |

---

_Este documento resume el trabajo colaborativo realizado durante la sesión del Taller 7: Opportunities & Solutions en el curso AREM - Universidad de La Sabana._
