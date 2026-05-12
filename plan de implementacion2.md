# 📘 PLAN DE IMPLEMENTACIÓN: "AEROPUERTO" – SISTEMA DE GESTIÓN AEROPORTUARIA
> **Alcance:** Desarrollo exclusivo para entornos de desarrollo/staging. Sin analíticas, sin Crashlytics, sin despliegue a producción. Multiplataforma (Android, iOS, Web, Windows). Roles: Admin, Staff Operativo, Pasajero. Estado: Provider. Backend: Firebase (Auth + Firestore + Storage). UI: Paleta azul profesional/corporativa (confianza, tecnología, aviación). Sin código en este documento.

---

## 🏗️ 1. Arquitectura y Estructura de Carpetas (Feature-First)
```
lib/
├── core/
│   ├── config/          # Entornos dev/staging, constantes, rutas base, configuración aeroportuaria
│   ├── theme/           # ThemeData, paleta azul corporativa, tipografía, espaciado, formas
│   ├── utils/           # Validadores, formateadores de hora/vuelo, helpers de fecha/moneda
│   └── services/        # Firebase init, Storage service, Cloud Functions (si aplica), notificaciones
├── features/
│   ├── auth/            # Login, registro, recuperación, RBAC claims (admin/staff/passenger)
│   ├── flights/         # Listado de vuelos, filtros, detalle, búsqueda por número/ruta/fecha
│   ├── bookings/        # Reservas, check-in, asignación de asientos, gestión de pasajeros
│   ├── operations/      # Gestión de puertas, terminales, tripulación, mantenimiento, retrasos
│   ├── admin/           # CRUD vuelos/aerolíneas/personal/puertas, inventario, reportes, configuración
│   └── profile/         # Datos pasajero, historial de vuelos, documentos, preferencias
├── shared/
│   ├── widgets/         # Botones, inputs, tarjetas de vuelo, loaders, snackbars, diálogos de confirmación
│   ├── models/          # Entidades Dart (inmutables, fromJson/toJson conceptual): Flight, Booking, Passenger
│   └── providers/       # AuthProvider, FlightProvider, BookingProvider, OperationsProvider, ThemeProvider
└── main.dart            # Punto de entrada, MultiProvider, MaterialApp.router, init
assets/
├── fonts/
├── images/              # Logos, placeholders de aeronaves, iconos de aeropuerto
└── icons/               # SVGs de avión, puerta, terminal, equipaje, seguridad
```

---

## 🔄 2. Mapeo Relacional → Firestore (NoSQL)
| Entidad SQL | Adaptación Firestore | Estrategia |
|-------------|----------------------|------------|
| `vuelo` | Colección `flights` | Documento con campos base. `segments` como array embebido para vuelos con escalas. |
| `aerolinea` | Colección `airlines` | Datos estáticos, lectura frecuente. Índices por `codigo_iata` y `nombre`. |
| `aeronave` | Colección `aircrafts` | Vinculada a vuelos. `config_asientos` como mapa embebido. |
| `puerta` / `terminal` | Colección `gates` / `terminals` | `gates` con referencia a `terminalId`. Estado en tiempo real (`disponible`, `ocupada`, `mantenimiento`). |
| `pasajero` | Colección `passengers` (vinculada a UID de Auth si es usuario registrado) | Perfil embebido. Documentos de viaje como subcolección `passengers/{id}/documents`. |
| `reserva` + `detalle_reserva` | Colección `bookings` | `passengers` como array de mapas. `seats` embebidos. Campos calculados (`subtotal`, `total`) guardados. |
| `asiento` | Campo `seatMap` dentro de `aircrafts` o subcolección `flights/{id}/seat_assignments` | Según volumen: embebido para configuración base, subcolección para asignaciones dinámicas por vuelo. |
| `tripulacion` | Colección `crew` | Vinculada a `employees`. Asignación a vuelos mediante subcolección `flights/{id}/crew_assignments`. |
| `equipaje` | Subcolección `bookings/{id}/baggage` | Documentos con `peso`, `dimensiones`, `tag_id`, `estado`, `ubicacion_actual`. |
| `mantenimiento` | Colección `maintenance_logs` | Vinculado a `aircrafts`. Historial de revisiones, reparaciones, próximo servicio. |
| `incidencia_vuelo` | Subcolección `flights/{id}/incidents` | `tipo` (retraso, cancelación, emergencia), `motivo`, `fecha`, `impacto`, `acciones_tomadas`. |
| `pago` + `tarifa` | Campos embebidos dentro de `bookings` | `payment: { metodo, estado, referencia, fecha }`, `fares: { base, impuestos, cargos }`. |
| `notificacion` | Colección `notifications` | Para pasajeros y staff. `tipo`, `prioridad`, `leido`, `fecha_envio`, `destinatarios`. |
| `empleado` + `rol_operativo` | Colección `employees` | Vinculado a Auth con custom claims. `rol: admin, supervisor, ground_staff, crew, maintenance`. |
| `proveedor_servicio` | Colección `service_providers` | Catering, limpieza, combustible, handling. Solo lectura/escritura admin. |

---

## 🔐 3. Autenticación y Control de Acceso (RBAC)
| Paso | Acción | Entregable |
|------|--------|------------|
| 3.1 | Registro/Login con email/password vía Firebase Auth | Flujo completo con validación de credenciales corporativas para staff |
| 3.2 | Asignación de Custom Claims (`admin`, `staff`, `passenger`) desde backend o Cloud Functions simuladas | Claims persistentes y accesibles en cliente para enrutamiento condicional |
| 3.3 | Interceptor de rutas (`go_router`) que redirige según rol: Admin→Dashboard, Staff→Operaciones, Passenger→Mis Vuelos | Navegación protegida y experiencial por perfil |
| 3.4 | Reglas de Firestore: lectura pública para estado de vuelos, escritura solo para `admin`/`staff`, acceso a `bookings` solo por pasajero propietario o staff autorizado | `.rules` validadas en emulador con casos de prueba por rol |
| 3.5 | Persistencia de sesión y cierre seguro con limpieza de estado local y revocación de tokens si aplica | Estado coherente post-logout, sin fugas de datos sensibles |

---

