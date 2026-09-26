# Plan del Proyecto — SaaS para Academias, Clubes y Escuelas

> **Nombre:** pendiente (`{producto}`). Finalistas: Crecemy, Cantemy, Nidemy, Cluppy, Crecy, Retoño, Acompaño.
> **Piloto:** Club Jakare (fútbol infantil, Paraguay). Moneda ₲ · `America/Asuncion` · español.
> Funcionalidades y lógica de negocio en detalle: `docs/PLAN_JAKARE.md`.

---

## 1. Alcance
SaaS multi-organización para academias, clubes, escuelas de formación y comisiones de padres (ACE).
- **Roles:** comisión (presidente, vice, secretario, tesorero, vocales, síndico — un rol por cargo, con mandato), admin, instructor, padre/tutor, alumno adulto.
- **Núcleo:** alumnos, familias, programas, grupos, horarios, asistencia, inscripciones, tarifas, cuotas, becas, mora, cobros, gastos, cuentas, avisos push, eventos, informes.
- **Módulos opcionales (feature flags):** comisión/actas/resoluciones, rifas, indumentaria, torneos, evaluaciones, SIFEN.
- **Vocabulario configurable** (Grupo = "Categoría" / "Nivel" / "Curso").
- **Fuera de alcance:** gestión académica de colegios formales.

## 2. Reglas de negocio clave
- Aislamiento total por organización; permisos por alcance (instructor → sus grupos; padre → sus hijos).
- Organización → Programa → Grupo → Horarios. Inscripción = alumno + grupo + temporada (varias por alumno).
- Tarifario por grupo/temporada con vigencia; cambios no alteran cargos emitidos.
- Cuenta corriente por jugador, vista consolidada por familia; pagos imputables a varios hijos; saldo a favor.
- Descuentos (hermanos, beca, convenio, pronto pago; % o fijo) y becas con aprobación; el cargo guarda base, ajustes y final.
- Mora configurable (vencimiento, gracia, recargo fijo/%, tope, exoneración con motivo) + recordatorios.
- Libro mayor inmutable (anulación con contra-movimiento); saldo = suma de movimientos.
- Comprobante del padre pendiente hasta validación del tesorero; gastos > umbral con doble aprobación.
- Mandatos que vencen solos; resoluciones publicadas a un grupo notifican y pueden generar cargos.

## 3. Stack (versiones verificadas 26/09/2026)
**Backend/panel:** PHP 8.5 · Laravel 13 · Filament 5.9 · Livewire 4 · Tailwind 4.3 · Vite 8 · Sanctum 4 · Fortify 1.40 · Filament Shield 4.3 + spatie/permission 8 · activitylog 5 · medialibrary 11 + S3 · dompdf 3 · kreait/laravel-firebase 7 · Pint · Sail.
**Datos e infraestructura (decididos):**
- **MariaDB 11.8 LTS** en dev, CI y prod (`DB_CONNECTION=mariadb`, `utf8mb4_uca1400_ai_ci`).
- **Redis 8 + Horizon 5** desde el día 1: colas y caché/locks. Sesiones en MariaDB.
- **Pest 5** (sobre PHPUnit 13) con arch tests.
- Producción: Docker Compose en VPS (como OpenSciRank).
**App:** Flutter 3.47 / Dart 3.13 · Riverpod 3 · go_router · dio · firebase_messaging · flutter_secure_storage.

## 4. Arquitectura
- Dos repos: `{producto}-api` (Laravel: API `/api/v1` + panel Filament `/admin` + Horizon + scheduler) y `{producto}-app` (Flutter).
- Tenancy en una BD: `organization_id` + trait `BelongsToOrganization` (global scope); tenancy nativo de Filament; header `X-Organization` en la API; usuario con roles distintos por organización.
- Dinero en enteros (₲) con value object `Money`.
- Regla: mismo motor y versión en todos los entornos; se actualizan juntos.
- Entidades: Organization, Membership, Season, Program, Group, Schedule, Session, Attendance, Student, Guardian, Family, Enrollment, Account, LedgerEntry, FeeConcept, Tariff, Charge, ChargeAdjustment, DiscountRule, Scholarship, LateFeePolicy, Payment, PaymentAllocation, PaymentProof, Expense, Supplier, Approval, Announcement, DeviceToken (+ módulos opcionales: Meeting, Minute, Resolution, Event, Fundraiser, ApparelCampaign).

