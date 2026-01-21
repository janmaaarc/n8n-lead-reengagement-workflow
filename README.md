# n8n WhatsApp Lead Re-engagement Workflow

Automated lead re-engagement system using n8n, WhatsApp (via Twilio), Google Sheets, Slack, and AI-powered response categorization.

## Overview

This project contains three interconnected n8n workflows designed to automatically re-engage leads that have been inactive for 90+ days through WhatsApp messaging.

### Workflows Included

| Workflow | Purpose |
|----------|---------|
| **WhatsApp Lead Re-engagement Drip Campaign** | Automated 5-day drip sequence to re-engage inactive leads |
| **WhatsApp Response Handler** | AI-powered processing of incoming WhatsApp replies |
| **Daily Report & Error Handler** | Daily summary reports and error recovery with retry logic |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Lead Re-engagement System                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │   Triggers   │    │   Actions    │    │    Integrations      │  │
│  ├──────────────┤    ├──────────────┤    ├──────────────────────┤  │
│  │ • Manual     │───▶│ • Filter     │───▶│ • Twilio (WhatsApp)  │  │
│  │ • Weekly     │    │   Leads      │    │ • Google Sheets      │  │
│  │ • Webhook    │    │ • Send       │    │ • Slack              │  │
│  │ • Daily 6PM  │    │   Messages   │    │ • Google Gemini AI   │  │
│  │ • Error      │    │ • Track      │    │                      │  │
│  └──────────────┘    │   Progress   │    └──────────────────────┘  │
│                      └──────────────┘                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Workflow Details

### 1. WhatsApp Lead Re-engagement Drip Campaign

**Purpose:** Automatically reach out to leads who haven't been contacted in 90+ days.

**Message Sequence:**
- **Day 1:** Initial outreach - friendly reconnection message
- **Day 3:** First follow-up - highlight updates and value
- **Day 5:** Final follow-up - last chance message
- **Day 7:** Mark as "No Response" if no reply

**Triggers:**
- Manual trigger (for testing)
- Weekly schedule (Monday 9 AM)
- Webhook: `POST /lead-reengagement-whatsapp`

**Flow:**
1. Read leads from Google Sheets (LeadData tab)
2. Filter for leads with `last_contact_date` > 90 days
3. Validate phone numbers (E.164 format)
4. Send WhatsApp message via Twilio
5. Track progress in LeadTracking sheet
6. Wait and check for responses between messages
7. Update final status and notify via Slack

### 2. WhatsApp Response Handler

**Purpose:** Process incoming WhatsApp replies and categorize them using AI.

**Response Categories:**

| Category | Examples | Result Tag |
|----------|----------|------------|
| INTERESTED | "Yes", "Sure", "OK", "Call me", "Sige" (Filipino) | Re-engaged |
| UNSUBSCRIBED | "Stop", "No", "Remove me", "Hindi" (Filipino) | Unsubscribed |
| QUESTION | "What is this?", "Who is this?", "Ano to?" | Needs Follow-up |
| UNCLEAR | "K", "Hmm", "Maybe" | Unclear |

**Flow:**
1. Receive WhatsApp webhook from Twilio
2. Parse phone number and message body
3. Find lead in LeadTracking sheet by phone
4. Send message to Google Gemini AI for analysis
5. Apply keyword matching as fallback
6. Update sheet with response and category
7. Send Slack notification
8. Respond to Twilio with empty TwiML

### 3. Daily Report & Error Handler

**Purpose:** Generate daily campaign summaries and handle workflow errors.

#### Daily Report
- **Schedule:** Daily at 6 PM
- **Webhook:** `POST /generate-report`
- **Metrics:** Total processed, in progress, re-engaged, unsubscribed, no response, needs review
- **Output:** Slack message with statistics and rates

#### Error Handler
- **Trigger:** Error Trigger (catches errors from linked workflows)
- **Retry Logic:** Up to 3 retries with exponential backoff (5s, 10s, 20s, max 60s)
- **Failure Action:** Send Slack alert + log to ErrorLog sheet

## Prerequisites

