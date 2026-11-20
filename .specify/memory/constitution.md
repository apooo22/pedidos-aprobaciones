# Constitución del Sistema de Pedidos y Aprobaciones (Order & Approval System)

## Core Principles

### I. Test-First (NO NEGOCIABLE)

**Declaración normativa**

- Todo código de producción DEBE estar precedido por una prueba que falle, siguiendo el ciclo
  Red → Green → Refactor.
- El orden de escritura DEBE ser: pruebas de contrato → pruebas de integración → pruebas
  unitarias de dominio → implementación.
- Las tareas generadas por `/speckit.tasks` DEBEN colocar cada prueba antes de la tarea de
  implementación que la satisface.
- El stack de pruebas DEBE ser: JUnit 6 (Jupiter), AssertJ, Mockito, Spring Boot Test,
  Testcontainers con PostgreSQL 16 real, RestAssured o `MockMvcTester` para la capa HTTP y
  ArchUnit para reglas de arquitectura. NO DEBE usarse H2 ni ninguna base de datos embebida.
- Las pruebas de dominio NO DEBEN depender de Spring ni de la base de datos.
- Las pruebas de integración con Testcontainers DEBEN residir en el source set `integrationTest`,
  declarado con JVM Test Suites de Gradle (`testing { suites { ... } }`); la tarea `check` DEBE
  depender de él.
- La cobertura medida con el plugin `jacoco` DEBE ser ≥ 90 % (líneas y ramas) en `:domain` y
  `:application`, y ≥ 80 % global. `jacocoTestCoverageVerification` DEBE estar encadenada a
  `check` y el build DEBE fallar por debajo de esos umbrales.
- El plugin `info.solidsoft.pitest` DEBE aplicarse al módulo `:domain`.

**Justificación**: el flujo de aprobación es lógica de negocio crítica (dinero, jerarquía,
segregación de funciones); las pruebas previas fijan el comportamiento esperado antes de
codificar y las pruebas de mutación demuestran que las aserciones detectan defectos reales.

**Criterio de verificación**

- `./gradlew check` ejecuta `test`, `integrationTest` y `jacocoTestCoverageVerification`, y
  falla si la cobertura no alcanza los umbrales.
- `./gradlew :domain:pitest` genera informe de mutación en cada PR.
- El historial de la PR muestra la prueba (en rojo) antes o junto con su implementación; el
  `tasks.md` lista pruebas antes de implementación.
- Una regla ArchUnit o la ausencia de dependencias en `:domain` demuestra que las pruebas de
  dominio no cargan Spring ni JPA.

### II. API-First

**Declaración normativa**

- El contrato OpenAPI 3.1 en `adapters/rest-in/src/main/resources/openapi/*.yaml` DEBE
  diseñarse, revisarse y versionarse ANTES de cualquier implementación del endpoint.
- Las interfaces y DTOs DEBEN generarse desde el contrato con el plugin `org.openapi.generator`
  (tarea `openApiGenerate`; `generatorName = "spring"`, `interfaceOnly = true`,
  `useJakartaEe = true`, `useTags = true` y la opción de Spring Boot compatible con la
  versión 4 que soporte el generador, p. ej. `useSpringBoot3 = true`). La salida DEBE ir a
  `build/generated/openapi` y registrarse como source set.
- `compileJava` DEBE depender de `openApiGenerate`. El código generado NO DEBE versionarse.
  NO DEBEN escribirse a mano controladores que diverjan del contrato.
- Convenciones REST obligatorias: recursos en plural, rutas en kebab-case, versionado en URI
  (`/api/v1`), paginación estándar (`page`, `size`, `sort` con metadatos de página) y cabecera
  `Idempotency-Key` en los POST críticos (crear, enviar, aprobar, rechazar, devolver, cancelar).
- Los errores DEBEN seguir RFC 9457 (Problem Details, `application/problem+json`) e incluir un
  código de error de negocio propio y estable.
