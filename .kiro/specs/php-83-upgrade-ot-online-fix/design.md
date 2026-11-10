# Design Document

## Overview

This design pins the local Docker PHP runtime to the exact production patch version (`php:8.3.36-apache`) and applies minimal, targeted patches to the ot-online CodeIgniter 3 (CI3) application so it loads without fatal PHP errors under PHP 8.3. Production runs **only** PHP 8.3.36, so the local environment must match that exact patch version to eliminate the class of bugs that pass locally (on PHP 8.0) but break in production.

The work has two tracks:

1. **Image track** — a one-line base-image change in `php/Dockerfile`, preserving existing customizations (mysqli, 32M upload/post limits).
2. **Application track** — an iterative, log-driven patch loop scoped strictly to `php/src/ot-online`. The CI3 `system/` core is not upgraded; only the specific files that produce fatal errors under PHP 8.3 are patched.

Verification is behavioral: rebuild via docker-compose, load `localhost:8080`, inspect container logs, patch the erroring file, repeat until no fatal PHP error remains.

### Base image availability (Requirement 1.3)

The tag `php:8.3.36-apache` follows the official `php` image's standard `{version}-apache` naming convention, and PHP 8.3.36 is a released patch version. A pre-flight check confirmed the tag naming is valid for the official Docker Hub `php` repository. The build step itself is the authoritative availability check: if `docker-compose build` fails to pull `php:8.3.36-apache` (manifest unknown), the Developer stops and reports the unavailability before any Dockerfile change is relied upon, rather than silently falling back to another tag.

## Architecture

```
docker-compose.yml  (php-apache-environment, host 8080 -> container 80)
        |
        | build: ./php/Dockerfile
        v
  Docker_Image  (FROM php:8.3.36-apache)  <-- SHARED by all mounted apps
        |
        | volume: ./php/src  ->  /var/www/html/
        v
  /var/www/html/
     |-- ot-online/     <-- CI3 app, PATCH SCOPE (this feature)
     |-- MockupKPGW/    <-- CI3 app, shares image, NOT patched here
     |-- MockupOPN/     <-- CI3 app, shares image, NOT patched here
```

Key architectural facts:

- A single Docker image serves all apps mounted under `php/src`. Changing the base image is a **shared** change: MockupKPGW and MockupOPN also move from PHP 8.0 to 8.3.36. This is documented as expected impact (Requirement 4.1), but code patches in this feature are confined to `ot-online` (Requirement 4.2).
- The app source is bind-mounted, not baked into the image. Application patches take effect on the next request without an image rebuild; only Dockerfile changes require a rebuild.
- The ot-online `system/` folder exists on disk but is gitignored (its contents are confirmed at runtime per the requirements). Its exact CI3 version and line numbers must be read from the running container, not assumed.

## Components and Interfaces

### Component 1: Dockerfile (image definition)

**Location:** `php-apache-mysql-docker/php/Dockerfile`

**Change:** Replace the base image line only.

```dockerfile
# Before
FROM php:8.0-apache
# After
FROM php:8.3.36-apache
```

**Preserved verbatim (Requirement 2):**

```dockerfile
RUN docker-php-ext-install mysqli && docker-php-ext-enable mysqli
RUN apt-get update && apt-get upgrade -y
RUN echo "post_max_size = 32M" >> /usr/local/etc/php/conf.d/custom.ini \
    && echo "upload_max_filesize = 32M" >> /usr/local/etc/php/conf.d/custom.ini
```

`docker-php-ext-install`/`docker-php-ext-enable` and the `conf.d/custom.ini` config path are stable across the official PHP 8.x image line, so these layers carry forward unchanged to 8.3.36.

**Interface / contract:**
- Input: build context `./php`.
- Output: an image whose `php -v` reports `8.3.36`, with mysqli loaded (`php -m | grep mysqli`) and `post_max_size`/`upload_max_filesize` both reporting `32M`.

### Component 2: ot-online application patches (CI3 compatibility)

**Location:** `php-apache-mysql-docker/php/src/ot-online` (and, only where a fatal error originates there, the gitignored `ot-online/system` core).

