# Plan: Sistema de Gestión de Clubes (SaaS) — Piloto: Club Jakare

> Documento de planificación. Todavía no hay código: aquí definimos funcionalidades,
> lógica de negocio y el orden de construcción.

## 1. Visión

Una plataforma **SaaS multi-club** para clubes de inferiores / academias deportivas.
El primer club (piloto) es **Jakare**. Cada club es un "inquilino" (tenant) con sus
propios datos, usuarios, finanzas y comunicaciones, aislados de los demás clubes.

**Stack (igual que OpenSciRank, repositorio nuevo)**
- **Backend:** Laravel 12 · PHP 8.2+ · MySQL 8.4 (dev) / MariaDB 11.8 (prod) · Laravel Sail (Docker) · colas en base de datos.
- **Panel web (comisión, tesorería, secretaría, admin):** Filament 5 + Livewire 4 + Tailwind 4.
- **Roles y permisos:** Filament Shield (spatie/laravel-permission).
- **Auditoría:** spatie/laravel-activitylog. **PDF** (actas, recibos, balances): barryvdh/laravel-dompdf. **QR:** endroid/qr-code.
- **Archivos:** S3 (aws-sdk). **Auth web:** Fortify.
- **Calidad:** Pint + PHPUnit + GitHub Actions (lint.yml, tests.yml) + Dockerfile/compose de producción.
- **Nuevo respecto a OpenSciRank:** Laravel Sanctum (API para la app) + Firebase Cloud Messaging (push).
- **App:** Flutter — **un solo código para Android, iPhone y web**.
  - Recomendación: la app Flutter para padres, técnicos y comisión "en movimiento"
    (avisos, pagos, asistencia, aprobar); el panel Filament para el trabajo de escritorio
    (actas, balances, carga masiva). Flutter web queda disponible, pero no reemplaza al panel.
- **Documento maestro de negocio:** `business-logic.md` en la raíz del repo nuevo (como en OpenSciRank).

---

## 2. Actores y roles

| Rol | Qué hace | Alcance |
|---|---|---|
| **Superadmin plataforma** | Da de alta clubes, planes, soporte | Todos los clubes |
| **Admin del club** | Configura el club, usuarios, categorías | Su club |
| **Presidente / Vice** | Aprueba resoluciones, gastos grandes, ve todo | Su club |
| **Secretario/a** | Reuniones, actas, resoluciones, comunicados | Su club |
| **Tesorero/a** | Cuotas, cobros, gastos, cuentas, balances | Su club |
| **Vocales / Síndico** | Lectura de actas e informes, auditoría | Su club |
| **Técnico / Coordinador** | Asistencia, convocatorias, avisos a su categoría | Sus categorías |
| **Padre / Tutor** | Ve a sus hijos, paga, recibe avisos, confirma asistencia | Sus hijos |
| **Jugador** (opcional, futuro) | Ve su agenda | Él mismo |

Una misma persona puede tener varios roles (ej.: un papá que es tesorero y además tutor).

---

## 3. Funcionalidades por módulo

### 3.1 Plataforma SaaS
- Alta de club (nombre, logo, colores, moneda, país, zona horaria).
- Planes de suscripción (por cantidad de jugadores / módulos).
- Onboarding: asistente inicial (crear categorías, cuentas, cuota base).
- Invitación de usuarios por email / link / código QR.

### 3.2 Club e institucional
- Datos del club, sedes y canchas (propias o alquiladas).
- Temporadas (ej.: 2026) — todo se agrupa por temporada.
- Documentos del club: estatuto, reglamento interno, archivos.

### 3.3 Comisión directiva
- Cargos: presidente, vicepresidente, secretario, tesorero, pro-tesorero, vocales, síndico.
- Mandatos con fecha de inicio/fin (historial de comisiones).
- Permisos derivados del cargo.

### 3.4 Reuniones, actas y resoluciones
- **Reunión:** convocatoria (fecha, lugar/virtual, orden del día), notificación a miembros.
- **Asistencia** y **quórum** automático.
- **Acta:** redacción, numeración correlativa, estados `borrador → en revisión → aprobada`, PDF.
- **Resoluciones:** numeradas (ej. `RES-2026-014`), vinculadas a un acta, con votación (a favor / en contra / abstención).
- **Publicación selectiva:** una resolución puede comunicarse a todo el club, a una o varias categorías, o quedar interna.
- **Acciones derivadas:** una resolución puede generar un cargo a cobrar (ej. "Cuota de torneo Sub-10: 50.000") o un evento.

