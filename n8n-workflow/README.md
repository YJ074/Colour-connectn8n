# CorrectColours n8n Workflow Documentation

## Overview
This workflow automates the CorrectColours PDF report generation and delivery system. It receives face scan data via webhook, performs colour analysis, generates a 25-page PDF report, and sends it to the user via email.

---

## Workflow Files

| File | Description |
|------|-------------|
| `correctcolours-workflow.json` | Main n8n workflow (importable) |
| `README.md` | This documentation |

---

## Node-by-Node Description

### NODE 1 — Webhook
**Type:** `n8n-nodes-base.webhook`  
**Purpose:** Entry point for incoming POST requests

- **HTTP Method:** POST only
- **Path:** `/correctcolours-webhook`
- **Response Mode:** On Received
- **Accepts JSON payload with:**
  - `name` (string)
  - `email` (string)
  - `country` (string)
  - `gender` (male | female | prefer_not_to_say)
  - `face_scan.front` (string - URL or base64)
  - `face_scan.left` (string - URL or base64)
  - `face_scan.right` (string - URL or base64)
  - `scan_status` (string)

---

### NODE 2 — Normalize Data (Set)
**Type:** `n8n-nodes-base.set`  
**Purpose:** Creates clean internal fields for workflow processing

**Output Fields:**
- `user.name`
- `user.email`
- `user.country`
- `user.gender`
- `images.front`
- `images.left`
- `images.right`
- `scan_status`

---

### NODE 3 — Validation Check (IF)
**Type:** `n8n-nodes-base.if`  
**Purpose:** Validates all required fields before processing

**Conditions (ALL must be TRUE):**
1. `scan_status === "approved"`
2. `user.name` is not empty
3. `user.email` is not empty
4. `user.gender` is not empty
5. `images.front` is not empty
6. `images.left` is not empty
7. `images.right` is not empty

**Outputs:**
- **TRUE branch:** Continues to Gender Switch
- **FALSE branch:** Stops workflow, returns 400 error

---

### NODE 4 — Gender Switch
**Type:** `n8n-nodes-base.switch`  
**Purpose:** Routes to gender-specific analysis prompts

**Routes:**
| Gender Value | Output Index |
|--------------|--------------|
| `male` | 0 |
| `female` | 1 |
| `prefer_not_to_say` | 2 (fallback) |

---

### NODE 5 — Analysis (OpenAI)
**Type:** `@n8n/n8n-nodes-langchain.openAi`  
**Purpose:** Single AI call to generate colour analysis

There are 3 analysis nodes (one per gender route):
- **Analysis - Male**
- **Analysis - Female**
- **Analysis - Neutral**

**Configuration:**
- Model: `gpt-4o`
- Temperature: `0` (deterministic)
- Max Tokens: `4096`

**Output Schema (STRICT JSON):**
```json
{
  "analysis": {
    "colours": {
      "core": [
        { "name": "colour_name", "hex": "#XXXXXX" }
      ],
      "avoid": [
        { "name": "colour_name", "hex": "#XXXXXX" }
      ]
    },
    "styling": {
      "necklines": {
        "recommended": ["neckline1", "neckline2"],
        "caution": ["neckline1"]
      },
      "fabric": {
        "recommended": ["fabric1", "fabric2"],
        "caution": ["fabric1"]
      }
    },
    "hair": {
      "natural_alignment": "Description of hair/skin undertone alignment",
      "undertone_alignment": "warm/cool/neutral",
      "recommended_colours": ["color1", "color2"],
      "avoid_colours": ["color1", "color2"],
      "recommended_styles": ["style1", "style2"]
    },
    "summary": {
      "season": "season_name",
      "undertone": "warm/cool/neutral",
      "contrast_level": "high/medium/low",
      "description": "Brief summary paragraph"
    }
  }
}
```

**IMPORTANT:** The `core` array contains EXACTLY 14 colours.

**Hair Colour Harmony Rules (Gender-Aware):**
- **Male:** Natural, professional, low-maintenance tones. Avoid extreme contrast unless facial contrast supports it.
- **Female:** Wider but harmonious palette. Undertone accuracy over trends.
- **Neutral/Prefer Not To Say:** Strict colour science with minimal stylistic bias.

---

### NODE 6 — Fetch HTML Template (HTTP Request)
**Type:** `n8n-nodes-base.httpRequest`  
**Purpose:** Fetches the full 25-page HTML template from GitHub

**Configuration:**
- Method: `GET`
- URL: Uses environment variable `PDF_TEMPLATE_URL`
- Response Format: `text`

---

### NODE 7 — Inject Data into HTML (Code)
**Type:** `n8n-nodes-base.code`  
**Purpose:** Replaces placeholders with user and analysis data

**Placeholder Replacements:**
| Placeholder | Source |
|-------------|--------|
| `{{USER_NAME}}` | `user.name` |
| `{{USER_EMAIL}}` | `user.email` |
| `{{USER_COUNTRY}}` | `user.country` |
| `{{USER_GENDER}}` | `user.gender` |
| `{{analysis.colours.core[0].name}}` | Analysis JSON |
| `{{analysis.colours.core[0].hex}}` | Analysis JSON |
| `{{analysis.styling.necklines.recommended[0]}}` | Analysis JSON |
| `{{analysis.hair.natural_alignment}}` | Analysis JSON |
| `{{analysis.hair.undertone_alignment}}` | Analysis JSON |
| `{{analysis.hair.recommended_colours[0]}}` | Analysis JSON |
| `{{analysis.hair.avoid_colours[0]}}` | Analysis JSON |
| ... (all indexed placeholders) | ... |

