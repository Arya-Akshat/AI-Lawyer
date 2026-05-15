# Database Schemas

The AI-Lawyer Platform uses MongoDB. Below are the definitions of all the collections (tables) and their schemas.

## 1. User Collection
The `User` collection stores information about all system users (Judges, Lawyers, and Citizens).

```javascript
const userSchema = new mongoose.Schema(
  {
    username: {
      type: String,
      required: true,
      unique: true,
      trim: true,
      index: true,
    },
    password: {
      type: String,
      required: true,
      trim: true,
    },
    refreshToken: {
      type: String,
    },
    role: {
      type: String,
      enum: ["user", "lawyer", "judge"],
      default: "lawyer",
    },
  },
  { timestamps: true }
);
```

## 2. Case Collection
The `Case` collection is the central data structure storing all case details. It is composed of several nested sub-schemas for organization and access control.

### Main Case Schema
```javascript
const CaseSchema = new mongoose.Schema(
  {
    CaseID: { type: String, required: true, unique: true, index: true },
    CaseName: { type: String, default: "Untitled Case" }, // LLM-generated case title for display
    LawyerID: { type: String, required: true, index: true },
    JudgeID: { type: String, required: true, index: true },
    UserID: { type: String, required: true, index: true },
    Evidence: { type: EvidenceClassSchema },
    Private: { type: PrivateSectionSchema },
    Public: { type: PublicSectionSchema },
  },
  { timestamps: true }
);
```

### Sub-Schemas used within Case

#### Evidence Schema
Categorizes all file paths and text representations of uploaded evidence.
```javascript
const EvidenceClassSchema = new mongoose.Schema(
  {
    photographs_and_videos: { type: [String] },
    official_reports: { type: [String] },
    contracts_and_agreements: { type: [String] },
    financial_records: { type: [String] },
    affidavits_and_statements: { type: [String] },
    digital_communications: { type: [String] },
    call_detail_records: { type: [String] },
    forensic_reports: { type: [String] },
    expert_opinions: { type: [String] },
    physical_object_descriptions: { type: [String] },
  },
  { _id: false }
);
```

#### Private Section Schema
Stores sensitive legal strategy and contact information visible only to authorized parties.
```javascript
const PrivateSectionSchema = new mongoose.Schema(
  {
    evidence_summary: { type: String },
    confidential_contacts: { type: [PersonDetailSchema] },
    privileged_communications: {
      type: Map,
      of: String,
    },
    legal_strategy_and_notes: { type: String },
  },
  { _id: false }
);
```

#### Public Section Schema
Stores court details, parties, and the general timeline accessible by all parties.
```javascript
const PublicSectionSchema = new mongoose.Schema(
  {
    court_details: {
      type: Map,
      of: String,
    },
    parties: {
      type: Map,
      of: [String],
    },
    case_type: { type: String },
    case_status: { type: String },
    case_summary: { type: String },
    timeline_of_proceedings: [
      {
        _id: false,
        date: { type: String },
        event: { type: String },
      },
    ],
  },
  { _id: false }
);
```

#### Person Detail Schema
A utility schema used for storing structured contact information.
```javascript
const PersonDetailSchema = new mongoose.Schema(
  {
    name: { type: String },
    role: { type: String },
    phone_number: { type: String },
    email_address: { type: String },
    address: { type: String },
  },
  { _id: false }
);
```

---

# System Design

## High-Level Architecture
The AI-Lawyer Platform utilizes a three-tier architecture augmented by specialized AI services and external integrations. 

### 1. Frontend (React / Vite)
- **Role:** Web-based user interface for Judges, Lawyers, and Citizens.
- **Tech Stack:** React 18, TypeScript, TailwindCSS, shadcn/ui.
- **Responsibilities:** 
  - Role-based dashboard rendering.
  - Document upload and case timeline visualization.
  - Interactive AI Counsel chat interface.

### 2. Express Backend (Node.js API)
- **Role:** Central orchestrator, authentication provider, and Telegram webhook receiver.
- **Tech Stack:** Node.js, Express, Mongoose, Redis.
- **Responsibilities:**
  - **Authentication:** JWT-based access and refresh token generation.
  - **Case Management:** CRUD operations for the MongoDB `Case` collection.
  - **Telegram Bot State Machine:** Uses Redis to manage user sessions as they upload case details and evidence via Telegram. 
  - **Routing:** Acts as a proxy and orchestrator, routing file processing requests to the Python AI Backend.

### 3. Python AI Backend (FastAPI)
- **Role:** Heavy-lifting AI service for document processing and RAG querying.
- **Tech Stack:** Python 3.13, FastAPI, LangGraph, FAISS, PyMuPDF.
- **Responsibilities:**
  - **Document Classification:** Parses uploaded evidence/documents, extracting text using OCR and PyMuPDF.
  - **Data Structuring:** Uses Gemini/Groq LLMs to automatically populate the complex Case schema (Public, Private, and Evidence sections).
  - **RAG System:** Chunks document text, generates embeddings via HuggingFace, and stores them in local FAISS vector stores (one per case).
  - **Agentic Workflow:** Employs LangGraph agents to answer legal queries, using the vector store for semantic search and falling back to Tavily web search if needed.

## Data Flow

### 1. Case Intake (via Telegram)
1. User interacts with the Telegram bot.
2. `backend_express` receives webhooks and tracks conversational state in **Redis**.
3. User uploads documents; `backend_express` temporarily saves them.
4. Once complete, `backend_express` sends the multipart data to `backend_lang` (`/classify`).
5. `backend_lang` parses and analyzes the documents using LLMs, returning structured JSON.
6. `backend_express` saves the structured JSON to **MongoDB**.

### 2. AI Counsel Query (RAG)
1. User submits a legal query via the Frontend.
2. Request is proxied through `backend_express` to `backend_lang` (`/rag/query`).
3. `backend_lang` invokes the LangGraph Agent.
4. Agent queries the **FAISS** vector database for similar document chunks.
5. If found, it formulates an answer. If not, it falls back to a web search.
6. The synthesized response is returned to the user.

## System Diagrams

### High-Level Architecture
![High-Level Architecture](images/High%20Level%20Archi.png)

### Case Intake Flow (via Telegram)
![Case Intake Flow (via Telegram)](images/Case%20Intake%20Flow.png)

### AI Counsel Query (RAG Flow)
![AI Counsel Query (RAG Flow)](images/AI%20Counsel%20Query.png)

## Component Interaction Diagram

![Detailed System Design](images/System%20Design.png)