## 📊 4. Gestión de Estado con Provider
| Ámbito | Responsabilidad | Alcance |
|--------|-----------------|---------|
| `AuthProvider` | Login, registro, claims, perfil, logout, verificación de rol | Raíz (`MultiProvider`) |
| `FlightProvider` | Listado de vuelos, filtros (fecha, ruta, aerolínea, estado), búsqueda, detalle en tiempo real | Raíz o feature `flights` |
| `BookingProvider` | Crear reserva, asignar asientos, check-in, gestión de pasajeros, cálculo de tarifas | Feature `bookings` |
| `OperationsProvider` | Gestión de puertas, asignación de tripulación, registro de incidencias, mantenimiento | Feature `operations` |
| `AdminProvider` | CRUD vuelos/aerolíneas/aeronaves/personal, reportes, configuración del sistema | Feature `admin` |
| `NotificationProvider` | Envío y recepción de notificaciones push/in-app para estados de vuelo | Raíz (acceso global) |
| `ThemeStateProvider` | Cambio de tonalidades azules, modo claro/oscuro, accesibilidad | Raíz |

> **Patrón de respuesta:** `ResultState<T>` con `idle`, `loading`, `success(data)`, `error(message)`. Todos los proveedores exponen listeners eficientes sin rebuilds innecesarios, con debounce para búsquedas de vuelos.

---

## 🎨 5. UI/UX: Tonalidades Azules Corporativas y Adaptabilidad
| Elemento | Especificación |
|----------|----------------|
| Paleta primaria | `#0A2540` (header/footer), `#1E3A8A` (primario), `#3B82F6` (acentos/estados activos), `#60A5FA` (hover), `#93C5FD` (fondos suaves) |
| Paleta secundaria | `#0EA5E9` (información de vuelo), `#22D3EE` (disponible/ok), `#F59E0B` (atención/retraso), `#EF4444` (crítico/cancelado) |
| Tipografía | `Inter` (cuerpo, legibilidad en pantallas), `Montserrat` (títulos, jerarquía), pesos 400/500/600/700 |
| Espaciado | Base 8px, sistema de 4pt para consistencia multiplataforma y alineación con estándares de aviación |
| Componentes | Botones con radius 8px (profesional), tarjetas de vuelo con sombra sutil, inputs con borde azul en foco, loaders con icono de avión animado |
| Responsividad | `LayoutBuilder` + `AdaptiveLayout`: móvil (check-in rápido), tablet (operaciones en piso), desktop (dashboard admin) |
| Accesibilidad | Contraste WCAG AA, `Semantics` para lectores de pantalla, tamaños de texto escalables, navegación por teclado en Web/Windows para staff |

---

## 🛫 6. Flujos de Usuario por Rol
### 👤 Pasajero
1. Registro/Login (opcional para consultas públicas) → Validación → Home con vuelos destacados
2. Búsqueda de vuelos: origen, destino, fecha, filtros (aerolínea, escala, precio)
3. Detalle de vuelo: horarios, aeronave, disponibilidad de asientos, servicios a bordo
4. Reserva: selección de pasajeros, asignación de asientos (mapa interactivo), equipaje, pago simulado (dev)
5. Check-in: selección de asientos restantes, generación de boarding pass (PDF/QR), notificación de puerta
6. Perfil: historial de vuelos, documentos guardados (pasaporte, visa), preferencias (comida, asiento)

### 👨‍✈️ Staff Operativo (Ground Crew, Tripulación)
1. Login con credenciales corporativas → Claims validados → Dashboard operativo
2. Vista de vuelos del día: estado en tiempo real, puertas asignadas, incidencias activas
3. Gestión de puertas: asignar/liberar puerta, registrar cambios, notificar a pasajeros
4. Check-in de pasajeros: escanear QR, validar documentos, asignar asientos, registrar equipaje
5. Registro de incidencias: retrasos, cancelaciones, emergencias, con flujo de aprobación
6. Comunicación: enviar notificaciones push a pasajeros afectados, actualizar pantallas de información

### 👨‍💼 Administrador
1. Login con credenciales admin → Claims validados → Panel de control completo
2. CRUD Vuelos: número, ruta, horarios, aeronave, tripulación, tarifas, estado
3. Gestión de Aerolíneas/Aeronaves: alta, edición, configuración de asientos, mantenimiento programado
4. Personal: alta de empleados, asignación de roles operativos, horarios, permisos
5. Reportes: ocupación, ingresos, incidencias, mantenimiento, exportación CSV/PDF
6. Configuración: terminales, puertas, proveedores de servicio, parámetros del sistema
7. Validación estricta en formularios, confirmaciones de acciones destructivas, auditoría de cambios

---

## 📦 7. Dependencias (`pubspec.yaml` - Conceptual)
| Categoría | Paquetes |
|-----------|----------|
| Core | `flutter`, `flutter_lints` |
| Firebase | `firebase_core`, `firebase_auth`, `cloud_firestore`, `firebase_storage` |
| Estado | `provider` |
| Enrutamiento | `go_router` |
| UI/Assets | `cached_network_image`, `flutter_svg`, `google_fonts`, `intl`, `fl_chart` (para reportes) |
| Utilidades | `shared_preferences`, `uuid`, `formz`, `equatable`, `qr_flutter` (boarding passes) |
| Dev/Testing | `mocktail`, `flutter_test`, `integration_test` |

> ✅ **Excluido explícitamente:** `firebase_analytics`, `firebase_crashlytics`, `firebase_remote_config`, herramientas de despliegue, CI/CD de producción.

---

## 🧪 8. Pruebas y Validación (Entorno Dev/Staging)
| Tipo | Alcance | Herramienta |
|------|---------|-------------|
| Unitarias | Modelos (Flight, Booking), lógica de cálculo de tarifas, validadores de horario, proveedores | `flutter test` |
| Widget | Tarjetas de vuelo, mapa de asientos, formularios de reserva, componentes de estado | `testWidgets` |
| Integración | Flujo completo: Búsqueda → Reserva → Check-in → Boarding Pass (simulado) | `integration_test` |
| Reglas Firestore | Simulación de lecturas/escrituras por rol: pasajero no puede modificar vuelo, staff solo operaciones asignadas | Firebase Emulator Suite |
| Performance | Rebuilds en listas de vuelos, carga de mapas de asientos, listeners en tiempo real de estado | Flutter DevTools (memory, frames, timeline) |
| Estrés | Simulación de 100+ vuelos activos, 50+ staff concurrentes, actualizaciones en tiempo real | Emulador + scripts de carga |