**Does NOT:**
- Alter HTML structure
- Add new placeholders
- Modify HALO logic

---

### NODE 8 — Render PDF (HTTP Request)
**Type:** `n8n-nodes-base.httpRequest`  
**Purpose:** Converts HTML to PDF using Browserless/Puppeteer

**Configuration:**
- URL: Uses environment variable `BROWSERLESS_URL`
- Authentication: HTTP Header Auth
- Format: A4
- Print Background: `true`
- Prefer CSS Page Size: `true`
- Margins: `0` (all sides)
- Wait Until: `networkidle0`
- Timeout: `120000ms`

**Supports:**
- 25 pages
- Dark backgrounds
- SVG via `<img>`
- Google Fonts
- A4 fixed size

**Output:** PDF binary

---

### NODE 9 — Send Email
**Type:** `n8n-nodes-base.emailSend`  
**Purpose:** Sends the PDF report to the user

**Configuration:**
- To: `user.email`
- Subject: "Your CorrectColours Report"
- Attachment: PDF binary from previous node

**Email Content:**
```
Dear [Name],

Thank you for using CorrectColours. Please find your personalised colour analysis report attached.

Your report includes:
- Your 14 core palette colours
- Styling recommendations
- Hair colour and style suggestions
- Colours to avoid

We hope you enjoy discovering your perfect colours!

Best regards,
The CorrectColours Team
```

---

### NODE 10 — Error Trigger & Handling
**Type:** `n8n-nodes-base.errorTrigger`  
**Purpose:** Handles any workflow failures

**Error Flow:**
1. **Error Trigger** catches any node failure
2. **Error Notification Email** sends:
   - "We're finishing your report and will send it shortly."
3. **Log Error** records error details with timestamp

**No retries implemented as per specification.**

---

## Required Credentials

| Credential Name | Type | Purpose | Required Fields |
|-----------------|------|---------|-----------------|
| **OpenAI API** | `openAiApi` | AI colour analysis | API Key |
| **SMTP** | `smtp` | Email delivery | Host, Port, User, Password |
| **Browserless API** | `httpHeaderAuth` | PDF rendering | Authorization header with API token |

### Credential Setup in n8n Cloud

1. **OpenAI API:**
   - Go to Settings → Credentials → Add Credential
   - Select "OpenAI API"
   - Enter your OpenAI API key

2. **SMTP:**
   - Go to Settings → Credentials → Add Credential
   - Select "SMTP"
   - Configure your email provider settings

3. **Browserless API:**
   - Go to Settings → Credentials → Add Credential
   - Select "Header Auth"
   - Name: `Authorization`
   - Value: `Bearer YOUR_BROWSERLESS_TOKEN`

---

## Environment Variables

Configure these in n8n Settings → Variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `PDF_TEMPLATE_URL` | GitHub raw URL for HTML template | `https://raw.githubusercontent.com/org/repo/main/template.html` |
| `BROWSERLESS_URL` | Browserless PDF endpoint | `https://chrome.browserless.io/pdf` |
| `SMTP_FROM_EMAIL` | Sender email address | `reports@correctcolours.com` |

---

## Importing the Workflow

1. Log in to n8n Cloud
2. Go to Workflows → Import from File
3. Select `correctcolours-workflow.json`
4. Configure credentials (replace placeholder IDs)
5. Set environment variables
6. Activate the workflow

---

## Confirmations

### ✅ Workflow is Deterministic
- Analysis uses `temperature: 0` for consistent AI outputs
- No random elements in data processing
- Same input will always produce same output structure
- Placeholder replacement is exact string matching

### ✅ 25 Pages Always Render
- Full HTML template is fetched in single GET request
- No page merging or reduction
- No content summarization
- PDF render uses:
  - `preferCSSPageSize: true` - respects page breaks in CSS
  - `printBackground: true` - preserves all backgrounds
  - `waitUntil: networkidle0` - ensures all assets load
  - Zero margins to preserve exact layout
- HTML structure is immutable (only placeholder replacement)

---

## Testing the Webhook

```bash
curl -X POST https://your-n8n-instance.app.n8n.cloud/webhook/correctcolours-webhook \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jane Doe",
    "email": "jane@example.com",
    "country": "United Kingdom",
    "gender": "female",
    "face_scan": {
      "front": "https://example.com/images/front.jpg",
      "left": "https://example.com/images/left.jpg",
      "right": "https://example.com/images/right.jpg"
    },
    "scan_status": "approved"
  }'
```

---

## Stop Conditions

The workflow will STOP and return 400 error if:
- `scan_status !== "approved"`
- `name` is missing or empty
- `email` is missing or empty
- `gender` is missing or empty
- Any `face_scan` image (front, left, right) is missing or empty

---

## What This Workflow Does NOT Do

As per specification:
- ❌ Merge pages
- ❌ Reduce pages
- ❌ Summarise content
- ❌ Generate visuals
- ❌ Modify HALO logic
- ❌ Modify HTML/CSS
- ❌ Store files permanently
- ❌ Add loops
- ❌ Add queues
- ❌ Add optimisations
- ❌ Retry failed operations

---

## Support

For issues with this workflow, check:
1. Credential configurations are correct
2. Environment variables are set
3. HTML template URL is accessible
4. Browserless service is operational
5. SMTP settings are valid
