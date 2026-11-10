# Implementation Plan: PHP 8.3 Upgrade & ot-online Fix

## Overview

Pin the local Docker PHP runtime to the exact production patch version (`php:8.3.36-apache`) and apply minimal, log-driven patches to the ot-online CodeIgniter 3 app so it loads without fatal PHP errors. Work proceeds in two tracks: a one-line base-image change in `php/Dockerfile` (shared image), then an iterative patch loop confined to `php/src/ot-online`. Each step builds on the previous: confirm the image is pullable, change the base image, rebuild and verify the runtime, then patch the app until it loads cleanly, and finally audit that the diff is scoped to ot-online.

## Tasks

- [x] 1. Pre-flight: confirm the target base image is pullable
  - Run `docker-compose build` (or `docker pull php:8.3.36-apache`) against the current context to confirm the `php:8.3.36-apache` tag resolves on Docker Hub
  - IF the tag returns a manifest/not-found error, STOP: report the unavailability and do not modify the Dockerfile or silently substitute another tag
  - Record the confirmation (tag pullable) before proceeding to task 2
  - _Requirements: 1.3_

- [x] 2. Change the Dockerfile base image to the production patch version
  - [x] 2.1 Edit `php/Dockerfile` FROM line
    - Replace `FROM php:8.0-apache` with `FROM php:8.3.36-apache`
    - Preserve verbatim the `docker-php-ext-install mysqli && docker-php-ext-enable mysqli` layer
    - Preserve verbatim the `post_max_size = 32M` and `upload_max_filesize = 32M` lines written to `conf.d/custom.ini`
    - Do not alter the build context or any other layer
    - _Requirements: 1.1, 2.1, 2.2, 2.3_

- [x] 3. Rebuild the container and verify the runtime
  - [x] 3.1 Rebuild and start the Container_Service
    - Run `docker-compose build` then `docker-compose up -d` for `php-apache-environment`
    - IF the build fails to pull the base image, return to task 1 and report
    - _Requirements: 5.1_
  - [ ]* 3.2 Smoke-check the PHP runtime version
    - Run `php -v` in the container; assert it reports exactly `8.3.36`
    - **Property 1: PHP runtime reports the production patch version**
    - **Validates: Requirements 1.1, 1.2**
  - [ ]* 3.3 Smoke-check preserved customizations
    - Run `php -m` and assert `mysqli` is listed
    - Run `php -i | grep -E 'post_max_size|upload_max_filesize'` (or inspect `conf.d/custom.ini`) and assert both report `32M`
    - **Property 2: Dockerfile customizations are preserved across the version change**
    - **Validates: Requirements 2.1, 2.2, 2.3**

- [x] 4. Checkpoint - runtime parity confirmed
  - Ensure the container runs PHP 8.3.36 with mysqli and 32M limits intact, ask the user if questions arise.

- [x] 5. Confirm the running CI3 source before patching
  - [x] 5.1 Record the actual CI3 version and entry path from the container
    - Load `http://localhost:8080` (and the ot-online entry route) and inspect `docker-compose logs php-apache-environment`
    - Read the CI3 version from the running container's `ot-online/system` core (do not assume a release); record it once for the patch loop
    - Base every subsequent patch on the confirmed runtime source file, not an assumed CI3 release
    - _Requirements: 3.4, 5.2_

- [x] 6. Iterative CI3 compatibility patch loop (repeat until no Fatal_PHP_Error)
  - [x] 6.1 Patch removed `each()` calls
    - For each fatal `each()` error, rewrite only the erroring line/loop using `foreach` / `current()` + `next()` + `key()` in the confirmed ot-online source file
    - Reload `localhost:8080`, inspect logs, repeat for each occurrence that surfaces
    - _Requirements: 3.1, 3.2, 3.3, 5.3_
  - [x] 6.2 Patch dynamic property creation deprecations
    - For each erroring class, apply the least invasive fix: add `#[\AllowDynamicProperties]` to the specific class, or declare the specific property; per erroring class only, not blanket across the core
    - Reload, inspect logs, repeat
    - _Requirements: 3.1, 3.2, 3.3, 5.3_
  - [x] 6.3 Patch null/stricter-type handling in string functions
    - For each erroring call site (e.g. `trim()`, `strlen()`, `htmlspecialchars()`, `preg_*`, `explode()` receiving `null`), null-coalesce the specific argument (`$x ?? ''`) at that site only
    - Reload, inspect logs, repeat
    - _Requirements: 3.1, 3.2, 3.3, 5.3_
  - [x] 6.4 Patch missing `ReturnTypeWillChange` attributes
    - For each internal-interface method (`ArrayAccess`, `Iterator`, `Countable`, `JsonSerializable`) lacking a native return type that triggers a fatal, add `#[\ReturnTypeWillChange]` above the specific implementing method
    - Reload, inspect logs, repeat
    - _Requirements: 3.1, 3.2, 3.3, 5.3_
  - [ ]* 6.5 Integration check: clean load with no fatal errors
    - GET `http://localhost:8080` and the ot-online entry route; assert a non-5xx response and zero `Fatal_PHP_Error` entries in `docker-compose logs`
    - This is the terminating condition of the loop; residual non-halting deprecation notices are acceptable
    - **Property 3: Application loads without fatal errors on the production runtime**
    - **Validates: Requirements 5.2, 5.3, 5.4**

- [ ] 7. Scope audit - confirm patches are confined to ot-online
  - [ ]* 7.1 Review the diff for scope confinement
    - Run `git status` / `git diff` and assert every changed application file resides under `php/src/ot-online`
    - Assert no file under MockupKPGW or MockupOPN was modified
    - **Property 4: Patch scope is confined to ot-online**
    - **Validates: Requirements 3.1, 3.3, 4.2**

- [x] 8. Final checkpoint - ensure the app loads cleanly
  - Ensure ot-online loads at `localhost:8080` with no Fatal_PHP_Error and the diff is scoped to ot-online, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional (smoke/integration/scope verification) and can be skipped for a faster MVP, but they are the behavioral checks that confirm the four correctness properties.
- The base-image change is shared: MockupKPGW and MockupOPN also move to PHP 8.3.36 (expected impact, Requirement 4.1), but code patches here stay inside ot-online (Requirement 4.2).
- Application patches take effect on the next request without an image rebuild (source is bind-mounted); only the Dockerfile change in task 2 requires a rebuild.
- The patch loop (task 6) is empirical and ordered by whichever fatal error surfaces first; the categories (6.1–6.4) are candidates to recognize in logs, not a fixed sequence. Only `Fatal_PHP_Error` gates completion.
- Each patch must be derived from the confirmed runtime source file (task 5), not an assumed CI3 release.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["2.1"] },
    { "id": 1, "tasks": ["3.1"] },
    { "id": 2, "tasks": ["3.2", "3.3"] },
    { "id": 3, "tasks": ["5.1"] },
    { "id": 4, "tasks": ["6.1"] },
    { "id": 5, "tasks": ["6.2"] },
    { "id": 6, "tasks": ["6.3"] },
    { "id": 7, "tasks": ["6.4"] },
    { "id": 8, "tasks": ["6.5"] },
    { "id": 9, "tasks": ["7.1"] }
  ]
}
```
