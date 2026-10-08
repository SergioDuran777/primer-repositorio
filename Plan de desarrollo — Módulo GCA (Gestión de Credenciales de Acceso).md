Plan de desarrollo — Módulo GCA (Gestión de Credenciales de Acceso)

> Estado: propuesta de diseño cerrada. Pendiente de aprobación para implementar.
> Proyecto: Prisma / SIM Industrial (repo `Azure_Helix`).
> Rama sugerida de trabajo: `feat/gca-credenciales`.

---

## 1. Contexto y objetivo

SIM Industrial paga múltiples servicios y plataformas (dominios, software, suscripciones,
correos corporativos, hosting/infra, portales externos). Hoy esas credenciales están dispersas:
en la cabeza de una o dos personas, en notas sueltas o en el `.env` de algún proyecto.

Si esa persona no está disponible, o un servicio vence sin que nadie lo note, la empresa pierde
acceso o paga de más. **GCA centraliza esto en un solo lugar controlado**, con contraseñas
cifradas y acceso auditado.

---

## 2. Alcance

**Incluye (Fase 1):**
- Registrar y administrar credenciales (servicio, proveedor, usuario, contraseña, responsable,
  fecha de vencimiento, notas).
- Guardar la contraseña **cifrada en reposo** (AES-256-GCM).
- **Revelar contraseñas por página/filtro** con una **sesión temporal de 60 s**, autorizada con
  un **segundo factor TOTP dedicado (Google Authenticator distinto al del login)**, obligatorio y
  no desactivable por el usuario.
- Alertas de vencimiento/renovación.
- Bitácora de quién **creó, vio, editó, reveló y copió** cada credencial.

**No incluye:**
- Las credenciales técnicas internas de Prisma (`JWT_SECRET`, `SUPABASE_KEY`, etc.): siguen en
  `.env` / Railway, fuera de este módulo.

---

## 3. Roles con acceso

```
Administrador TI (Luis) · SuperAdmin · Gerente · Dir. Administrativo
```

- Son exactamente los `FULL_ACCESS_ROLES` actuales (`constants/accessRoles.js`, backend y
  frontend en espejo). `authorizeRoles` y `checkModuleAccess` ya los dejan pasar; el resto no ve
  el ítem de menú ni la ruta.
- Todos estos roles ya tienen **MFA de login obligatorio** (`MFA_REQUIRED_ROLES`). GCA añade su
  propio segundo factor por encima.
- Ningún otro rol debe ver este módulo: es información crítica de la empresa, no de un área
  operativa específica.

---

## 4. Flujo completo, paso a paso

### A. Enrolamiento del 2º factor GCA (una vez por usuario)
1. El usuario autorizado ingresa a `/gca`; el backend detecta que no tiene el factor GCA activo.
2. Se le exige escanear un **QR** con etiqueta `PRISMA - GCA (Credenciales)` — entrada separada en
   Google Authenticator, **distinta** a la del login (`PRISMA - SIM Industrial`).
3. Confirma un primer código válido → queda habilitado.
4. **No puede desactivarlo.** Si pierde el dispositivo, un `Administrador TI` / `SuperAdmin`
   regenera el secreto (con registro en auditoría).

### B. Registrar una credencial
1. En `/gca` → "Nueva credencial".
2. Completa servicio, proveedor, usuario, contraseña, responsable, fecha de vencimiento y notas.
3. El backend **cifra** la contraseña (IV aleatorio + `authTag`, AAD = id) y guarda solo el
   ciphertext. Nunca el texto plano.
4. Evento `CREATED` en la bitácora.

### C. Consultar (sin ver contraseña)
1. El listado y el detalle muestran **metadatos**: nombre, categoría, proveedor, usuario,
   responsable y vencimiento. Las contraseñas se ven como puntos.
2. Evento `VIEWED`.

### D. Modo revelado (por página/filtro, 60 s)
1. En la página filtrada, botón **"Ver contraseñas"**.
2. Se abre un **modal único** que pide el **código del authenticator GCA**.
3. El backend valida el código y emite una **sesión de revelado de 60 s**
   (`gca_reveal_session`, atada al usuario). Queda registro `REVEALED { scope, count, ip }`.
4. El frontend descifra y muestra **las contraseñas de esa página/filtro**, con un
   **contador invisible** en marcha (no se muestra cuenta atrás).
5. A los **60 s** las contraseñas se ocultan solas y el token del servidor expira → hay que
   **volver a autenticar** con el código para verlas de nuevo.
6. Mientras la sesión siga viva, no se vuelve a pedir el código (esa es la agilidad: no
   autenticar contraseña por contraseña).
7. El botón "Copiar" cuenta como acceso y se audita (`COPIED`).

