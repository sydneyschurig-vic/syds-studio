[README.md](https://github.com/user-attachments/files/26449380/README.md)
# syds-studio

`syds-studio` is an AI-powered Slack bot for marketing and design requests. It helps teammates find existing materials in Google Drive, and if something does not exist yet, it turns the request into a structured creative brief and sends it to `#syds-studio-requests`.

## What It Does

- Searches Google Drive for existing marketing assets like case studies, customer graphics, logos, data sheets, product graphics, integration guides, and e-books
- Uses Anthropic to understand what the user is asking for and extract search intent
- Returns matching files directly in Slack
- Falls back to a guided intake flow when a user needs something new created
- Compiles the request into a polished mini brief and posts it to the design request channel

## How It Works

When someone messages the bot in Slack, it classifies the request into one of three buckets:

1. `existing_asset`
2. `new_request`
3. `unclear`

If the message is asking for an existing asset, the bot:

- identifies the best matching content category
- extracts 1 to 3 useful search keywords
- searches the correct Google Drive folder
- returns matching files, or shows everything in that folder if no exact match is found

If the message is asking for a new design request, the bot:

- asks for a few quick details
- gathers the response in-thread or in DM
- formats the request into a clean creative brief
- posts the final brief into `#syds-studio-requests`

## Supported Asset Categories

- Case Studies
- Customer Graphics
- Customer Logos
- Data Sheets
- Guides & E-books
- Integration Guides
- Product Graphics
- Vic.ai Logos

## Example Prompts

People can use natural language, for example:

- `Do you have the Diesel Direct case study?`
- `Send me the black Vic.ai logo PNG`
- `Do we have a GoCardless customer logo?`
- `I need a one-pager created for an upcoming event`

## Tech Stack

- Python
- Slack Bolt for Python
- Anthropic API
- Google Drive API
- Service account authentication

## Project Structure

```text
syds-studio/
├── app.py
├── requirements.txt
├── service-account-key.json
└── README.md
```

## Setup

### 1. Clone the project

```bash
git clone <your-repo-url>
cd syds-studio
```

### 2. Create and activate a virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Add your environment variables

Create a `.env` file in the project root with:

```env
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
SLACK_APP_TOKEN=xapp-...
ANTHROPIC_API_KEY=...
```

### 5. Add your Google service account key

Place your Google service account credentials file in the project root as:

```text
service-account-key.json
```

### 6. Enable Google Drive API

Make sure the Google Drive API is enabled in the Google Cloud project connected to your service account.

If your files live in a shared drive, make sure:

- the service account has access to that shared drive or folder
- the Drive API is enabled for the correct Google Cloud project

### 7. Run the bot

```bash
python3 app.py
```

## Slack Behavior

The bot responds to:

- direct messages
- `@mentions` in Slack channels

## Notes

- The bot uses an in-memory session to track users during the brief intake flow
- Existing asset searches rely on exact Drive folder names mapped in the app
- Google Drive errors are handled with clearer user-facing messages, including API access and permission issues

## Future Improvements

- persist brief sessions in a database instead of memory
- add fuzzy matching and richer Drive search
- support file type filters like PNG, PDF, and SVG
- add analytics for most-requested assets
- support approvals or triage logic before posting new briefs
