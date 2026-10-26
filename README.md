# MR Attorneys Inc website

Website for MR Attorneys Inc, a law firm at 265 Pretorius Street, Pretoria Central.
A static front end plus a small Java (Spring Boot) back end that validates the
contact form and emails each enquiry to the firm.

Live site: https://www.mathewrasekgwaloattorneysinc.co.za

## Repository layout

```
backend/                       Spring Boot API (Java 21, Maven wrapper)
  Dockerfile                   Multi-stage build used by Render
  pom.xml
  src/main/java/com/mrattorneys/contactapi/
    ContactApiApplication.java Entry point
    ContactController.java     POST /api/contact, CORS, honeypot check
    ContactRequest.java        Request record + validation rules
    EmailService.java          Sends the enquiry through JavaMailSender
  src/main/resources/application.properties
  src/test/                    Default context-load test
frontend/                      Static site published as-is
  index.html                   Page, SEO tags, JSON-LD structured data, form
  script.js                    Form submit handler (API_URL lives here)
  style.css
  favicon.svg, favicon-32.png, apple-touch-icon.png
  robots.txt, sitemap.xml
  google<token>.html           Search Console ownership file (do not delete)
```

## Stack

- Java 21, Spring Boot 4.1.x, Maven wrapper (`mvnw`, `mvnw.cmd`)
- Dependencies: Spring Web, Validation, Java Mail Sender
- Email delivery: Brevo SMTP relay
- Hosting: Render (static site + Docker web service)
- Domain: domains.co.za

## API

`POST /api/contact` with `Content-Type: application/json`

| Field | Rules |
| --- | --- |
| `name` | required, max 100 characters |
| `email` | required, valid email, max 200 |
| `phone` | optional, max 30 |
| `message` | required, max 2000 |
| `hpField` | honeypot, must be empty (see below) |

Responses:

- `200 {"status":"received"}` the enquiry was emailed (or silently dropped, see honeypot)
- `400` validation failed (blank or oversize field, invalid email)
- `500 {"status":"error"}` the email could not be sent; the cause is logged as
  `Email failed: ...`

Example:

```
curl -X POST https://mr-attorneys-api.onrender.com/api/contact \
  -H "Content-Type: application/json" \
  -d '{"name":"Test","email":"test@example.com","phone":"123","message":"Hello"}'
```

### Spam protection

The form contains a hidden input named `hpField`, positioned off-screen and
excluded from tab order. People never fill it in; simple bots often do. If it is
non-blank, the controller returns `200 received` without sending anything, so the
bot gets no signal that it was caught. Field length limits cap the size of
submissions. There is no rate limiting or CAPTCHA; if spam becomes a problem,
consider Cloudflare Turnstile or a rate limiter.

### CORS

`@CrossOrigin` on `ContactController` lists the origins allowed to call the API:

- `http://localhost:5500` and `http://127.0.0.1:5500` (VS Code Live Server)
- the Render address of the static site
- the production domain, with and without `www`

Origins must match exactly (scheme included, no trailing slash). A new site
address must be added here or the browser will block the form.

## Configuration

The API reads its mail settings from environment variables. Set them in the
Render dashboard (service > Environment). Never commit values, keys or `.env`
files.

| Variable | Purpose |
| --- | --- |
| `MAIL_USERNAME` | Brevo SMTP login (ends in `@smtp-brevo.com`) |
| `MAIL_PASSWORD` | Brevo SMTP key (not an API key) |
| `MAIL_FROM` | Sender address verified in Brevo |
| `MAIL_TO` | Inbox that receives enquiries |

`application.properties` maps these to `spring.mail.*` and `app.mail.*`, uses
`smtp-relay.brevo.com` on port **2525** with STARTTLS, and sets
`server.port=${PORT:8080}` so it follows the port Render assigns.

Email details: `from` is `MAIL_FROM`, `to` is `MAIL_TO`, `reply-to` is the
visitor's address (so replying in the inbox answers them), and the subject is
fixed ("New website inquiry") so visitor input never reaches a header.

