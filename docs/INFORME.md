# Informe técnico: despliegue en la nube de un microservicio de gestión de usuarios

- **Proyecto:** `mauricio-users-management-microservice`
- **Autor:** Mauricio Vergara (GitHub: [MauricioFonck](https://github.com/MauricioFonck))
- **Fecha:** 18 de noviembre de 2026
- **Repositorio:** <https://github.com/MauricioFonck/mauricio-users-management-microservice>
- **API desplegada:** <https://mauricio-users-management-microservice.onrender.com>
- **Base del proyecto y guías:** instructor John Carlos Arrieta Arrieta (repositorio original `arrietajohn/users-management-microservice`)

---

## 1. Resumen ejecutivo

Se tomó un microservicio Spring Boot de gestión de usuarios (arquitectura hexagonal, autenticación JWT) y se llevó desde el equipo local hasta un entorno productivo en la nube, siguiendo cinco guías paso a paso. El resultado es una API REST pública, con base de datos remota, despliegue continuo desde GitHub y envío automático de correos transaccionales.

| Guía | Objetivo | Resultado |
|------|----------|-----------|
| 1 | Clonar el repositorio y levantar el servicio en local | Cumplido: compilación correcta, 194 pruebas en verde, API operativa con MySQL en Docker |
| 2 | Integrar una base de datos relacional remota (Aiven) | Cumplido: esquema cargado y API conectada a Aiven |
| 3 | Desplegar en un PaaS (Render) con CI/CD | Cumplido: servicio en vivo, redespliegue automático en cada `push` a `main` |
| 4 | Migrar la base de datos a TiDB Cloud | Cumplido: la API desplegada escribe en TiDB con SSL estricto |
| 5 | Notificaciones por correo con Brevo (API REST) | Cumplido: correo "Tu cuenta ha sido creada" enviado desde la API en producción |

Todos los cambios se versionaron en Git bajo la identidad del autor, y cada paso quedó documentado en la carpeta `docs/` del repositorio.

## 2. Arquitectura y tecnologías

| Capa | Tecnología |
|------|-----------|
| Lenguaje y framework | Java 17, Spring Boot 3.3.5, Maven (Maven Wrapper) |
| Arquitectura | Hexagonal (puertos y adaptadores) con diseño guiado por el dominio |
| Seguridad | Spring Security con tokens JWT firmados (HMAC-SHA256) |
| Base de datos local | MySQL 8.4 en Docker (`compose.yaml`) |
| Base de datos remota | Aiven MySQL (Guía 2) y luego TiDB Cloud Starter (Guía 4) |
| Despliegue | Render (servicio web con Docker, plan gratuito) |
| Correo | Brevo, API REST transaccional con reintentos y circuit breaker (Resilience4j) |
| Documentación de la API | Swagger UI / OpenAPI 3 |

```text
Cliente (Swagger / HTTP)
        │ HTTPS
        ▼
Render (contenedor Docker, Spring Boot)  ──HTTPS──►  Brevo (envío de correos)
        │ JDBC + TLS (VERIFY_IDENTITY)
        ▼
TiDB Cloud (MySQL compatible)
```

## 3. Desarrollo paso a paso

### 3.1 Guía 1 — Entorno local

Se hizo *fork* del repositorio original a la cuenta del autor con el nombre `mauricio-users-management-microservice`, se clonó y se configuró la identidad de Git solo para este repositorio. El equipo tenía Java 26, pero el proyecto requiere Java 17; se usó un JDK 17 portable sin modificar el Java global. La base de datos local se levantó con Docker en lugar de instalar MySQL.

![Contenedor MySQL en Docker](img/02-mysql-docker.png)

![Tabla users creada por schema.sql](img/03-mysql-tablas.png)

La compilación con el wrapper de Maven terminó en `BUILD SUCCESS` con **194 pruebas sin fallos**, y el servicio respondió correctamente en `/health`.

![Compilación exitosa y pruebas](img/05-build-success.png)

![Endpoint /health en local](img/06-health-up.png)

### 3.2 Guía 2 — Base de datos remota (Aiven)

Se creó un servicio MySQL 8.4 en Aiven con el plan de desarrollo (gratuito durante el período de prueba), se ejecutó `schema.sql` y se comprobó que la aplicación funcionaba contra la base remota, apagando incluso el MySQL local para demostrarlo.

![Creación del servicio MySQL en Aiven](img/09-aiven-crear-servicio-mysql.png)

![Información de conexión de Aiven](img/10-aiven-connection-information.png)

![Esquema cargado en Aiven](img/18-aiven-esquema-cargado.png)

![Aplicación conectada a Aiven](img/19-app-conectada-a-aiven.png)

### 3.3 Guía 3 — Despliegue en Render con CI/CD

Se activó la autenticación de dos factores en GitHub, se creó la cuenta de Render con inicio de sesión de GitHub y se conectó el repositorio. Render detecta el `Dockerfile`, compila la imagen y publica el servicio; a partir de ahí, cada `push` a la rama `main` dispara un nuevo despliegue automático.

![2FA habilitado en GitHub](img/11-github-2fa-habilitado.png)

![Selección del repositorio en Render](img/21-render-seleccionar-repo.png)

![Configuración: Docker, rama main](img/23-render-nombre-docker-main.png)

![Plan gratuito y variables de entorno](img/24-render-plan-free-y-variables.png)

![Estado del despliegue: Deploy succeeded, Live](img/28-render-deploy-succeeded-live.png)

El servicio quedó disponible en la URL pública. Se verificó el estado, la documentación interactiva y los endpoints protegidos:

![Endpoint /health en Render](img/33-render-health.png)

![Swagger UI de la API desplegada](img/34-render-swagger-ui.png)

![Pruebas contra la URL pública](img/26-render-pruebas-en-la-nube.png)

### 3.4 Guía 4 — Migración a TiDB Cloud

Se creó una instancia TiDB Cloud Starter (plan gratuito sin vencimiento, compatible con MySQL), se cargó el mismo esquema y se cambiaron las variables de conexión, con `DB_SSL_MODE=VERIFY_IDENTITY` para validar el certificado del servidor.

![Creación del recurso en TiDB](img/14-tidb-crear-recurso-starter.png)

![Esquema cargado en TiDB](img/30-tidb-esquema-cargado.png)

![SQL Editor de TiDB Cloud con la tabla users](img/32-tidb-sql-editor-schemas.png)

![Aplicación conectada a TiDB](img/31-app-conectada-a-tidb.png)

Verificación en producción: al crear un usuario por la URL pública de Render, la fila apareció directamente en TiDB, lo que confirma que la API desplegada ya no usa Aiven.

### 3.5 Guía 5 — Notificaciones por correo con Brevo

Render bloquea los puertos SMTP en el plan gratuito, por lo que se integró la rama `feature/brevo-rest-email-resilience` del repositorio original, que reemplaza el adaptador SMTP por uno que consume la **API REST de Brevo** por HTTPS. Se resolvieron tres conflictos de fusión (`.gitignore`, `render.yaml`, `application.properties`), se verificaron 199 pruebas en verde y se integró mediante un *Pull Request*.

![Claves de API en Brevo](img/16-brevo-claves-api.png)

![Configuración SMTP/API de Brevo](img/17-brevo-smtp.png)

![IP no autorizadas: bloqueo desactivado](img/29-brevo-ips-autorizadas-desactivado.png)

![Pull Request #1 fusionado](img/37-github-pull-request-1.png)

Prueba extremo a extremo: se creó un usuario desde la API pública y Brevo registró el envío del correo transaccional «Tu cuenta ha sido creada — Gestion de Usuarios».

## 4. Control de versiones

Todos los commits se realizaron con la identidad `MauricioFonck`. El historial muestra cada paso documentado y el *merge* del Pull Request.

![Historial de commits en GitHub](img/38-github-commits.png)

![Repositorio en GitHub](img/35-github-repo.png)

![Carpeta docs con la bitácora](img/36-github-docs-bitacora.png)

## 5. Matriz de verificación

| Requisito | Evidencia | Estado |
|-----------|-----------|--------|
| El proyecto compila y pasa las pruebas | Figura «Compilación exitosa»: 194 pruebas (199 tras la integración de Brevo) | Cumplido |
| API operativa en local con base de datos | `/health` y login contra MySQL en Docker | Cumplido |
| Base de datos remota funcionando | Esquema en Aiven y aplicación conectada | Cumplido |
| Servicio desplegado y accesible en Internet | URL pública, `/health` y Swagger UI | Cumplido |
| CI/CD desde GitHub | Estado *Live*; cada `push` a `main` redespliega | Cumplido |
| API protegida con JWT | `GET /api/users` con token → 200; sin token → 401 | Cumplido |
| Migración a TiDB | Esquema en TiDB y escritura verificada desde la API pública | Cumplido |
| Correo transaccional en producción | Registro de envío en Brevo y Pull Request #1 fusionado | Cumplido |
| Autoría y trazabilidad | Historial de commits bajo `MauricioFonck` | Cumplido |

## 6. Problemas encontrados y soluciones

| Problema | Causa | Solución |
|----------|-------|----------|
| El proyecto pide Java 17 y el equipo tenía Java 26 | Spring Boot 3.3.5 y Lombok no son fiables con versiones tan nuevas | JDK 17 portable usado solo en la sesión de compilación |
| La instalación con `winget` quedó detenida | Requería permisos de administrador (UAC) | Descarga del JDK en formato ZIP, sin instalador |
| La API usaba la clave JWT de desarrollo en Render | Al crear el servicio a mano no se definió `JWT_SECRET` | Variable `JWT_SECRET` aleatoria de 32 bytes; verificado que los tokens ya no validan con la clave por defecto |
| Brevo respondía HTTP 401 | Brevo bloquea llamadas desde IP no autorizadas | Se desactivó el bloqueo para claves API y SMTP (entorno de taller) |
| Brevo rechazaría el remitente | La cuenta solo tenía verificado un remitente, no el Gmail | Se usó el remitente verificado de la cuenta |
| Conflictos al integrar la rama de Brevo | Cambios en `.gitignore`, `render.yaml` y `application.properties` | Resolución manual, conservando la lectura del `.env` local y las variables `BREVO_*` |
| SMTP no funciona en Render (plan gratuito) | Render bloquea los puertos 25, 465 y 587 | Migración a la API REST de Brevo por HTTPS |

## 7. Seguridad y buenas prácticas

- **Secretos fuera del repositorio:** contraseñas, claves de API y `JWT_SECRET` viven en el archivo `.env` local (ignorado por Git) y en las variables de entorno de Render. Se comprobó que ninguna clave aparece en el repositorio.
- **Conexión cifrada a la base de datos:** TLS obligatorio y `VERIFY_IDENTITY` en TiDB.
- **Autenticación JWT:** los endpoints de usuarios devuelven 401 sin token válido.
- **Limitaciones reconocidas (taller):** el administrador inicial conserva las credenciales por defecto del script; el bloqueo de IP de Brevo se desactivó; el correo de bienvenida incluye una contraseña temporal. En un entorno real se cambiarían las credenciales del administrador, se autorizarían solo las IP necesarias y se enviaría un enlace de un solo uso en lugar de la contraseña.
- **Rotación de claves:** las claves usadas durante el desarrollo se rotan al cerrar el proyecto.

## 8. Conclusiones

El microservicio quedó funcionando de punta a punta en producción: API REST pública, base de datos remota con conexión segura, despliegue continuo desde GitHub y notificaciones por correo. El proceso dejó dos aprendizajes prácticos: las diferencias entre ejecutar en local y en un PaaS (puertos SMTP bloqueados, variables de entorno que no se heredan), y la importancia de verificar cada integración contra el sistema real, por ejemplo, confirmando directamente en TiDB que la API desplegada escribe allí.

## Anexo A — Evidencias clave para la revisión

| N.º | Evidencia | Qué demuestra |
|-----|-----------|---------------|
| 1 | Compilación exitosa y pruebas (§3.1) | El proyecto compila y pasa 194 pruebas |
| 2 | `/health` y Swagger UI en Render (§3.3) | El servicio está desplegado y accesible |
| 3 | Estado *Deploy succeeded / Live* (§3.3) | Despliegue exitoso en Render |
| 4 | Esquema y SQL Editor en TiDB (§3.4) | La base de datos remota existe con la tabla `users` |
| 5 | Pruebas contra la URL pública (§3.3) | Login, endpoint protegido (200) y acceso sin token (401) |
| 6 | Pull Request #1 e historial de commits (§3.5, §4) | Integración de Brevo y autoría de los cambios |
| 7 | Brevo: claves e IP autorizadas (§3.5) | Servicio de correo configurado |

## Anexo B — Documentación completa

El detalle de cada paso (comandos, decisiones y capturas adicionales) está en la carpeta [`docs/`](README.md) del repositorio, desde `paso-01-fork-y-clonacion.md` hasta `paso-09-tidb.md`.
