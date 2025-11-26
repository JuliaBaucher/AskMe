# 📘 AskMe -- Technical Specification Document

# 1. Project Overview

**AskMe** is an internal support chatbot designed for Amazon forecasting
teams (VSP / VeSPER).\
It delivers instant answers to forecasting, tooling, workflows, and
documentation questions.\
The system combines:

-   Conversational AI using Amazon Bedrock\
-   Retrieval-Augmented Generation (RAG)\
-   An admin interface for knowledge updates\
-   Automatic embedding generation\
-   A serverless backend (Lambda + API Gateway + S3)\
-   A lightweight GitHub Pages frontend

AskMe is built for scalability, low cost, and zero-maintenance
operations.

------------------------------------------------------------------------

# 2. High-Level Architecture



┌────────────────────────────────────────┐  
│ GitHub Pages Frontend │  
│ - index.html (Chat UI) │  
│ - admin.html (KB Upload UI) │  
└───────────────────┬────────────────────┘   

│ HTTPS  
▼  
┌────────────────────────────────────────┐  
│ API Gateway (HTTP API) │  
│ /chat /upload │  
└──────────────┬───────────────┬─────────┘  
│ │  
│ │ (Bearer Admin Token)  
│ ▼  
│ ┌────────────────────────────┐  
│ │ Lambda: AdminUpload │  
│ │ - Stores files in S3 │  
│ └──────────────┬────────────┘  
│ │  
│ ▼ Trigger  
│ ┌────────────────────────────┐  
│ │ Lambda: EmbeddingBuilder │  
│ │ - Chunk + embed using │  
│ │ Amazon Titan Embeddings │  
│ └──────────────┬────────────┘  
│ │  
│ ▼  
│ S3 Bucket (Knowledge Base)  
│ raw/.md embeddings/.json  
│  
▼  
┌──────────────────────────────────────────┐  
│ Lambda: ChatBackend (RAG + Bedrock LLM) │  
│ - Loads embeddings │  
│ - Vector similarity search │  
│ - Calls Claude / Titan │  
└───────────────────┬──────────────────────┘  
▼  
┌──────────────────────────────┐  
│ Amazon Bedrock Foundation │  
│ Models: │  
│ - Claude 3.5 Sonnet (chat) │  
│ - Titan Embeddings G1 Text │  
└──────────────────────────────┘  



# 3. AWS Resources Required

## 3.1 Amazon S3 Bucket

    askme-kb
     └── kb/
         ├── raw/
         └── embeddings/



## 3.2 Lambda Functions

### 1. ChatBackend Lambda  
Main inference and RAG logic.

Responsibilities:
- Parse user message  
- Load embeddings from S3  
- Perform cosine similarity  
- Build enriched RAG prompt  
- Call Amazon Bedrock LLM  
- Return `{ reply: "..." }`  

Environment variables:

Environment:

    BUCKET=askme-kb
    BEDROCK_CHAT_MODEL=anthropic.claude-3-5-sonnet-20240620
    BEDROCK_EMBED_MODEL=amazon.titan-embed-text-v1
    SYSTEM_MESSAGE="You are AskMe..."

### 2. AdminUpload Lambda  
Secure interface for knowledge updates.

Responsibilities:
- Validate admin token  
- Accept `filename`, `content`, `token`  
- Write file into `kb/raw/`  

Environment variables:


    ADMIN_TOKEN=<secure-password>
    BUCKET=askme-kb


---

### 3. EmbeddingBuilder Lambda  
Auto-triggered on file uploads.

Responsibilities:
1. Load raw markdown  
2. Chunk text  
3. Generate Titan embeddings  
4. Save vectors to `kb/embeddings/*.json`  

Environment variables:


    BUCKET=askme-kb
    BEDROCK_EMBED_MODEL=amazon.titan-embed-text-v1

------------------------------------------------------------------------



# 4. API Gateway

AskMe uses an **HTTP API** with:


POST /chat → ChatBackend
POST /upload → AdminUpload

### CORS:
Access-Control-Allow-Origin: https://<github-pages-domain>
Access-Control-Allow-Methods: POST, OPTIONS
Access-Control-Allow-Headers: Content-Type

### Chat Request:
{ "message": "How does the VSP cycle work?" }

### Chat Response:
{ "reply": "Here is how the cycle works..." }
------------------------------------------------------------------------

# 5. Frontend

## index.html

Fetch example:

   fetch("https://<api-id>.execute-api.<region>.amazonaws.com/prod/chat", {
method: "POST",
headers: { "Content-Type": "application/json" },
body: JSON.stringify({ message: userText })
});



---

## admin.html – Knowledge Base Editor
Fields:
- Document name  
- Content text area  
- Admin token  
- Upload button  

Upload request:

POST /upload
{
"filename": "vsp_process.md",
"content": "Full text here...",
"token": "ADMIN_TOKEN"
}

------------------------------------------------------------------------

# 6. RAG Workflow

1.  User sends message\
2.  Embeddings loaded from S3\
3.  Cosine similarity\
4.  Build augmented prompt\
5.  Bedrock inference\
6.  Response returned



The knowledge retrieval pipeline operates as follows:

### **1. User sends a question**  
Frontend → `/chat` endpoint → ChatBackend Lambda  

### **2. ChatBackend loads all embeddings**  
Reads JSON vectors from S3 `embeddings/`.

### **3. Compute cosine similarity**  
Selects the most relevant knowledge chunks.

### **4. Construct augmented prompt**  

Format:
SYSTEM MESSAGE

Relevant context:
<top matching chunks>

User question:
<user message>

Assistant:


### **5. Bedrock model inference**  
Claude 3.5 Sonnet recommended for best reasoning.

### **6. Response returned to UI**  
Frontend displays the assistant’s answer.

