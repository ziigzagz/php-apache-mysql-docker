# Requirements Document

## Introduction

The production server for the ot-online application runs exclusively on PHP 8.3.36. The local Docker environment currently builds from `php:8.0-apache`, producing a version mismatch that allows code to pass locally while breaking in production. This feature pins the local Docker PHP runtime to the exact production patch version (8.3.36) and applies minimal, targeted patches to the ot-online CodeIgniter 3 application so that it loads without fatal PHP errors under PHP 8.3. The CodeIgniter core/system folder is not upgraded; only the specific breakages that prevent the application from loading are addressed. Success is defined as the application loading cleanly at `localhost:8080` after a container rebuild, matching production PHP 8.3.36 behavior.

## Glossary

- **Docker_Image**: The container image built from `php-apache-mysql-docker/php/Dockerfile`.
- **Dockerfile**: The build file located at `php-apache-mysql-docker/php/Dockerfile`.
- **Production_PHP_Version**: PHP 8.3.36, the exact patch version running on the production server.
- **Target_Base_Image**: The Docker Hub image tag `php:8.3.36-apache`.
- **OT_Online_App**: The CodeIgniter 3 application located at `php-apache-mysql-docker/php/src/ot-online`.
- **CI3_Core**: The bundled CodeIgniter 3 `system/` folder of the OT_Online_App, which may be gitignored and whose contents are confirmed at runtime.
- **Container_Service**: The `php-apache-environment` service defined in `php-apache-mysql-docker/docker-compose.yml`, exposed on host port 8080.
- **Developer**: The person performing the upgrade, rebuild, and verification.
- **Build_System**: The Docker and docker-compose tooling that builds the Docker_Image and runs the Container_Service.
- **Shared_Apps**: The other applications mounted under `php-apache-mysql-docker/php/src` (including MockupKPGW and MockupOPN) that share the same Docker_Image.
- **Fatal_PHP_Error**: A PHP error that halts request execution, including parse errors, uncaught exceptions, and E_ERROR-level failures. Deprecation notices that do not halt execution are not Fatal_PHP_Errors.

## Requirements

### Requirement 1: Pin PHP runtime to production patch version

**User Story:** As a developer, I want the local Docker PHP runtime pinned to the exact production patch version, so that code behaves identically between local and production environments.

#### Acceptance Criteria

1. THE Dockerfile SHALL set the base image to the Target_Base_Image `php:8.3.36-apache`.
2. WHEN the Docker_Image is built, THE Build_System SHALL produce a container whose PHP runtime reports version 8.3.36.
3. IF the Target_Base_Image `php:8.3.36-apache` is not available on Docker Hub, THEN THE Developer SHALL report the unavailability before modifying the Dockerfile.

### Requirement 2: Preserve existing Dockerfile customizations

**User Story:** As a developer, I want the existing Dockerfile customizations preserved, so that file upload limits and database connectivity continue to work after the version change.

#### Acceptance Criteria

1. THE Dockerfile SHALL install and enable the mysqli PHP extension.
2. THE Dockerfile SHALL set `post_max_size` to 32M in the PHP configuration.
3. THE Dockerfile SHALL set `upload_max_filesize` to 32M in the PHP configuration.

### Requirement 3: Minimal targeted patches for PHP 8.3 compatibility

**User Story:** As a developer, I want the ot-online CodeIgniter 3 application patched with minimal targeted changes, so that it runs under PHP 8.3 without a full framework upgrade.

#### Acceptance Criteria

1. WHEN a Fatal_PHP_Error arising from a PHP 8.3 incompatibility is identified in the OT_Online_App, THE Developer SHALL apply a targeted patch to the specific file that produces the error.
2. THE Developer SHALL limit patches to the specific errors and deprecations that prevent the OT_Online_App from loading, including dynamic property deprecation, the removed `each()` function, stricter type handling, and missing `ReturnTypeWillChange` attributes.
3. THE Developer SHALL leave the CI3_Core system folder unchanged except where a targeted patch is required to resolve a Fatal_PHP_Error that prevents the OT_Online_App from loading.
4. WHEN the actual CodeIgniter 3 version and breaking files are confirmed at runtime, THE Developer SHALL base each patch on the confirmed runtime source files.

### Requirement 4: Shared image safety and patch scoping

**User Story:** As a developer, I want the shared Docker image change to remain safe for the other mounted applications, so that changing the base image does not silently break unrelated apps.

#### Acceptance Criteria

1. THE Developer SHALL document that the Docker_Image change applies to all Shared_Apps mounted under the source directory.
2. THE Developer SHALL scope application-code patches to the OT_Online_App.

### Requirement 5: Verification by rebuild and load

**User Story:** As a developer, I want the application verified by rebuilding and loading it, so that I can confirm it matches production PHP 8.3.36 behavior.

#### Acceptance Criteria

1. WHEN the Dockerfile and application patches are complete, THE Developer SHALL rebuild the Container_Service using docker-compose.
2. WHEN the Container_Service is running, THE Developer SHALL load the OT_Online_App at `localhost:8080`.
3. WHILE Fatal_PHP_Errors appear in the container logs when loading the OT_Online_App, THE Developer SHALL apply additional targeted patches and rebuild until no Fatal_PHP_Error remains.
4. THE OT_Online_App SHALL load at `localhost:8080` without a Fatal_PHP_Error on a PHP 8.3.36 runtime.
