import os
import json
import anthropic
from slack_bolt import App
from slack_bolt.adapter.socket_mode import SocketModeHandler
from dotenv import load_dotenv
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError

load_dotenv()

# Initialize Slack app
app = App(
    token=os.environ["SLACK_BOT_TOKEN"],
    signing_secret=os.environ["SLACK_SIGNING_SECRET"]
)

# Initialize Anthropic client
claude = anthropic.Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

# Google Drive root folder ID
DRIVE_FOLDER_ID = "0ACvBIz1NXgADUk9PVA"
SERVICE_ACCOUNT_FILE = os.path.join(os.path.dirname(__file__), "service-account-key.json")

# Slack channel where completed briefs get posted
BRIEFS_CHANNEL = "syds-studio-requests"

# In-memory store for tracking users mid-brief conversation
brief_sessions = {}

# Map of category keywords to subfolder names (exactly as named in Drive)
CATEGORY_FOLDER_MAP = {
    "case studies": "Case Studies",
    "case study": "Case Studies",
    "customer graphics": "Customer Graphics",
    "customer graphic": "Customer Graphics",
    "customer logos": "Customer Logos",
    "customer logo": "Customer Logos",
    "data sheets": "Data Sheets",
    "data sheet": "Data Sheets",
    "datasheet": "Data Sheets",
    "guides": "Guides & E-books",
    "ebooks": "Guides & E-books",
    "e-books": "Guides & E-books",
    "ebook": "Guides & E-books",
    "e-book": "Guides & E-books",
    "integration guides": "Integration Guides",
    "integration guide": "Integration Guides",
    "product graphics": "Product Graphics",
    "product graphic": "Product Graphics",
    "vic.ai logos": "Vic.ai Logos",
    "vic.ai logo": "Vic.ai Logos",
    "logo": "Vic.ai Logos",
    "logos": "Vic.ai Logos",
}


def get_drive_service():
    """Build and return a Google Drive service using service account credentials."""
    creds = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE,
        scopes=["https://www.googleapis.com/auth/drive.readonly"]
    )
    return build("drive", "v3", credentials=creds)


def drive_list(drive, **kwargs):
    """Run a Drive files.list call with shared-drive support enabled."""
    return drive.files().list(
        supportsAllDrives=True,
        includeItemsFromAllDrives=True,
        **kwargs
    ).execute()


def get_subfolder_id(drive, folder_name):
    """Find the ID of a named subfolder inside the root Drive folder."""
    results = drive_list(
        drive,
        q=f"'{DRIVE_FOLDER_ID}' in parents and mimeType='application/vnd.google-apps.folder' and name='{folder_name}' and trashed=false",
        fields="files(id, name)",
        pageSize=5
    )
    folders = results.get("files", [])
    return folders[0]["id"] if folders else None


def search_files_in_folder(drive, folder_id, keywords):
    """Search for files inside a folder matching any of the keywords."""
    keyword_conditions = " or ".join([f"name contains '{kw}'" for kw in keywords])
    query = f"'{folder_id}' in parents and ({keyword_conditions}) and trashed=false"
    results = drive_list(
        drive,
        q=query,
        fields="files(id, name, mimeType, webViewLink)",
        pageSize=10
    )
    return results.get("files", [])


def list_all_files_in_folder(drive, folder_id):
    """List all files in a folder."""
    results = drive_list(
        drive,
        q=f"'{folder_id}' in parents and mimeType!='application/vnd.google-apps.folder' and trashed=false",
        fields="files(id, name, mimeType, webViewLink)",
        pageSize=20
    )
    return results.get("files", [])


def explain_drive_error(error):
    """Translate Google Drive API errors into user-facing messages."""
    if isinstance(error, HttpError):
        status = getattr(error.resp, "status", None)
        details = ""
        try:
            details = error.content.decode("utf-8", errors="ignore")
        except Exception:
            details = str(error)

        if status == 403 and "accessNotConfigured" in details:
            return (
                "I couldn't search Google Drive because the Drive API is not enabled for the Google Cloud "
                "project tied to this service account. Enable the Google Drive API for that project, wait a "
                "minute or two, and try again."
            )

        if status == 403:
            return (
                "I reached Google Drive, but this bot doesn't currently have permission to search those files. "
                "Please make sure the service account has access to the shared drive or folder."
            )

    return "I had trouble searching Drive right now. Try again in a moment!"