---

## 📅 9. Roadmap de Desarrollo (Dev/Staging)
| Semana | Foco | Entregable |
|--------|------|------------|
| 1 | Setup, estructura, tema azul corporativo, routing base | Proyecto inicial, `main.dart`, `core/`, `theme/`, configuración de emuladores |
| 2 | Firebase Auth + RBAC + Providers base | Login multi-rol, claims, `AuthProvider`, rutas protegidas por perfil |
| 3 | Firestore mapping + Módulo de Vuelos + Búsqueda | `FlightProvider`, listado paginado, filtros por fecha/ruta/aerolínea, búsqueda en tiempo real |
| 4 | Reservas + Asignación de Asientos + Check-in simulado | `BookingProvider`, mapa de asientos interactivo, generación de boarding pass (QR), flujo de pago dev |
| 5 | Panel Operativo (Staff): gestión de puertas, incidencias, notificaciones | Formularios de incidencia, actualización en tiempo real de estado de vuelos, push notifications simuladas |
| 6 | Panel Admin (CRUD completo) + Reportes + Perfil Pasajero | CRUD de vuelos/aerolíneas/personal, gráficos de ocupación, historial de pasajero, tests de integración |
| 7 | Optimización, accesibilidad, responsive Web/Windows | `AdaptiveLayout` para desktop operativo, DevTools, validación cross-platform, soporte para teclado |
| 8 | Documentación, empaquetado dev, revisión final | README técnico, checklist de despliegue staging, build de prueba para cada plataforma |

---

## ✅ Checklist de Validación Pre-Implementación
- [ ] Esquema Firestore alineado con entidades aeroportuarias relacionales
- [ ] Custom claims definidos para `admin`, `staff`, `passenger` con permisos granulares
- [ ] Reglas de seguridad probadas en emulador: aislamiento de datos por rol
- [ ] Providers con `ResultState` y listeners eficientes (debounce en búsquedas)
- [ ] Rutas protegidas por `go_router` según rol y estado de operación
- [ ] Paleta azul corporativa aplicada a `ThemeData` y componentes clave (tarjetas de vuelo, estados)
- [ ] Dependencias limpias (sin analíticas, sin prod, sin paquetes de monitoreo externo)
- [ ] Estructura de carpetas feature-first lista para escalabilidad y mantenimiento
- [ ] Plan de pruebas unitarias/widget definido para lógica crítica (asignación de asientos, cálculo de tarifas)
- [ ] Documentación de decisiones técnicas iniciada: por qué subcolección vs array embebido en cada caso

---

## 📊 1. Estructura de Colecciones Firestore (Mapeo Relacional → NoSQL)

> 🔹 **Nota de arquitectura:** Firestore es orientado a documentos. Las claves foráneas se modelan como `references` o `strings` con ruta de colección. Las relaciones 1:N se gestionan mediante **subcolecciones** o **arrays embebidos** según el volumen de lecturas y la frecuencia de actualización. Se prioriza la inmutabilidad de datos históricos (ej: estado de vuelo en un momento dado) y la validación en reglas de seguridad.

