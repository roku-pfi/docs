# Figuras del E90 que cambiaron (2026-10-23)

Archivos nuevos en `docs/diagrams/png/` (diagramas, vector en `svg/`) y
`docs/screenshots/` (capturas del sistema en AWS). No se editó el informe.

| Figura del E90 | Reemplazo | Qué cambió |
|---|---|---|
| 3.16 Flujo de login (1 de 2) | `diagrams/png/01-login-sequence.png` | `client_id` y organización; política base y modo del motor; el PDP decide sólo con Valkey (sin Postgres); bloqueo por IP y pausa de cuenta; la decisión se registra en el diario (Valkey) en lugar del outbox; paso adicional sólo con passkey, `SECOND_FACTOR_MISSING`, confirmación del paso adicional (el perfil aprende recién ahí). 37 pasos. |
| 3.17 Flujo de login (2 de 2) | `diagrams/png/02-session-authorization.png` | Igual en lo esencial; la aplicación cliente es Roku Bank. Pasos 38–47. |
| 3.18 Arquitectura desplegada | `diagrams/png/03-architecture.png` | De k3d a AWS: CloudFront (3 hosts), ALB, EKS 1.35 Graviton, RDS PostgreSQL, ElastiCache Valkey, Lambdas atacantes en 3 regiones, ECR + GitHub Actions; dos aplicaciones cliente (Roku Bank, Kaze Labs); relay del diario en lugar del outbox. |
| 3.19 Esquema `rba_idp` (directorio) | `diagrams/png/04-schema-idp-dir.png` | Tablas `tenants` y `tenant_domains`; `tenant_id` en usuarios, grupos, aplicaciones y roles; `client_id`; `last_login_at`. |
| 3.20 Esquema `rba_idp` (sesión) | `diagrams/png/05-schema-idp-runtime.png` | Tabla `activation_codes`; `sessions.enroll_until`; `mfa_challenges.denied_reason`. |
| 3.21 Esquemas `rba_decision`, `rba_profile`, `rba_audit` | `diagrams/png/06-schema-ops.png` | Sin `outbox`; nueva `tenant_policies`; `decisions` suma `tenant_id`, `enforced_action`, `engine_mode`, `simulated`, `login`, `step_up_passed_at`; `tenant_id` en historial y archivo. |
| §3.6.4 Perfil en Redis (sin figura) | `diagrams/png/07-valkey-keys.png` (nueva) | Valkey ya no es sólo el perfil: población, diario de decisiones (clave + stream) y contadores de fallos/IPs bloqueadas. |
| 3.8 Login: credenciales, paso adicional, bloqueo | `screenshots/01`–`04` | Tres estilos de login (marca del banco, RokuSec claro, consola oscura) y el paso adicional con passkey. El bloqueo no se capturó: con el motor en monitoreo no se aplica. |
| 3.9 / 3.10 Aplicación demo y 403 | `screenshots/05`, `07`–`09` | Roku Bank nuevo; los roles se muestran en Kaze Labs (viewer sin "Crear proyecto", 403 en Configuración, admin con acceso). |
| 3.11 Rastro de intentos | `screenshots/11`, `12` | Consola Carbon: lista de Eventos y detalle de un intento desde Tokio con sus razones. |
| 3.12 Bandas y modo monitoreo | `screenshots/13` | Página Política (política base, modo del motor, bandas, acciones). |
| 3.13–3.15 Usuarios, roles, grupos | `screenshots/14`–`16` | Consola Carbon. Extra: `10` Resumen, `17` Aplicaciones, `18` Efectividad. |

- **Las figuras 3.8–3.15 del E90 son wireframes de diseño:** siguen siendo
  válidos como etapa de diseño. Las capturas muestran el resultado construido;
  pueden ir junto a ellos o reemplazarlos.
- **En las capturas, la IP real del autor se reemplazó** por una dirección de
  documentación (`2001:db8::4f2a`, RFC 3849). El resto son datos de demo.
- **Sin cambio:** 2.1–2.3 (estado del arte), 3.1–3.7 (encuesta) y A.1
  (cronograma tentativo; si se quiere, puede rehacerse con las fechas reales).
- Los diagramas usan la tipografía y el azul de la marca actual (IBM Plex, `#1466E5`).