- Un cambio incompatible en el contrato DEBE publicarse como nueva versión mayor de la API
  (`/api/v2`); `openapi-diff` en CI DEBE bloquear cambios incompatibles en la versión vigente.
- Las pruebas de contrato DEBEN validar las respuestas reales contra el esquema OpenAPI.

**Justificación**: el contrato es la fuente única de verdad para consumidores y productores;
generarlo elimina la deriva entre documentación y código.

**Criterio de verificación**

- `git ls-files build/` está vacío y `compileJava` depende de `openApiGenerate`.
- Cada controlador en `:adapters:rest-in` implementa una interfaz generada (regla ArchUnit).
- El job de CI `openapi-diff` contra la rama principal está en verde.
- Las pruebas de contrato fallan si una respuesta no cumple el esquema.

### III. Arquitectura Hexagonal (Puertos y Adaptadores)

**Declaración normativa**

- El proyecto DEBE ser un build Gradle multi-módulo cuyas dependencias hagan cumplir la
  arquitectura en compilación:
  - `:domain` — plugin `java-library`. Entidades, value objects, agregados (`Pedido`,
    `Aprobacion`, `FlujoAprobacion`), eventos de dominio y excepciones de negocio. Java puro:
    NO DEBE depender de Spring, JPA ni Jackson.
  - `:application` — depende solo de `:domain`. Un caso de uso por clase, puertos de entrada
    (`in`) y de salida (`out`) y orquestación transaccional.
  - `:adapters:rest-in` — depende de `:application` y del código OpenAPI generado; implementa
    las interfaces generadas.
  - `:adapters:persistence-out` — depende de `:application`, Spring Data JPA y Flyway.
  - `:bootstrap` — ÚNICO módulo con el plugin `org.springframework.boot`; ensambla los
    adaptadores y produce el jar ejecutable.
- Las dependencias entre módulos DEBEN declararse con `implementation`; usar `api` requiere
  justificación escrita en el plan.
- Las dependencias DEBEN apuntar siempre hacia el dominio. ArchUnit DEBE verificar las reglas de
  paquetes como segunda línea de defensa.
- Las entidades JPA NO DEBEN ser el modelo de dominio; se mapean con MapStruct o mappers
  explícitos.
- La máquina de estados del pedido y las reglas de aprobación DEBEN residir en el dominio, con
  invariantes protegidas por el agregado raíz.
- La descomposición DEBE guiarse por DDD táctico: agregados, repositorios como puertos de
  salida y eventos de dominio.

**Justificación**: aislar el núcleo de negocio de frameworks permite probarlo sin
infraestructura, sustituir adaptadores y evitar que detalles técnicos contaminen las reglas.

**Criterio de verificación**

- `:domain/build.gradle.kts` no declara dependencias de Spring, Jakarta Persistence ni Jackson.
- Solo `:bootstrap` aplica `org.springframework.boot`.
- Las pruebas ArchUnit (capas, ciclos, ubicación de `@Entity`) pasan en `check`.
- Ningún tipo anotado con `@Entity` es referenciado desde `:domain` o `:application`.

### IV. Stack Tecnológico Obligatorio

**Declaración normativa**

- Java 25 (LTS): records para value objects y DTOs, sealed interfaces para estados y
  resultados, pattern matching en `switch` y virtual threads habilitados
  (`spring.threads.virtual.enabled=true`). NO DEBEN usarse APIs preview en producción.
- Spring Boot 4.x (Spring Framework 7, Jakarta EE 11) con Web MVC, Validation, Data JPA,
  Security (OAuth2 Resource Server con JWT) y Actuator.
- PostgreSQL 16 con esquema gestionado EXCLUSIVAMENTE por Flyway (migraciones versionadas e
  inmutables una vez fusionadas). `spring.jpa.hibernate.ddl-auto` NO DEBE tener un valor
  distinto de `validate`.