### 3.5 Academia (jugadores y categorías)
- **Categorías:** por año de nacimiento (ej. Sub-8, Sub-10…), con técnicos asignados, horarios y cancha.
- **Jugadores:** datos personales, foto, documento, fecha de nacimiento, talle, posición.
- **Ficha médica:** alergias, grupo sanguíneo, contacto de emergencia, apto médico con vencimiento.
- **Tutores / familia:** un tutor puede tener varios hijos; un hijo puede tener varios tutores.
- **Inscripción:** solicitud online del padre → revisión → aprobación → se genera el cargo de inscripción.
- Estados del jugador: `pendiente`, `activo`, `becado`, `suspendido`, `baja`.
- Pase automático de categoría al cambiar de temporada.

### 3.6 Entrenamientos y asistencia
- Horarios recurrentes por categoría (ej. lun/mié/vie 17:00–18:30, 3 veces por semana).
- Generación automática de sesiones de entrenamiento.
- Técnico toma asistencia desde la app (presente / ausente / justificado).
- Suspensión de práctica (lluvia, etc.) → aviso push inmediato a la categoría.
- Reporte de asistencia por jugador/categoría.

### 3.7 Eventos, torneos y partidos
- Evento: torneo, amistoso, festival, reunión de padres, cena.
- Convocatoria de jugadores; el padre **confirma / rechaza** asistencia.
- Costo asociado opcional (genera cargo a las familias convocadas).
- Resultados y fixture (futuro).

### 3.8 Indumentaria
- Campaña de indumentaria (camiseta, short, buzo…), precio, fecha límite.
- Pedido por jugador con talle; genera cargo.
- Estado de entrega.

### 3.9 Finanzas (núcleo del sistema)
- **Cuentas:** banco(s), caja efectivo, billetera digital. Cada una con saldo.
- **Conceptos de cobro:** inscripción, cuota mensual, torneo, indumentaria, rifa, otros.
- **Cuenta corriente por familia/jugador:**
  - *Cargos* (lo que se debe) y *pagos* (lo que se abonó).
  - Generación automática de la cuota mensual el día X de cada mes.
  - Descuentos (hermanos, becas parciales/totales), recargos por mora (opcional).
- **Cobros:**
  - Registro manual por el tesorero (efectivo / transferencia).
  - El padre sube el **comprobante de transferencia** desde la app → el tesorero lo valida.
  - Pasarela de pago online (fase posterior; según país: Bancard/Pagopar, Mercado Pago, etc.).
  - Recibo automático (PDF) enviado al padre.
- **Gastos / egresos:**
  - Alquiler de cancha (gasto recurrente a proveedor), árbitros, materiales, transporte, etc.
  - Cada gasto: cuenta de salida, categoría de gasto, proveedor, comprobante adjunto.
  - Aprobación según monto (ej.: > X requiere presidente + tesorero).
- **Transferencias entre cuentas** (ej. caja → banco).
- **Proveedores** (dueño de la cancha, tienda de indumentaria…).
- **Presupuesto** anual por rubro (fase posterior).
- **Conciliación bancaria** (fase posterior).

### 3.10 Actividades de recaudación (rifas, eventos)
- **Rifa:** cantidad de números, precio, premios, fecha de sorteo.
- Talonarios asignados a familias/jugadores; registro de números vendidos.
- Rendición: cada familia rinde lo vendido → ingreso a cuenta.
- Sorteo y publicación de ganadores.
- Resultado neto de la actividad (ingresos − costos de premios).
- Otras actividades: cantina, pollada, festival (ingresos y gastos agrupados por actividad).

### 3.11 Informes
- Balance / estado de ingresos y egresos por período (mes, trimestre, temporada).
- Flujo de caja y saldo por cuenta.
- Morosidad: quién debe, cuánto y desde cuándo.
- Resultado por categoría y por actividad (ej. rifa).
- Informe para asamblea (memoria y balance) en PDF.
- Exportación a Excel / PDF.

### 3.12 Comunicación y notificaciones
- **Avisos** segmentados: todo el club, una categoría, varias, un evento, una familia.
- Tipos: general, torneo, indumentaria, pago, suspensión de práctica, resolución.
- **Push** (FCM) + email; confirmación de lectura ("visto por 34 de 40 familias").
- Recordatorios automáticos: cuota por vencer, cuota vencida, evento mañana.
- Calendario del club/categoría.
- Encuestas simples (ej. "¿Qué día prefieren para la reunión?").
- Preferencias de notificación por usuario.

### 3.13 Transversales
- Auditoría: quién creó/modificó/anuló cada registro (clave en finanzas).
- Adjuntos en cualquier entidad.
- Búsqueda global.
- Multi-idioma (español primero).

---

## 4. Reglas de negocio clave