| Colección / Subcolección | Campo | Tipo Firestore | Restricciones / Default | Mapeo Relacional |
|--------------------------|-------|----------------|--------------------------|------------------|
| **`flights`** | `id` | `string` | PK auto-generado | `vuelo.id_vuelo` |
| | `flightNumber` | `string` | `required`, `unique`, formato IATA (ej: AA123) | `vuelo.numero_vuelo` |
| | `airlineId` | `reference` | `required` → `airlines` | FK → aerolínea |
| | `aircraftId` | `reference` | `required` → `aircrafts` | FK → aeronave |
| | `origin` | `map` | `{airportCode: string, name: string, terminal: string}` | `vuelo.origen` |
| | `destination` | `map` | `{airportCode: string, name: string, terminal: string}` | `vuelo.destino` |
| | `departureTime` | `timestamp` | `required` | `vuelo.hora_salida` |
| | `arrivalTime` | `timestamp` | `required` | `vuelo.hora_llegada` |
| | `status` | `string` | `enum: scheduled, boarding, departed, arrived, delayed, cancelled` | `vuelo.estado` |
| | `gateId` | `reference` | `nullable` → `gates` | FK → puerta asignada |
| | `basePrice` | `number` | `required`, `2 decimales` | `vuelo.precio_base` |
| | `availableSeats` | `number` | `default: 0`, calculado | `vuelo.asientos_disponibles` |
| | `createdAt` | `timestamp` | `auto` | `vuelo.created_at` |
| **`flights/{id}/segments`** *(para escalas)* | `order` | `number` | `required`, secuencial | `vuelo_segmento.orden` |
| | `airportCode` | `string` | `required`, IATA | `segmento.aeropuerto` |
| | `arrivalTime`, `departureTime` | `timestamp` | `required` | `segmento.horas` |
| | `layoverDuration` | `number` | `nullable`, minutos | `segmento.duracion_escala` |
| **`airlines`** | `id` | `string` | PK auto | `aerolinea.id_aerolinea` |
| | `name` | `string` | `required`, `max: 100` | `aerolinea.nombre` |
| | `iataCode` | `string` | `unique`, `max: 3`, `required` | `aerolinea.codigo_iata` |
| | `logoUrl` | `string` | `nullable` (Storage) | `aerolinea.logo` |
| | `country` | `string` | `nullable` | `aerolinea.pais` |
| | `isActive` | `boolean` | `default: true` | `aerolinea.activa` |
| **`aircrafts`** | `id` | `string` | PK auto | `aeronave.id_aeronave` |
| | `model` | `string` | `required`, `max: 50` | `aeronave.modelo` |
| | `registration` | `string` | `unique`, `max: 20` | `aeronave.matricula` |
| | `airlineId` | `reference` | `required` → `airlines` | FK → aerolínea |
| | `seatConfig` | `map` | `{total: number, classes: {first: n, business: n, economy: n}}` | `aeronave.config_asientos` |
| | `seatMap` | `array<map>` | `[{row: 1, seats: [{label: "1A", class: "first", available: true}]}]` | `aeronave.mapa_asientos` |
| | `status` | `string` | `enum: active, maintenance, retired` | `aeronave.estado` |
| **`gates`** | `id` | `string` | PK auto | `puerta.id_puerta` |
| | `terminalId` | `reference` | `required` → `terminals` | FK → terminal |
| | `name` | `string` | `required`, `max: 10` (ej: "A12") | `puerta.nombre` |
| | `status` | `string` | `enum: available, occupied, maintenance` | `puerta.estado` |
| | `currentFlightId` | `reference` | `nullable` → `flights` | FK → vuelo asignado |
| | `facilities` | `array<string>` | `nullable` (ej: ["jetway", "power", "wifi"]) | `puerta.servicios` |
| **`terminals`** | `id` | `string` | PK auto | `terminal.id_terminal` |
| | `name` | `string` | `required`, `max: 50` | `terminal.nombre` |
| | `code` | `string` | `unique`, `max: 5` (ej: "T1") | `terminal.codigo` |
| | `gatesCount` | `number` | `default: 0` | `terminal.num_puertas` |
| | `facilities` | `array<string>` | `nullable` | `terminal.servicios` |
| **`passengers`** | `id` | `string` | PK = Auth UID (si registrado) o auto | `pasajero.id_pasajero` |
| | `firstName`, `lastName` | `string` | `required`, `max: 80` | `pasajero.nombre/apellido` |
| | `email` | `string` | `unique`, `required` | `pasajero.email` |
| | `phone` | `string` | `nullable`, `max: 20` | `pasajero.telefono` |
| | `passportNumber` | `string` | `nullable`, `encrypted` | `pasajero.pasaporte` |
| | `frequentFlyerNumber` | `string` | `nullable` | `pasajero.numero_frecuente` |
| | `preferences` | `map` | `{seat: string, meal: string, notifications: boolean}` | `pasajero.preferencias` |
| | `isActive` | `boolean` | `default: true` | `pasajero.activo` |
| **`passengers/{uid}/documents`** | `id` | `string` | PK auto | `documento.id_documento` |
| | `type` | `string` | `enum: passport, visa, id_card` | `documento.tipo` |
| | `number` | `string` | `required`, `encrypted` | `documento.numero` |
| | `expiryDate` | `timestamp` | `required` | `documento.vencimiento` |
| | `fileUrl` | `string` | `nullable` (Storage) | `documento.archivo` |
| **`bookings`** | `id` | `string` | PK auto | `reserva.id_reserva` |
| | `flightId` | `reference` | `required` → `flights` | FK → vuelo |
| | `passengerId` | `reference` | `required` → `passengers` | FK → pasajero principal |
| | `bookingReference` | `string` | `unique`, `max: 6`, formato PNR | `reserva.codigo_reserva` |
| | `status` | `string` | `enum: confirmed, checked_in, boarded, cancelled, no_show` | `reserva.estado` |
| | `passengers` | `array<map>` | `[{passengerId, seat, baggageAllowance, specialRequests}]` | `detalle_reserva` |
| | `fares` | `map` | `{base: number, taxes: number, fees: number, total: number}` | `reserva.tarifas` |
| | `payment` | `map` | `{method, amount, status, reference, date}` | `pago` |
| | `createdAt` | `timestamp` | `auto` | `reserva.fecha_creacion` |
| **`bookings/{id}/baggage`** | `id` | `string` | PK auto | `equipaje.id_equipaje` |
| | `passengerIndex` | `number` | `required` (índice en array passengers) | FK → pasajero en reserva |
| | `type` | `string` | `enum: carry_on, checked, special` | `equipaje.tipo` |
| | `weight` | `number` | `nullable`, kg | `equipaje.peso` |
| | `tagId` | `string` | `unique`, generado | `equipaje.codigo_tag` |
| | `status` | `string` | `enum: registered, loaded, in_transit, delivered, lost` | `equipaje.estado` |
| | `location` | `string` | `nullable` (ej: "Terminal 1, Cinta 3") | `equipaje.ubicacion` |
| **`employees`** | `id` | `string` | PK = Auth UID | `empleado.id_empleado` |
| | `firstName`, `lastName` | `string` | `required` | `empleado.nombre/apellido` |
| | `email` | `string` | `unique`, `required` | `empleado.email` |
| | `role` | `string` | `enum: admin, supervisor, ground_staff, crew, maintenance` | `empleado.rol` |
| | `assignedTerminal` | `reference` | `nullable` → `terminals` | FK → terminal asignada |
| | `shift` | `map` | `{start: timestamp, end: timestamp, days: array}` | `empleado.turno` |
| | `isActive` | `boolean` | `default: true` | `empleado.activo` |
| **`maintenance_logs`** | `id` | `string` | PK auto | `mantenimiento.id_registro` |
| | `aircraftId` | `reference` | `required` → `aircrafts` | FK → aeronave |
| | `type` | `string` | `enum: routine, repair, inspection, emergency` | `mantenimiento.tipo` |
| | `description` | `string` | `required`, `max: 500` | `mantenimiento.descripcion` |
| | `performedBy` | `reference` | `required` → `employees` | FK → empleado responsable |
| | `startDate`, `endDate` | `timestamp` | `required` | `mantenimiento.fechas` |
| | `status` | `string` | `enum: scheduled, in_progress, completed, deferred` | `mantenimiento.estado` |
| | `nextServiceDate` | `timestamp` | `nullable` | `mantenimiento.proximo_servicio` |
| **`notifications`** | `id` | `string` | PK auto | `notificacion.id_notificacion` |
| | `type` | `string` | `enum: flight_status, gate_change, boarding, emergency, promo` | `notificacion.tipo` |
| | `priority` | `string` | `enum: low, medium, high, critical` | `notificacion.prioridad` |
| | `title`, `body` | `string` | `required` | `notificacion.titulo/mensaje` |
| | `targetAudience` | `map` | `{roles: array, flightIds: array, passengerIds: array}` | `notificacion.destinatarios` |
| | `sentAt` | `timestamp` | `auto` | `notificacion.fecha_envio` |
| | `readBy` | `array<string>` | `default: []`, UIDs de lectores | `notificacion.leidos` |