---

# 7. IAM & Security

### ChatBackend
- `s3:GetObject`  
- `bedrock:InvokeModel`  

### AdminUpload
- `s3:PutObject`  

### EmbeddingBuilder
- `s3:GetObject`, `s3:PutObject`  
- `bedrock:InvokeModel`  

### API Access
- `/chat` is public  
- `/upload` requires correct admin token  

### S3
- Server-side encryption enabled  
- Least-privilege IAM principles applied  

---

# 8. Future Enhancements

- Add document editing & deletion  
- Support PDF ingestion with text extraction  
- Add conversation memory  
- Add analytics for user queries  
- Optional vector index with OpenSearch or Kendra  

---

# 9. Deliverables Checklist

✔ AskMe Bedrock-powered chatbot  
✔ RAG engine with S3 knowledge base  
✔ Automatic embedding generator  
✔ GitHub Pages UI (chat + admin)  
✔ API Gateway with `/chat` and `/upload`  
✔ Three Lambda functions  
✔ IAM-secure serverless architecture  
✔ Ready for Amazon forecasting use cases  

---

# 10. Prompt

Prompt:

Design and implement a clean, modern HTML page for an internal Amazon support chatbot. The page is fully dedicated to the chatbot experience.

Purpose & Context

This tool is for Amazon employees to get insights on how forecasting models work. 

The tone should be professional, friendly, and efficient, reflecting Amazon’s internal product style.

The layout must make the chat experience the clear focus of the page.

Layout & Structure

Header (Top Bar)

Left: Amazon logo (assume a provided logo image, e.g. amazon-logo.png).

Use alt text like: "Amazon logo".

Right: Small text label such as: “AskMe - AI assistant to expplain the VSP and Vesper forecasting models ”.

Background: subtle, light tone; keep it minimal and non-distracting.

Header should stay fixed at the top on scroll.

Intro Section (Above the Chat, Short & Simple)

A narrow content strip below the header with:

A short title, e.g.: “Understand everything about VSP and VESPER forecasts”.

2–3 short sentences explaining the tool, for example:

“This is an AI assistant to explain the VSP and Vesper forecasting models ”

“Ask a question in natural language and get quick, AI-powered answers.”



Add 3 small hint chips or bullet examples, e.g.:

“WHat is VSP and VEPSER?”
“WHat causes accuracy issues in the preductions? ”
“WHat actions are required to be taken to improve accuracy? ”



Main Chat Area (The Focus)

The chat area should dominate the page (especially on desktop).

Use a two-part layout:

A large chat window occupying most of the viewport.

A message input bar fixed at the bottom of the chat area.

Chat window behavior:

Show conversation as stacked message bubbles:

User messages right-aligned.

Assistant messages left-aligned, with a subtle icon or avatar indicating “bot”.

Use clear visual contrast between user and bot bubbles.

Support scroll for longer conversations (chat area should be vertically scrollable, not the whole page where possible).

Empty state:

When there is no conversation yet, show soft placeholder text like:

“Ask any questions about the VSP and VESPER forecasting models”

Input Area

A wide text input field at the bottom of the chat panel.

Placeholder text: “Type your question about the VSP and VESPER forecasting models…”.

A primary Send button to the right (icon of a paper plane is fine).

Hitting Enter sends the message; Shift+Enter creates a new line.



Status & Error Feedback

When the assistant is thinking / waiting for backend response:

Show a small “Assistant is thinking…” indicator in the chat area (e.g. animated dots bubble).

On network/API error:

Show a friendly inline error message in the chat panel:

“Something went wrong contacting the support assistant. Please try again.”

Keep the user’s input in the field so they can resend easily.

Footer (Optional, Very Minimal)

A very small footer line:

“For urgent or sensitive issues, contact your usual support channels.”

Keep it understated and non-distracting.

Visual Style & UX Details

Use a simple, enterprise-friendly design:

Light background, high contrast text.

Rounded corners on chat bubbles and input.

Generous spacing; avoid clutter.

Typography:

Clean sans-serif font.

Title slightly larger; body text normal size; hints maybe slightly smaller.

Colors:

Neutral base (whites / light grays).

Accent color(s) can be inspired by Amazon’s brand (e.g. subtle orange for buttons or highlights), but do not make the page loud.

Make it fully responsive:

On desktop: the chat area should be large and centered, with a reasonable max width.

On mobile: the chat should use almost the full screen width; header remains visible; input bar is easy to tap.

Behavior & API Integration

The page must be a standalone HTML file using vanilla JS (no frameworks required unless you choose a very lightweight approach).

On “Send”:

Immediately render the user’s message bubble in the chat window.

Then call a backend API over HTTPS.

API Contract:

Use fetch to POST to a chat endpoint, e.g.:

POST /chat

Headers:

Content-Type: application/json

Request body JSON:

{ "message": "<user message text>" }


The backend uses Amazon Bedrock; the frontend should not expose any internal model names.

Expected success response (JSON):

{ "reply": "<assistant answer>" }


On success:

Append the assistant’s reply as a new message bubble.

Scroll the chat view to the bottom.

On error (non-200 or network failure):

Show an assistant-style bubble with an error message.

Optionally allow a quick “Try again” action that resends the last user message.

Accessibility

Use semantic HTML where possible.

Ensure sufficient color contrast for text and buttons.

Support keyboard navigation:

Tab between input and send button.

Enter to send / Shift+Enter for new line.

Add aria-label attributes on buttons and important interactive elements (e.g. “Send message to support assistant”).

Deliverables

A single index.html file with:

Inline or linked CSS.

JavaScript embedded or linked (clear structure).

Code should be readable and easy to adapt:

Clearly mark where to configure the API URL.

Add comments explaining main sections (header, intro, chat, API call, error handling).