Why port 2525: Render's free web services block outbound traffic on the usual SMTP
ports (25, 465, 587). Port 2525 currently works on the free tier. Paid instances
can use 587, and a paid instance also avoids cold starts.

## Hosting and deployment

**API:** Render web service `mr-attorneys-api`, runtime Docker, root directory
`backend`, branch `main`, Frankfurt region. The Dockerfile builds with
`./mvnw -DskipTests package` (the default test boots the whole app and needs the
mail variables) and runs the jar on Eclipse Temurin 21.

**Static site:** Render static site `mr-attorneys-site`, root directory `frontend`,
publish directory `.`, no build command.

Both services auto-deploy when `main` changes. A commit that only touches
`frontend/` redeploys the site; one that touches `backend/` redeploys the API.

**Free tier behaviour:** the free API instance sleeps after about 15 minutes of
inactivity, and the first request afterwards can take 50 seconds or more. The
form shows a generic error if the request fails. Upgrading the API service to a
paid instance (Compute > instance type) removes this and is recommended for
production.

**DNS (domains.co.za):**

- `A` record, root domain, to the IP address shown in Render's Custom Domains
- `CNAME` record, `www`, to `mr-attorneys-site.onrender.com`
- Any AAAA records should be removed; do not delete MX or TXT records
- Render issues the HTTPS certificate automatically; the root domain redirects to `www`

Render's Hobby workspace includes two custom domains (root and `www`).

## Search (SEO)

`frontend/index.html` has the page title, meta description, canonical link, Open
Graph tags and JSON-LD (`LegalService`) with the firm's name, address, phone and
practice areas. `robots.txt` allows crawling and points to `sitemap.xml`. Search
Console ownership is verified with the HTML file in `frontend/`; keep it in place.
Keep the phone number and address consistent between the page, the JSON-LD and
any Google Business Profile.

## Running locally

1. Install JDK 21.
2. In a terminal, set the four environment variables, for example in PowerShell:
   `$env:MAIL_USERNAME="..."` (repeat for the others).
3. From `backend/`, run `./mvnw spring-boot:run` (`mvnw.cmd` on Windows).
4. Open `frontend/index.html` with VS Code Live Server (port 5500) and change
   `API_URL` in `frontend/script.js` to `http://localhost:8080/api/contact`.
   Change it back before committing.

## Common tasks

- **New website address:** add it to `@CrossOrigin`, commit, wait for the API deploy.
- **New recipient:** change `MAIL_TO` in Render and let the service restart.
- **New Brevo account:** verify a sender in Brevo, generate an SMTP key, update
  all four variables, send a test, then delete the old key.
- **Edit page content or SEO tags:** edit `frontend/index.html`, then request
  indexing for the home page in Search Console.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Form says "Could not reach the server"; console shows a CORS error | Site address missing or misspelled in `@CrossOrigin`, or API still deploying/asleep |
| Form says "Something went wrong" | API returned 400 or 500; check Render Logs for `Email failed:` |
| `Authentication failed` in logs | Wrong `MAIL_USERNAME` or `MAIL_PASSWORD`, an API key used instead of an SMTP key, or login and key from different Brevo accounts |
| Connection timeout to `smtp-relay.brevo.com` | Outbound SMTP port blocked (use 2525 on the free tier, or upgrade) |
| Build fails with a missing placeholder | A `MAIL_*` variable is not set |
| Certificate error on a custom domain | DNS still propagating, or CAA/AAAA records interfering |

## Ownership and handover notes

- The Brevo account (SMTP credentials) belongs to the client.
- The GitHub repository and Render workspace currently sit under the developer's
  accounts. Render's Hobby plan is single-user; team access needs a paid
  workspace plan, or collaboration can happen through GitHub.
- The domain is registered by the client at domains.co.za; DNS changes need access
  to that account.