- [n8n](https://n8n.io/) instance (self-hosted or cloud)
- [Twilio](https://www.twilio.com/) account with WhatsApp sandbox or business number
- [Google Cloud](https://console.cloud.google.com/) project with Sheets API and Gemini API enabled
- [Slack](https://slack.com/) workspace with incoming webhooks or bot

## Setup Instructions

### 1. Google Sheets Setup

Create a Google Sheet with the following tabs:

#### LeadData (source data)
| Column | Description |
|--------|-------------|
| name | Lead's full name |
| phone | Phone number (with country code) |
| email | Email address |
| last_contact_date | Date of last contact (YYYY-MM-DD) |

#### LeadTracking (progress tracking)
| Column | Description |
|--------|-------------|
| workflow_id | Unique workflow execution ID |
| name | Lead's name |
| phone | Cleaned phone number |
| email | Email address |
| original_last_contact | Original last contact date |
| days_since_contact | Days since last contact |
| sequence_started | When drip sequence started |
| current_status | In Progress / Completed / Needs Review |
| sequence_day | Current day in sequence (1, 3, 5, 7) |
| channel | WhatsApp |
| last_sms_sent | Timestamp of last message |
| last_sms_type | day1_initial / day3_followup / day5_final |
| twilio_sid | Twilio message SID |
| response_text | Lead's response |
| response_category | INTERESTED / UNSUBSCRIBED / QUESTION / UNCLEAR |
| final_tag | Re-engaged / Unsubscribed / No Response / Unclear |
| notes | Additional notes |
| response_received_at | When response was received |
| completed_at | When sequence completed |

#### ErrorLog (error tracking)
| Column | Description |
|--------|-------------|
| timestamp | When error occurred |
| workflow_name | Name of failed workflow |
| workflow_id | ID of failed workflow |
| execution_id | Execution ID |
| error_node | Node that failed |
| error_message | Error details |
| retry_count | Number of retry attempts |
| status | Failed - Retries Exhausted |

### 2. Twilio Setup

1. Create a Twilio account at [twilio.com](https://www.twilio.com/)
2. Enable the WhatsApp Sandbox (for testing) or apply for a WhatsApp Business number
3. Note your:
   - Account SID
   - Auth Token
   - WhatsApp number (e.g., `+14155238886` for sandbox)
4. Configure the webhook URL for incoming messages to point to your n8n webhook

### 3. Slack Setup

1. Create a Slack app at [api.slack.com](https://api.slack.com/)
2. Add the `chat:write` scope
3. Install the app to your workspace
4. Note the Bot Token and Channel ID where notifications should be sent

### 4. Google Gemini Setup

1. Enable the Gemini API in Google Cloud Console
2. Create an API key
3. Note the API key for n8n credential setup

### 5. n8n Import & Configuration

1. **Import the workflow:**
   - Open your n8n instance
   - Go to Workflows > Import from File
   - Select `workflow.json`
   - Import each workflow from the `workflows` array

2. **Configure credentials:**
   Replace all placeholders in the workflows:

   | Placeholder | Replace With |
   |-------------|--------------|
   | `YOUR_GOOGLE_SHEET_ID` | Your Google Sheets document ID (from URL) |
   | `YOUR_LEAD_TRACKING_SHEET_GID` | GID of LeadTracking tab |
   | `YOUR_SLACK_CHANNEL_ID` | Slack channel ID (e.g., C0A2V3QHC30) |
   | `YOUR_SLACK_CREDENTIAL_ID` | n8n Slack credential ID |
   | `YOUR_TWILIO_CREDENTIAL_ID` | n8n Twilio credential ID |
   | `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | n8n Google Sheets credential ID |
   | `YOUR_GEMINI_CREDENTIAL_ID` | n8n Gemini API credential ID |
   | `+1XXXXXXXXXX` | Your Twilio WhatsApp number |

3. **Set up credentials in n8n:**
   - Google Sheets: OAuth2 connection
   - Twilio: Account SID + Auth Token
   - Slack: Bot Token
   - Google Gemini: API Key

4. **Link error handler:**
   - In each workflow's settings, set "Error Workflow" to the Daily Report & Error Handler workflow

5. **Activate workflows:**
   - Activate all three workflows
   - Test with manual trigger first

## Phone Number Format

The workflow assumes US phone numbers (+1) by default.

**For other countries (e.g., Philippines +63):**

Update the `Filter 90+ Days & Validate` node:

```javascript
// Change this:
const e164Phone = phone.startsWith('1') ? `+${phone}` : `+1${phone}`;

// To this (if numbers already include country code):
const e164Phone = `+${phone}`;
```

## Testing

1. **Manual Test:**
   - Add a test lead to LeadData with `last_contact_date` > 90 days ago
   - Trigger the drip campaign manually
   - Check LeadTracking for the new entry
   - Verify WhatsApp message is received

2. **Response Test:**
   - Reply to the WhatsApp message
   - Check if response is categorized correctly
   - Verify Slack notification is received

3. **Report Test:**
   - Trigger the report webhook: `POST /generate-report`
   - Check Slack for the summary report

## Customization

### Message Templates

Edit the `Prepare Day X WhatsApp` nodes to customize messages:

```javascript
const message = `Hi ${firstName}! 👋

Your custom message here.

👉 Reply *YES* to schedule
👉 Reply *STOP* to opt out`;
```

### AI System Prompt

Edit the `AI Agent` node's system message to adjust categorization rules:

```
You are a WhatsApp Response Analyzer...
// Customize categories, examples, and rules
```

### Schedule

Edit the `Weekly Schedule` and `Daily Report` nodes to change timing:

```javascript
// Weekly Monday 9 AM
"expression": "0 9 * * 1"

// Daily 6 PM
"expression": "0 18 * * *"
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Messages not sending | Check Twilio credentials and WhatsApp sandbox setup |
| Leads not found | Verify Google Sheets credential and sheet/tab names |
| AI not categorizing | Check Gemini API key and quota |
| No Slack notifications | Verify Slack bot token and channel ID |
| Webhook not receiving | Check n8n webhook URL is publicly accessible |

## Security Notes

- Never commit real API keys or credentials
- Use n8n's built-in credential storage
- Rotate credentials periodically
- Ensure webhook endpoints use HTTPS
- Consider rate limiting for production use

## License

MIT License - feel free to use and modify for your own lead re-engagement campaigns.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

## Support

For issues or questions:
- Open a GitHub issue
- Check [n8n community](https://community.n8n.io/)
- Review [Twilio WhatsApp docs](https://www.twilio.com/docs/whatsapp)
# n8n-lead-reengagement-workflow