### E. Vencimientos
1. Un cron diario revisa `fecha_vencimiento`.
2. Avisa por email/push al responsable y a los roles admin a 30/15/7/1 días, sin repetir el mismo
   aviso.

### F. Auditoría
1. Cada acción queda en `gca_acceso_log`.
2. El panel global permite filtrar por actor, credencial, acción y fecha.

---

## 5. Pantallas / vistas

1. **Bandeja `/gca`** — tabla con nombre, categoría, proveedor, usuario, responsable, badge de
   vencimiento, estado y acciones. Búsqueda y filtros. Incluye el botón **"Ver contraseñas"** que
   activa el modo revelado de la página/filtro actual.
2. **Enrolamiento 2º factor GCA** — QR + confirmación (obligatorio antes de poder revelar).
3. **Modal de revelado** — solo pide el código del authenticator GCA.
4. **Formulario crear/editar** — al editar, el campo secreto vacío = "no cambiar".
5. **Vencimientos próximos** — lista ordenada (y badge en el menú).
6. **Historial de accesos** — por credencial y panel global.
7. **Ítem en Sidebar**, grupo **Config**, visible solo para los 4 roles.

---

## 6. Modelo de datos (alto nivel)

### `gca_credentials`
| Campo | Descripción |
|---|---|
| `id` | uuid pk |
| `nombre` | ej. "Dominio simindustrial.com" |
| `categoria` | enum: Correo / Dominio / Suscripción / Software / Hosting / Otro |
| `proveedor` | Brevo, GoDaddy, Railway, Supabase, AutoDesk, … |
| `url_login` | enlace al portal |
| `usuario` | correo/usuario de la cuenta |
| `secreto_cifrado` | ciphertext AES-256-GCM |
| `secreto_iv` | IV único por registro |
| `secreto_auth_tag` | tag de autenticación GCM |
| `key_version` | versión de clave (rotación) |
| `responsable_id` | FK a `users` |
| `fecha_vencimiento` | nullable |
| `renovacion_automatica` | bool |
| `notas` | texto libre (sin secretos) |
| `estado` | activa / archivada |
| `created_by`, `updated_by` | FK a `users` |
| `created_at`, `updated_at` | timestamps |
| `deleted_at` | soft delete |

### `gca_totp`
| Campo | Descripción |
|---|---|
| `user_id` | FK a `users` |
| `secret_cifrado` | semilla TOTP (cifrada con la clave GCA) |
| `secret_iv`, `secret_auth_tag` | cifrado |
| `enabled` | bool |
| `enrolled_at` | timestamp |
| `last_used_step` | anti-replay del TOTP |

### `gca_acceso_log`
| Campo | Descripción |
|---|---|
| `id` | uuid pk |
| `credential_id` | FK |
| `actor_id`, `actor_email` | quién |
| `accion` | CREATED / UPDATED / ARCHIVED / VIEWED / **REVEALED** / COPIED |
| `ip_address`, `user_agent` | contexto |
| `metadata` | jsonb (campos cambiados, scope/count; **nunca valores**) |
| `created_at` | timestamp |

> **RLS activado** en las tres tablas, sin políticas para `anon`/`public`/`authenticated`.
> El acceso es solo desde el backend con `service_role`.

---

## 7. Seguridad — diseño del revelado de secretos

El secreto **es recuperable** (una persona necesita leerlo), por lo que **no se hashea**: se cifra
de forma simétrica.

### 7.1 Cifrado en reposo
- **AES-256-GCM** con el módulo `crypto` nativo de Node (sin librería nueva).
- Clave maestra de 32 bytes en `GCA_ENCRYPTION_KEY` (base64), **fuera de la base de datos**
  (Railway). Si falta, el módulo falla en voz alta y no arranca (mismo patrón que `JWT_SECRET`).
- Por registro: **IV aleatorio de 12 bytes** + `authTag`. AAD = `id` de la credencial (evita mover
  el ciphertext a otra fila).
- **Rotación de clave:** `key_version` + keyset (`GCA_ENCRYPTION_KEYS_JSON`), se cifra con la
  versión actual y se descifra con la que indique la fila.

### 7.2 Segundo factor dedicado (TOTP)
- Semilla separada de la de login, cifrada con la misma clave GCA.
- Etiqueta distinta en el authenticator; **el código de login no sirve** para revelar.
- **Obligatorio y no desactivable** por el usuario; solo TI/SuperAdmin puede regenerarlo.
- Configurado con ventana estricta (`window: 0`) y **anti-replay** (`last_used_step`).

### 7.3 Sesión de revelado (60 s)
- El código se pide **una sola vez** y habilita ver las contraseñas de la página/filtro durante
  **60 segundos**.
