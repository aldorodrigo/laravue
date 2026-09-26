# Plan de Implementación — SaaS para Academias, Clubes y Escuelas

> Nombre del producto: **pendiente** (finalistas: Crecemy, Cantemy, Nidemy, Cluppy, Crecy, Retoño, Acompaño).
> En este documento se usa `{producto}` como marcador.
> Piloto: **Club Jakare** (fútbol infantil, Paraguay).
> Lógica de negocio y funcionalidades: ver `docs/PLAN_JAKARE.md`.

---

## 1. Alcance del producto

SaaS multi-organización para **academias, clubes y escuelas de formación** (deporte, danza,
música, idiomas, etc.) y **comisiones de padres (ACE)**.

- **Núcleo común:** alumnos, responsables/familias, programas, grupos, horarios, asistencia,
  instructores, inscripciones, tarifas, cuotas, descuentos/becas, mora, cobros, gastos,
  cuentas, avisos push, eventos, informes.
- **Módulos opcionales** (feature flags por organización/plan): comisión directiva y actas,
  rifas/actividades, indumentaria, torneos/convocatorias, evaluaciones/niveles,
  facturación electrónica (SIFEN).
- **Vocabulario configurable:** el código usa nombres genéricos; cada organización elige
  sus etiquetas (Grupo → "Categoría" en un club, "Nivel" en una academia de danza).
- Gestión académica de colegios formales (materias, boletines, MEC): **fuera de alcance**.

---

## 2. Stack — últimas versiones estables (verificadas al 26/09/2026)

### Backend + panel web (misma base que OpenSciRank, actualizada)

| Componente | Versión | Notas |
|---|---|---|
| PHP | **8.5** | Laravel 13 exige ≥ 8.3; spatie/activitylog 5 exige ≥ 8.4 |
| Laravel | **13.x** (13.33) | OpenSciRank está en 12 → aquí arrancamos en 13 |
| Filament | **5.x** (5.9) | Panel de administración + multi-tenancy nativo |
| Livewire | **4.x** (4.4) | Requerido por Filament 5 |
| Tailwind CSS | **4.x** (4.3) | |
| Vite | **8.x** (8.3) | |
| Laravel Sanctum | **4.x** (4.3) | Tokens para la app Flutter |
| Laravel Fortify | **1.40** | Login/2FA web (igual que OpenSciRank) |
| Filament Shield | **4.3** | Roles y permisos en el panel (compatible con Filament 5 y Laravel 13) |
| spatie/laravel-permission | **8.x** (8.3) | Base de Shield |
| spatie/laravel-activitylog | **5.x** (5.1) | Auditoría (clave en finanzas) |
| spatie/laravel-medialibrary | **11.x** | Adjuntos: comprobantes, fotos, actas |
| barryvdh/laravel-dompdf | **3.1** | Recibos, actas y balances en PDF |
| kreait/laravel-firebase | **7.x** (7.2) | Push notifications (FCM) |
| Laravel Horizon | **5.x** | *Opcional:* monitor de colas si pasamos a Redis |
| Laravel Sail | **1.68** | Docker local (runtime PHP 8.5) |
| Laravel Pint | **1.32** | Formato de código |
| PHPUnit | **13.x** | Igual que OpenSciRank (alternativa: Pest 5) |
| Base de datos | **MySQL 8.4 LTS** (dev) / **MariaDB 11.8 LTS** (prod) | Igual que OpenSciRank |
| Colas / caché / sesión | Base de datos al inicio → Redis cuando crezca | |
| Archivos | S3 compatible | |
| Node.js | **24 LTS** | Para Vite |

### App móvil y web

| Componente | Versión | Notas |
|---|---|---|
| Flutter | **3.47** (stable, 18/09/2026) | Android, iOS y web con un solo código |
| Dart | **3.13** | |
| Estado | Riverpod 3 | |
| Navegación | go_router | |
| HTTP | dio | |
| Almacenamiento seguro | flutter_secure_storage | Token Sanctum |
| Push | firebase_messaging + flutter_local_notifications | |
| i18n | flutter_localizations + intl | Español (PY) primero |

---

## 3. Arquitectura

```
┌─────────────────────┐      ┌──────────────────────────────┐
│ App Flutter         │      │ Laravel 13                   │
│ (Android/iOS/Web)   │─────▶│  /api/v1  (Sanctum)          │
│ padres · técnicos · │ HTTPS│  /admin   (Filament 5)       │──▶ MySQL / MariaDB
│ comisión            │      │  colas · scheduler · PDFs    │──▶ S3 (archivos)
└─────────▲───────────┘      └──────────────┬───────────────┘
          │  push                           │
          └──────── Firebase Cloud Messaging◀┘
```