1. **Aislamiento por club:** todo registro pertenece a un `club_id`; nadie ve datos de otro club.
2. **Permisos por alcance:** técnico → solo sus categorías; padre → solo sus hijos y sus cargos.
3. **Finanzas inmutables:** un movimiento no se borra; se **anula** con un contra-movimiento y queda en auditoría.
4. **Saldo de cuenta = suma de sus movimientos** (no se edita a mano).
5. **Imputación de pagos:** un pago se aplica a uno o varios cargos (primero los más antiguos, o elección manual).
6. **Cuota mensual automática** para jugadores `activo`; no para `becado` total ni `baja`.
7. **Descuento por hermanos** configurable (ej. 2º hijo −20 %, 3º −50 %).
8. **Comprobante subido por padre** queda `pendiente` hasta que el tesorero lo aprueba; recién ahí impacta en la cuenta.
9. **Gasto > umbral** requiere doble aprobación.
10. **Acta aprobada** no se edita; se corrige con una nueva resolución.
11. **Resolución publicada a categoría** → notificación automática a los tutores de esa categoría.
12. **Categoría por año de nacimiento**, recalculada al abrir una nueva temporada.
13. **Rifa:** un número no puede venderse dos veces; la familia rinde contra los números asignados.

---

## 5. Modelo de datos (entidades principales)

```
Club ─┬─ Temporada, ConfiguracionMora
      ├─ Disciplina ── Categoria
      ├─ Tarifa (concepto + categoría + temporada), ReglaDescuento, Beca
      ├─ Familia ── Jugador ── Inscripcion (categoría + temporada)
      ├─ Usuario ── RolEnClub (cargo, alcance)
      ├─ Categoria ─┬─ Horario ── SesionEntrenamiento ── Asistencia
      │             └─ TecnicoCategoria
      ├─ Jugador ── TutorJugador ── Tutor(Usuario)
      │     └─ FichaMedica, Inscripcion
      ├─ Reunion ─┬─ AsistenciaReunion
      │           └─ Acta ── Resolucion ── Votacion
      ├─ Evento ── Convocatoria (jugador, confirmación)
      ├─ Cuenta (banco/caja) ── Movimiento
      ├─ ConceptoCobro ── Cargo ── Imputacion ── Pago ── Comprobante
      ├─ Gasto ── Proveedor, CategoriaGasto, Aprobacion
      ├─ Actividad (rifa/evento) ── Talonario ── NumeroRifa
      ├─ CampañaIndumentaria ── PedidoIndumentaria
      ├─ Aviso ── Destinatario ── Lectura
      └─ Adjunto, Auditoria
```

---

## 6. Hoja de ruta paso a paso

### Fase 0 — Definiciones (ahora)
- [ ] Validar este listado de funcionalidades.
- [x] Responder preguntas abiertas (sección 7).
- [ ] Decidir repositorio y estructura (API + panel + app).

### Fase 1 — MVP para Jakare (lo mínimo para usarlo de verdad)
1. Repo nuevo con base OpenSciRank (Laravel 12, Filament 5, Sail) + Sanctum + multi-club + roles/permisos.
2. Club, temporada, disciplinas, categorías, horarios.
3. Jugadores, tutores, inscripción (carga por el club).
4. Avisos segmentados + push FCM (app Flutter para padres: login, mis hijos, avisos).
5. Finanzas básicas: cuentas, tarifario por categoría, cuota mensual automática, descuentos/becas, mora, pagos, gastos (incl. alquiler de cancha).
6. Informe básico: ingresos/egresos por mes y saldo por cuenta; lista de morosos.

### Fase 2 — Gestión institucional y deportiva
7. Comisión y cargos; reuniones, actas y resoluciones con PDF.
8. Resoluciones publicadas a categorías → notificación.
9. Entrenamientos, asistencia desde app del técnico, suspensión con aviso.
10. Eventos/torneos con confirmación de asistencia de padres.
11. Comprobante de pago subido por el padre + validación del tesorero + recibo PDF.

### Fase 3 — Recaudación y reportes avanzados
12. Rifas (talonarios, rendición, sorteo).
13. Indumentaria (campañas, pedidos, talles).
14. Informes avanzados (por categoría, por actividad, memoria y balance para asamblea, Excel/PDF).
15. Aprobación de gastos por umbral, presupuesto.

### Fase 4 — SaaS comercial
16. Alta autoservicio de clubes, planes y cobro de suscripción.
17. Personalización de marca por club, landing page.
18. Pasarela de pagos online para cuotas.
19. Conciliación bancaria, WhatsApp, estadísticas.

---

## 7. Decisiones tomadas

| Tema | Decisión |
|---|---|
| País / moneda | **Paraguay, Guaraníes (PYG)** |
| Cuenta corriente | **Por jugador, con vista consolidada por familia** |
| Cuota | **Distinta por categoría**; preparado para **varias disciplinas** (fútbol hoy, pádel u otras después) |
| Descuentos / becas | **Sí, configurables** |
| Recargo por mora | **Configurable por club** |
| Comisión | **Cada cargo es un rol** con permisos propios (presidente, tesorero, secretario…) |
| App | **Flutter** (Android, iPhone, web) + panel **Filament** |
| Repositorio | **Nuevo**, con la misma configuración técnica que OpenSciRank |

