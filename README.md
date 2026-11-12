# The Collision Justice

The full-stack application has two parts:

- `client/` — React, Vite, the intake form, request tracking, and administrator UI.
- `server/` — Laravel 12, PostgreSQL, encrypted lead records, administrator authentication, intake email notifications, and PDFs.

The public pages are implemented in React. The Laravel API is a separate backend; no CMS or CMS plugin is required. The source theme files were not in the supplied project, so the public pages are a recreation rather than a line-by-line theme conversion.

The public pages include a spring-follow cursor accent, subtle pointer-based hero motion, and scroll reveals. The browser's native cursor remains enabled, touch layouts do not get a custom cursor, and system reduced-motion settings disable the effects.

## Local development

Requirements: Node.js 20+, PHP 8.2+, Composer, PostgreSQL, and PHP extensions `openssl`, `pdo_pgsql`, `mbstring`, `xml`, and `dom`.

1. Create a PostgreSQL database and dedicated database user.
2. In `server/`, copy `.env.example` to `.env`, set the database credentials and `APP_URL`, and set `SESSION_SECURE_COOKIE=false` **only for local HTTP development**. For a local PostgreSQL instance without a TLS certificate, set `DB_SSLMODE=prefer`; use verified TLS for a remote database.
3. Install and initialize the server:

   ```powershell
   cd server
   composer install
   php artisan key:generate
   php artisan migrate
   php artisan app:create-admin admin@example.com "Site Administrator"
   php artisan serve --host 127.0.0.1
   ```

   The admin command prompts for a password; use 12 to 72 bytes. This creates the initial owner account; there is no shared or default login. After signing in at `/admin/login`, the owner can create editor and viewer accounts. Editors can view and update lead details; viewers can only view leads. Only the owner can create accounts, and additional owners must be provisioned from the server console.
4. In `client/`, copy `.env.example` to `.env`. Install and run the client:

   ```powershell
   cd client
   npm install
   npm run dev
   ```

   Vite proxies `/api` requests to the local Laravel server. The main website opens at `/`; use `/about-us`, `/faq`, `/contact-us`, and `/get-a-quote` for the other primary pages. `/intake` is an alias for the form, `/track` checks a Lead ID, and `/admin/login` opens the administrator sign-in.
5. Open the Vite URL and use `/admin/login` for the administrator sign-in.

## Company branding

The company logo and accident images are included in `client/public/`; the PDF receipt uses a print-friendly JPEG version of the logo from the same directory. The home page also embeds a map to **310 Oakley Avenue, Lynchburg, VA 24501** and provides a directions link. The map is loaded from Google Maps and is allowed by the server's content-security policy.

## Intake email

Every accepted intake is encrypted and saved in PostgreSQL, then the full submission—including contact, accident, and consent details—is sent to `ADMIN_EMAIL` (defaults to `Info@thecollisionjustice.com`). The customer receipt reports whether the notification was sent; if mail is unavailable, the lead remains saved and the receipt tells the customer to call with their Lead ID.

Configure `MAIL_HOST`, `MAIL_PORT`, `MAIL_SCHEME`, `MAIL_USERNAME`, `MAIL_PASSWORD`, and `MAIL_FROM_ADDRESS` for the company's authenticated SMTP provider. Set `ADMIN_EMAIL` to the inbox that should receive leads. Use the provider's verified TLS settings and sender domain. Do not send real customer submissions through an unconfigured development mail server.

## Jornaya and TrustedForm

The intake form can load the account-specific Jornaya Create campaign and TrustedForm Certify scripts. After those accounts are approved, copy `client/.env.example` to `client/.env` and set:

- `VITE_JORNAYA_CAMPAIGN_KEY` to the lowercase campaign UUID from the Jornaya web campaign.
- `VITE_TRUSTEDFORM_SCRIPT_URL` to the HTTPS `api.trustedform.com` script URL from the TrustedForm Certify setup snippet. Configure the script to use the `xxTrustedFormCertUrl` field and tagged consent.

These are public browser-script settings, not API secrets. Never put vendor passwords or private API keys in `VITE_` variables. Rebuild the client after changing them. The scripts load after the intake form is mounted; the form tags the opt-in language, checkbox, and submit action for consent capture and masks customer-entered fields as sensitive. The server validates and stores the TrustedForm URL and Jornaya token inside the encrypted lead data. Admin details, CSV exports, and notification emails include both IDs. The customer PDF includes the logo, generated `TCJ-...` Lead ID, and clearly labeled Jornaya LeadID and TrustedForm certificate fields; either vendor field reads "Not collected" if no ID was submitted. The Jornaya ID is only collected when the campaign key is configured and the vendor script successfully populates the form's LeadID field.

Submissions remain accepted if either vendor script fails or produces no ID. Missing IDs are flagged for review in the admin panel and shown as not collected in the notification email. Verify both integrations using each vendor's official implementation self-test before sending real leads.

The existing consent wording is preserved and tagged as the offer's consent language and opt-in. Because it refers to Collision Justice and independently retained attorneys, confirm the exact advertiser-name disclosure/tagging with your compliance counsel and lead partners; the integration does not guess or rewrite legal consent language.

## VPS deployment

Build the client (`npm ci && npm run build` in `client/`), install PHP dependencies (`composer install --no-dev --optimize-autoloader` in `server/`), configure PostgreSQL and authenticated SMTP, then set a private production `server/.env`. Generate a unique `APP_KEY`; never copy it into source control. Keep `APP_DEBUG=false`, `SESSION_SECURE_COOKIE=true`, and use a database account with access only to this application's database.

Configure PHP-FPM and Caddy using `server/deploy/Caddyfile`. Replace the example hostname and filesystem/FPM paths. Caddy obtains and renews HTTPS certificates; only expose ports 80/443 publicly and restrict PostgreSQL/PHP-FPM to trusted local/private interfaces. Then run `php artisan migrate --force`, `php artisan config:cache`, and `php artisan view:cache`. Do not use `php artisan serve` in production.

## Security and operations

- Laravel's session authentication uses server-side, encrypted sessions and `Secure`, `HttpOnly`, `SameSite=Lax` cookies. Login passwords are never placed in cookies or local storage.
- The admin panel requires an explicitly provisioned account. Owners can create editor and viewer accounts; editors can update lead details, while viewers are read-only. Server middleware enforces these permissions, and submitted consent records are preserved during lead edits. Login and public submission/tracking routes are rate-limited; form validation is enforced on the server.
- Intake data is encrypted at rest with Laravel's authenticated encryption cast. Keep `APP_KEY` private, back it up separately, and plan key rotation before changing it; losing the key makes stored lead details unreadable.
- Database fields are encrypted, so admin search intentionally supports Lead ID and status rather than plaintext PII.
- PDF bearer links and administrator PDFs use no-store responses. Administrator CSV exports require an authenticated session and neutralize spreadsheet formula prefixes.
- Configure and verify authenticated TLS SMTP before launch. Intake notification emails contain full contact, accident, and consent details; restrict the recipient mailbox and access to it accordingly.
- HTTPS is terminated by Caddy, not by a custom Express server. Keep the application and database patched, monitor failed logins, protect backups, define data retention/deletion, and obtain appropriate privacy/legal review before production use.
- Automated tests are in `server/tests/Feature`. Run them with `composer test` after installing the PHP dependencies. This environment does not include PHP or Composer, so the Laravel runtime and PostgreSQL migration still need to be tested on a PHP-capable machine.
