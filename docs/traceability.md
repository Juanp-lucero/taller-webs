# Matriz de trazabilidad

Esta matriz precede a la implementación. Las filas registran el diseño y la verificación previstos; no afirman que una función, prueba, medición o despliegue ya exista. En cada fase se agregarán rutas concretas, comandos ejecutados y resultados observados.

## Requerimientos funcionales

| ID | Requisito | Implementación prevista | Prueba prevista | Evidencia prevista | Estado |
|---|---|---|---|---|---|
| RF-1 | Recibir actas idempotentemente y responder inequívocamente incluso si se pierde la confirmación | Caso de uso de recepción; script Lua atómico; claves `idem` y `acta+versión`; recibo original en reintentos | Pruebas de respuestas y Redis real con cinco reenvíos y fallo entre registro y confirmación | `docs/evidence/rf-1-idempotency.*` | Pendiente |
| RF-2 | Aplicar correcciones y converger ante versiones fuera de orden | Modelo de versiones y deltas; Lua rechaza versiones obsoletas y registra conflictos de contenido | Versiones 1, 3, 2; órdenes equivalentes; conflicto con contenido diferente en igual versión | `docs/evidence/rf-2-convergence.*` | Pendiente |
| RF-3 | Mantener acumulados atómicos de mesa, puesto, municipio y departamento, con porcentajes informados | Escritura indivisible en Lua; claves jerárquicas y contadores versionados | Integración concurrente Redis que comprueba ausencia de estados parciales | `docs/evidence/rf-3-atomic-aggregates.*` | Pendiente |
| RF-4 | Agregar fan-out, enviar por lotes, ceder al Event Loop y medir retraso | Ventana inicial configurable de dos segundos; deltas por tópico; lotes con cesión; métrica de retraso | Carga reproducible y comparación con 2.560 millones de mensajes ingenuos | `docs/evidence/rf-4-fanout.*` | Pendiente |
| RF-5 | Entregar a suscriptores de su municipio o puesto | `Map<Topic, Set<Connection>>` y canales Redis por tópico | Entrega selectiva y medición de alta, baja, recorrido y canal global frente a tópicos | `docs/evidence/rf-5-topics.*` | Pendiente |
| RF-6 | Recuperar eventos desde el último identificador o enviar una fotografía versionada | Redis Streams con IDs monótonos, retención y recorte configurables; cursor o snapshot | Desconexión de 30 segundos y cursores dentro y fuera de retención | `docs/evidence/rf-6-recovery.*` | Pendiente |
| RF-7 | Cola priorizada y asignación exclusiva recuperable de actas dudosas | Cola Redis; bloqueo con dueño, expiración, renovación por actividad y token de cercado | Dos auditores concurrentes; reasignación y rechazo de token antiguo | `docs/evidence/rf-7-audit-locks.*` | Fuera de fase 1 |
| RF-8 | Mostrar presencia distribuida y expirar conexiones fantasma | Latidos y claves Redis con TTL; estado del coordinador | Integración Redis con expiración de presencia | `docs/evidence/rf-8-presence.*` | Fuera de fase 1 |
| RF-9 | Controlar clientes lentos y recuperar sin desbordar búferes | Medición del búfer; snapshot o desconexión con código de continuidad | Cliente deliberadamente lento alcanza umbral y se recupera | `docs/evidence/rf-9-backpressure.*` | Fuera de fase 1 |
| RF-10 | Degradarse sin Redis y evitar avalanchas de reconexión | No confirmar escrituras sin Redis; degradación visible; cola local; retroceso exponencial con jitter | Caídas de Redis y red; comparación de reconexiones por segundo | `docs/evidence/rf-10-degradation.*` | Parcial en fase 1 |

## Requerimientos técnicos