**Strategy:** Patch the exact file named in each fatal error, nothing more. Each patch is derived from the confirmed runtime source file (read from the container), not from an assumed CI3 release (Requirement 3.4). The confirmed CI3 version is recorded once at the start of the loop (copyright headers suggest the 3.1.11–3.1.13 range; the running container is authoritative).

### Component 3: Verification harness (manual, log-driven)

**Tooling:** docker-compose + container log inspection.

- Rebuild: `docker-compose build` then `docker-compose up -d` (run manually by the Developer; the base-image change requires a rebuild).
- Load: HTTP GET `http://localhost:8080` (and the ot-online entry path).
- Observe: `docker-compose logs php-apache-environment` and the Apache/PHP error stream for fatal errors.

## Data Models

This feature changes configuration and source files, not runtime data. The relevant "models" are the artifacts under change control.

| Artifact | Path | Change type | Scope |
|---|---|---|---|
| Base image tag | `php/Dockerfile` (FROM) | Edit (one line) | Shared image |
| mysqli + ini layers | `php/Dockerfile` | Preserve | Shared image |
| CI3 core files | `ot-online/system/**` | Targeted patch only on fatal error | ot-online only |
| CI3 app files | `ot-online/application/**` | Targeted patch only on fatal error | ot-online only |

### PHP 8.3 breakage categories (expected, to be confirmed at runtime)

CI 3.1.x predates PHP 8.1–8.3. The likely fatal-error categories, each mapped to its minimal patch, are:

1. **Removed `each()` function** — removed in PHP 8.0. Any surviving `each()` call is a fatal `Error`. Patch: rewrite the loop using `foreach` / `current()` + `next()` + `key()` on the specific line.
2. **Dynamic property creation deprecation (PHP 8.2+)** — assigning to undeclared object properties. In CI3 this surfaces across core classes and loaded libraries/models. Deprecations alone are not fatal, but combined with CI3's custom error handler converting notices to exceptions they can halt a request. Patch options, least invasive first: add `#[\AllowDynamicProperties]` to the specific class that errors, or declare the specific property. Patches are applied per erroring class, not blanket across the core.
3. **Stricter type / null-related deprecations in string functions** — e.g. passing `null` to `trim()`, `strlen()`, `htmlspecialchars()`, `preg_*`, `explode()`. Fatal only where CI3's handler escalates them or where a `null` reaches a context that throws a `TypeError`. Patch: null-coalesce the specific argument (`$x ?? ''`) at the erroring call site.
4. **Missing `ReturnTypeWillChange` attributes** — classes implementing internal interfaces (`ArrayAccess`, `Iterator`, `Countable`, `JsonSerializable`) whose methods lack a native return type trigger deprecations in 8.1+. Patch: add `#[\ReturnTypeWillChange]` above the specific implementing method.

These are candidate categories to recognize in logs; the actual set and order are discovered empirically. Only errors that are `Fatal_PHP_Error` (parse errors, uncaught exceptions, E_ERROR) gate completion — non-halting deprecation notices do not.

## Error Handling

### Build-time

- **Base image not pullable:** `docker-compose build` reports a manifest/not-found error for `php:8.3.36-apache`. Handling: stop, report unavailability to the user (Requirement 1.3), do not substitute a different tag silently.
- **Extension/config layer failure:** if `docker-php-ext-install mysqli` or the ini `echo` fails on the new base, surface the build log; the install/enable mechanism and `conf.d` path are expected to be stable on 8.3.36.

### Runtime (the iterative patch loop)

```
build + up  ->  GET localhost:8080  ->  read logs
                                           |
                      no fatal error ------+------ fatal error
                           |                            |
                        DONE                 identify erroring file:line
                                             read that file FROM the container
                                             apply minimal patch (ot-online scope)
                                             reload (rebuild only if Dockerfile changed)
                                             loop
```

- Each iteration addresses exactly one erroring file; after patching, reload to surface the next error (errors are often sequential — the first fatal masks later ones).
- Patches never touch MockupKPGW or MockupOPN source. If a shared-core file under `ot-online/system` is patched, the change lives inside the ot-online copy only and does not affect the other apps' own `system/` folders.
- Termination: the loop ends when a full page load produces no `Fatal_PHP_Error` in the logs (Requirement 5.4). Residual non-fatal deprecation notices are acceptable and out of scope.

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

