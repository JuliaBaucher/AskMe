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

+----------------------------------------------------+
| GitHub Pages Frontend |
| - index.html (Chat UI) |
| - admin.html (KB Upload UI) |
+-----------------------------+----------------------+
|
| HTTPS
v
+----------------------------------------------------+
| API Gateway (HTTP API) |
| Routes: /chat /upload |
+------------------+------------------+--------------+
| |
| | (Bearer Admin Token)
| v
| +----------------------------+
| | Lambda: AdminUpload |
| | - Stores files in S3 |
| +--------------+-------------+
| |
| v (S3 Trigger)
| +----------------------------+
| | Lambda: EmbeddingBuilder |
| | - Chunk + embed using |
| | Amazon Titan Embeddings |
| +--------------+-------------+
| |
| v
| +--------------------------+
| | S3 Knowledge Base |
| | - kb/raw/.md |
| | - kb/embeddings/.json |
| +------------+-------------+
| |
v v
+---------------------------------------------------------+
| Lambda: ChatBackend (RAG + Bedrock LLM) |
| - Loads embeddings |
| - Vector similarity search |
| - Calls Claude / Titan |
+----------------------------+----------------------------+
|
v
+---------------------------------------------------------+
| Amazon Bedrock Foundation Models |
| - Claude 3.5 Sonnet (chat) |
| - Titan Embeddings G1 Text |
+---------------------------------------------------------+
------------------------------------------------------------------------

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


------------------------------------------------------------------------

# 7. IAM & Security

-   Least privilege IAM\
-   Token-protected admin endpoint\
-   HTTPS only\
-   S3 encryption
