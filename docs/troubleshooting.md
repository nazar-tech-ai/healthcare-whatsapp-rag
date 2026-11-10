# Troubleshooting

## "Test" in Meta works, but real WhatsApp messages never reach n8n
The Meta app is in **Development** mode. In that mode only test webhooks sent from the dashboard are delivered. Switch the app to **Live** (App settings → Basic needs a valid **Privacy Policy URL** and an app category).

- The Privacy Policy URL must be publicly reachable without login. A Google Doc `/edit` link is rejected: use *File → Share → Publish to web* and paste the `/pub` link.
- Send the message **to the business/test number** from your personal WhatsApp, not the other way round.

## No execution appears in n8n
- The Callback URL in Meta must be the WhatsApp Trigger's **Production URL**, and the workflow must be **Published**. If you re-create the trigger node the URL changes.
- The `messages` field must be **Subscribed** (Configuration → Webhook fields).
- Only one WhatsApp Trigger can exist per Meta app. Deactivate older workflows that use another trigger.
- Check the WhatsApp Business Account is subscribed to the app.

## Pinecone: "This index has a document schema, so writes must go through the documents API"
The index was created as an integrated-embedding / document-schema index. n8n needs a **standard dense index** with your own vectors. Create a new index with custom settings, dimension equal to your embedding size (3072 for the Gemini embedding used here), metric `cosine`.

## Bot replies "I couldn't find enough information in the healthcare documents"
1. Open a record in Pinecone and check its metadata: `patientPhone` must equal the sender's WhatsApp number (digits only, with country code).
2. The insert and search nodes must use the **same embedding model** and the same index.
3. Search `topK` should be about 8 with the reranker's Top N about 4.
4. The Pinecone search description must not tell the model to add an MRN, the filter already restricts by phone.

## Replies stop after about a day
The temporary access token from "API Setup" expires after 24 hours. Use a System User token with no expiry.

## Postgres chat memory errors
- Enable SSL in the Postgres credential (Neon requires it) and use the direct (non-pooled) host.
- The memory node's Session ID must be a custom key built from the sender's `wa_id`. An empty or shared key mixes different patients' conversations.

## Non-text message gets no reply
The *WhatsApp - Send Text Only Notice* node needs a WhatsApp API credential selected, like the main reply node.

## Executions count rises faster than the number of messages
Meta sends sent/delivered/read status events to the same webhook field, and each one starts an execution. They are filtered out early, but still count.