def extract_search_intent(user_message):
    """Use Claude to extract the category and specific search keywords from the user's message."""
    categories = list(set(CATEGORY_FOLDER_MAP.values()))
    categories_str = ", ".join(categories)

    response = claude.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=200,
        system=(
            f"You help extract search intent from a user's message asking for design assets.\n\n"
            f"Available categories: {categories_str}\n\n"
            f"Return a JSON object with two fields:\n"
            f"- 'category': the best matching category from the list above, or null if unclear\n"
            f"- 'keywords': a list of 1-3 specific search keywords from the message (e.g. customer name, color, type)\n\n"
            f"Examples:\n"
            f"'I need the Diesel Direct case study' -> {{\"category\": \"Case Studies\", \"keywords\": [\"Diesel\", \"Diesel Direct\"]}}\n"
            f"'send me the black vic.ai logo png' -> {{\"category\": \"Vic.ai Logos\", \"keywords\": [\"black\"]}}\n"
            f"'do you have a GoCardless customer logo?' -> {{\"category\": \"Customer Logos\", \"keywords\": [\"GoCardless\"]}}\n"
            f"Respond with only the JSON, no explanation."
        ),
        messages=[{"role": "user", "content": user_message}]
    )

    try:
        return json.loads(response.content[0].text.strip())
    except Exception:
        return {"category": None, "keywords": []}


def format_file_list(files):
    """Format a list of Drive files into a readable Slack message."""
    lines = []
    for f in files:
        link = f.get("webViewLink", "")
        name = f.get("name", "Untitled")
        lines.append(f"• <{link}|{name}>")
    return "\n".join(lines)


def classify_request(user_message):
    """Classify the request as existing_asset, new_request, or unclear."""
    response = claude.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=50,
        system=(
            "You are a classifier for a design studio Slack bot. Given a message, respond with ONLY one of these words:\n"
            "- 'existing_asset' if the user is asking for any design asset, file, logo, case study, graphic, or document\n"
            "- 'new_request' if the user is asking for something new to be designed or created\n"
            "- 'unclear' if unrelated or too vague\n"
            "Respond with only the single word, no punctuation."
        ),
        messages=[{"role": "user", "content": user_message}]
    )
    return response.content[0].text.strip().lower()


def compile_brief(original_request, details):
    """Compile a final brief from the original request + the user's answers."""
    response = claude.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=800,
        system=(
            "You are a creative director. Based on the original request and the details provided, "
            "compile a complete, final mini creative brief for the design team. "
            "Use this exact format with Slack markdown:\n\n"
            "*🎨 New Design Request*\n\n"
            "*What's needed:* \n"
            "*Who is it for:* \n"
            "*Specs/Sizes:* \n"
            "*Due Date:* \n"
            "*Additional Details:* \n\n"
            "Fill in every field based on the information provided. "
            "If a field is still missing, write 'Not specified'. "
            "Do NOT ask any more questions — just produce the completed brief."
        ),
        messages=[{
            "role": "user",
            "content": f"Original request: {original_request}\n\nAdditional details provided: {details}"
        }]
    )
    return response.content[0].text.strip()


def post_brief_to_channel(brief, requester_name):
    """Post the completed brief to the design requests channel."""
    try:
        result = app.client.chat_postMessage(
            channel=BRIEFS_CHANNEL,
            text=f"*New request from {requester_name}:*\n\n{brief}"
        )
        print(f"✅ Posted to #{BRIEFS_CHANNEL}: {result['ts']}")
        return True
    except Exception as e:
        print(f"❌ Error posting to #{BRIEFS_CHANNEL}: {e}")
        return False


def get_user_name(user_id):
    """Get the display name of a Slack user."""
    try:
        result = app.client.users_info(user=user_id)
        return result["user"]["profile"].get("display_name") or result["user"]["real_name"]
    except Exception:
        return "Someone"