### Repositorios (dos, independientes)
- `{producto}-api` → Laravel: API + panel Filament + scheduler + colas.
- `{producto}-app` → Flutter.

Motivo: ciclos de release distintos (tiendas vs servidor), CI distintos, y la base Laravel
queda igual a OpenSciRank en la raíz del repo (Sail, compose, Dockerfile).

### Multi-organización (tenancy)
- **Una base de datos**, columna `organization_id` en cada tabla del dominio.
- Trait `BelongsToOrganization` con *global scope* + asignación automática al crear.
- **Filament:** tenancy nativo (`->tenant(Organization::class)`), URL `/admin/{organization}`.
- **API:** header `X-Organization` (o el slug en la ruta), validado contra las membresías del usuario.
- Un usuario puede pertenecer a **varias organizaciones** con **roles distintos** en cada una.
- Tests obligatorios de aislamiento: un usuario nunca ve datos de otra organización.

### Dinero
- Montos en **enteros** (guaraníes sin decimales); moneda definida por organización.
- Value object `Money` + cast Eloquent; formateo `₲ 150.000`.
- **Libro mayor inmutable:** los movimientos no se editan ni se borran; se anulan con un contra-movimiento.

---

## 4. Modelo de dominio (nombres en código)

| Módulo | Entidades |
|---|---|
| Plataforma | `Organization`, `Plan`, `Subscription`, `Feature` (flags), `Terminology` (etiquetas) |
| Personas | `User`, `Membership` (usuario + organización + roles), `Invitation` |
| Académico | `Season`, `Program` (disciplina), `Group` (categoría/nivel), `Schedule`, `Session`, `Attendance` |
| Alumnos | `Student`, `Guardian`, `Family`, `StudentGuardian`, `MedicalRecord`, `Enrollment` |
| Staff | `Instructor` (vía `Membership`), `GroupInstructor` |
| Gobierno *(opcional)* | `BoardPosition`, `BoardTerm`, `Meeting`, `MeetingAttendance`, `Minute`, `Resolution`, `Vote` |
| Eventos | `Event`, `EventCallup` (convocatoria + confirmación) |
| Finanzas | `Account`, `LedgerEntry`, `Transfer`, `FeeConcept`, `Tariff`, `Charge`, `ChargeAdjustment`, `DiscountRule`, `Scholarship`, `LateFeePolicy`, `Payment`, `PaymentAllocation`, `PaymentProof`, `Receipt` |
| Egresos | `Expense`, `ExpenseCategory`, `Supplier`, `Approval` |
| Recaudación *(opcional)* | `Fundraiser`, `RaffleBook`, `RaffleTicket` |
| Indumentaria *(opcional)* | `ApparelCampaign`, `ApparelItem`, `ApparelOrder` |
| Comunicación | `Announcement`, `AnnouncementAudience`, `AnnouncementRead`, `DeviceToken`, `NotificationPreference` |
| Transversal | `Media` (medialibrary), `Activity` (activitylog) |

---

## 5. Hoja de ruta — Fase 1 (MVP para Jakare)

Sprints de ~2 semanas. Cada sprint termina con tests verdes en CI y demo.

### Sprint 0 — Fundaciones
- [ ] Definir nombre (o usar nombre en clave temporal) y crear los 2 repos.
- [ ] `laravel new` (Laravel 13) + Sail con PHP 8.5 + MySQL 8.4 + Mailpit.
- [ ] Filament 5, Shield, Sanctum, Fortify, activitylog, medialibrary, dompdf.
- [ ] Pint + PHPUnit 13 + GitHub Actions (`lint.yml`, `tests.yml`) copiados/adaptados de OpenSciRank.
- [ ] `Dockerfile.production` + `compose.production.yaml` (MariaDB 11.8) como OpenSciRank.
- [ ] `CLAUDE.md` + `business-logic.md` (documento maestro) + `DESIGN.md`.
- [ ] `flutter create` + estructura (features/, core/), Riverpod, go_router, dio, flavors dev/prod.

### Sprint 1 — Organizaciones, usuarios y roles
- [ ] `Organization` + tenancy en Filament y API.
- [ ] `Membership` con roles: owner/admin, presidente, vice, secretario, tesorero, vocal, síndico, instructor, responsable.
- [ ] Mandatos con fecha de inicio/fin (roles de comisión se desactivan solos).
- [ ] Invitaciones por email/link/QR.
- [ ] Feature flags por organización y vocabulario configurable.
- [ ] Tests de aislamiento entre organizaciones.