- La expiración la aplica el **servidor** (token `gca_reveal`, `purpose` propio, atado a
  `user_id` + `token_version`, TTL 60 s). El contador del frontend es solo UX.
- Al expirar: el frontend borra las contraseñas de memoria y las oculta; el token ya no sirve.
- Cada apertura y cada lote quedan auditados con actor, IP y fecha.

### 7.4 Resistencia al volcado de la base de datos
- Quien robe un `pg_dump` o la `service_role key` obtiene **solo ciphertext**.
- Sin `GCA_ENCRYPTION_KEY` (que vive en el entorno de despliegue, no en la BD) no puede leer nada.
- Los listados seleccionan columnas explícitas y **nunca** incluyen los campos del secreto.

### 7.5 Cabeceras y transporte
- `Cache-Control: no-store` en las respuestas del módulo.
- Los secretos viajan solo por `POST`, nunca por URL/query.
- `morgan` (hoy `tiny`/`dev`) no loguea body; el reveal no debe loguear body.

---

## 8. Arquitectura técnica (encaje en Prisma)

Se integra siguiendo el patrón de los módulos existentes (SENA/Admin):

**Backend**
- `backend/src/services/gca/` → lógica de negocio + `gcaCrypto.js` (cifrado).
- `backend/src/controllers/gca/`.
- `backend/src/routes/gca/gca.routes.js`.
- `backend/src/validations/gca.validation.js` (zod).
- `backend/src/services/background/gcaExpiryAlerts.service.js` + registro en `jobs/cron.js`.
- `server.js`: `app.use('/api/gca', authMiddleware, gcaRoutes);`

**Frontend**
- `frontend/src/pages/gca/`.
- `frontend/src/services/gcaService.js`.
- `App.jsx`: `lazyWithRetry` + `ProtectedRoute allowedRoles=[los 4]`.
- `frontend/src/config/modulePermissions.js`: entrada `'/gca'`.
- `Sidebar.jsx`: ítem dentro del grupo **Config**.

### Endpoints
```
GET    /api/gca                     listado (sin secretos)
GET    /api/gca/:id                 detalle (sin secreto)
POST   /api/gca                     crear
PUT    /api/gca/:id                 editar (secreto vacío = no cambiar)
DELETE /api/gca/:id                 archivar
GET    /api/gca/vencimientos        próximos a vencer
GET    /api/gca/:id/historial       accesos de esa credencial
GET    /api/gca/auditoria           panel global

POST   /api/gca/totp/setup          QR del 2º factor GCA
POST   /api/gca/totp/verify-setup   confirmar enrolamiento
POST   /api/gca/totp/reset          (TI) regenerar 2º factor de un usuario

POST   /api/gca/revelar/sesion      { code } → abre sesión de 60 s
POST   /api/gca/revelar/lote        { ids } → id→secreto de la página (sesión viva)
```

---

## 9. Alertas de vencimiento

- `gcaExpiryAlerts.service.js` corre en `cron.schedule` diario (patrón `dueDateAlerts` /
  `hseqAlerts`).
- Avisa al `responsable_id` + roles admin con umbrales 30/15/7/1 días.
- No reenvía el mismo aviso; respeta `renovacion_automatica`.

---

## 10. Recomendaciones de seguridad adicionales (a tener en cuenta al construir)

### 🔴 Críticas

1. **La clave maestra es el nuevo punto único de falla.**
   - `GCA_ENCRYPTION_KEY` protege *todas* las contraseñas a la vez.
   - Si se pierde → la empresa pierde todo. Guardar **copia offline** (sobre sellado / gestor de
     secretos de la empresa), no solo en Railway.
   - Si se filtra → se compromete todo. Tratarla con más celo que la `service_role` de Supabase.
   - Fase 2: evaluar **envelope encryption con KMS** para no tener el valor crudo en el `.env`.

2. **Continuidad — no repetir la dependencia de una sola persona.**
   - Enrolar al menos **2–3 personas** (Luis + Gerente + Dir. Administrativo) desde el día 1.
   - Escribir un **runbook de recuperación**: pérdida de clave, pérdida de todos los
     authenticators, salida de un admin.

### 🟠 Vulnerabilidades técnicas a cubrir

3. **Anti-replay del TOTP**: ventana estricta + registrar el último time-step usado.
4. **Rate-limit y bloqueo propios del factor GCA** (no depender solo del login).
5. **Confusión de tokens JWT**: `purpose='gca_reveal'` distinto y verificado estrictamente;
   incluir y validar `token_version`; atar a `user_id` (evaluar IP/user-agent).
6. **CSRF**: no existe protección CSRF en el backend hoy. Usar `SameSite=Strict` + token CSRF
   en endpoints de mutación/revelado, o pasar el reveal por header `Authorization`.