def process_message(message_text, say, user_id):
    """Core logic: classify request and respond appropriately."""
    clean_text = " ".join(
        word for word in message_text.split()
        if not word.startswith("<@")
    ).strip()

    if not clean_text:
        say(
            "Hey! I'm Syd's Studio bot 🎨 Here's what I can help with:\n\n"
            "• *Find existing assets* — case studies, customer graphics, customer logos, data sheets, "
            "product graphics, Vic.ai logos, integration guides, or e-books\n"
            "• *Submit a new design request* — I'll put together a brief for the design team\n\n"
            "What do you need?"
        )
        return

    # Check if user is mid-brief conversation
    if user_id in brief_sessions and brief_sessions[user_id]["step"] == "collecting":
        original_request = brief_sessions[user_id]["original_request"]
        brief = compile_brief(original_request, clean_text)
        requester_name = get_user_name(user_id)
        posted = post_brief_to_channel(brief, requester_name)
        del brief_sessions[user_id]

        if posted:
            say(
                f"✅ Your brief has been submitted to #{BRIEFS_CHANNEL}! "
                f"The design team will pick it up from there.\n\n"
                f"Here's what was sent:\n\n{brief}"
            )
        else:
            say(
                f"I put together your brief but had trouble posting to #{BRIEFS_CHANNEL}. "
                f"Here it is — you can paste it there manually:\n\n{brief}"
            )
        return

    intent = classify_request(clean_text)

    if intent == "existing_asset":
        say("Let me search for that... 🔍")

        search_intent = extract_search_intent(clean_text)
        category = search_intent.get("category")
        keywords = search_intent.get("keywords", [])

        print(f"Search intent: category={category}, keywords={keywords}")

        if not category:
            say(
                "I wasn't sure which category to look in. I have:\n\n"
                "• Case Studies\n• Customer Graphics\n• Customer Logos\n"
                "• Data Sheets\n• Guides & E-books\n• Integration Guides\n"
                "• Product Graphics\n• Vic.ai Logos\n\n"
                "Can you be more specific?"
            )
            return

        try:
            drive = get_drive_service()
            folder_id = get_subfolder_id(drive, category)

            if not folder_id:
                say(f"I couldn't find the *{category}* folder in Drive. It may have been moved or renamed.")
                return

            if keywords:
                files = search_files_in_folder(drive, folder_id, keywords)
            else:
                files = list_all_files_in_folder(drive, folder_id)

            if files:
                file_list = format_file_list(files)
                say(f"Here's what I found in *{category}*:\n\n{file_list}")
            else:
                all_files = list_all_files_in_folder(drive, folder_id)
                if all_files:
                    file_list = format_file_list(all_files)
                    say(
                        f"I couldn't find an exact match for *{', '.join(keywords)}* in {category}. "
                        f"Here's everything in that folder:\n\n{file_list}\n\n"
                        f"If what you need doesn't exist yet, I can submit a design request for you!"
                    )
                else:
                    say(
                        f"I couldn't find anything in *{category}*. "
                        f"Would you like me to submit a new design request?"
                    )

        except Exception as e:
            print(f"Drive search error: {e}")
            say(explain_drive_error(e))

    elif intent == "new_request":
        brief_sessions[user_id] = {
            "step": "collecting",
            "original_request": clean_text
        }
        say(
            "Got it! To put together a brief for the design team, I just need a few quick details:\n\n"
            "1. *Who is this for?* (client name, team, or audience)\n"
            "2. *Specs/sizes?* (dimensions, file format, e.g. 1080x1080 PNG)\n"
            "3. *Due date?*\n"
            "4. *Anything else?* (colors, messaging, tone, examples)\n\n"
            "Reply with as much as you know and I'll take it from there!"
        )

    else:
        say(
            "Hey! I'm Syd's Studio bot 🎨 Here's what I can help with:\n\n"
            "• *Find existing assets* — case studies, customer graphics, customer logos, data sheets, "
            "product graphics, Vic.ai logos, integration guides, or e-books\n"
            "• *Submit a new design request* — I'll put together a brief for the design team\n\n"
            "What do you need?"
        )


# Handle @mentions in channels
@app.event("app_mention")
def handle_mention(event, say):
    message_text = event.get("text", "")
    user_id = event.get("user", "")
    process_message(message_text, say, user_id)


# Handle direct messages
@app.event("message")
def handle_dm(event, say):
    if event.get("channel_type") == "im" and not event.get("bot_id"):
        message_text = event.get("text", "")
        user_id = event.get("user", "")
        process_message(message_text, say, user_id)


if __name__ == "__main__":
    print("⚡ Syd's Studio bot is running!")
    handler = SocketModeHandler(app, os.environ["SLACK_APP_TOKEN"])
    handler.start()