- Gradle 9.x con Kotlin DSL (`build.gradle.kts`, `settings.gradle.kts`). El Gradle Wrapper DEBE
  versionarse y validarse con `gradle/actions/wrapper-validation`.
- Java Toolchains obligatorio:
  `java { toolchain { languageVersion = JavaLanguageVersion.of(25) } }`.
- Todas las versiones DEBEN centralizarse en `gradle/libs.versions.toml`; NO DEBEN existir
  versiones hardcodeadas en los build scripts.
- La convención común (Java 25, JaCoCo, Spotless, pruebas) DEBE compartirse mediante convention
  plugins en `build-logic/` (included build). NO DEBEN usarse `allprojects` ni `subprojects`.
- Las dependencias Spring DEBEN alinearse con el BOM de Spring Boot 4 vía
  `platform(libs.spring.boot.bom)` o el plugin `io.spring.dependency-management`.
- Builds reproducibles: `org.gradle.configuration-cache=true` y `org.gradle.caching=true` en
  `gradle.properties`, y dependency locking para las configuraciones de producción.
- Bean Validation obligatoria en el borde; records en lugar de Lombok; `Optional` en lugar de
  `null` en retornos de APIs públicas.

**Justificación**: un stack fijo y actual reduce la variabilidad, aprovecha las capacidades
del lenguaje para modelar el dominio y garantiza builds deterministas.

**Criterio de verificación**

- `./gradlew build --configuration-cache` termina sin advertencias de incompatibilidad.
- Búsqueda de `version = "` o coordenadas con versión literal en `*.gradle.kts` sin resultados.
- Búsqueda de `allprojects`/`subprojects`/`lombok`/`--enable-preview` sin resultados.
- Existen `gradle/wrapper/gradle-wrapper.jar`, `gradle.lockfile` y la validación del wrapper
  en CI; `ddl-auto` es `validate` en todos los perfiles.

### V. Contenerización con Docker

**Declaración normativa**

- El `Dockerfile` DEBE ser multi-stage:
  - Build: `gradle:9-jdk25` (o JDK 25 con `./gradlew`), copiando primero
    `settings.gradle.kts`, `gradle/`, `build-logic/` y los `build.gradle.kts` para aprovechar
    el caché de capas, y ejecutando `./gradlew :bootstrap:bootJar --no-daemon`.
  - Extracción de capas con `java -Djarmode=tools -jar app.jar extract --layers --launcher`.
  - Runtime: JRE 25 mínima (distroless o `eclipse-temurin:25-jre-alpine`), usuario no root y
    `HEALTHCHECK` contra `/actuator/health`.
- Alternativa permitida: `./gradlew bootBuildImage` (Cloud Native Buildpacks), solo si se
  documenta en el plan.
- `docker-compose.yml` DEBE definir el entorno local: app, `postgres:16-alpine` y, si aplica,
  pgAdmin o Mailpit. Las variables de entorno DEBEN residir en `.env` (no versionado; se
  versiona `.env.example`), nunca en el código.
- La configuración DEBE externalizarse (12-Factor). NO DEBE haber secretos en el repositorio.
- Testcontainers DEBE usar la misma imagen y versión de PostgreSQL que producción.

**Justificación**: imágenes pequeñas, reproducibles y sin privilegios reducen superficie de
ataque y garantizan paridad entre desarrollo, pruebas y producción.

**Criterio de verificación**

- `docker build` termina en CI; la imagen final no ejecuta como UID 0 y declara `HEALTHCHECK`.
- `docker compose up` levanta la app y `/actuator/health` responde `UP`.
- La etiqueta de imagen PostgreSQL es idéntica en `docker-compose.yml` y en Testcontainers
  (constante única o propiedad compartida).
- El escaneo de secretos en CI (p. ej. gitleaks) no reporta hallazgos; `.env` está en
  `.gitignore`.

