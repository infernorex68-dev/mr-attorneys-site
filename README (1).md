# MR Attorneys Inc website

Website for MR Attorneys Inc, a law firm in Pretoria Central. A static front end
plus a small Java (Spring Boot) back end that validates the contact form and
emails each enquiry to the firm.

Live site: https://www.mathewrasekgwaloattorneysinc.co.za

## Repository layout

```
backend/                       Spring Boot API (Java 21, Maven wrapper)
  Dockerfile                   Multi-stage build used for deployment
  pom.xml
  src/main/java/com/mrattorneys/contactapi/
    ContactApiApplication.java Entry point
    ContactController.java     POST /api/contact and CORS configuration
    ContactRequest.java        Request record and validation rules
    EmailService.java          Sends the enquiry through JavaMailSender
  src/main/resources/application.properties
  src/test/                    Default context-load test
frontend/                      Static site published as-is
  index.html                   Page, SEO tags, structured data, contact form
  script.js                    Form submit handler (API_URL is set here)
  style.css
  favicon.svg, favicon-32.png, apple-touch-icon.png
  robots.txt, sitemap.xml
```

## Stack

- Java 21, Spring Boot 4.1.x, Maven wrapper (`mvnw`, `mvnw.cmd`)
- Dependencies: Spring Web, Validation, Java Mail Sender
- Email delivery: Brevo SMTP relay
- Hosting: Render (static site and a Docker web service)

## API

`POST /api/contact` with `Content-Type: application/json`

| Field | Rules |
| --- | --- |
| `name` | required, max 100 characters |
| `email` | required, valid email, max 200 |
| `phone` | optional, max 30 |
| `message` | required, max 2000 |

Responses:

- `200 {"status":"received"}` the request was accepted
- `400` validation failed (blank or oversize field, invalid email)
- `500 {"status":"error"}` the email could not be sent; the cause is logged

The form also includes a hidden honeypot field, and field length limits cap the
size of submissions, to filter out simple spam.

### CORS

`@CrossOrigin` on `ContactController` lists the origins allowed to call the API:
the local development addresses, the hosting provider's address for the site, and
the production domain (with and without `www`). Origins must match exactly
(scheme included, no trailing slash). A new site address must be added here, or
the browser will block the form.

## Configuration

Mail settings come from environment variables, set in the hosting dashboard.
Never commit values, keys or `.env` files.

| Variable | Purpose |
| --- | --- |
| `MAIL_USERNAME` | SMTP login |
| `MAIL_PASSWORD` | SMTP key |
| `MAIL_FROM` | Verified sender address |
| `MAIL_TO` | Inbox that receives enquiries |

`application.properties` maps these to `spring.mail.*` and `app.mail.*`, uses
the Brevo SMTP relay on port 2525 with STARTTLS, and sets
`server.port=${PORT:8080}` so the app follows the port the host assigns.

The email's reply-to is the visitor's address, and the subject is fixed so
visitor input never reaches a mail header.

Note on the port: free hosting instances may block the standard SMTP ports (25,
465, 587), and port 2525 works around that. Paid instances can use 587.

## Hosting and deployment

- **API:** Docker web service, root directory `backend`, branch `main`. The
  Dockerfile builds with `./mvnw -DskipTests package` (the default test boots the
  whole app and needs the mail variables) and runs the jar on Eclipse Temurin 21.
- **Static site:** static site service, root directory `frontend`, publish
  directory `.`, no build command.

Both services auto-deploy when `main` changes. A change under `frontend/`
redeploys the site; a change under `backend/` redeploys the API.

On a free instance, the API sleeps after a period of inactivity, and the first
request afterwards can be slow. A paid instance avoids this and is recommended for
production.

**DNS:** an `A` record for the root domain and a `CNAME` for `www`, both pointing
at the hosting provider. Remove any AAAA records, and leave MX and TXT records
alone. HTTPS certificates are issued automatically, and the root domain redirects
to `www`.

## Search (SEO)

`frontend/index.html` contains the page title, meta description, canonical link,
Open Graph tags and JSON-LD (`LegalService`) with the firm's details.
`robots.txt` allows crawling and points to `sitemap.xml`. Keep the firm's name,
address and phone number consistent between the page, the structured data and any
business listings.

## Running locally

1. Install JDK 21.
2. Set the four environment variables in your terminal, for example in
   PowerShell: `$env:MAIL_USERNAME="..."` (repeat for the others).
3. From `backend/`, run `./mvnw spring-boot:run` (`mvnw.cmd` on Windows).
4. Open `frontend/index.html` with VS Code Live Server (port 5500) and set
   `API_URL` in `frontend/script.js` to `http://localhost:8080/api/contact`.
   Change it back before committing.

## Common tasks

- **New website address:** add it to `@CrossOrigin`, commit, and wait for the API
  to redeploy.
- **New recipient:** change `MAIL_TO` in the hosting dashboard and let the service
  restart.
- **New mail account:** verify a sender, generate an SMTP key, update all four
  variables, send a test, then delete the old key.
- **Edit page content or SEO tags:** edit `frontend/index.html`, then request
  indexing for the home page in Search Console.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| Form says "Could not reach the server"; console shows a CORS error | Site address missing or misspelled in `@CrossOrigin`, or the API is still deploying or waking up |
| Form says "Something went wrong" | The API returned 400 or 500; check the service logs for `Email failed:` |
| `Authentication failed` in logs | Wrong SMTP login or key, an API key used instead of an SMTP key, or login and key from different accounts |
| Connection timeout to the SMTP host | Outbound SMTP port blocked on the host |
| Build fails with an unresolved placeholder | A `MAIL_*` variable is not set |