---

## 📦 2. Dependencias `pubspec.yaml` (Listado Conciso)

```yaml
dependencies:
  flutter:
    sdk: flutter

  # 🔥 Firebase Core + Servicios
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.4
  firebase_storage: ^12.3.2

  # 🔄 Estado & Enrutamiento
  provider: ^6.1.2
  go_router: ^14.2.7

  # 🎨 UI & Assets
  cached_network_image: ^3.4.1
  flutter_svg: ^2.0.10
  google_fonts: ^6.2.1
  intl: ^0.19.0
  fl_chart: ^0.69.0  # Para gráficos de reportes admin

  # 🛠️ Utilidades & Persistencia Local
  shared_preferences: ^2.3.2
  uuid: ^4.5.0
  formz: ^0.7.0
  equatable: ^2.0.5
  qr_flutter: ^4.1.0  # Generación de boarding passes

dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter
  flutter_lints: ^5.0.0
  mocktail: ^1.0.4
```

> ✅ **Notas técnicas:**
> - Todas las versiones son estables y compatibles con Flutter `3.22+` / `3.24+` (2024-2026).
> - Se excluyeron explícitamente: `firebase_analytics`, `firebase_crashlytics`, `firebase_remote_config`, y cualquier paquete de despliegue o monitoreo de producción.
> - `formz` y `equatable` facilitan validación de formularios operativos y comparación de modelos sin boilerplate.
> - `go_router` maneja rutas declarativas, parámetros complejos (ej: `/flights?origin=MEX&destination=CUN&date=2026-01-15`), y protección por `redirect` basada en `AuthProvider`.
> - `qr_flutter` permite generar boarding passes en formato QR para simulación de check-in y boarding.

---

## 📌 Siguientes Pasos Recomendados
1. **Validación de estructura de asientos:** Confirma si prefieres `seatMap` como **array embebido** en `aircrafts` (recomendado para configuración base) o como subcolección `flights/{id}/seat_assignments` para asignaciones dinámicas por vuelo con historial.
2. **Reglas de seguridad:** Puedo generar el bloque `.rules` de Firestore alineado a roles `admin`/`staff`/`passenger` con validaciones de tipo, rangos de fecha y aislamiento de datos sensibles.
3. **Flujo de Providers:** Esquema detallado de `ChangeNotifier` con `ResultState<T>` y listeners optimizados por feature, incluyendo debounce para búsquedas de vuelos y actualización en tiempo real de estado.
4. **Emuladores:** Configuración paso a paso de `firebase emulators:start` para Auth + Firestore + Storage en dev/staging, con scripts de seed para datos de prueba (vuelos, aeronaves, personal).

¿Deseas que proceda con las **reglas de seguridad por rol**, la **configuración de emuladores**, o la **estructura detallada de los Providers** antes de pasar a la generación técnica?

---

# 🎨 Guía de Diseño Visual: "Aeropuerto" – Sistema de Diseño UI/UX

> **Concepto:** Sistema de gestión aeroportuaria con estética profesional, tecnológica y orientada a la eficiencia operativa.  
> **Paleta base:** Tonalidades azules corporativas (confianza, precisión, aviación).  
> **Plataforma:** Flutter multiplataforma (Android, iOS, Web, Windows).  
> **Enfoque:** Accesibilidad WCAG AA, legibilidad en entornos de alta presión, adaptabilidad responsive para móvil (pasajero), tablet (staff en piso) y desktop (admin).

---

## 🎨 1. Paleta de Colores (Sistema de Diseño Azul Corporativo)

### 🔵 Colores Primarios (Marca Institucional)
| Nombre | Hex | Uso Principal | Contraste sobre blanco |
|--------|-----|---------------|------------------------|
| `primaryDark` | `#0A2540` | AppBar, footer, headers de sección, textos críticos | ✅ AAA |
| `primary` | `#1E3A8A` | Botones primarios, estados activos, enlaces, indicadores de selección | ✅ AA |
| `primaryLight` | `#3B82F6` | Hover, acentos secundarios, iconos interactivos, bordes de foco | ✅ AA |
| `primarySoft` | `#60A5FA` | Fondos de tarjetas, estados de carga suave, separators | ✅ AA |
| `primaryPale` | `#93C5FD` | Placeholders, bordes sutiles, estados deshabilitados | ✅ AA |

### ⚪ Colores Neutros (Base Operativa)
| Nombre | Hex | Uso |
|--------|-----|-----|
| `background` | `#FFFFFF` | Fondos de pantallas, tarjetas de información |
| `surface` | `#F8FAFC` | Fondos de secciones, paneles laterales, áreas de formulario |
| `border` | `#E2E8F0` | Bordes de inputs, divisores de tabla, contenedores |
| `textPrimary` | `#0F172A` | Títulos, información crítica, números de vuelo |
| `textSecondary` | `#475569` | Subtítulos, descripciones, etiquetas de formulario |
| `textDisabled` | `#94A3B8` | Texto deshabilitado, placeholders, estados inactivos |

### 🟢🟡🔴 Colores de Estado (Feedback Operativo)
| Estado | Hex | Uso en Contexto Aeroportuario |
|--------|-----|-------------------------------|
| `onTime` / `success` | `#10B981` | Vuelo a tiempo, check-in completado, puerta asignada |
| `delayed` / `warning` | `#F59E0B` | Retraso moderado, atención requerida, stock bajo de asientos |
| `cancelled` / `error` | `#EF4444` | Vuelo cancelado, error crítico, acción destructiva |
| `boarding` / `info` | `#3B82F6` | Embarque en progreso, información general, notificación |
| `maintenance` | `#8B5CF6` | Aeronave en mantenimiento, puerta fuera de servicio |

