# 🤖 Telegram Invoice Bot — Setup & Usage Guide

An n8n workflow that lets you create **customers**, **products** and **invoices** via Telegram text or voice messages. Data is automatically saved to Google Sheets and invoices are generated as Google Docs documents.

---

## 📋 Prerequisites

Before you start you'll need:

- [n8n](https://n8n.io) installed (locally with Docker or in the cloud)
- A **Telegram** account
- An **OpenAI** account with an API key
- A **Google** account (Google Sheets + Google Drive + Google Docs)
- A domain on **Cloudflare** (to expose n8n publicly over HTTPS)

---

## 🗂️ Project Structure

| Service | Purpose |
|---|---|
| Telegram | Receive text and voice messages from the user |
| OpenAI Whisper | Transcribe voice messages to text |
| OpenAI GPT-4o-mini | Classify the user's intent |
| Google Sheets | Store customers, products and invoices |
| Google Drive | Save invoice documents |
| Google Docs | Generate invoices from a template |

---

## 🚀 Step-by-step Setup

### 1. Install n8n with Docker

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e WEBHOOK_URL=https://your-domain.com \
  -e N8N_EDITOR_BASE_URL=https://your-domain.com \
  -e N8N_HOST=your-domain.com \
  -e N8N_PROTOCOL=https \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n:latest
```

Replace `your-domain.com` with your actual domain.

### 2. Expose n8n with Cloudflare Tunnel

Telegram requires a public HTTPS URL to deliver messages.

1. Go to [one.dash.cloudflare.com](https://one.dash.cloudflare.com)
2. **Networks → Tunnels → Create a tunnel**
3. Name it `n8n` and follow the steps
4. Under **Published application routes** add:
   - Domain: `your-domain.com`
   - Service: `http://localhost:5678`
5. Copy the tunnel token and run the Cloudflare container:

```bash
docker run -d \
  --name cloudflared-n8n \
  --restart unless-stopped \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run --token YOUR_TUNNEL_TOKEN
```

### 3. Create your Telegram bot

1. Open Telegram and search for `@BotFather`
2. Send `/newbot`
3. Give it a name and a username (must end in `bot`)
4. Save the **token** BotFather gives you

### 4. Set up Google Sheets

Create a spreadsheet in Google Sheets with **3 tabs** with exactly these columns:

**Tab: Customers**
| ID | Nombre | Empresa | Email | Telefono | Direccion | Fecha_Alta |

**Tab: Products**
| ID | Nombre | Descripcion | Precio | Unidad | IVA | Fecha_Alta |

**Tab: Invoices**
| Numero | Fecha | Cliente | Base_Imponible | IVA | Total | Doc_URL | Estado |

> ⚠️ Format the **ID** and **Telefono** columns in the Customers tab as **Plain text** to prevent Google Sheets from treating values as formulas.

Copy the **spreadsheet ID** from the URL:
`docs.google.com/spreadsheets/d/`**`THIS_IS_THE_ID`**`/edit`

### 5. Create the invoice template in Google Docs

Create a new Google Docs document named `Invoice Template` with this exact content — placeholders must match exactly including the double curly braces:

```
Invoice nº {{NUMERO_FACTURA}}          Date: {{FECHA}}

Customer: {{CLIENTE_NOMBRE}} — {{CLIENTE_EMPRESA}}
Email: {{CLIENTE_EMAIL}}
Address: {{CLIENTE_DIRECCION}}

─────────────────────────────────────
{{LINEAS}}
─────────────────────────────────────

Subtotal:    {{BASE_IMPONIBLE}} €
VAT (21%):   {{IVA}} €
TOTAL:       {{TOTAL}} €

Notes: {{NOTAS}}
```

Copy the **document ID** from the URL:
`docs.google.com/document/d/`**`THIS_IS_THE_ID`**`/edit`

### 6. Create a folder for invoices in Google Drive

1. In Google Drive create a folder named `Invoices BOT`
2. Open the folder and copy the **ID** from the URL:
`drive.google.com/drive/folders/`**`THIS_IS_THE_ID`**

### 7. Configure Google credentials in Google Cloud Console

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project or use an existing one
3. Under **APIs & Services → Library** enable:
   - Google Sheets API
   - Google Drive API
   - Google Docs API
4. Under **APIs & Services → Credentials** create an **OAuth 2.0 Client ID** (type: Web application)
5. Under **Authorized redirect URIs** add:
   ```
   https://your-domain.com/rest/oauth2-credential/callback
   ```

---

## ⚙️ Workflow Configuration in n8n

### Import the workflow

1. Open n8n at `https://your-domain.com`
2. Go to **Overview → Import workflow**
3. Select the `telegram-invoice-bot.json` file

### Set up credentials

Create these credentials in n8n (**Settings → Credentials → Add credential**) using exactly these names, as the workflow references them by name:

| Name in n8n | Type | Required data |
|---|---|---|
| `Telegram Facturas` | Telegram API | BotFather token |
| `OpenAi account` | OpenAI API | OpenAI API Key |
| `Google Sheets account` | Google Sheets OAuth2 API | Client ID + Secret → Sign in with Google |
| `Google Drive account` | Google Drive OAuth2 API | Client ID + Secret → Sign in with Google |
| `Google Docs account` | Google Docs OAuth2 API | Client ID + Secret → Sign in with Google |

> ⚠️ For Google OAuth to work, always access n8n via `https://your-domain.com`, never via `localhost`.

### Replace the placeholder IDs

Once imported, find and replace these placeholders in the corresponding nodes:

| Placeholder | Node | Replace with |
|---|---|---|
| `YOUR_TELEGRAM_BOT_TOKEN` | `Get File Path` and `Download Audio` (URL field) | Your BotFather token |
| `YOUR_GOOGLE_SHEET_ID` | `Create Customer in Sheets`, `Create Product in Sheets`, `Find Customer`, `Save Invoice in Sheets` | Your Google Sheet ID |
| `YOUR_GOOGLE_DOCS_TEMPLATE_ID` | `Create Google Docs Document` (File ID field) | Your Invoice Template ID |
| `YOUR_GOOGLE_DRIVE_FOLDER_ID` | `Create Google Docs Document` (Folder ID field) | Your Invoices BOT folder ID |

---

## 💬 How to use the bot

Once the workflow is published (click **Publish** in n8n), find your bot on Telegram and send messages like:

**Create a customer:**
```
I want to create a new customer: name John Smith, company Acme Ltd,
email john@acme.com, phone 612345678, address 1 Main Street London
```

**Create a product:**
```
Create a product: name Consulting, description monthly advisory service,
price 500, VAT 21%
```

**Create an invoice:**
```
I want to create an invoice for John Smith with 2 units of Consulting
at 500€ each. Notes: payment within 30 days.
```

You can also send **voice messages** — the bot will automatically transcribe them with Whisper before processing.

---

## 🔄 How it works

```
Telegram message
       ↓
  Is Audio? ──── Yes ──→ Download → Transcribe with Whisper
       ↓ No                                  ↓
  Extract Text                        Merge Text
       ↓_______________________________↓
                  GPT-4o-mini
            (classifies the intent)
                       ↓
        ┌──────────────┼──────────────┐
     Customer       Product        Invoice
        ↓              ↓               ↓
   Google Sheets  Google Sheets   Find Customer
        ↓              ↓               ↓
  Telegram Reply  Telegram Reply  Calculate amounts
                                       ↓
                                 Copy Drive template
                                       ↓
                                 Fill Docs placeholders
                                       ↓
                                 Save to Sheets
                                       ↓
                                 Telegram reply with link
```

---

## 🛠️ Troubleshooting

**Bot is not responding**
- Check that the workflow is **Published** (green dot next to the name in n8n)
- Verify that the Cloudflare Tunnel is active (HEALTHY status in Zero Trust)
- Check that the n8n Docker container is running

**Google OAuth error (Unauthorized)**
- Always access n8n via `https://your-domain.com`, never via `localhost`
- Check that the redirect URI is added in Google Cloud Console

**Invoice placeholders are not replaced**
- Placeholders in the template must be exactly `{{PLACEHOLDER_NAME}}` with no extra spaces

**Telegram says "Bad Request: bad webhook"**
- The n8n URL must be HTTPS. Verify that the Cloudflare Tunnel is active and that `WEBHOOK_URL` is set in the Docker container

**Phone number shows #ERROR! in Google Sheets**
- Select the Phone column → Format → Number → Plain text

---

## 📁 Files included

- `telegram-invoice-bot.json` — ready-to-import n8n workflow (no sensitive data)
- `README.md` — this guide

---

## 📄 License

Free to use for personal and commercial projects.