### VI. Seguridad y Auditoría

**Declaración normativa**

- La autenticación DEBE realizarse con JWT (OAuth2 Resource Server) y la autorización DEBE
  combinar rol (`SOLICITANTE`, `APROBADOR`, `ADMINISTRADOR`, `AUDITOR`) con propiedad del
  recurso, mediante `@PreAuthorize` y reglas de dominio.
- Segregación de funciones: un SOLICITANTE NO DEBE poder aprobar su propio pedido. Es una
  invariante del agregado y DEBE tener una prueba de dominio y una de integración.
- Toda transición de estado DEBE registrarse en una tabla de auditoría inmutable (solo
  inserción) con: actor, instante, estado anterior, estado nuevo y comentario.
- Los agregados DEBEN usar control de concurrencia optimista (`@Version` en la entidad de
  persistencia y versión en el agregado) para impedir aprobaciones duplicadas.
- Las entradas DEBEN validarse en el borde, y el diseño DEBE cubrir el OWASP API Security
  Top 10 (en especial BOLA, autenticación rota y consumo ilimitado de recursos).

**Justificación**: el sistema autoriza gasto empresarial; la trazabilidad y la segregación de
funciones son requisitos de control interno y de auditoría.

**Criterio de verificación**

- Existe una prueba que intenta la autoaprobación y espera el error de negocio
  correspondiente.
- Una migración Flyway revoca `UPDATE`/`DELETE` sobre la tabla de auditoría o lo impide con
  trigger; una prueba de integración lo verifica.
- Una prueba de integración con dos aprobaciones concurrentes produce exactamente una
  transición y un conflicto (HTTP 409 Problem Details).
- Cada endpoint tiene pruebas de acceso denegado por rol y por propiedad.

### VII. Observabilidad y Calidad

**Declaración normativa**

- Los logs DEBEN ser JSON estructurado, incluir `correlationId` (MDC) y NO DEBEN contener datos
  sensibles (tokens, credenciales, datos personales).
- Métricas con Micrometer expuestas a Prometheus; trazas con OpenTelemetry.
- Análisis estático con `com.diffplug.spotless` (google-java-format), `com.github.spotbugs` y
  `org.sonarqube`; el quality gate de Sonar DEBE estar en verde.
- El pipeline de CI DEBE ejecutar
  `./gradlew clean build integrationTest jacocoTestCoverageVerification spotlessCheck`,
  `openapi-diff` y el build de la imagen Docker. Todos DEBEN estar en verde para fusionar una PR.
- CI DEBE usar `gradle/actions/setup-gradle` para cachear dependencias.

**Justificación**: un sistema de aprobaciones requiere diagnosticar incidentes por pedido y
por petición; las puertas de calidad automáticas evitan regresiones.

**Criterio de verificación**

- Una prueba verifica que una petición genera logs JSON con `correlationId` propagado.
- `/actuator/prometheus` expone métricas; hay trazas exportadas en el entorno local.
- La protección de rama exige los checks de CI listados en verde.

### VIII. Simplicidad

**Declaración normativa**

- YAGNI: NO DEBEN introducirse microservicios, mensajería ni caché sin un requisito explícito
  en la spec.
- El sistema DEBE comenzar como monolito modular hexagonal.
- Toda complejidad adicional DEBE justificarse en la sección "Complexity Tracking" del plan,
  indicando la alternativa más simple descartada y por qué.

**Justificación**: la complejidad prematura incrementa el coste de cambio sin aportar valor
demostrado.

**Criterio de verificación**

- El Constitution Check del plan lista cada componente de infraestructura con su requisito de
  origen; cada desviación aparece en "Complexity Tracking".
- El repositorio produce un único artefacto desplegable (`:bootstrap`).

## Contexto de Dominio y Restricciones

