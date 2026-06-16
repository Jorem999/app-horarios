# PRD: App de Horarios — D'la Nonna

## Problem Statement

D'la Nonna es una panadería artesanal en Ambato, Ecuador. El control de horarios de los empleados se maneja actualmente de forma manual o inexistente. Jorge, el dueño, no tiene visibilidad de quién entra, quién sale, y si se cumplen los horarios asignados a cada empleado. Esto genera problemas de operación, costos imprevistos y falta de información para la toma de decisiones.

Jorge es ex-programador (creó Nexus ERP en VFP, 100K líneas) y quiere construir un sistema moderno, modular y evolutivo. En lugar de migrar el ERP completo, la estrategia es construir apps modulares empezando por Horarios, la necesidad más urgente de la panadería.

## Solution

Aplicación web progresiva (PWA) para el control de marcaciones de entrada y salida de empleados. Corre sobre un celular viejo fijo en la zona de producción, conectado por WiFi local. Los empleados marcan ingresando su número de cédula. El sistema registra eventos puros (timestamp + empleado + tipo de marcación) y el control de cumplimiento se procesa de forma diferida contra los regímenes de cada empleado.

Arquitectura: frontend web standalone (HTML/CSS/JS) con capacidad de operar offline. Backend en Python con PostgreSQL. Despliegue inicial en el NAS de la panadería.

## User Stories

1. Como empleado, quiero marcar entrada con mi cédula, para que quede registrada mi hora de llegada.
2. Como empleado, quiero marcar salida con mi cédula, para que quede registrada mi hora de salida.
3. Como empleado, quiero ver mis marcaciones del día en la pantalla después de marcar, para confirmar que se registraron correctamente.
4. Como administrador, quiero registrar empleados con nombre, cédula y régimen asignado, para que puedan marcar.
5. Como administrador, quiero definir regímenes de horario (fijo y flexible), para adaptarme a distintos tipos de empleado.
6. Como administrador, quiero asignar un horario base a cada empleado con régimen fijo (ej: 08:00-17:00), para calcular cumplimiento.
7. Como administrador, quiero ver el reporte diario de marcaciones, para saber quién llegó temprano/tarde y quién faltó.
8. Como administrador, quiero ver el resumen mensual de marcaciones por empleado, para procesar la nómina.
9. Como administrador, quiero recibir notificaciones de llegadas tarde o ausencias, para poder tomar acciones correctivas.
10. Como administrador, quiero ajustar manualmente una marcación incorrecta (ej: olvido de marcar), para mantener los registros precisos.
11. Como administrador, quiero exportar los datos de marcaciones a CSV/Excel, para integrarlos con sistemas de nómina.
12. Como dueño, quiero que el sistema funcione sin internet (offline-first), para que las marcaciones nunca se pierdan por problemas de conectividad.
13. Como dueño, quiero que el sistema sea operable por empleados con nivel básico de tecnología, para minimizar errores y capacitación.
14. Como administrador, quiero que cada marcación quede registrada con sello de tiempo inalterable, para evitar disputas.
15. Como administrador, quiero poder ver el historial de marcaciones de cualquier empleado en un rango de fechas, para auditoría.
16. Como empleado, quiero una interfaz táctil grande y simple, para marcar rápido sin confusiones.
17. Como administrador, quiero definir tolerancias de minutos para llegadas tarde, para manejar casos realistas sin falsos positivos.
18. Como dueño, quiero que el sistema pueda escalar a más módulos (producción, ventas, recetas), para construir un ecosistema completo.

## Implementation Decisions

- **Metodología:** SDD (Spec Driven Development). Primero la especificación, luego la implementación. Cada historia de usuario guía una iteración.
- **Modelo de datos:** Marcaciones como eventos puros e inmutables. Cada marcación es un registro con: id, empleado_id, timestamp, tipo (entrada/salida), dispositivo, metadata. El control de cumplimiento se calcula por proyección diferida contra el régimen del empleado, no al momento de marcar.
- **Regímenes de horario:**
  - *Fijo:* El empleado tiene un horario base (ej: 08:00-17:00). El sistema evalúa si cada marcación cumple respecto a ese horario, con tolerancia configurable.
  - *Flexible:* Sin horario base. Solo se registran las horas trabajadas (entrada-salida). Útil para personal con horarios variables.
