# Paso 01 — Fork y clonación del repositorio (Guía 1)

## Qué se hizo
1. Se creó la carpeta de trabajo `mauricio-vergara/`.
2. Se hizo fork de `https://github.com/arrietajohn/users-management-microservice` a la cuenta `MauricioFonck`, renombrado a `mauricio-users-management-microservice`:
   ```
   gh repo fork arrietajohn/users-management-microservice --fork-name mauricio-users-management-microservice --clone
   ```
3. Se configuró la identidad de Git solo para este repo (config local):
   ```
   git config user.name "MauricioFonck"
   git config user.email "mauricioandresvergarafonseca@gmail.com"
   ```
4. Remotos resultantes: `origin` = tu fork, `upstream` = repo original.

## Prerrequisitos detectados en el equipo
| Herramienta | Estado |
|-------------|--------|
| Git 2.55 | OK |
| Java | 26.0.2.1 instalado; la guía pide JDK 17 (pendiente verificar compatibilidad con el `pom.xml`) |
| Maven | No está en PATH; se usará `mvnw.cmd` (Maven Wrapper) del proyecto |
| Docker | 29.6.2 OK |
| MySQL Server / Workbench | No detectado |
| IntelliJ IDEA | No detectado |

## Capturas
_Pendiente: agregar en `docs/img/` (por ejemplo `01-fork.png`)._
