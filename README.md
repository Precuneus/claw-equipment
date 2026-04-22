# claw.equipment

Coming-soon landing page for Claw. Captures early-access email signups with client-side encryption.

## What it is

A single-page teaser site built with Astro 5. Visitors enter their email against a dark sci-fi backdrop; the address is encrypted in-browser with AES-GCM and POSTed to a Google Sheet via a Google Apps Script web app. No plaintext email ever leaves the browser.

## Tech stack

- [Astro 5](https://astro.build) — static site generation
- TypeScript (strict)
- Web Crypto API — AES-GCM 256 encryption, PBKDF2 key derivation
- Google Apps Script — serverless endpoint that writes rows to a Google Sheet
- GitHub Pages — hosting
- Google Fonts — Cinzel, Crimson Pro, JetBrains Mono

## Local development

```bash
npm install

# Fill in both secrets
echo "PUBLIC_ENCRYPTION_SALT=your-secret-salt" > .env
echo "PUBLIC_APPS_SCRIPT_URL=https://script.google.com/macros/s/.../exec" >> .env

npm run dev       # dev server with hot reload
npm run build     # production build → dist/
npm run preview   # serve the built dist/ locally
```

## Environment variables

| Variable | GitHub secret | Description |
| --- | --- | --- |
| `PUBLIC_ENCRYPTION_SALT` | `ENCRYPTION_SALT` | Salt for PBKDF2 key derivation. Must match the value used when decrypting. |
| `PUBLIC_APPS_SCRIPT_URL` | `APPS_SCRIPT_URL` | Deployed Google Apps Script web app URL that receives signup POSTs. |

The workflow reads both secrets and writes them to `.env` at build time.

## Google Sheets setup

1. Create a Google Sheet.
2. Open **Extensions → Apps Script**, paste the function below, then click **Deploy → New deployment** (type: Web app, Execute as: Me, Who can access: Anyone).
3. Copy the deployment URL and add it as a GitHub repository secret named `APPS_SCRIPT_URL`.

```javascript
function doPost(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const { ts, data } = JSON.parse(e.postData.contents);
  sheet.appendRow([new Date(ts).toISOString(), data]);
  return ContentService.createTextOutput('ok').setMimeType(ContentService.MimeType.TEXT);
}
```

Each submission adds a row: `[ISO timestamp, encrypted base64 blob]`.

## Deployment

Pushes to `main` trigger the [GitHub Actions workflow](.github/workflows/deploy.yml), which:

1. Installs dependencies (`npm ci`)
2. Writes `.env` from the `ENCRYPTION_SALT` and `APPS_SCRIPT_URL` repository secrets
3. Builds the site (`npm run build`)
4. Deploys `dist/` to GitHub Pages

Custom domain `claw.equipment` is configured via `public/CNAME`.

## Decrypting signups

Emails are encrypted with AES-GCM before leaving the browser; the sheet stores only ciphertext.

To decrypt:

1. Open `decrypt.html` locally in a browser.
2. Enter the `PUBLIC_ENCRYPTION_SALT` value.
3. Copy the `data` column from the sheet, paste it into the textarea (one blob per line).
4. Click **Decrypt**.

`decrypt.html` also accepts the old localStorage JSON format (`[{"ts":...,"data":"..."}]`) for backwards compatibility.

## Project structure

```
src/
  pages/index.astro    # landing page + encryption + sheet POST logic
  layouts/Base.astro   # HTML shell, fonts, meta tags
  styles/global.css    # CSS variables, reset
public/
  actor-figure.png     # hero image
  CNAME                # custom domain
decrypt.html           # offline email decryption utility
.github/workflows/
  deploy.yml           # GitHub Pages CI/CD
```