- **Hardware:** Celular/tablet viejo fijo en la zona de producción, siempre cargando, conectado vía WiFi local al NAS. No usar celulares personales del personal.
- **Input:** Número de cédula (vía teclado numérico táctil). No se requiere credenciales complejas.
- **Frontend:** HTML/CSS/JS plano (sin framework), con capacidad standalone (funciona sin servidor backend usando localStorage). La interfaz debe ser táctil, grande y simple — no asumir destreza tecnológica del usuario.
- **Backend:** Python (Flask/FastAPI) + PostgreSQL. API REST para consultas de administración. Despliegue inicial en contenedor Docker en el NAS (192.168.100.44).
- **Persistencia:** Los eventos de marcación se almacenan localmente (localStorage/IndexedDB) y se sincronizan con el backend cuando hay conexión. En el servidor, PostgreSQL almacena los registros definitivos.
- **Evolución:** Este es el primer módulo de un sistema mayor. El diseño debe permitir añadir módulos de producción, ventas, recetas e insumos sin reescribir. El frontend puede evolucionar a una SPA con framework si el alcance lo justifica.
- **Autenticación inicial:** Solo por cédula + validación contra el registro de empleados. El sistema asume que el dispositivo físico está en un entorno controlado (zona de producción, solo personal autorizado tiene acceso).
- **No migrar Nexus ERP:** Se construye desde cero para la panadería, adaptado a sus necesidades reales. Las lecciones del Nexus informan el diseño pero no se reutiliza código.

## Testing Decisions

- **Enfoque:** Probar comportamiento a través de la interfaz pública, no implementación interna. Los tests deben sobrevivir a refactors.
- **Seams de prueba:**
  1. *API REST de backend:* Probar endpoints de creación/consulta de marcaciones y empleados con pruebas de integración contra PostgreSQL de prueba.
  2. *Frontend standalone:* Probar el flujo de marcación completo simulando eventos de click/teclado en el DOM. Usar el seam del módulo de persistencia (inyectar un storage fake).
  3. *Lógica de control diferido:* Probar en aislamiento la función que evalúa cumplimiento: dados eventos + régimen, determinar si hubo tardanza, ausencia, etc. Este es un seam natural de módulo profundo.
- **Qué probar:**
  - Flujo de marcación (entrada → salida → visualización)
  - Cálculo de cumplimiento vs régimen fijo y flexible
  - Sincronización offline → online
  - Reportes diarios y mensuales
  - Casos borde: marcación doble, día sin marcar, empleado no registrado
- **Qué NO probar:**
  - Implementación interna de almacenamiento (localStorage, DB)
  - Detalles de renderizado visual (selectores CSS específicos)

## Out of Scope

- Gestión de nómina completa (solo se exportan datos, no se calcula pago)
- Módulo de producción, ventas, recetas e insumos (futuras iteraciones)
- Aplicación móvil nativa (la PWA es suficiente para la fase inicial)
- Biometría (huella, facial) — la cédula es el identificador
- Integración con sistemas contables externos
- Portal de empleados con self-service (cambio de datos personales, solicitudes)
- Multi-sucursal (una sola panadería por ahora)

## Further Notes

- **Inspiración:** Las lecciones del Nexus ERP (VFP) informan las decisiones — evitar diseño sobreingenierizado, priorizar lo que realmente se usa en el día a día de la panadería.
- **Contexto de Jorge:** Panadero artesanal, masón, ex-programador. El horario de trabajo es 13:00-20:00 en la panadería, con disponibilidad de programación en las mañanas (después de las 06:30, después de medicación).
- **Filosofía:** Construir para hoy, diseñar para mañana. El módulo de horarios debe estar operativo rápido (semanas, no meses) pero con suficiente calidad como para que los módulos siguientes se construyan sobre la misma base.
- **Repositorio:** https://github.com/Jorem999/app-horarios