## 5. Hoja de ruta
### Fase 1 — MVP Jakare (sprints de ~2 semanas)
0. **Fundaciones:** repos; Laravel 13 + Sail `--with=mariadb,redis,mailpit` (PHP 8.5); Filament, Shield, Sanctum, Fortify, Horizon, activitylog, medialibrary, dompdf; Pest 5 + arch test de tenancy; CI con MariaDB 11.8 y Redis 8 reales + Pint; compose de producción (app, MariaDB, Redis AOF, Horizon, scheduler); `CLAUDE.md`, `business-logic.md`, `DESIGN.md`; Flutter base.
1. **Organizaciones y roles:** tenancy, membresías, invitaciones (email/link/QR), cargos con mandato, feature flags, vocabulario, tests de aislamiento.
2. **Académico:** temporadas, programas, grupos, horarios, alumnos, tutores, familias, ficha médica, inscripciones con estados, importación Excel.
3. **Finanzas:** cuentas y libro mayor, tarifario, cuota mensual automática (lock Redis + índice único), descuentos, becas, mora.
4. **Cobros y gastos:** pagos con imputación y saldo a favor, recibo PDF, gastos con aprobación, proveedores, transferencias, informes (balance, saldos, morosos; PDF/Excel).
5. **API y avisos:** API v1 (OpenAPI), avisos segmentados con lectura, push por lotes vía Horizon, recordatorios de cuotas.
6. **App Flutter:** login, selector de organización, mis hijos, estado de cuenta, recibos, avisos; web + Android (interno) + iOS (TestFlight).
7. **Piloto Jakare:** carga de datos reales, capacitación, invitación a padres, correcciones.

### Fase 2 — Institucional y deportiva
Comisión, reuniones, actas y resoluciones (PDF, votación, publicación a grupos); asistencia desde app del instructor; suspensión de prácticas; eventos/torneos con confirmación; comprobante de pago del padre + validación.

### Fase 3 — Recaudación e informes
Rifas (talonarios, rendición, sorteo), indumentaria, informes por grupo/actividad, memoria y balance para asamblea, presupuesto.

### Fase 4 — SaaS comercial
Alta autoservicio, planes y suscripciones, landing, pagos online (Bancard / Pagopar), SIFEN, WhatsApp, conciliación bancaria.

## 6. Calidad y operación
- Tests obligatorios: reglas de dinero, aislamiento entre organizaciones, arch tests (Pest).
- CI: lint + tests por PR (API); `flutter analyze` + `flutter test` (app).
- Auditoría en finanzas y roles; datos de menores con acceso mínimo; ficha médica restringida.
- Horizon con alerta de jobs fallidos; backups diarios cifrados de MariaDB con restauración probada; Redis con AOF.
- Cuentas: Firebase, Google Play (USD 25), Apple Developer (USD 99/año), S3, dominio.

## 7. Decisiones
✅ Paraguay/₲ · ✅ cuenta por jugador + familia · ✅ cuota por grupo, multi-disciplina · ✅ descuentos/becas/mora configurables · ✅ un rol por cargo con mandato · ✅ Laravel 13 + Filament 5 + Flutter 3.47 · ✅ MariaDB 11.8 en todos lados · ✅ Redis + Horizon desde el inicio · ✅ Pest 5 · ✅ dos repos nuevos.
⏳ Nombre del producto · ⏳ hosting (¿mismo servidor que OpenSciRank?) · ⏳ datos de Jakare (jugadores, categorías, técnicos, pago de cancha).

## 8. Próximos pasos
1. ~~Guardar este plan en el repo.~~ ✅
2. Elegir nombre o usar uno provisorio.
3. Crear los dos repos en GitHub (usuario, o dar acceso).
4. Arrancar Sprint 0.
