# Plan práctico para pasar Control Horario a datos compartidos y roles

Este documento resume cómo aplicar las mejoras solicitadas en el orden recomendado, aprovechando Firebase para evitar que los datos se queden en el `localStorage` del navegador.

## 1. Datos compartidos con Firestore
- **Crear proyecto en Firebase** y activar Firestore en modo production.
- **Añadir el SDK** a `index.html` (scripts de `firebase-app` y `firebase-firestore`).
- **Configurar la app** pegando el bloque de `firebaseConfig` y llamando a `initializeApp` + `getFirestore`.
- **Modelar colecciones**:
  - `fichajes` (uid, empleadoId, inicio, fin, ubicacion opcional, creadoPor).
  - `vacaciones` (empleadoId, rangoFecha, estado, comentarios).
  - `agenda` (titulo, fecha/hora, repeticion, checklist, notas, recordatorio).
  - `empleados` (nombre, rol, pin opcional, email opcional, horasDisponibles, limitesVacaciones).
- **Sustituir `localStorage`** por consultas a Firestore y `onSnapshot` para que admin y empleados vean cambios en tiempo real.
- **Reglas iniciales**: permitir lectura/escritura al usuario autenticado sobre sus propios documentos y lectura parcial para admins; pegar las reglas básicas que proporciona Firebase como punto de partida y luego afinar por colecciones.

## 2. Login real y roles
- Opción rápida: **PIN por empleado** guardado en la colección `empleados` (campo `pin`). Se valida contra Firestore y se mantiene un `sessionId` en `sessionStorage`.
- Opción robusta: **Firebase Auth (email/contraseña)**; cada usuario se registra y se almacena su `uid` en la colección `empleados` junto al campo `rol` (`admin`, `empleado`).
- **Control de acceso en UI**: mostrar/ocultar botones de administración según el rol y validar en reglas de Firestore para evitar modificaciones no autorizadas.

## 3. Exportar a Excel/PDF
- Añadir un botón que construya CSV/Excel a partir de los datos en memoria (tras leer de Firestore). Librerías ligeras: `SheetJS` para Excel o `window.print` + CSS para PDF.
- Exportaciones sugeridas:
  - Vacaciones por persona y rango.
  - Fichajes por mes (con cálculo de horas trabajadas y detección de fichajes incompletos).
  - Agenda semanal.

## 4. Notificaciones automáticas
- **Email**: usar la extensión de Firebase “Email Trigger” (envía correo cuando se crea/actualiza un documento). Disparadores recomendados:
  - Nueva solicitud de vacaciones.
  - Aprobación/rechazo de solicitud.
  - Fichaje con más de X horas sin cierre.
  - Evento de agenda para “hoy”.
- **In-app**: escuchar con `onSnapshot` y pintar banners/toasts; no requiere permisos del sistema.

## 5. Control horario a prueba de despistes
- Temporizador que compruebe fichajes abiertos con más de *X* horas; crear un documento de alerta o enviar notificación.
- Añadir botón **“Corregir fichaje” (solo admin)** que abre modal con motivo y guarda la corrección en `fichajes` más un registro en `auditoria`.
- Recordatorio al final de turno: programar verificación al detectar horario previsto en `agenda` o `empleados`.

## 6. Geolocalización con radio permitido
- Al pulsar “Fichar”, usar `navigator.geolocation.getCurrentPosition`.
- Validar distancia respecto a una **lista de ubicaciones permitidas** almacenada en `empleados` o `config/ubicaciones` con un radio (ej. 200 m).
- Si falla o está fuera de rango: pedir motivo o foto (guardar referencia en `fichajes` o `incidencias`).

## 7. Calendario visual y reglas de vacaciones
- Mostrar un calendario (ej. librería `litepicker` o `fullcalendar`) con los rangos de `vacaciones` por empleado.
- **Reglas automáticas** al crear solicitud:
  - Bloquear si excede días disponibles (`empleados.horasDisponibles` o `diasDisponibles`).
  - Detectar solapes con otros rangos en `vacaciones` con estado aprobado.
  - Bloquear festivos/días no disponibles (colección `festivos`).
  - Límites por temporada configurables por rol admin.

## 8. Agenda compartida mejorada
- Campos adicionales en `agenda`: `repeticion` (daily/weekly/monthly), `recordatorioMin`, `notas`, `checklist` (array de items con `done`).
- Al guardar, generar próximas ocurrencias básicas si hay repetición semanal/mensual.
- Recordatorios: disparar notificación in-app o email el mismo día (`recordatorioMin` antes del evento).

## 9. Histórico y auditoría
- Crear colección `auditoria` con: `tipo`, `entidadId`, `accion`, `usuario`, `timestamp`, `cambios`.
- Registrar eventos clave: aprobar/rechazar vacaciones, editar fichaje, ajustes de contador de vacaciones o ubicación permitida.
- Mostrar un log filtrable en el panel admin.

## 10. Instalación como PWA
- Añadir `manifest.webmanifest` con nombre, iconos y colores de la app.
- Registrar un `service-worker` básico para caché de shell y modo offline de lectura (sin cachear datos sensibles).
- Activar `display: standalone` y probar instalación en móvil.

## Siguiente paso recomendado
1. Integrar Firebase (config + Firestore) y migrar datos desde `localStorage`.
2. Activar login por PIN o Auth con roles.
3. Añadir exportación y notificaciones.

Con estas tres entregas la app pasa de depender del navegador a ser multi-dispositivo, con control de acceso y visibilidad inmediata para admin y empleados.