This feature is a configuration + environment-parity change verified behaviorally (rebuild, load, inspect logs). Most acceptance criteria are infrastructure/setup checks (SMOKE) or one-off process steps rather than input-varying logic, so they are validated by integration/smoke verification rather than property-based tests. The prework below records that classification; the properties that follow are the few statements that hold universally and are worth asserting.

### Property 1: PHP runtime reports the production patch version

For any successful build of the Docker_Image, the resulting container's reported PHP version equals exactly `8.3.36`.

**Validates: Requirements 1.1, 1.2**

### Property 2: Dockerfile customizations are preserved across the version change

For any build of the Docker_Image after the base-image change, the container has the mysqli extension loaded and reports `post_max_size = 32M` and `upload_max_filesize = 32M`.

**Validates: Requirements 2.1, 2.2, 2.3**

### Property 3: Application loads without fatal errors on the production runtime

For any load of the OT_Online_App at `localhost:8080` on the PHP 8.3.36 runtime after the patch loop terminates, no Fatal_PHP_Error appears in the container logs.

**Validates: Requirements 5.2, 5.3, 5.4**

### Property 4: Patch scope is confined to ot-online

For any patch applied in this feature, the modified file resides under `php/src/ot-online`; no file under MockupKPGW or MockupOPN is modified.

**Validates: Requirements 3.1, 3.3, 4.2**

## Testing Strategy

Because the deliverable is environment parity and a cleanly loading app — not a reusable pure-logic module — verification is integration/smoke oriented, not property-based.

### Smoke checks (single execution)

- **Runtime version (Property 1):** after rebuild, run `php -v` in the container; assert `8.3.36`.
- **Extension + limits (Property 2):** run `php -m` (expect `mysqli`) and `php -i | grep -E 'post_max_size|upload_max_filesize'` (expect `32M` each), or inspect `conf.d/custom.ini`.

### Integration checks (1–3 representative loads)

- **Clean load (Property 3):** GET `http://localhost:8080` and the ot-online entry route; assert a non-5xx response and zero `Fatal_PHP_Error` entries in `docker-compose logs`. This is the terminating condition of the patch loop.
- **Scope audit (Property 4):** after the loop, review the diff (`git status` / `git diff`) to confirm all changed application files are under `ot-online`, and that MockupKPGW/MockupOPN sources are untouched.

### Why not property-based testing here

- The base-image and ini settings are one-time configuration: behavior does not vary with input, so 100 randomized iterations add no value over a single assertion (SMOKE).
- "Loads without fatal errors" is verified against the real container and external services (Apache, MySQL), not a pure function; representative loads are the right granularity (INTEGRATION).
- The CI3 patches are corrective edits to specific files, not a new algorithm with a universal input/output contract; their correctness is confirmed by the app loading, not by generated inputs.

### Acceptance Criteria Testing Prework (classification)

- **1.1 / 1.2 base image + reported version** — Classification: SMOKE. One-time build/config fact; asserted once via `php -v`. (Captured as Property 1.)
- **1.3 image unavailability reporting** — Classification: SMOKE/process. Verified by the build step's success/failure and the Developer's stop-and-report action; not input-varying.
- **2.1 / 2.2 / 2.3 mysqli + 32M limits** — Classification: SMOKE. One-time config facts asserted via `php -m` / `php -i`. (Captured as Property 2.)
- **3.1 / 3.2 / 3.3 / 3.4 targeted patches** — Classification: EXAMPLE/process. Each patch is a specific corrective edit confirmed by the app loading and by scope audit; no universal input/output property beyond scope confinement. (Scope captured as Property 4.)
- **4.1 shared-image documentation** — Classification: non-testable (documentation statement; satisfied by this design's Architecture section).
- **4.2 patch scoping to ot-online** — Classification: PROPERTY (holds for every patch). (Captured as Property 4.)
- **5.1 rebuild** — Classification: SMOKE/process step.
- **5.2 / 5.3 / 5.4 load without fatal error** — Classification: INTEGRATION. Verified against the running container; termination condition of the loop. (Captured as Property 3.)