7. **Fuga por UI / portapapeles**: ocultar también al perder foco la pestaña (`blur` /
   `visibilitychange`), limpiar estado al navegar, no usar almacenamiento persistido, advertir
   sobre el historial de portapapeles de Windows y auditar cada copia.
8. **Bitácora a prueba de manipulación**: tabla `insert-only` (sin UPDATE/DELETE) y con RLS que
   niegue a anon/authenticated. Incluir request id. **Alertas de anomalía** (reveals fuera de
   horario, IP nueva, muchos seguidos, fallos repetidos de TOTP).
9. **Errores sin filtrar**: un fallo de descifrado se registra en servidor sin valor ni clave;
   al cliente, mensaje genérico.
10. **Validación (zod)**: `categoria` como enum cerrado; `url_login` con esquema validado
    (bloquear `javascript:`/`data:`); no concatenar input en filtros `.or()` de Supabase;
    escapar con `escapeHtml` en emails de vencimiento.

### 🟡 Hallazgos concretos en el repo

11. **`.gitignore` incompleto**: solo ignora `.env`, `.env.local`, `.env.*.local`; **no** ignora
    `.env.production`, `.env.staging`, etc. Corregir a:
    ```gitignore
    .env
    .env.*
    !.env.example
    ```
    Y recordar: los `.env.example` deben llevar solo placeholders; cualquier secreto que ya haya
    estado en git **debe rotarse** (la historia no se borra al eliminar el archivo).

12. **Separación de funciones (Fase 2)**: hoy los 4 roles podrían crear/editar/revelar. Evaluar
    separar "editar" de "revelar" y pedir confirmación extra para rotar una credencial.

### 🟢 Buenas prácticas

13. **Cifrado autenticado bien hecho** con test (vitest): round-trip correcto y que un ciphertext
    manipulado **falle** (GCM debe detectarlo). Script de re-cifrado para rotación.
14. **Modelar negocio**: `renovacion_automatica` no alerta igual; distinguir "sin vencimiento" de
    "vencido"; tipo de secreto (password / API key / token).
15. **Cumplimiento (ya referencian ISO en el código)**: esto cae en ISO 27001 **A.5.15/A.8.24
    (cifrado)**, **A.5.17 (info de autenticación)** y **A.8.16 (monitoreo)**. Documentar el
    runbook ayuda en auditoría.

---

## 11. Fase 2 (fuera de la v1)

- Permisos por credencial ("quién más puede ver/usar") y separación editar/revelar.
- Tags/grupos y dashboard de renovaciones/gastos.
- Rotación de clave automatizada con gestor de secretos (KMS/Infisical/Doppler) — envelope
  encryption.
- Semilla TOTP/2FA de las cuentas externas.
- Import CSV, adjuntar comprobantes, exportación cifrada.
- Doble aprobación para reset del 2º factor.
- Códigos de recuperación de respaldo para el 2º factor.

---

## 12. Checklist de implementación (Fase 1)

- [ ] Decidir y generar `GCA_ENCRYPTION_KEY` + copia offline + runbook de recuperación.
- [ ] Crear tablas en Supabase (`gca_credentials`, `gca_totp`, `gca_acceso_log`) con RLS.
- [ ] `gcaCrypto.js` (cifrar/descifrar AES-256-GCM) + tests (round-trip y tamper).
- [ ] Servicios, controladores, rutas y validaciones zod.
- [ ] Registrar `/api/gca` en `server.js`.
- [ ] Enrolamiento del 2º factor GCA (QR, verify-setup, reset por TI).
- [ ] CRUD + auditoría (`gca_acceso_log` insert-only).
- [ ] Sesión de revelado de 60 s + `revelar/lote` por página/filtro + anti-replay.
- [ ] Alertas de vencimiento en `cron.js`.
- [ ] Frontend: página, servicio, ruta protegida, `modulePermissions`, Sidebar.
- [ ] Ocultar en `blur`/expiración; limpiar estado y portapapeles.
- [ ] Corregir `.gitignore` y rotar secretos que hayan estado en git.

---

## 13. Decisiones cerradas

| Tema | Decisión |
|---|---|
| Roles con acceso | Administrador TI, SuperAdmin, Gerente, Dir. Administrativo |
| "Luis (Tech Lead)" | Administrador TI |
| Cifrado | AES-256-GCM, clave en `GCA_ENCRYPTION_KEY` |
| 2º factor para revelar | Google Authenticator **distinto** al del login, obligatorio |
| Desactivar 2º factor | No permitido al usuario (solo reset por TI) |
| Duración de revelado | 60 segundos |
| Alcance de "ver contraseñas" | Página/filtro actual (no toda la base) |
| Contraseña de seguridad | No en v1 (el authenticator es la autorización) |
ca-modulo-credenciales.md…]()