## 8. Lógica de negocio derivada de las decisiones

### 8.1 Guaraníes
- Montos como **enteros** (el guaraní no usa decimales): `150000` → se muestra `₲ 150.000`.
- Redondeos de descuentos y recargos al guaraní entero (configurable: a 500 o 1.000).
- Zona horaria `America/Asuncion`, idioma español (Paraguay).
- Pagos online (fase 4): **Bancard** (vPOS / QR) y/o **Pagopar**. Transferencias y giros (Tigo Money, Personal) como pago manual con comprobante.
- Comprobantes: el club emite **recibos internos**. La factura electrónica (SIFEN) queda como opción futura.

### 8.2 Disciplinas y categorías
```
Club → Disciplina (Fútbol, Pádel…) → Categoría/Grupo (Sub-8, Sub-10… / Pádel Inicial…) → Horarios
```
- **Inscripción** = jugador + categoría + temporada. Un jugador puede tener **varias inscripciones** (ej.: fútbol y pádel).
- Cada inscripción genera sus propios cargos.
- Criterio de categoría configurable por disciplina: por año de nacimiento (fútbol) o por nivel (pádel).

### 8.3 Tarifario
- **Tarifa** = concepto (inscripción, cuota mensual, torneo…) + categoría + temporada + monto + vigencia.
- Un cambio de tarifa no altera cargos ya emitidos; aplica desde la fecha de vigencia.
- La tarifa se puede aprobar mediante resolución de comisión (queda vinculada).

### 8.4 Cuenta corriente por jugador y familia
- Todo **cargo** pertenece a un **jugador** (y a su inscripción).
- La **familia** agrupa jugadores y tutores; su saldo = suma de saldos de sus jugadores.
- Un **pago** lo hace un tutor/familia y se **imputa** a cargos de uno o varios hijos
  (por defecto, los más antiguos primero; el tesorero puede elegir).
- Saldo a favor (pagó de más): queda como crédito de la familia y se aplica al próximo cargo.

### 8.5 Descuentos y becas (configurables)
- **Regla de descuento:** tipo (hermanos, beca, convenio, pronto pago, otro), porcentaje **o** monto fijo, conceptos a los que aplica, vigencia.
- **Hermanos:** posición del hijo (2º, 3º…) → porcentaje, según las inscripciones activas de la familia.
- **Beca:** asignada a un jugador (parcial % o total), con motivo, vigencia y **aprobación** (resolución o presidente + tesorero).
- Orden de aplicación configurable; tope: el cargo nunca queda negativo.
- El cargo guarda el **detalle**: monto base, descuentos aplicados y monto final (trazabilidad).

### 8.6 Mora (configurable)
- Parámetros por club (y opcionalmente por concepto): día de vencimiento, días de gracia, tipo de recargo (fijo / %), frecuencia (una vez / por mes), tope.
- Se puede desactivar por completo, o exonerar un cargo puntual (con motivo y auditoría).
- Recordatorios automáticos: X días antes del vencimiento, el día del vencimiento y cada N días de atraso.
- Opcional: aviso al técnico o bloqueo de inscripción a torneos si hay deuda > N meses (configurable, nunca automático sin decisión del club).

### 8.7 Roles de la comisión
- Cada cargo (presidente, vice, secretario, pro-secretario, tesorero, pro-tesorero, vocal, síndico) es un **rol** con permisos editables (Filament Shield).
- La asignación tiene **mandato** (desde / hasta): al vencer, el rol se desactiva solo.
- Aprobaciones con reglas por rol: ej. gasto > umbral → tesorero **y** presidente; beca → presidente.
- Una persona puede ser a la vez tutor y miembro de la comisión (la app muestra ambos perfiles).

## 9. Próximos pasos

1. Crear el repositorio nuevo (nombre sugerido: `clubes` o `jakare`) con la base de OpenSciRank:
   Laravel 12 + Filament 5 + Sail + Pint + PHPUnit + GitHub Actions + Docker de producción.
2. Escribir `business-logic.md` a partir de este plan.
3. Migraciones y modelos de la **Fase 1**: clubes, disciplinas, categorías, jugadores, familias, tarifas, cargos, pagos, cuentas, gastos.
4. Panel Filament para la tesorería y la secretaría.
5. API con Sanctum y app Flutter para padres (login, mis hijos, estado de cuenta, avisos push).

### Preguntas que quedan
- Nombre del producto SaaS (y del repo nuevo).
- Cantidad aproximada de jugadores, categorías y técnicos de Jakare.
- ¿Los técnicos cobran del club? (si es así, agregamos "pagos a técnicos" como gasto recurrente).
- ¿El alquiler de cancha es mensual fijo o por hora/uso?
