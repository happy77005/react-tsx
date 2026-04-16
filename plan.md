Plan: Intelligent Email Control Center - MVP 1

1. Project Foundation & Dependencies
Install core dependencies: @supabase/Bolt Database-js, react-router-dom, axios or ky for HTTP requests
Install UI libraries: tailwindcss and shadcn/ui (or equivalent) for the clean modern dashboard style
Install Google OAuth library: @react-oauth/google for handling Gmail authentication flow
Set up folder structure: src/components, src/pages, src/hooks, src/services, src/types, src/utils
Initialize Local Indexed DB Database

2. Local Database Schema Setup 
Create messages table with fields: id, user_id, provider_message_id, thread_id, sender, subject, body_snippet, received_at, raw_payload_ref, category, priority_score, priority_bucket, classification_reason
Create tasks table with fields: id, message_id, user_id, title, priority, status, created_at
Create users table to store Google OAuth profile and access token references
Enable Row Level Security (RLS) on all tables so users only see their own data
Create a classification_cache table to avoid re-classifying the same emails


3. Google OAuth & Gmail Integration
Configure Google OAuth consent screen credentials in the app using VITE_GOOGLE_CLIENT_ID
Build an AuthContext provider that manages login state, stores the Google access token in Bolt Database session
Request Gmail OAuth scopes: gmail.readonly and openid email profile
Build a GmailService class that uses the Gmail REST API to:
Fetch inbox message list with pagination
Fetch full message details (sender, subject, snippet, body, timestamp, threadId)
Normalize raw Gmail response into the canonical Message schema
Store fetched and normalized messages into the Bolt Database messages table
Implement local-first indexing: check Bolt Database before re-fetching from Gmail API to avoid redundant calls

4. Classification Engine
Build a ClassificationService that processes each message through two layers:
Rule-based layer: regex patterns for known categories (payment keywords, delivery tracking, rejection phrases, promotional language patterns, spam indicators)
LLM layer: call an AI provider (OpenAI or Anthropic) with a structured prompt to classify the email into: rejection, payment_success, order_delivery, action_required, spam_promo, low_priority
Generate a classification_reason string (e.g., "promotional language detected", "action verb found") for explainability
Build a PriorityScoringService that calculates priority_score using:
Category weight (action_required = highest, spam_promo = lowest)
Urgency keyword scan (deadline, urgent, expires, confirm, verify)
Sender importance (known domains, reply-to patterns)
Map score to priority bucket: critical, high, medium, low
Persist classification and score back to the messages table in indexed DB


6. Task Extraction Engine

Build a TaskExtractionService that scans messages classified as action_required or with critical/high priority
Use regex patterns to detect actionable language: scheduled appointments, payment deadlines, confirmation requests, form completions
For ambiguous cases, use an LLM prompt to extract a clean task title from the email
Create Task records in the Bolt Database tasks table linked to the originating message
Assign task priority based on the parent message's priority bucket
6. UI: Three-Panel Dashboard

Build a root layout with a left sidebar for navigation and a main content area
Panel 1 - Unified Inbox: display all fetched messages sorted by received_at, show sender, subject, snippet, category badge, and priority badge; clicking a message opens a detail view
Panel 2 - Priority Feed: filter and display only critical and high priority messages; include visual indicators (color-coded priority badges); show a quick summary of how many items need attention
Panel 3 - Task Panel: list all extracted tasks with title, source email, priority, and status; allow marking tasks as done (status update only, no deletion)
Detail view for individual messages: show full classification result, priority score, and the explainability reason string
7. Low Priority Cleanup Feature

Build a "Low Priority" section that groups all low bucket messages
Display a count header: "These N emails are low priority"
Render a checklist where all items are pre-selected by default
Allow the user to uncheck/deselect specific emails they want to keep visible
Provide a "Mark as Reviewed" or "Archive" action button for the remaining selected items (update status in Bolt Database, UI removes them from the list)
Persist user's deselection choices so they survive page refresh
8. Explainability Layer

On each message card and in the detail view, display a "Why this priority?" tooltip or inline section
Show the classification_reason field in plain English
Display contributing factors as a short list (e.g., "Promotional language detected", "No action verb found", "Sender matches known newsletter pattern")
Show the category label alongside the reason for full transparency
9. Background Sync & State Management

Build a useEmailSync hook that triggers Gmail fetch on first login and on manual refresh
Use React Context or Zustand for global state: current user, messages list, tasks list, sync status
Show a loading/syncing indicator when fetching from Gmail
Handle Gmail API rate limits gracefully with retry logic and user-visible feedback
10. plan.md Creation

Write a comprehensive plan.md file at the project root documenting:
Phase 1: Foundation, auth, DB schema
Phase 2: Gmail ingestion and normalization
Phase 3: Classification engine and priority scoring
Phase 4: Task extraction
Phase 5: UI panels and explainability
Phase 6: Low priority cleanup and polish
Each phase lists its goals, deliverables, and dependencies on prior phases
Clarifying Questions:

For the LLM classification layer, do you have a preferred AI provider (OpenAI GPT-4o, Anthropic Claude, or Gemini), or should the plan include whichever is most cost-effective for a demo?

For the Gmail sync frequency, should the initial fetch pull the last N emails (e.g., last 100 or 200), or should it fetch all inbox emails up to a configurable limit?

For the "archive" action in the low priority cleanup, should this only update the status in your local Bolt Database DB (safe, no changes to Gmail), or should it eventually write back to Gmail to actually archive in Gmail as well (requires gmail.modify scope)?