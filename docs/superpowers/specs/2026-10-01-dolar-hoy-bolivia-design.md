# Diseño: App Dólar Hoy Bolivia — cotizaciones y alertas de precio

- **Fecha:** 2026-10-01
- **Estado:** aprobado por el usuario (diseño en chat)
- **Plataforma v1:** solo Android
- **Base:** plantilla Flutter Admin Panel UI (paquete `admin`, tema oscuro)

## 1. Objetivo

App móvil que muestre el precio del dólar en Bolivia en dos mercados (oficial BCB y
paralelo/blue), permita al usuario definir un precio de interés y le envíe una
notificación cuando el dólar alcance ese precio, incluso con la app cerrada.

## 2. Alcance

### En alcance (v1)

- Pantalla de cotizaciones: oficial y paralelo, con variación y última actualización.
- Pantalla de alertas: crear, editar, pausar/activar y eliminar múltiples alertas.
- Revisión periódica en segundo plano (~15 min) con `workmanager`.
- Notificación local cuando se cumple una alerta.
- Persistencia local de alertas con `shared_preferences`.
- Estilo visual coherente con la plantilla actual (oscuro, Poppins).

### Fuera de alcance (v1)

- iOS y web.
- Backend / push remoto (FCM).
- Gráfico de histórico (posible futuro: migrar persistencia a SQLite).
- Conversor de divisas, cuentas de usuario, autenticación.

## 3. Fuente de datos

API pública **DolarApi.com Bolivia**, sin autenticación:

| Endpoint | Contenido |
|---|---|
| `GET https://bo.dolarapi.com/v1/dolares/oficial` | Tipo de cambio oficial (BCB) |
| `GET https://bo.dolarapi.com/v1/dolares/blue` | Dólar paralelo (blue) |

Respuesta (ambos): `{ compra, venta, casa, nombre, moneda, fechaActualizacion }`.

Criterios del servicio:

- Timeout de 10 s; reintentos: 1 reintento inmediato en falla.
- Cache corto en memoria (60 s) para evitar llamadas duplicadas al abrir pestañas.
- La API no provee histórico: para calcular la "variación vs. ayer" se guarda un
  snapshot diario (`fecha → {compra, venta}`) en `shared_preferences` (clave
  `daily_snapshots`, máximo 30 días). Si no hay snapshot del día anterior, la
  variación no se muestra.
- Si fallan ambas llamadas, la UI muestra el último valor conocido con la marca
  "desactualizado" y un botón de reintentar.

## 4. Arquitectura

```
lib/
├── main.dart                    # init: workmanager, notificaciones, Provider
├── constants.dart               # colores (existente)
├── responsive.dart              # breakpoints (existente)
├── models/
│   ├── exchange_rate.dart       # Rate: compra, venta, fechaActualizacion, tipo
│   └── price_alert.dart         # PriceAlert (ver §5)
├── services/
│   ├── dollar_api.dart          # HTTP a dolarapi, timeout, cache 60 s
│   ├── alert_store.dart         # CRUD de alertas en shared_preferences (JSON)
│   └── background_service.dart  # tarea periódica: fetch → evaluar → notificar
├── providers/
│   ├── rates_provider.dart      # estado de cotizaciones + refresh + snapshots diarios
│   └── alerts_provider.dart     # CRUD de alertas + disparo/pausa
└── screens/
    ├── main/main_screen.dart    # shell con BottomNavigationBar
    ├── rates/                   # tarjetas oficial/paralelo + variación
    └── alerts/                  # lista de alertas + diálogo de creación
```

### Dependencias nuevas

`provider`, `http`, `shared_preferences`, `workmanager`,
`flutter_local_notifications`, `intl`.

### Navegación

- Móvil (<850 px): `BottomNavigationBar` con dos pestañas (Cotizaciones, Alertas).
- Tablet/desktop (≥850 px): se conserva el `SideMenu` lateral con las mismas dos
  secciones, usando la clase `Responsive` existente.
- `Header` simplificado: título de sección + "última actualización hace X min";
  se eliminan el buscador y el perfil de demo.

## 5. Modelo de alerta