### 🌙 Modo Oscuro (Opcional - Escalable para Ops Nocturnas)
| Elemento | Hex Claro | Hex Oscuro |
|----------|-----------|------------|
| Background | `#FFFFFF` | `#0F172A` |
| Surface | `#F8FAFC` | `#1E293B` |
| Text Primary | `#0F172A` | `#F1F5F9` |
| Text Secondary | `#475569` | `#94A3B8` |
| Border | `#E2E8F0` | `#334155` |
| Card Shadow | `rgba(0,0,0,0.1)` | `rgba(0,0,0,0.3)` |

---

## 🔤 2. Tipografía (Google Fonts - Legibilidad Operativa)

### Fuentes Principales
| Uso | Fuente | Peso | Tamaño Base | Line Height | Notas |
|-----|--------|------|-------------|-------------|-------|
| Títulos H1 (Dashboard) | `Montserrat` | 700 (Bold) | 28px / 1.75rem | 1.2 | Números de vuelo, métricas clave |
| Títulos H2 (Secciones) | `Montserrat` | 600 (SemiBold) | 22px / 1.375rem | 1.3 | Encabezados de módulo |
| Títulos H3 (Tarjetas) | `Montserrat` | 600 (SemiBold) | 18px / 1.125rem | 1.4 | Títulos de vuelo, pasajero |
| Cuerpo (Información) | `Inter` | 400 (Regular) | 15px / 0.9375rem | 1.5 | Descripciones, detalles operativos |
| Números / Códigos | `Inter` | 500 (Medium) | 16px / 1rem | 1.4 | Números de vuelo, PNR, tags de equipaje |
| Botones / Acciones | `Inter` | 500 (Medium) | 14px / 0.875rem | 1.4 | Textos de botones, enlaces de acción |
| Caption / Metadata | `Inter` | 400 (Regular) | 12px / 0.75rem | 1.4 | Horarios, códigos de aeropuerto, estados |

### Escala Responsiva (Mobile → Desktop)
```dart
// Escala conceptual para adaptabilidad operativa
H1: 24px (mobile) → 26px (tablet) → 28px (desktop)
H2: 18px → 20px → 22px
Body: 14px → 15px → 16px  // Legibilidad crítica en pantallas de staff
Numbers: 16px (fixed)     // Números de vuelo siempre legibles
Button: 14px (fixed)      // Consistencia en acciones
```

---

## 🧱 3. Sistema de Espaciado y Layout (Eficiencia Operativa)

### Grid Base (8pt System - Alineación con Estándares de UI Aeroportuaria)
| Token | Valor | Uso Operativo |
|-------|-------|---------------|
| `space-1` | 4px | Micro-espaciado entre iconos y texto, indicadores de estado |
| `space-2` | 8px | Padding interno de chips de estado, botones pequeños |
| `space-3` | 12px | Espaciado entre elementos relacionados (ej: hora y puerta de vuelo) |
| `space-4` | 16px | Padding estándar de tarjetas de vuelo, contenedores de formulario |
| `space-5` | 24px | Separación entre secciones en dashboard, márgenes de módulo |
| `space-6` | 32px | Márgenes entre bloques principales en desktop admin |
| `space-7` | 40px | Espaciado hero en landing de búsqueda de vuelos |
| `space-8` | 48px | Márgenes grandes en reportes y vistas de análisis |

### Layout Responsive por Rol
| Breakpoint | Ancho | Columnas | Gutter | Margen | Uso Principal |
|------------|-------|----------|--------|--------|---------------|
| Mobile | < 600px | 4 | 16px | 16px | Pasajero: búsqueda rápida, check-in móvil |
| Tablet | 600–1024px | 8 | 24px | 24px | Staff operativo: gestión de puertas, incidencias en piso |
| Desktop | ≥ 1024px | 12 | 32px | 48px | Admin: dashboard completo, reportes, configuración |

---

## 🧩 4. Componentes UI Específicos para Aeroportuario

### 🔘 Botones Operativos
| Variante | Fondo | Texto | Borde | Radius | Estados |
|----------|-------|-------|-------|--------|---------|
| Primary (Acción Crítica) | `#1E3A8A` | `#FFFFFF` | none | 8px | hover: `#3B82F6`, pressed: `#1E3A8A` + sombra interna |
| Secondary (Acción Secundaria) | `transparent` | `#1E3A8A` | `#1E3A8A` 2px | 8px | hover: fondo `#EFF6FF` |
| Status (Estado de Vuelo) | Dinámico (`success`/`warning`/`error`) | `#FFFFFF` | none | 20px (pill) | Solo visual, no interactivo |
| Outline (Acciones de Lista) | `transparent` | `#475569` | `#E2E8F0` 1px | 8px | hover: borde `#3B82F6` |
| Disabled | `#E2E8F0` | `#94A3B8` | none | 8px | no interacción, opacidad 70% |

### ✈️ Tarjeta de Vuelo (Flight Card - Componente Central)
```
┌─────────────────────────────────────┐
│ [AA] American Airlines              │ ← Logo + nombre aerolínea, Inter 14px SemiBold
│ ─────────────────────────────────── │
│ AA123 • MEX → CUN                   │ ← Montserrat 16px Bold, código y ruta
│                                     │
│ 🕐 08:30 → 10:45 • 2h 15m          │ ← Inter 14px, horarios y duración
│ 🚪 Puerta: A12 • Terminal 2        │ ← Icono + texto, estado asignado
│                                     │
│ [🟢 A tiempo]                      │ ← Chip de estado con color dinámico
│                                     │
│ Asientos disponibles: 12           │ ← Progreso visual o texto
│ Precio desde: $1,299 MXN           │ ← Inter 16px Bold, color primary
│                                     │
│ [🔍 Ver Detalle]  [🎫 Reservar]    │ ← Botones outline y primary, ancho adaptativo
└─────────────────────────────────────┘
```

### 🪑 Mapa de Asientos Interactivo (Booking Flow)
- **Grid visual:** Filas (1-30) × Columnas (A-F), con pasillo central
- **Estados por asiento:** 
  - `disponible`: borde `#E2E8F0`, fondo `#FFFFFF`
  - `seleccionado`: borde `#3B82F6` 3px, fondo `#EFF6FF`
  - `ocupado`: fondo `#F1F5F9`, icono de persona, no interactivo
  - `premium`: borde dorado `#F59E0B`, badge "Business"
