---
inclusion: auto
name: CI3 PHP 8.x Migration Playbook
description: Playbook for running a CodeIgniter 3 (CI3) app on PHP 8.1/8.2/8.3 under Docker (apache + mysqli). Use when upgrading the PHP base image, or fixing "dynamic property is deprecated", "headers already sent", "Swal is not defined" / missing node_modules assets, Windows backslash asset paths, or CI3 routing/case-sensitivity issues on Linux. Based on the ot-online fix.
---

# CI3 on PHP 8.x — Migration & Fix Playbook

Context distilled from fixing the ot-online CodeIgniter 3.1.13 app on php:8.3-apache.
Reuse this when another CI3 project hits the same class of problems.

## 0. Environment assumptions

- App runs in Docker: php:8.x-apache, with the mysqli extension installed and enabled.
- Source is BIND-MOUNTED into the container (e.g. ./php/src -> /var/www/html/), so PHP and
  view edits take effect on the next request WITH NO REBUILD. Only a Dockerfile change needs
  a container image rebuild.
- Linux filesystem is CASE-SENSITIVE and uses "/" as the only path separator.
- ENVIRONMENT = development in index.php means error_reporting(-1), so every deprecation
  notice is echoed into the HTML body. This is what turns harmless notices into visible
  "A PHP Error was encountered" blocks and triggers downstream "headers already sent".

## 1. Pin the PHP base image (Dockerfile)

- Change only the FROM line to the exact production patch version, e.g. FROM php:8.3.36-apache.
- Preserve verbatim: the "docker-php-ext-install mysqli && docker-php-ext-enable mysqli" layer
  and any post_max_size / upload_max_filesize custom.ini lines.
- Confirm the tag is pullable first (docker pull php:8.3.36-apache) before editing.
- Rebuild the container image, then start it.
- Verify inside the container: php -v (exact version), php -m shows mysqli,
  php -i shows post_max_size and upload_max_filesize at the expected values.

## 2. Confirm the actual CI3 version from the running source

- Read system/core/CodeIgniter.php -> const CI_VERSION. Do NOT assume a release.
  (ot-online = 3.1.13.) Base every patch on the on-disk runtime source file.

## 3. "Creation of dynamic property X::$y is deprecated" (PHP 8.2+) — THE BIG ONE

Root cause: CI3 magic-loads libraries/models onto controllers/models as undeclared
properties. PHP 8.2 deprecates dynamic properties. In development mode the notice is echoed,
output starts early, then any header() / redirect / session_start() in a controller
__construct() fails with "Cannot modify header information - headers already sent", so the
page hangs showing a wall of errors (this is why e.g. an auth redirect silently breaks).

Fix (least invasive, per erroring class — NOT a blanket core edit).
Add the attribute immediately above the class declaration:

    #[\AllowDynamicProperties]
    class Admin extends CI_Controller { ... }

Apply to EVERY class that magic-loads properties, which in practice means:

- ALL controllers in application/controllers/*.php (each class X extends CI_Controller).
- ALL models in application/models/*.php (each class X extends CI_Model).
- The CI core classes that error on the entry route: CI_URI (system/core/URI.php),
  CI_Router (system/core/Router.php), CI_Loader (system/core/Loader.php), and the active DB
  driver, e.g. CI_DB_mysqli_driver (system/database/drivers/mysqli/mysqli_driver.php).

PITFALL: Patching only the default controller (e.g. Home) makes the landing page look clean,
but every OTHER controller route still breaks. Patch them ALL. Verify each route WITHOUT the
"index.php/" prefix (that prefix can mask the real behavior) — hit the real rewritten URL,
e.g. http://localhost:8080/\<app\>/Admin.

Idempotent bulk-apply (run in the controllers dir; repeat for models using CI_Model):

    for f in *.php; do
      cls=$(basename "$f" .php)
      perl -0pi -e "s/(?<!AllowDynamicProperties\]\n)class ${cls} extends CI_Controller/#[\\\\AllowDynamicProperties]\nclass ${cls} extends CI_Controller/" "$f"
    done

Alternative (broader, less surgical): add #[\AllowDynamicProperties] to the base
CI_Controller / CI_Model in system/core once. The per-class approach above was preferred here.

Always run "php -l" on each edited file afterward.

## 4. "headers already sent" in a controller __construct

- This is almost always a SYMPTOM of section 3 (a deprecation was echoed before a header()/
  redirect/session call). Fix the dynamic-property notices and this disappears.
- Typical trigger lines: header("Refresh:0; url=".base_url()."Login/...") or session_start()
  inside __construct().

## 5. "Swal is not defined" / missing JS & CSS (node_modules 404s)

Two independent causes, both must be fixed:

a) node_modules not installed. It is gitignored. On a fresh checkout the local libraries
   (sweetalert2, datatables, apexcharts, highcharts, jszip, pdfmake, moment, ...) are 404.
   Install them (host usually has Node; the container usually does NOT):

       cd <app-root>   # where package.json lives
       npm install --no-audit --no-fund

   Because source is bind-mounted, the files appear in the container immediately. Remember to
   run npm install on production too (node_modules is not in git). Projects often ship a
   "คำสั่งติดตั้ง.txt" / "ติดตั้ง module.bat" note for this.

b) Windows backslash paths in views. application/views/css.php and js.php (and other views)
   reference assets like node_modules\sweetalert2\dist\sweetalert2.min.js. Backslashes are NOT
   path separators on Apache/Linux, so the browser requests a bad URL. Convert all backslashes
   to forward slashes in those include files:

       perl -pi -e 's/\\/\//g' application/views/css.php application/views/js.php

   (Safe here because those files are pure lists of link/script paths.)

Verify over HTTP (expect 200):

    curl -s -o /dev/null -w "%{http_code}\n" \
      "http://localhost:8080/<app>/node_modules/sweetalert2/dist/sweetalert2.min.js"

## 6. CI3 routing & URL gotchas on Linux

- The .htaccess rewrite "RewriteRule ^(.*)$ index.php/$1 [L]" requires Apache mod_rewrite
  enabled and AllowOverride All. With $config['index_page'] = '' the app EXPECTS clean URLs,
  so mod_rewrite must work.
- Case sensitivity: base_url and link casing must match the real folder/route case (e.g. use
  "ot-online", not "Ot_Online") or assets/routes 404 on Linux.
- A controller with NO index() method returns a CI3 404 at /<app>/<Controller> — that is
  expected routing behavior, not a PHP crash.
- base_url is often host-conditional in config.php; keep the localhost branch pointing at the
  Docker port, e.g. http://localhost:8080/\<app\>/.

## 7. Other PHP 8.x breakages to scan for (empirical, log-driven)

Reload the route, read the response body plus the container logs, fix the first FATAL, repeat.
Only true fatals gate completion (parse error, uncaught Error/TypeError, call to undefined).
Candidates:

- each() removed in PHP 8.0 -> rewrite with foreach / current()+next()+key().
- Passing null to string funcs (trim, strlen, htmlspecialchars, explode, preg_*) is deprecated
  in 8.1, or a TypeError if strict -> null-coalesce the specific argument: $x ?? ''.
- Internal-interface methods (ArrayAccess, Iterator, Countable, JsonSerializable) lacking a
  native return type -> add #[\ReturnTypeWillChange] on that method. (ot-online had none, but
  other CI3 apps / third-party libraries do.)

## 8. Verification workflow (host shell may hang after commands)

The host shell in this environment intermittently hangs after a command completes. Work around
it: run the command in the background writing to a file, then read the file with a file-reading
tool instead of relying on shell stdout.

    (curl -s -m 20 "http://localhost:8080/<app>/Admin" -o /tmp/resp.html \
      -w "%{http_code}\n" > /tmp/out.txt 2>&1; echo DONE >> /tmp/out.txt) &

Then inspect /tmp/resp.html for: "A PHP Error was encountered", "headers already sent",
"Fatal error", "Uncaught", "Call to a member function ... on null". Zero of these = clean.

## 9. Scope discipline

- Keep application patches inside the one app folder (e.g. php/src/ot-online).
- A shared base-image change does move sibling apps to the new PHP too (expected), but do not
  edit sibling apps' code.
- node_modules/ stays gitignored; don't commit it.