| ID | Requisito | Implementación prevista | Prueba prevista | Evidencia prevista | Estado |
|---|---|---|---|---|---|
| RT-1 | Ocho instancias detrás de balanceador y URL pública | Selección justificada de plataforma, ocho réplicas y health checks; solo tras aprobación de costos | Consultas repetidas al identificador de instancia y revisión del despliegue | `docs/evidence/rt-1-deployment.*` | Bloqueado: despliegue y costos requieren aprobación |
| RT-2 | Balanceo sin sesiones pegajosas | Estado compartido en Redis; sin afinidad | Verificar instancias distintas y continuidad al reconectar | `docs/evidence/rt-2-no-sticky-sessions.*` | Pendiente |
| RT-3 | Coordinación entre instancias exclusivamente por Redis | Puertos desacoplados del adaptador Redis; sin comunicación servidor a servidor | Inspección de dependencias y prueba con instancias aisladas | `docs/evidence/rt-3-redis-backplane.*` | Pendiente |
| RT-4 | Fan-out por lotes, cediendo el Event Loop y con retraso medido | Planificador de lotes y métrica por instancia | Carga reproducible con muestreo por instancia | `docs/evidence/rt-4-event-loop.*` | Pendiente |
| RT-5 | Transporte elegido por audiencia; mínimo dos transportes coexistentes | WebSocket para jurados; público y verificación según mediciones; SSE y polling comparativos | p50/p95/p99, memoria por conexión, mensajes y bytes/s, CPU y recuperación | `docs/evidence/rt-5-transports.*` | Pendiente |
| RT-6 | Cola de actas del jurado ante conectividad intermitente | IndexedDB, clave de idempotencia estable y reenvío al recuperar red | Corte y restauración de conectividad | `docs/evidence/rt-6-offline-queue.*` | Pendiente |
| RT-7 | Métricas operativas visibles por instancia | Conexiones, mensajes/s, retraso del Event Loop, memoria/conexión, actas/min y latencia de publicación | Medición durante carga, sin datos inventados | `docs/evidence/rt-7-operations.*` | Fuera de fase 1 |
| RT-8 | Prueba reproducible de 20.000 conexiones y perfil de jornada | Herramienta de carga para ráfagas, reenvíos, dudosas y correcciones | Ejecución contra despliegue autorizado; mediciones separadas de proyecciones | `docs/evidence/rt-8-load-test.*` | Fuera de fase 1; requiere capacidad aprobada |
| RT-9 | Página pública para teléfono de gama baja y reporte de INP real | Interfaz móvil optimizada y medición bajo emulación y carga | Captura reproducible de INP, sin inventar valores | `docs/evidence/rt-9-mobile-inp.*` | Fuera de fase 1 |
| RT-10 | URL pública verificable sin intervención del grupo | Publicación reproducible | Verificación externa tras aprobación de costos y configuración | `docs/evidence/rt-10-public-url.*` | Bloqueado: despliegue y costos requieren aprobación |

## Papeles obligatorios de Redis

| Papel | Implementación prevista | Verificación y evidencia | Estado |
|---|---|---|---|
| Backplane de difusión | Pub/Sub por tópico para distribución; Streams para continuidad | Comparar tráfico por tópico y global: `docs/evidence/redis-backplane.*` | Pendiente |
| Historial para recuperación | Redis Streams con IDs monótonos, retención y recorte definidos | Continuidad y memoria medida: `docs/evidence/redis-streams.*` | Pendiente |
| Acumulados | Totales jerárquicos y ranking actualizable | Correcciones y lectura consistente: `docs/evidence/redis-aggregates.*` | Pendiente |
| Operación atómica compuesta | Lua versionado para validar, revertir, aplicar, agregar y registrar evento | Integración Redis sin lecturas parciales: `docs/evidence/redis-lua-atomicity.*` | Pendiente |
| Idempotencia | Registro de `idem` y registro duradero `acta+versión` | Reenvío después de expiración de clave corta: `docs/evidence/redis-idempotency.*` | Pendiente |
| Bloqueo distribuido | Dueño, TTL, renovación por actividad y token de cercado | Reasignación y rechazo de auditor obsoleto: `docs/evidence/redis-locks.*` | Fuera de fase 1 |
| Presencia | Latidos con TTL y estado distribuido | Expiración automática comprobada: `docs/evidence/redis-presence.*` | Fuera de fase 1 |
| Límite de tasa | Contador por credencial y ventana con respuesta explícita | Umbral y respuesta de exceso comprobados: `docs/evidence/redis-rate-limit.*` | Pendiente |

## Supuestos que deben quedar visibles

- La sección 7 del informe no está incluida. Se medirán WebSocket, SSE y polling en p50/p95/p99, memoria por conexión, mensajes y bytes por segundo, CPU y tiempo de recuperación.
- `acta` es un identificador opaco. No se deriva la mesa de su contenido: el ejemplo usa `0042` en el código y `4201` en el campo mesa.
- La persistencia de Redis, garantías de recuperación y pérdidas posibles ante caída se documentarán y probarán; una configuración por sí sola no se considerará demostración.
- No se marcarán mediciones ni URL como logradas sin evidencia ejecutada. Los valores de 20.000 y 400.000 conexiones se reportarán por separado; no se extrapolarán mediciones.
