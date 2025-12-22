# Required Credentials for CorrectColours Workflow

This document lists all credentials required to run the n8n workflow.

---

## 1. OpenAI API Credential

**Purpose:** AI-powered colour analysis from face scan images

**Credential Type:** HTTP Header Auth  
**Name:** `OpenAI API Header`

**Configuration:**
| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer sk-your-openai-api-key` |

**How to Get:**
1. Go to https://platform.openai.com/api-keys
2. Create a new API key
3. Copy the key (starts with `sk-`)

**n8n Setup:**
1. Settings → Credentials → Add Credential
2. Select "Header Auth"
3. Name: `Authorization`
4. Value: `Bearer YOUR_API_KEY`
5. Save

---

## 2. SMTP Credential

**Purpose:** Sending PDF reports via email

**Credential Type:** SMTP  
**Name:** `SMTP`

**Configuration (varies by provider):**

### Gmail Example:
| Field | Value |
|-------|-------|
| Host | `smtp.gmail.com` |
| Port | `587` |
| User | `your-email@gmail.com` |
| Password | App Password (not regular password) |
| SSL/TLS | `true` |

### SendGrid Example:
| Field | Value |
|-------|-------|
| Host | `smtp.sendgrid.net` |
| Port | `587` |
| User | `apikey` |
| Password | Your SendGrid API key |
| SSL/TLS | `true` |

### Amazon SES Example:
| Field | Value |
|-------|-------|
| Host | `email-smtp.us-east-1.amazonaws.com` |
| Port | `587` |
| User | Your SES SMTP username |
| Password | Your SES SMTP password |
| SSL/TLS | `true` |

**n8n Setup:**
1. Settings → Credentials → Add Credential
2. Select "SMTP"
3. Fill in your provider details
4. Save

---

## 3. Browserless API Credential

**Purpose:** Rendering HTML to PDF with Puppeteer/Playwright

**Credential Type:** HTTP Header Auth  
**Name:** `Browserless API`

**Configuration:**
| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Basic YOUR_BROWSERLESS_TOKEN` |

**How to Get:**
1. Go to https://www.browserless.io/
2. Sign up for an account
3. Get your API token from dashboard

**Alternative Services:**
- **Puppeteer Cloud:** https://www.puppeteercloud.com/
- **Headless Chrome Service:** Self-hosted option

**n8n Setup:**
1. Settings → Credentials → Add Credential
2. Select "Header Auth"
3. Name: `Authorization`
4. Value: `Basic YOUR_TOKEN` (or Bearer depending on service)
5. Save

---

## 4. Google Sheets OAuth2 Credential

**Purpose:** Logging workflow executions to operational dashboard

**Credential Type:** Google Sheets OAuth2 API  
**Name:** `Google Sheets OAuth2`

**How to Get:**
1. Go to Google Cloud Console: https://console.cloud.google.com/
2. Create a new project or select existing
3. Enable Google Sheets API
4. Create OAuth 2.0 credentials (Web application)
5. Add n8n callback URL to authorized redirect URIs

**n8n Setup:**
1. Settings → Credentials → Add Credential
2. Select "Google Sheets OAuth2 API"
3. Enter Client ID and Client Secret from Google Cloud Console
4. Click "Sign in with Google" and authorize
5. Save

---

## Environment Variables

Set these in n8n Settings → Variables:

| Variable | Description | Required |
|----------|-------------|----------|
| `PDF_TEMPLATE_URL` | GitHub raw URL for the 25-page HTML template | Yes |
| `BROWSERLESS_URL` | PDF rendering service endpoint | Yes |
| `SMTP_FROM_EMAIL` | Sender email address | Yes |
| `GOOGLE_SHEET_ID` | Google Sheet document ID | Yes |
| `GOOGLE_SHEET_NAME` | Sheet tab name (default: Sheet1) | No |

**Example Values:**
```
PDF_TEMPLATE_URL = https://raw.githubusercontent.com/your-org/correctcolours/main/templates/report.html
BROWSERLESS_URL = https://chrome.browserless.io/pdf
SMTP_FROM_EMAIL = reports@correctcolours.com
```

---

## Credential IDs in Workflow JSON

After creating credentials in n8n, you need to update the workflow JSON with your credential IDs:

### Locations to Update:

1. **OpenAI Nodes** (Analysis - Male/Female/Neutral):
```json
"credentials": {
  "httpHeaderAuth": {
    "id": "YOUR_OPENAI_CREDENTIAL_ID",
    "name": "OpenAI API Header"
  }
}
```

2. **Browserless Node** (Render PDF):
```json
"credentials": {
  "httpHeaderAuth": {
    "id": "YOUR_BROWSERLESS_CREDENTIAL_ID",
    "name": "Browserless API"
  }
}
```

3. **Email Nodes** (Send Email, Error Email):
```json
"credentials": {
  "smtp": {
    "id": "YOUR_SMTP_CREDENTIAL_ID",
    "name": "SMTP"
  }
}
```

---

## Finding Credential IDs in n8n

1. Go to Settings → Credentials
2. Click on the credential you created
3. Look at the URL - the ID is in the path
   - Example: `/credentials/AbCdEf123` → ID is `AbCdEf123`

Or use n8n API:
```bash
curl -X GET "https://your-n8n-instance.app.n8n.cloud/api/v1/credentials" \
  -H "X-N8N-API-KEY: your-api-key"
```

---

## Security Best Practices

1. **Never commit API keys to version control**
2. **Use n8n's credential system** - don't hardcode keys in workflow JSON
3. **Rotate API keys periodically**
4. **Use environment variables** for sensitive URLs
5. **Limit API key permissions** where possible
   - OpenAI: Create key with only chat completions access
   - SMTP: Use app-specific passwords

---

## Cost Considerations

| Service | Estimated Cost per Execution |
|---------|------------------------------|
| OpenAI GPT-4o | ~$0.01-0.05 (depending on image sizes) |
| Browserless | ~$0.005-0.02 per PDF |
| Email (SendGrid) | Free tier: 100/day |
| Email (SES) | ~$0.0001 per email |

**Estimated total per report:** $0.02-0.08

---

## Troubleshooting

### OpenAI Errors
- **401 Unauthorized:** Check API key format (Bearer prefix)
- **429 Rate Limit:** Add retry logic or upgrade plan
- **500 Error:** Check if model name is correct

### SMTP Errors
- **Authentication failed:** Check password/app password
- **Connection refused:** Check port and SSL settings
- **Recipient rejected:** Verify sender domain configuration

### Browserless Errors
- **Timeout:** Increase timeout in workflow settings
- **PDF empty:** Check HTML content is valid
- **Fonts missing:** Ensure Google Fonts URLs are accessible