- **Dominio**: sistema empresarial en el que los usuarios crean pedidos (órdenes de compra o de
  servicio) sujetos a un flujo de aprobación configurable por monto, centro de costo y
  jerarquía organizacional.
- **Estados canónicos del pedido**: `BORRADOR`, `ENVIADO`, `EN_APROBACION`, `APROBADO`,
  `RECHAZADO`, `DEVUELTO`, `CANCELADO`. Flujo de referencia:
  `BORRADOR → ENVIADO → EN_APROBACION → APROBADO | RECHAZADO | DEVUELTO`, con `CANCELADO` como
  estado terminal. Los estados DEBEN modelarse como sealed interface o enum en `:domain` y toda
  transición no permitida DEBE rechazarse con una excepción de negocio.
  TODO(TRANSICIONES_PEDIDO): la tabla completa (destino de `DEVUELTO`, estados desde los que se
  permite `CANCELADO`) se fija en la spec mediante `/speckit-clarify`.
- **Roles**: `SOLICITANTE` (crea y envía sus pedidos), `APROBADOR` (decide en su nivel),
  `ADMINISTRADOR` (configura flujos y reglas), `AUDITOR` (solo lectura, incluida auditoría).
- **Lenguaje ubicuo**: los nombres del dominio (`Pedido`, `Aprobacion`, `FlujoAprobacion`,
  `CentroCosto`, etc.) DEBEN usarse de forma consistente en spec, contrato OpenAPI y código.
- **Importes**: DEBEN representarse con un value object monetario basado en `BigDecimal` y
  moneda explícita; NO DEBE usarse `double`/`float`.

## Flujo de Desarrollo y Quality Gates

1. **specify / clarify**: la spec describe el qué y el porqué, sin detalles de implementación;
   las ambigüedades se resuelven antes del plan.
2. **plan**: DEBE incluir un "Constitution Check" que evalúe los principios I–VIII antes de la
   Fase 0 y de nuevo tras el diseño (Fase 1), y la sección "Complexity Tracking".
3. **contrato**: el YAML OpenAPI se fusiona/revisa antes de cualquier código del endpoint.
4. **tasks**: orden obligatorio por historia: pruebas de contrato → integración → dominio →
   implementación; ninguna tarea de implementación precede a su prueba.
5. **implement**: ciclo Red → Green → Refactor; cada PR referencia la spec y las tareas.
6. **Puertas de fusión**: CI completo en verde (Principio VII), cobertura y mutación según el
   Principio I, `openapi-diff` sin rupturas y revisión de cumplimiento constitucional aprobada
   por al menos un revisor.

## Governance

- **Supremacía**: esta constitución prevalece sobre cualquier otra práctica, guía o
  preferencia. Ante conflicto, se aplica la constitución o se enmienda formalmente.
- **Constitution Check**: `/speckit.plan` DEBE verificar cada principio antes y después del
  diseño; una violación no justificada bloquea el avance a `/speckit.tasks`.
- **Proceso de enmienda**: toda enmienda DEBE presentarse como PR que documente motivo, impacto
  (principios, plantillas y código afectados) y plan de migración; requiere aprobación del
  responsable técnico y actualización del Sync Impact Report.
- **Versionado semántico**:
  - MAJOR: se elimina o redefine de forma incompatible un principio o regla de gobernanza.
  - MINOR: se agrega un principio o sección, o se amplía materialmente una guía.
  - PATCH: aclaraciones, redacción o correcciones no semánticas.
- **Cumplimiento**: cada PR DEBE incluir la revisión de cumplimiento; se realiza una auditoría
  trimestral de la constitución y de su aplicación, registrando hallazgos y acciones.
- **Plantillas dependientes**: `plan-template.md`, `spec-template.md` y `tasks-template.md`
  DEBEN mantenerse alineadas con esta constitución tras cada enmienda.

**Version**: 1.0.0 | **Ratified**: 2026-11-20 | **Last Amended**: 2026-11-20
