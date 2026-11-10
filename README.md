# Healthcare Patient Guide: WhatsApp RAG Bot (n8n)

A WhatsApp assistant that answers patients' questions about **their own** medication plan (dose, timing, food instructions, missed doses, side effects, follow-ups, emergency contacts). It runs on [n8n](https://n8n.io) and uses retrieval-augmented generation (RAG) over the patients' PDF medication schedules.

> **Disclaimer:** This is a demo/prototype, not a medical device. It only repeats what is written in the uploaded plans and tells users to contact their doctor for any change. Review privacy and legal requirements before using it with real patient data.

## How it works

```
                        ┌──────────────── Ingestion (run once per PDF batch) ────────────────┐
Upload form (Basic Auth)│                                                                    │
   ──► Split files ──► Extract PDF text ──► Chunk by CLAUSE + patient header ──► Gemini embeddings ──► Pinecone
                                                                                                          ▲
                        └────────────────────────────────────────────────────────────────────┘          │
                                                                                                          │ filter: patientPhone = sender's number
WhatsApp user ──► WhatsApp Trigger ──► text message? ──► AI Agent (Gemini) ──► Pinecone search (+ Cohere rerank)
                                          │                    │  ▲
                                          │ no                 │  └── Postgres chat memory (per phone number)
                                          ▼                    ▼
                               "Please send text"      WhatsApp reply
```

Key design points:

- **Patient identity = the sender's WhatsApp number.** Each chunk is stored with `patientPhone` metadata (digits only, e.g. `923001234567`). At query time the Pinecone search is filtered with the sender's `wa_id`, which WhatsApp sets and the user cannot spoof. The model only ever sees that patient's chunks.
- **Clause-aware chunking.** PDFs are split on `CLAUSE n` headings and every chunk starts with a header `[Patient | MRN | Plan date | Ref]`, so the medication table stays in one chunk and chunks are never ambiguous between patients.
- **Roman Urdu / Urdu / English.** The agent rewrites the question into an English search query and answers in the user's language.
- **Non-text messages** (voice, images) get a short "please send text" reply. Delivery/read status events are ignored.
- **Reliability.** Retry on the agent, a fallback Gemini model, Cohere reranking, temperature 0.

## Repository layout

```
.
├── workflow/healthcare-patient-guide-whatsapp-rag.json   # n8n workflow (credentials removed)
├── samples/                                               # 3 sample PDFs with fictional phone numbers and CNICs
├── docs/troubleshooting.md                                # problems we hit and how to fix them
├── .gitignore
└── README.md
```

## Requirements

| Service | Used for | Notes |
|---|---|---|
| n8n (Cloud or self-hosted) | Orchestration | One WhatsApp Trigger per Meta app |
| Meta WhatsApp Cloud API | Messaging | App must be in **Live** mode for real messages |
| Pinecone | Vector store | **Standard (non-integrated) dense index**, dimension must equal your embedding size (3072 with the Gemini embedding node used here), metric `cosine` |
| Google Gemini API | Chat model + embeddings | Model names are set in the Gemini nodes, change to what your key supports |
| Cohere | Reranker | Free trial key is enough |
| Postgres (e.g. Neon) | Chat memory | SSL required, use the direct (non-pooled) connection |

## Setup

### 1. Pinecone
Create a **standard** index (custom settings, *not* the integrated-embedding / document-schema type), dimension `3072`, metric `cosine`, and name it `medication-n8n` (or change it in the three Pinecone nodes).

### 2. Postgres chat memory
Create a free Postgres database (for example on Neon) and note host, database, user, password. The memory node creates the `chat_histories` table on first use.

### 3. Import the workflow
In n8n: **Workflows → Import from file →** `workflow/healthcare-patient-guide-whatsapp-rag.json`. Then create and select credentials for every node that shows a warning:

- Google Gemini (PaLM) API: chat model, fallback model, both embedding nodes
- Pinecone API: insert node and search node
- Cohere API: reranker
- Postgres: chat memory (SSL on)
- HTTP Basic Auth: upload form (choose your own username and password)
- WhatsApp API (access token + business account ID) and WhatsApp OAuth (trigger)

Replace `YOUR_PHONE_NUMBER_ID` in the two WhatsApp send nodes with your Phone Number ID.

### 4. Meta / WhatsApp
1. In your Meta app, add the WhatsApp product and copy the **Production URL** of the WhatsApp Trigger node into **Configuration → Callback URL**; set the verify token and **Verify and save**.
2. Subscribe to the **`messages`** webhook field.
3. Switch the app to **Live** (requires a public Privacy Policy URL and an app category). In Development mode Meta only delivers test webhooks.
4. Create a **System User** token (`whatsapp_business_messaging`, `whatsapp_business_management`, never expires) and use it in the WhatsApp API credential. Temporary tokens expire after 24 hours.
5. Publish the n8n workflow.

### 5. Load patient plans
Open the upload form's Production URL, log in with the Basic Auth credentials and upload the PDFs. Upload each file **once**, re-uploading creates duplicate records (enable *Clear Namespace* on the insert node for a single re-load, then turn it off again).

The sample PDFs in `samples/` use fictional numbers (`+92-300-0000001` to `+92-300-0000003`). To test with **your own** WhatsApp number, either edit a copy of the PDF, or set `TEST_PHONES` in the *Chunk By Clause With Patient Header* node, for example:

```js
const TEST_PHONES = { 'MRN-2026-51067': '923XXXXXXXXX' };
```

Use digits only, with country code and no `+`. Do not commit real numbers.

### 6. Test
From the phone registered for a patient, message the business number:

| Message | Expected |
|---|---|
| `Amlodipine kitni dafa leni hai` | Dose from that patient's plan only |
| A medicine that belongs to another patient | "not found" style answer, never the other patient's data |
| A voice note or image | "Please send your question as text" reply |
| An unregistered number | "no registered medication plan" reply |

## Privacy and security notes

- Never commit real patient PDFs, tokens, API keys or phone numbers (`.gitignore` blocks `*.pdf` outside `samples/`).
- The upload form is protected with Basic Auth, keep its URL and password private.
- Chat history is stored in Postgres, decide on retention and access before real use.
- Patient identification relies on the phone number only. Add stronger verification for production.
- Every WhatsApp status event (sent/delivered/read) triggers an n8n execution, which counts toward plan limits.

## Troubleshooting

See [docs/troubleshooting.md](docs/troubleshooting.md).