### Sprint 2 — Estructura académica y alumnos
- [ ] Temporadas, programas (fútbol; luego pádel), grupos, horarios.
- [ ] Alumnos, responsables, familias, ficha médica.
- [ ] Inscripciones (alumno + grupo + temporada), estados (`pendiente`, `activo`, `becado`, `suspendido`, `baja`).
- [ ] Importación desde Excel (para cargar Jakare rápido).

### Sprint 3 — Núcleo financiero
- [ ] Cuentas (banco, caja, billetera) y libro mayor inmutable.
- [ ] Conceptos y tarifario por grupo/temporada con vigencia.
- [ ] Generación automática de cuotas mensuales (comando programado, idempotente).
- [ ] Descuentos (hermanos, convenio) y becas con aprobación.
- [ ] Política de mora configurable (vencimiento, gracia, recargo fijo/%, tope, exoneración).
- [ ] Tests exhaustivos de reglas de dinero.

### Sprint 4 — Cobros, gastos e informes
- [ ] Pagos con imputación a cargos (FIFO o manual), saldo a favor por familia.
- [ ] Recibo PDF automático.
- [ ] Gastos con comprobante, proveedores (ej. dueño de la cancha), categorías, aprobación por umbral.
- [ ] Transferencias entre cuentas.
- [ ] Informes: ingresos/egresos por período, saldo por cuenta, morosidad. Exportar PDF/Excel.

### Sprint 5 — API y notificaciones
- [ ] API v1 (Sanctum): login, organizaciones del usuario, hijos, estado de cuenta, avisos, perfil.
- [ ] Avisos segmentados (organización, programa, grupo, familia) + confirmación de lectura.
- [ ] Firebase: registro de dispositivos, envío por colas, preferencias.
- [ ] Recordatorios automáticos de cuotas (antes/el día/después del vencimiento).
- [ ] Documentación de la API (OpenAPI).

### Sprint 6 — App Flutter MVP
- [ ] Login + selector de organización.
- [ ] "Mis hijos": grupo, horarios, estado de cuenta, historial de pagos, recibos.
- [ ] Avisos y notificaciones push.
- [ ] Build web + Android (prueba interna) + iOS (TestFlight).

### Sprint 7 — Piloto Jakare
- [ ] Carga de datos reales (categorías, jugadores, familias, tarifas, cuentas).
- [ ] Capacitación a comisión y técnicos.
- [ ] Invitación a padres y seguimiento de adopción.
- [ ] Ronda de correcciones.

---

## 6. Fases siguientes (resumen)

- **Fase 2:** comisión, reuniones, actas y resoluciones (PDF, votación, publicación a grupos);
  asistencia desde la app del instructor; eventos/torneos con confirmación; comprobante de
  pago subido por el padre y validado por el tesorero.
- **Fase 3:** rifas, indumentaria, informes avanzados (por grupo/actividad, memoria y balance
  para asamblea), presupuesto.
- **Fase 4 (SaaS comercial):** alta autoservicio de organizaciones, planes y cobro de
  suscripción, landing, pagos online de cuotas (Bancard / Pagopar), SIFEN opcional, WhatsApp.

---

## 7. Calidad, seguridad y operación

- **Tests:** feature tests por módulo; reglas de dinero y aislamiento entre organizaciones son obligatorias.
- **CI:** lint + tests en cada PR (backend); `flutter analyze` + `flutter test` (app).
- **Auditoría:** activitylog en todo lo financiero y en cambios de roles.
- **Datos de menores:** acceso mínimo por rol, ficha médica visible solo para roles autorizados, backups cifrados.
- **Backups:** diarios de base de datos + S3, con restauración probada.
- **Producción:** VPS con Docker (igual que OpenSciRank), HTTPS, dominio propio.
- **Cuentas necesarias:** Firebase, Google Play Console (USD 25 único), Apple Developer (USD 99/año), S3, dominio.

---

## 8. Decisiones pendientes

1. Nombre del producto (y de los repos).
2. Hosting de producción (¿el mismo servidor/proveedor que OpenSciRank?).
3. PHPUnit (como OpenSciRank) o Pest 5.
4. ¿Redis desde el inicio o colas en base de datos (como OpenSciRank)?
5. Datos de Jakare: cantidad de jugadores, categorías, técnicos; cómo se paga el alquiler de la cancha.