- **Leyenda fija:** Explicación visual de estados, accesible por tooltip
- **Interacción:** Tap para seleccionar, doble tap para deseleccionar, arrastre para selección múltiple (desktop)

### 🧾 Formularios Operativos (Check-in, Incidencias, Registro)
| Elemento | Estilo Base | Estado Focus | Estado Error | Estado Disabled |
|----------|-------------|--------------|--------------|-----------------|
| TextField | Borde `#E2E8F0`, radius 8px, padding 12px, Inter 15px | Borde `#3B82F6` 2px, sombra suave `0 0 0 3px rgba(59,130,246,0.2)` | Borde `#EF4444`, icono de error a la derecha, helper text rojo | Fondo `#F8FAFC`, texto `#94A3B8`, cursor not-allowed |
| Label | `Inter 14px`, `#475569`, margin-bottom 4px | Color `#1E3A8A`, peso 500 | Color `#EF4444`, icono de advertencia | Color `#94A3B8` |
| Helper Text | `Inter 12px`, `#64748B`, margin-top 4px | — | Color `#EF4444`, icono informativo | — |
| Dropdown / Select | Mismo que TextField, con ícono de flecha | Borde `#3B82F6`, flecha azul | Borde `#EF4444`, mensaje de validación | Opacidad 50%, sin interacción |
| Checkbox / Radio | Borde `#E2E8F0`, check `#1E3A8A`, tamaño 20px | Borde `#3B82F6`, sombra suave | Borde `#EF4444`, icono de error | Opacidad 50%, sin animación |

### 🗂️ Filtros y Chips de Búsqueda de Vuelos
- **Chips de fecha:** `Inter 13px`, padding `8px 16px`, border-radius `20px`, borde `#E2E8F0`
- **Estado seleccionado:** Fondo `#EFF6FF`, texto `#1E3A8A`, borde `#3B82F6`, icono de check pequeño
- **Filtros activos:** Badge con contador en esquina superior derecha del chip, color `primarySoft`
- **Búsqueda avanzada:** Panel expandible con campos de aerolínea, rango de precio, escalas, con botón "Aplicar" y "Limpiar"

### 🎫 Boarding Pass Digital (QR + Información Esencial)
```
┌─────────────────────────────────────┐
│ [LOGO AEROLÍNEA]                    │
│ ─────────────────────────────────── │
│ PASAJERO: Juan Pérez                │ ← Montserrat 16px Bold
│ VUELO: AA123 • MEX → CUN           │
│ FECHA: 15 Ene 2026 • 08:30         │
│ PUERTA: A12 • ASIENTO: 12A         │
│ GRUPO DE EMBARQUE: 3               │
│                                     │
│ ┌─────────────────────────┐        │
│ │ [QR CODE 200x200px]    │        │ ← Generado con qr_flutter, fondo blanco
│ └─────────────────────────┘        │
│                                     │
│ [📱 Guardar en Wallet] [🖨️ Imprimir] │ ← Botones secundarios, ancho completo
└─────────────────────────────────────┘
```

---

## 🖼️ 5. Guías para Imágenes y Assets Aeroportuarios

### Fotografía / Ilustración de Aeronaves y Aeropuertos
| Especificación | Valor |
|----------------|-------|
| Ratio | 16:9 (banners), 4:3 (tarjetas), 1:1 (iconos de aerolínea) |
| Resolución mínima | 1200x675px (banners), 400x400px (iconos) |
| Formato recomendado | WebP (con fallback JPG/PNG) |
| Estilo | Profesional, iluminación clara, fondos neutros o degradados azules |
| Uso | Banners promocionales, iconos de aerolínea, placeholders de aeronave |

### Iconografía Operativa
- **Estilo:** Lineal minimalista, grosor de trazo 1.5–2px, consistente con Material Design
- **Tamaño base:** 20px para UI general, 24px para acciones principales, 32px para estados de vuelo
- **Color por defecto:** `#475569`, activo/seleccionado: `#1E3A8A`, estado crítico: `#EF4444`
- **Biblioteca recomendada:** `flutter_svg` + set personalizado de iconos aeroportuarios (avión, puerta, terminal, equipaje, seguridad, reloj, usuario) o `lucide_icons` con extensión

### Ilustraciones para Estados Vacíos y Onboarding
- **Estilo:** Vectorial moderno, trazos limpios, acentos en azul corporativo, toques de color de estado según contexto
- **Uso:** 
  - "No hay vuelos con esos filtros" → Ilustración de avión en tierra con lupa
  - "Sin reservas aún" → Pasajero con maleta mirando pantalla de salidas
  - "Check-in completado" → Boarding pass con check verde animado
- **Formato:** SVG para escalabilidad, máximo 50KB por asset, con variantes para modo claro/oscuro

---

## ♿ 6. Accesibilidad y Experiencia de Usuario Operativa

### Contraste y Legibilidad Crítica
- ✅ Todos los textos sobre fondos cumplen WCAG AA (mínimo 4.5:1), AAA para información crítica (números de vuelo, estados)
- ✅ Tamaños de texto escalables con `MediaQuery.textScaler`, sin romper layouts operativos
- ✅ Iconos con `Semantics` para lectores de pantalla, con labels descriptivos ("Vuelo AA123, a tiempo, puerta A12")
- ✅ Números de vuelo y códigos PNR siempre en fuente monoespaciada o con `fontFeature: [FontFeature.tabularFigures()]` para alineación

### Feedback Visual y Táctil para Entornos de Presión
| Acción | Feedback Operativo |
|--------|-------------------|
| Tap en botón de acción crítica | Escala 0.98 + sombra reducida + ripple azul suave + vibración corta (mobile) |
| Carga de lista de vuelos | Skeleton screens con shimmer en tarjetas de vuelo, no spinner genérico |
| Éxito en check-in | Snackbar verde con icono check + sonido suave (configurable) + actualización inmediata de UI |
| Error en asignación de puerta | Banner rojo persistente en topo de pantalla con botón "Reintentar" y código de error para soporte |
| Sin conexión (staff en piso) | Banner informativo fijo + modo offline limitado (lectura de datos cacheados, acciones en cola para sincronizar) |
| Actualización en tiempo real de estado de vuelo | Animación sutil de pulso en el chip de estado + notificación toast no intrusiva |