```dart
class PriceAlert {
  String id;            // UUID generado al crear
  double target;        // precio objetivo en Bs
  bool triggerOnRise;   // true = "sube hasta", false = "baja hasta"
  bool isParallel;      // true = paralelo/blue, false = oficial
  bool enabled;         // activa / pausada
  DateTime createdAt;
}
```

Reglas de negocio:

- Al crear una alerta se solicita el permiso de notificaciones si no fue concedido
  (Android 13+).
- Disparo: `triggerOnRise && venta >= target` → cumplida;
  `!triggerOnRise && venta <= target` → cumplida. Se compara contra `venta`.
- Al dispararse: la alerta pasa a `enabled = false` (pausada) y se emite una
  notificación local. El usuario decide reactivarla.
- Todas las alertas se serializan a JSON en `shared_preferences` bajo la clave
  `price_alerts`.

## 6. Segundo plano y notificaciones

### Workmanager

- `registerPeriodicTask` con `flexInterval` de 15 min (mínimo en Doze), registrado
  una sola vez en `main()` con `isPeriodic: true` y nombre fijo
  (`alert_check_task`) para evitar duplicados.
- Flujo de la tarea: consultar ambos endpoints → evaluar alertas activas →
  persistir cambios → notificar si corresponde → terminar.
- Sin conectividad: terminar limpio; el próximo ciclo reintenta.
- Registrar `RECEIVE_BOOT_COMPLETED` en el `AndroidManifest.xml` para persistir
  entre reinicios.

### Notificaciones (`flutter_local_notifications`)

- Canal `alertas_dolar`, importancia `Importance.high`, creado en el arranque.
- Texto: *"💵 Dólar paralelo alcanzó Bs 13,00 (ahora Bs 13,10)"* (o "oficial").
- Tocar la notificación abre la app directamente en la pestaña de Alertas.
- Permiso `POST_NOTIFICATIONS` solicitado en runtime al crear la primera alerta.

### Permisos Android

- `POST_NOTIFICATIONS` (Android 13+)
- `RECEIVE_BOOT_COMPLETED`

## 7. Pantallas

### Cotizaciones

- Tarjeta **Oficial (BCB)**: compra/venta, variación vs. ayer (flecha verde/roja),
  fecha de vigencia.
- Tarjeta **Paralelo (blue)**: compra/venta, variación, badge con brecha % respecto
  al oficial.
- Acciones: pull-to-refresh y botón de refrescar.
- Estados: cargando (skeleton), error (reintentar + caché si existe).

### Alertas

- Lista de tarjetas: "Paralelo ≥ Bs 13,00" con switch activa/pausada y eliminar.
- FAB "+" → bottom sheet: campo de precio, dirección (sube hasta / baja hasta),
  mercado (paralelo / oficial), guardar.
- Lista vacía: mensaje con CTA para crear la primera alerta.

## 8. Manejo de errores

| Caso | Comportamiento |
|---|---|
| API caída con caché disponible | Mostrar último valor + "desactualizado" |
| API caída sin caché | Estado de error + reintentar |
| Sin conectividad en background | La tarea termina; reintenta en el próximo ciclo |
| Permiso de notificación denegado | La alerta se guarda igual; aviso en UI de que no llegará notificación |
| Reloj del teléfono alterado | El disparo depende solo del precio actual, no de la hora |

## 9. Testing

- **Unit:** parser de respuestas de dolarapi; lógica de evaluación de alertas
  (sube/baja × paralelo/oficial); CRUD de `alert_store`.
- **Widget:** pantalla de cotizaciones con datos inyectados; pantalla de alertas
  vacía y con alertas de ejemplo.
- **Manual:** `flutter analyze` + `flutter test`; en dispositivo/emulador, crear
  una alerta a un precio ya superado y verificar notificación en ≤15 min con la
  app cerrada.

## 10. Criterios de aceptación

1. La app muestra oficial y paralelo con su variación al abrirla (o estado de error).
2. Se pueden crear varias alertas con dirección y mercado; sobreviven al reinicio.
3. Con la app cerrada, se recibe una notificación al cumplirse una alerta.
4. La alerta cumplida queda pausada automáticamente.
5. `flutter analyze` y `flutter test` pasan sin errores.
