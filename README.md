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

# UX Design Prompt: Amazon Internal Support Chatbot

## Project Overview
Design a professional, user-friendly HTML page for Amazon's internal employee support chatbot named "AskMe". This tool is specifically designed to help Amazon employees understand how VSP (Vendor Sourcing Portal) and VESPER forecasting models work. The page should be fully dedicated to the chatbot experience and serve as an educational resource for forecasting insights. This page should fetch an API to call the backend. 
fetch("https://mjnq8gk6vh.execute-api.eu-north-1.amazonaws.com/prod/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ message: userText })
});


Return HTML in one editor block.

## Key Requirements

### Visual Identity
- Incorporate Amazon's official logo prominently in the header
- Use Amazon's brand colors: primarily Amazon Orange (#FF9900) and black/dark gray
- Maintain a clean, professional aesthetic that aligns with Amazon's internal tools
- Ensure the design feels trustworthy and authoritative

### Layout & Structure
- **Chat-First Design**: The chat interface should dominate the page (occupy 75-85% of viewport)
- **Fixed Header**: Sticky top bar containing:
  - Amazon logo on the left (use `amazon-logo.png` with alt text "Amazon logo")
  - Tool name/title on the right: "AskMe - AI assistant to explain the VSP and Vesper forecasting models"
  - Subtle, light background that doesn't distract from content
- **Brief Intro Section**: Narrow content strip immediately below header with:
  - Short, impactful title: "Understand everything about VSP and VESPER forecasts"
  - Optional: 1-2 sentence description of what the chatbot can help with
  - Minimal height to maximize chat window space
- **Dominant Chat Window**: Full-height, scrollable chat interface below intro
- **Responsive Design**: Primarily optimized for desktop monitors (internal tool)

### Chat Interface Components
- Clear message input field with send button
- Scrollable message history with distinct styling for:
  - User messages (right-aligned, different background)
  - Bot responses (left-aligned, different background)
- Timestamp indicators
- Typing indicator when bot is processing
- Loading states for API calls
- Error handling UI for failed requests

### Content & Messaging
**Primary Use Case**: Educational tool for understanding forecasting models
- What the chatbot can help with:
  - Explaining how VSP forecasting models work
  - Clarifying VESPER model methodology
  - Answering technical questions about forecasting logic
  - Providing insights into model parameters and outputs
- Availability: Real-time AI-powered responses via AWS Bedrock backend

**Intro Section Content**:
- Title: "Understand everything about VSP and VESPER forecasts"
- Brief subtitle (optional): "Ask me anything about how our forecasting models work"

### Technical Integration
- API integration with AWS Bedrock backend (following patterns from provided README files)
- Backend API endpoint structure similar to index.html implementation in documentation
- RESTful API calls with proper headers and authentication
- Request format should match:
  ```javascript
  {
    "message": "user question",
    "sessionId": "unique-session-id",
    // additional parameters as per README
  }
  ```
- Proper error handling and retry logic
- Loading states during API requests
- Message streaming support if the backend provides it (typical for Bedrock responses)
- Session management to maintain conversation context

### User Experience Features
- Auto-focus on input field when page loads
- Enter key to send messages
- Clear conversation button
- Smooth scrolling to latest message
- Visual feedback for all interactions
- Accessible design (keyboard navigation, screen reader support)

### Tone & Voice
- **Professional yet approachable**: This is an internal educational tool, not customer-facing
- **Friendly and efficient**: Reflecting Amazon's internal product style
- **Technical but accessible**: Can discuss complex forecasting concepts clearly
- **Helpful and solution-oriented**: Focused on helping employees understand models
- **Aligned with Amazon's Leadership Principles**: Learn and Be Curious, Dive Deep
- Clear and concise messaging that respects employees' time

## Design Specifications

### Page Structure
1. **Header (Fixed Top Bar)**
   - Height: 60-70px
   - Left side: Amazon logo (amazon-logo.png, ~40px height)
   - Right side: Tool identifier "AskMe - AI assistant to explain the VSP and Vesper forecasting models"
   - Background: Subtle light tone (#FAFAFA or similar)
   - Stays fixed on scroll

2. **Intro Section (Minimal)**
   - Max height: 80-100px
   - Centered or left-aligned title: "Understand everything about VSP and VESPER forecasts"
   - Optional brief description below title
   - Clear visual separation from chat area

3. **Chat Interface (Primary Focus)**
   - Occupies remaining viewport height (calc(100vh - header - intro))
   - Fixed height with internal scroll
   - Message bubbles clearly differentiated
   - Input area fixed at bottom of chat container

### Recommended Color Palette
- Primary: Amazon Orange (#FF9900)
- Secondary: Amazon Black (#232F3E)
- Background: White (#FFFFFF) or light gray (#F5F5F5)
- Chat bubbles: Light blue (#E3F2FD) for bot, light gray (#F1F1F1) for user
- Text: Dark gray (#333333) for primary, medium gray (#666666) for secondary

### Typography
- Clean, readable sans-serif font (Arial, Helvetica, or Amazon Ember if available)
- Clear hierarchy between headers, body text, and chat messages
- Minimum 14px font size for body text, 16px for chat messages

### Spacing & Layout
- Generous whitespace to reduce cognitive load
- Clear separation between introduction and chat interface
- Comfortable padding around chat messages
- Minimum 44px touch targets for interactive elements

### Interactive States
- Hover effects on buttons
- Focus states for keyboard navigation
- Disabled states when processing
- Loading animations that don't distract

## Success Metrics
The design should optimize for:
- Quick time-to-first-interaction
- Clear understanding of chatbot capabilities
- Minimal friction in sending messages
- Professional appearance that builds trust
- Fast perceived performance

## Implementation Notes
- Single HTML file with embedded CSS and JavaScript
- Use Fetch API for backend communication
- Handle authentication tokens if required by Bedrock API
- Implement proper CORS handling
- Add error boundaries for API failures
- Consider rate limiting on client side

## Out of Scope (Future Enhancements)
- Multi-language support
- File upload capabilities
- Conversation history persistence
- User authentication UI
- Analytics dashboard
