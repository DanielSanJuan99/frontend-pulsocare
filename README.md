# 🫀 PulsoCare – Frontend

Aplicación web para el **monitoreo continuo de pacientes en hospitalización domiciliaria**, desarrollada en **Angular 21**. Permite a médicos y enfermeros hacer seguimiento de signos vitales extraídos del dataset clínico **MIMIC-IV**, configurar umbrales de alarma personalizados, y acceder al histórico de mediciones. Los familiares disponen de una vista simplificada de los signos vitales de su ser querido, y los administradores gestionan usuarios y revisan la bitácora de auditoría.

La autenticación está delegada completamente a **Azure AD B2C** mediante MSAL, y el rol de cada usuario se determina por el atributo `jobTitle` configurado en Microsoft Entra ID.

---

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Tecnologías Utilizadas](#tecnologías-utilizadas)
- [Arquitectura del Proyecto](#arquitectura-del-proyecto)
- [Roles y Control de Acceso](#roles-y-control-de-acceso)
- [Módulos y Funcionalidades](#módulos-y-funcionalidades)
- [Signos Vitales Monitoreados](#signos-vitales-monitoreados)
- [Escala NEWS2](#escala-news2)
- [Requisitos Previos](#requisitos-previos)
- [Instalación y Ejecución Local](#instalación-y-ejecución-local)
- [Variables de Entorno](#variables-de-entorno)
- [Rutas de la Aplicación](#rutas-de-la-aplicación)
- [Despliegue](#despliegue)
- [Pruebas](#pruebas)
- [GitHub Actions – Sincronización del Fork](#github-actions--sincronización-del-fork)

---

## Descripción General

PulsoCare consume datos clínicos reales del dataset **MIMIC-IV** (Medical Information Mart for Intensive Care), replicados y reproducidos en tiempo real mediante un componente de backend llamado "replayer". El frontend muestra esos datos en tiempo real y se refresca automáticamente cada 8 segundos sin intervención del usuario.

El sistema opera en tres capas de acceso diferenciadas:

- **Médico / Enfermero** – Monitoreo completo de sus pacientes asignados, configuración de umbrales de alarma individuales por signo vital, historial paginado con filtros y exportación a Excel.
- **Familiar** – Vista de solo lectura de los signos vitales del paciente asignado, con indicadores visuales de estado (normal, atención, crítico).
- **Administrador** – Creación y gestión de cuentas de usuario (médicos y familiares), registro de pacientes, asignación de cuidadores, y revisión de la bitácora de auditoría de accesos clínicos.

---

## Tecnologías Utilizadas

| Tecnología | Versión | Uso |
|---|---|---|
| Angular | 21.2.x | Framework principal |
| TypeScript | 5.9.x | Lenguaje de desarrollo |
| Tailwind CSS | 4.3.x | Estilos utilitarios |
| MSAL Angular / Browser | 5.3.x / 5.15.x | Autenticación con Azure AD B2C |
| @ng-icons/lucide | 33.2.x | Íconos de la interfaz |
| RxJS | 7.8.x | Programación reactiva |
| write-excel-file | 4.1.x | Exportación del histórico a Excel |
| Vitest | 4.0.x | Framework de pruebas unitarias |
| Prettier | 3.8.x | Formateo de código |
| Docker + Nginx | — | Contenerización y despliegue |
| AWS Amplify | — | Despliegue en producción |

---

## Arquitectura del Proyecto

```
src/
└── app/
    ├── core/
    │   ├── auth/
    │   │   ├── roles.config.ts       # Mapeo jobTitle → rol, catálogos de parentesco
    │   │   └── token.interceptor.ts  # Adjunta el ID token de B2C a cada petición
    │   ├── guards/
    │   │   └── rol.guard.ts          # Protege rutas por rol (ADMIN / MEDICO / FAMILIAR)
    │   ├── models/                   # DTOs e interfaces TypeScript
    │   │   ├── paciente.dto.ts
    │   │   ├── consultas.dto.ts      # LecturaDTO, AlertaDTO, catálogo y límites de signos
    │   │   ├── news2.dto.ts          # Escala NEWS2
    │   │   ├── usuario.dto.ts
    │   │   ├── umbral.dto.ts
    │   │   ├── asignacion.dto.ts
    │   │   └── bitacora.dto.ts
    │   └── services/
    │       ├── auth.store.ts         # Estado de sesión, sincronización con el backend
    │       ├── consultas.service.ts  # Lecturas, alertas, umbrales, NEWS2, bitácora
    │       ├── paciente.service.ts   # CRUD de pacientes
    │       └── notificaciones.service.ts  # Toasts de éxito / error / info
    │
    ├── pages/
    │   ├── login/                    # Pantalla de inicio y botón de login B2C
    │   ├── recuperar-password/       # Flujo de recuperación de contraseña B2C
    │   ├── admin/
    │   │   ├── admin-store.ts        # Estado y operaciones del panel de administración
    │   │   ├── admin-tabs/           # Navegación entre pestañas de admin
    │   │   ├── crear-usuario/        # Formulario de alta de médicos y familiares
    │   │   ├── crear-paciente/       # Formulario de registro de pacientes
    │   │   └── auditoria/            # Bitácora de accesos clínicos
    │   ├── medico/
    │   │   ├── pacientes/            # Lista de pacientes asignados al médico
    │   │   ├── paciente-detalle/     # Ficha con signos vitales en tiempo real y alertas
    │   │   ├── historico/            # Historial paginado con filtros y exportación Excel
    │   │   └── umbrales/             # Configuración de límites de alarma por signo vital
    │   └── familiar/
    │       └── signos-vitales/       # Vista simplificada de signos vitales
    │
    └── shared/
        ├── topbar/                   # Barra superior con nombre, rol y botón de cierre de sesión
        ├── vitals-board/             # Panel completo de signos vitales con auto-refresco
        ├── vital-card/               # Tarjeta individual de un signo vital
        ├── news2-panel/              # Panel de la escala de alerta temprana NEWS2
        ├── notificaciones/           # Componente de avisos tipo toast
        └── ecg-trace/                # Animación SVG de traza ECG decorativa
```

---

## Roles y Control de Acceso

La aplicación define tres roles internos derivados del atributo `jobTitle` configurado en Microsoft Entra ID:

| `jobTitle` en Entra ID | Rol interno | Ruta de entrada |
|---|---|---|
| `Administrador` | `ADMIN` | `/admin/usuarios` |
| `Medico` o `Enfermero` | `MEDICO` | `/medico/pacientes` |
| `Familiar` | `FAMILIAR` | `/familiar/signos-vitales` |

> Los enfermeros se homologan al rol `MEDICO` porque comparten las mismas vistas de monitoreo. El rol se determina al iniciar sesión y no se modifica durante la sesión activa.

### Flujo de autenticación

1. El usuario hace clic en "Ingresar" en la pantalla de login.
2. MSAL redirige a Azure AD B2C (`B2C_1_SIGN_IN`), que gestiona el login (incluyendo recuperación de contraseña).
3. Al regresar, el `AuthStore` llama a `POST /api/auth/registro` con los claims del token para registrar o sincronizar la cuenta en el backend.
4. El backend responde con el `UsuarioDTO` completo (incluido el rol real de la base de datos).
5. El `rolGuard` verifica que el rol del usuario tenga acceso a la ruta solicitada y redirige a la ruta principal del rol si no lo tiene.
6. El `tokenInterceptor` adjunta el ID token de B2C en el header `Authorization: Bearer <token>` de cada petición al backend.

> Si el backend devuelve 403, el usuario ve un mensaje explicativo en la pantalla de login: "Tu cuenta está desactivada. Comunícate con el administrador."

---

## Módulos y Funcionalidades

### 🔐 Login y Recuperación de Contraseña
- Inicio de sesión delegado a Azure AD B2C.
- Recuperación de contraseña mediante flujo B2C (sin lógica propia en el frontend).
- Mensajes de error explicativos cuando la cuenta está desactivada o hay problemas de sesión.

### 🛠️ Panel de Administración (`/admin`)
Accesible solo para el rol `ADMIN`.

**Gestión de Usuarios** (`/admin/usuarios`):
- Crear cuentas de médicos y familiares (la cuenta se crea tanto en el backend como en Azure B2C).
- Al crear un usuario, se muestra la contraseña temporal generada por B2C para entregársela al nuevo usuario.
- Activar o desactivar usuarios (baja lógica: no se eliminan por integridad referencial).
- Asignar y desasignar pacientes a usuarios.
- Ver los pacientes asignados a cada usuario.

**Gestión de Pacientes** (`/admin/pacientes`):
- Registrar nuevos pacientes con datos demográficos (nombre, fecha de nacimiento, sexo, modalidad, estado).
- Cambiar el estado de un paciente (ESTABLE, CRÍTICO, OBSERVACIÓN, ALTA). Un paciente dado de alta deja de ser monitorizado.

**Bitácora de Auditoría** (`/admin/auditoria`):
- Visualizar el registro de acciones clínicas: quién accedió a qué datos, cuándo y desde qué IP.
- Las acciones registradas incluyen: ver paciente, ver historial, definir/editar/eliminar umbrales, iniciar sesión.

### 🩺 Panel Médico (`/medico`)
Accesible para los roles `MEDICO` y `Enfermero` (homologado).

**Lista de Pacientes** (`/medico/pacientes`):
- Ver todos los pacientes asignados al médico con su estado clínico actual en tiempo real.
- Cada paciente muestra el peor estado entre todos sus signos vitales (normal / atención / crítico).

**Detalle del Paciente** (`/medico/pacientes/:id`):
- Panel de signos vitales con refresco automático cada 8 segundos.
- Escala NEWS2 calculada en el backend y actualizada junto con las lecturas.
- Panel de alertas activas con posibilidad de reconocerlas.
- Acceso rápido al histórico y a la configuración de umbrales.

**Histórico de Lecturas** (`/medico/pacientes/:id/historico`):
- Tabla paginada (50 filas por página) con todas las lecturas del paciente.
- Filtros por signo vital, rango de fechas y hora del día.
- Ordenamiento por columna (fecha, signo, valor) delegado al backend.
- Exportación del resultado filtrado a un archivo Excel (`.xlsx`).
- Cada lectura muestra su estado (Normal / Atención / Crítico) comparado con los umbrales del paciente.

**Configuración de Umbrales** (`/medico/pacientes/:id/umbrales`):
- Definir límites normales y críticos personalizados para cada uno de los 8 signos vitales.
- Los umbrales se guardan con baja lógica: se mantiene el historial de cambios.
- Restaurar cualquier signo a sus valores por defecto (baja lógica del umbral personalizado).
- Los cambios quedan registrados en la bitácora con el id del médico que los realizó.

### 👨‍👩‍👧 Vista Familiar (`/familiar`)
Accesible solo para el rol `FAMILIAR`.

**Signos Vitales** (`/familiar/signos-vitales`):
- Panel de signos vitales de lectura del paciente asignado, con refresco automático.
- Indicadores visuales de estado (verde / amarillo / rojo) sin mostrar valores de riesgo clínico que requieran contexto médico.
- Si el familiar no tiene un paciente asignado, se muestra un mensaje para que contacte al administrador.

---

## Signos Vitales Monitoreados

| Código | Nombre | Rango normal | Unidad |
|---|---|---|---|
| `FC` | Frecuencia cardíaca | 60 – 100 | lpm |
| `SPO2` | Saturación de oxígeno | 95 – 100 | % |
| `PAS` | Presión arterial sistólica | 90 – 120 | mmHg |
| `PAD` | Presión arterial diastólica | 60 – 80 | mmHg |
| `TEMP` | Temperatura | 36.0 – 37.5 | °C |
| `FR` | Frecuencia respiratoria | 12 – 20 | rpm |
| `GCS` | Nivel de conciencia (Glasgow) | 15 / 15 | puntos |
| `O2SUP` | Oxígeno suplementario | Sin oxígeno | Sí / No |

Los límites por defecto mostrados en la tabla son los valores de referencia estándar del sistema. Cada médico puede ajustarlos individualmente por paciente desde la pantalla de umbrales.

---

## Escala NEWS2

La aplicación muestra el puntaje **NEWS2** (National Early Warning Score 2) calculado en el backend a partir de las últimas lecturas de cada paciente. Este puntaje es una señal de apoyo clínico, no un diagnóstico.

| Nivel de riesgo | Total NEWS2 | Acción recomendada |
|---|---|---|
| **BAJO** | 0 – 4 | Monitoreo habitual |
| **MEDIO** | 5 – 6 (o 3 en un solo parámetro) | Revisar al paciente |
| **ALTO** | 7 o más | Revisión inmediata |

La escala evalúa 7 parámetros con un puntaje máximo de 20 puntos. La bandera roja se activa cuando un solo signo vital supera 3 puntos, independientemente del total.

---

## Requisitos Previos

- [Node.js](https://nodejs.org/) v20 o superior
- [npm](https://www.npmjs.com/) v11.9.0 (declarado como `packageManager`)
- [Angular CLI](https://angular.dev/tools/cli) v21

```bash
npm install -g @angular/cli@21
```

---

## Instalación y Ejecución Local

```bash
# 1. Clonar el repositorio
git clone https://github.com/DanielSanJuan99/frontend-pulsocare
cd frontend-pulsocare

# 2. Instalar dependencias
npm ci

# 3. Iniciar el servidor de desarrollo
npm start
```

La aplicación estará disponible en: **http://localhost:4200**

> El servidor recarga automáticamente ante cualquier cambio en el código fuente. Para que el login de B2C funcione en local, la URL `http://localhost:4200/` debe estar registrada como URI de redirección en el registro de la aplicación en Azure AD B2C.

---

## Variables de Entorno

La URL base de la API se configura en `src/environments/`:

| Archivo | Entorno | `apiUrl` |
|---|---|---|
| `environment.development.ts` | Local | `http://localhost:8080/api` |
| `environment.ts` / `environment.production.ts` | Producción | `https://c7ja6bpsz3.execute-api.us-east-1.amazonaws.com/api` |

Angular selecciona el archivo de entorno automáticamente según la configuración de build (`--configuration=development` para local, `--configuration=production` para producción).

### Configuración de Azure AD B2C

Los siguientes valores están hardcodeados en `app.config.ts` y deben actualizarse si se cambia el tenant o el registro de la aplicación:

| Parámetro | Valor actual |
|---|---|
| `CLIENT_ID` | `bbc1023b-e89e-4fd1-925c-141f8d7d148c` |
| `authority` | `https://pulsocareduoc.b2clogin.com/pulsocareduoc.onmicrosoft.com/B2C_1_SIGN_IN` |
| `SCOPE_API` | `https://pulsocareduoc.onmicrosoft.com/pulsocare-api/acceso.total` |

> En producción, considera externalizar estos valores a variables de entorno de Amplify o al `environment.production.ts` para evitar exponer configuración sensible en el repositorio.

---

## Rutas de la Aplicación

| Ruta | Componente | Roles permitidos |
|---|---|---|
| `/` | `Login` | Público |
| `/recuperar-password` | `RecuperarPassword` | Público |
| `/admin/usuarios` | `CrearUsuario` | ADMIN |
| `/admin/pacientes` | `CrearPaciente` | ADMIN |
| `/admin/auditoria` | `Auditoria` | ADMIN |
| `/medico/pacientes` | `Pacientes` | MEDICO |
| `/medico/pacientes/:id` | `PacienteDetalle` | MEDICO |
| `/medico/pacientes/:id/historico` | `Historico` | MEDICO |
| `/medico/pacientes/:id/umbrales` | `Umbrales` | MEDICO |
| `/familiar/signos-vitales` | `SignosVitalesFamiliar` | FAMILIAR |
| `**` | Redirige a `/` | — |

Todas las rutas bajo `/admin`, `/medico` y `/familiar` requieren sesión activa en Azure B2C (`MsalGuard`) y el rol correspondiente (`rolGuard`). Una sesión válida con rol incorrecto redirige a `/` en vez de mostrar un error 403.

---

## Despliegue

### Docker (local o servidor propio)

El proyecto incluye un `Dockerfile` multietapa que compila la aplicación con Node 20 y la sirve con Nginx.

```bash
# Construir la imagen
docker build -t pulsocare-frontend .

# Ejecutar el contenedor
docker run -p 80:80 pulsocare-frontend
```

La aplicación estará disponible en: **http://localhost**

La configuración de Nginx (`nginx.conf`) redirige todas las rutas a `index.html` para soportar el enrutamiento de SPA.

### AWS Amplify (producción)

El proyecto incluye `amplify.yml` con la configuración de build para AWS Amplify. Amplify detecta automáticamente este archivo y ejecuta `npm ci` + `npm run build`, publicando el contenido de `dist/front-pulsocare/browser/`.

```yaml
# amplify.yml (resumen)
baseDirectory: dist/front-pulsocare/browser
preBuild: npm ci
build: npm run build
```

Para desplegar, conecta el repositorio en la consola de AWS Amplify y configura las variables de entorno necesarias si corresponde.

---

## Pruebas

El proyecto usa **Vitest** como framework de pruebas (integrado con Angular CLI).

```bash
# Ejecutar pruebas unitarias
npm test
```

Las pruebas cubren:
- `AuthStore` – sincronización con el backend, manejo de rechazos y cierre de sesión.
- `rolGuard` – verificación de roles, manejo de recarga de página con sesión cacheada.
- `roles.config` – mapeo de `jobTitle` a rol y de nombre de rol a clave interna.
- `VitalCard` – renderizado de tarjetas de signos vitales y estados visuales.
- `VitalsBoard` – panel de monitoreo y lógica de estado agregado.
- `ConsultasService` – llamadas al API de lecturas, alertas y umbrales.
- `CrearUsuario` – formulario de creación de usuarios.
- `App` – inicialización y ciclo de vida de la aplicación.

---

## GitHub Actions – Sincronización del Fork

El repositorio incluye un workflow (`.github/workflows/sync-fork.yml`) que mantiene este fork sincronizado con el repositorio upstream original, rebasando automáticamente los commits de configuración de despliegue sobre los cambios de upstream.

- **Frecuencia:** cada 15 minutos (o manualmente desde la pestaña *Actions*).
- **Estrategia:** `git rebase upstream/main` para preservar el orden de commits.
- **En caso de conflicto:** el workflow falla con un mensaje de error explicativo y debe resolverse manualmente con `git fetch upstream && git rebase upstream/main`.