### Navegación y Jerarquía por Rol
- **Mobile (Pasajero):** BottomNavigationBar (4 ítems: Inicio, Mis Vuelos, Check-in, Perfil), drawer para configuración y ayuda
- **Tablet (Staff):** NavigationRail lateral izquierdo con iconos + texto, área principal para vista operativa, panel derecho para detalles contextuales
- **Desktop (Admin):** AppBar con menú horizontal + búsqueda global, sidebar para módulos, área principal con grid de widgets/dashboard
- **Breadcrumbs:** En desktop para navegación profunda (ej: `Dashboard > Vuelos > AA123 > Pasajeros`), con enlaces clickeables

---

## 📱 7. Adaptabilidad Multiplataforma Específica

| Plataforma | Ajustes Específicos para Contexto Aeroportuario |
|------------|-------------------------------------------------|
| **Android** | Respetar gestos de retroceso, soporte para back button físico, Material 3 tokens, notificaciones push con canal prioritario para estados de vuelo |
| **iOS** | SafeArea para notch/dynamic island, gestos de swipe para navegación, tipografía San Francisco como fallback, integración con Wallet para boarding passes (simulado en dev) |
| **Web** | Hover states visibles para acciones de staff, URLs amigables con `go_router` para compartir enlaces de vuelo, soporte para teclado (tab navigation) para operaciones rápidas, impresión optimizada de boarding passes |
| **Windows** | Soporte para foco con teclado (crítico para staff en mostradores), redimensionamiento fluido de ventanas, menús contextuales con clic derecho para acciones rápidas en listas de vuelos, integración con impresoras térmicas (simulado) |

---

## 🧪 8. Estados de UI por Componente Crítico (Ejemplo: Tarjeta de Vuelo)

```
[Estado: Idle / Cargado]
→ Imagen/logo aerolínea cargado, horarios visibles, chip de estado con color correspondiente, botón "Reservar" habilitado si hay asientos

[Estado: Loading / Búsqueda]
→ Skeleton: rectángulo gris animado (shimmer) en área de logo, líneas grises para texto de ruta/horarios, botón deshabilitado

[Estado: Success / Reserva Confirmada]
→ Tarjeta con borde verde sutil, badge "✓ Confirmado", botón cambia a "Ver Boarding Pass", notificación toast de éxito

[Estado: Error / Sin Disponibilidad]
→ Tarjeta con overlay gris 20%, texto "Sin asientos disponibles" en diagonal, botón "Notificarme si hay cancelaciones" habilitado

[Estado: Delayed / Retraso]
→ Chip de estado cambia a amarillo `#F59E0B`, texto "Retrasado: nueva salida 09:15", icono de reloj animado, notificación push simulada

[Estado: Offline / Sin Conexión]
→ Banner fijo en topo "Modo offline: mostrando datos cacheados", tarjetas con opacidad 80%, acciones de escritura deshabilitadas con tooltip explicativo
```

---

## 🎯 9. Checklist de Implementación Visual en Flutter

- [ ] `ThemeData` configurado con paleta azul corporativa completa (primarios, neutros, estados operativos)
- [ ] `TextTheme` con `Montserrat` (títulos/jerarquía) e `Inter` (cuerpo/legibilidad) vía `google_fonts`
- [ ] Sistema de espaciado con constantes (`AppSpacing.space4`, etc.) y extensión de `EdgeInsets`
- [ ] Componentes reutilizables: `FlightCard`, `StatusChip`, `SeatMapWidget`, `AppTextField`, `OperationButton`
- [ ] Skeletons personalizados para listas de vuelos, mapa de asientos y dashboard de admin
- [ ] Soporte para modo oscuro preparado en `ThemeData` (aunque no se active inicialmente en staging)
- [ ] Assets optimizados: imágenes WebP, iconos SVG, fuentes subseteadas para reducir tamaño de build
- [ ] Pruebas de contraste con herramientas como `flutter contrast_checker` o plugins de accesibilidad
- [ ] Documentación de componentes en `README.md` o Storybook interno con ejemplos por rol y estado
- [ ] Guías de micro-interacciones documentadas: animaciones de estado, transiciones entre vistas, feedback táctil

---

## 🖼️ Visualización Rápida (Mockup Conceptual - Dashboard Staff)

```
┌─────────────────────────────────────────────────┐
│ [LOGO] Aeropuerto Intl.    [🔔3] [👨‍✈️ Juan]   │ ← AppBar primaryDark, notificaciones y perfil
├─────────────────────────────────────────────────┤
│  Vuelos de Hoy • Terminal 2                      │ ← Título de sección, Montserrat 22px
│  [🕐 Ahora] [📅 Hoy] [🔍 Buscar vuelo...]       │ ← Tabs de filtro + búsqueda global
├─────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────┐    │
│  │ AA123 • MEX→CUN • 🟢 A tiempo          │    │
│  │ 🕐 08:30→10:45 • 🚪 Puerta A12         │    │
│  │ 👥 142/180 asientos • [⚙️ Gestionar]   │    │ ← Tarjeta de vuelo con acción de staff
│  └─────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────┐    │
│  │ DL456 • MEX→JFK • 🟡 Retrasado 25min   │    │
│  │ 🕐 10:00→14:30 • 🚪 Puerta B07         │    │
│  │ 👥 89/220 asientos • [⚠️ Ver Incidencia]│    │ ← Estado warning con acción crítica
│  └─────────────────────────────────────────┘    │
│  ... (lista scrollable)                         │
├─────────────────────────────────────────────────┤
│ [🗓️ Calendario] [📊 Reportes] [⚙️ Configuración] │ ← BottomNavBar para navegación rápida
└─────────────────────────────────────────────────┘
```

---

> ✅ **Nota Final:** Este plan mantiene **idéntica estructura, profundidad y formato** al ejemplo "Urba & Flow", adaptando cada sección al dominio aeroportuario. Todas las decisiones técnicas priorizan el entorno **dev/staging**, sin analíticas ni despliegue a producción. ¿Te gustaría que genere ahora el archivo `firestore.rules` con las reglas de seguridad por rol, o prefieres revisar primero la estructura de los Providers?
