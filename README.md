

## Overall canvas structure

On the canvas, create **separate clusters** (visually spaced sections) for:

1. **Input & Setup (Step 0)**
2. **Research (Step 1)**
3. **Analysis / Agents (Step 2)**
4. **Writing / Draft Generation (Step 3)**
5. **Fact & Content Management (Step 4)**
6. **On‑page SEO (Step 5)**
7. **Approval (Step 6)**
8. **Images (Step 7)**
9. **WP Draft & Distribution (Steps 8–10)**
10. **SoMe Content & Media (Step 10)**
11. **Final SoMe Approval & Posting (Steps 11–13)**

Use **Sticky Notes** (as in the AI examples & advanced AI docs) to label each cluster in human terms: “Research bots”, “Writers”, “Fact checker”, “SEO bots”, etc. [[AI fallback](https://docs.n8n.io/advanced-ai/examples/human-fallback/)]

Where steps are “semi‑detached”, you can visually separate them and (optionally) imagine them as separate workflows later; for the prototype, keep them on one canvas.

---

## Step 0 – Human input: brand + market

**Goal:** Human tells the system “Betsson + Norway”.

**Nodes:**

- **n8n Form** (or **Manual Trigger** while prototyping)
  - Fields: `brand`, `market` (dropdown or free text). [[Form templates](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/#templates-and-examples)]
- **Set / Edit Fields** node
  - Normalize into e.g. `brandName`, `country`, `locale`.

**Sticky note (yellow):**  
“CEO / marketer tells the system which brand & market we’re building a page for.”

---

## Step 1 – Research agents (SERP + brand + PAA)

### 1.1 Research Agent 1 – SERP (Google/Bing)

**Pattern:** AI Agent + SerpAPI tool.

**Nodes:**

- **AI Agent** (root AI agent node)  
  - Prompt: “Research competitors for {{ $json.brandName }} in {{ $json.country }}. Use SerpAPI to get top 10 organic results on Google and Bing and return URLs & titles.” [[AI agent chat](https://blog.n8n.io/llm-agents/#how-to-create-an-llm-agent-workflow-with-n8n)]
- **OpenAI Chat Model** node connected as **language model** to Agent. [[LLM agent setup](https://blog.n8n.io/llm-agents/#how-to-create-an-llm-agent-workflow-with-n8n); [OpenAI node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/#openai-node)]
- **SerpAPI (Google Search) tool** node connected as **ai_tool** to Agent. [[SerpApi tool](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolserpapi/#serpapi-google-search-node)]

Agent returns a list of competitor URLs + metadata.

### 1.2 Capture content from each site

**Nodes:**

- **Split Out** (to iterate URLs one by one). [[Scrape + summarize example](https://blog.n8n.io/how-to-scrape-data-from-a-website/#bonus-can-chatgpt-scrape-the-web)]
- **HTTP Request** node
  - GET each URL, store HTML in `binary` or `json.html`. [[Web scraping with HTTP Request](https://blog.n8n.io/how-to-scrape-data-from-a-website/#bonus-can-chatgpt-scrape-the-web)]
- **HTML Extract** node
  - Extract `main content`, `title`, etc with CSS selectors. [[HTML Extract](https://blog.n8n.io/how-to-scrape-data-from-a-website/#bonus-can-chatgpt-scrape-the-web)]

### 1.3 Store content (SERP research memory)

Use Supabase as the central store.

- **Supabase Vector Store** node – **Insert Documents**
  - Store: page text as `content`, plus metadata: `brand`, `market`, `url`, `type: "competitor_page"`. [[Supabase vector store](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoresupabase/#supabase-vector-store-node); [Insert mode](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoresupabase/#node-parameters)]
- Optionally also a plain **Supabase** node – “Create row” in a table for easy reporting. [[Supabase node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.supabase/#supabase-node)]

**Sticky note:** “Research Agent 1: pulls top 10 competitors from Google/Bing, scrapes their pages, and stores them in Supabase as long‑term knowledge.”

---

### 1.4 Research Agent 2 – Brand‑specific info

From the Excel: welcome bonus, T&Cs, license, local payments.

You can either:

- Reuse search + scraping on the brand’s own domain, or
- Directly scrape known brand URLs (welcome page, T&Cs, payments page).

**Nodes:**

- **AI Agent** + **OpenAI** + **SerpAPI tool** again, but with a prompt specialized for:
  - “Search for '{{ brandName }} {{ market }} welcome bonus', 'terms and conditions', 'license', 'payment methods'. Return candidate URLs for each category.”
- Follow with:
  - **Split Out** → **HTTP Request** → **HTML Extract** (per URL).
- **Supabase Vector Store – Insert Documents**
  - metadata: `type: "brand_bonus" | "brand_terms" | "brand_payments" | ...`.

**Sticky note:** “Research Agent 2: gathers first‑party info about the brand – welcome bonus, T&Cs, license, payments – stored with rich metadata in Supabase.”

---

### 1.5 Research Agent 3 – PAA (People Also Ask)

If your SerpAPI config returns PAA, the Agent can ask for them; otherwise you approximate via search.

**Nodes:**

- Same **AI Agent** + **SerpAPI tool** pattern, but prompt:
  - “Collect People Also Ask‑style questions users search for around '{{ brandName }} {{ market }} casino’.”
- Store questions in **Supabase** (table `paa_questions`) with `brand`, `market`, `question`, `source`. [[Supabase node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.supabase/#supabase-node)]

---

## Step 2 – Analyse content (specialized “agents”)

Conceptually, each of the “Bonus / Payment / History / Regulation / Slots / WD / Support / Responsible Gaming” agents is:

> “Take relevant pages from Supabase Vector Store + run an AI analysis to extract structured fields.”

**Common pattern for each sub‑agent:**

1. **Supabase Vector Store – Get Many / Retrieve Documents (As Tool for AI Agent)** with:
   - **Table Name** set to your embeddings table.
   - Query prompt: “bonus info for {{ brandName }} in {{ market }}” etc. [[Vector Store retrieve modes](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoresupabase/#node-parameters)]
2. **AI Agent** with **OpenAI** model
   - System prompt: “You are the Bonus Casino Agent. From the retrieved documents, extract: Welcome bonus amount, wagering requirements, expiry, notable constraints. Output strict JSON with keys ...”
3. **Set / Edit Fields** node
   - Clean up and map final JSON shape.
4. **Supabase** node
   - “Create/Update row” in a `casino_analysis` table with columns like `bonus`, `payments`, `history`, etc.

Do this as **parallel branches** (Bonus Agent, Payment Agent, History Agent, Regulation Agent, Slots Agent, WD Agent, Support Agent, Responsible Gaming Agent).

To recombine:

- **Merge** node in **Mode: Combine / Matching Fields**, matching on e.g. `brandName + market` so you end up with a **single aggregated record** per casino before writing it to Supabase. [[Merge – Matching Fields](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/#node-parameters); [Try‑it example](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/#try-it-out-a-step-by-step-example)]

**Sticky note:** “These are specialist analysts: Bonus, Payments, History, Regulation, Games, Withdrawals, Support, Responsible Gambling. Each reads from the shared Supabase research store and produces structured fields for writers.”

---

## Step 3 – Writer agents (12 sections)

Here you can impress the CEO with a line of “Writer” nodes.

**Pattern per writer:**

- **OpenAI node – Generate a Model Response** or **Generate a Chat Completion** (Text resource). [[OpenAI text ops](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/text-operations/#generate-a-model-response)]
  - Prompt uses:
    - `brandName`, `market`
    - Fields from Step 2 analysis (bonus, licenses, payments, etc)
  - Each writer is responsible for a section:
    - Writer 1: headline + intro
    - Writer 2: welcome bonus text
    - …
    - Writer 12: FAQ (can use PAA questions from Step 1.5).

When done:

- **Merge** node (**Mode: Append** then later **Combine by Position** or simply join text via a **Code** node) to consolidate into a **full draft object**:
  - `sections: { headline, intro, bonus, payments, history, ... }`. [[Merge examples](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/#templates-and-examples)]

**Sticky note:** “Each Writer Agent uses a consistent template + the structured analysis to generate a specific section of the review. Together they form a full casino review draft.”

---

## Step 4 – Fact Manager & Content Manager

### 4.1 Fact Manager (hallucination check)

Use a second model or Perplexity to cross‑check content vs research.

**Nodes:**

- **Perplexity node – Message a Model**  
  - Prompt: “Here is the research summary (from Supabase) and here is the section draft. Identify any factual errors or hallucinations and answer true/false for ‘safe_to_publish’ plus a list of issues.” [[Perplexity node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.perplexity/#perplexity-node)]
- **IF** node
  - Condition: `safe_to_publish === true`.
  - **True:** pass section forward.
  - **False:** loop back to relevant Writer node(s).

Looping can be modeled with an explicit loop as in n8n docs: connect node output back to earlier Writer nodes, controlled by IF. [[Loop until condition](https://docs.n8n.io/flow-logic/looping/#loop-until-a-condition-is-met)]

Also add a **Slack** “Send and Wait for Approval” node if the Fact Manager finds repeated hallucinations:

- **Slack node – Message → Send and Wait for Approval**  
  - Slack text: “Fact Manager flags hallucinations in sections 3 & 8 for {{ brandName }} – human review required.” [[Slack operations](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/#slack-node)]

### 4.2 Content Manager – combine full draft

**Nodes:**

- **OpenAI node** (Text → Generate a Model Response)
  - Prompt: “You are the Content Manager. Given all section drafts (JSON), merge them into a single long‑form casino review according to this template (H1/H2 structure, order, transitions). Do not invent facts.”
- Output: `fullDraftHtml` and `fullDraftPlain`.

**Sticky note:** “Fact Manager cross‑checks content against research; if hallucinations persist, it escalates to human in Slack. Content Manager merges all validated sections into a polished long‑form article.”

---

## Step 5 – On‑page SEO agents

Treat each as a small specialized AI pass over the draft.

**Common input:** `fullDraftHtml` from Step 4.

Per SEO agent, use an **OpenAI** text operation node with a narrowly defined prompt and return updated content or metadata:

1. **On page SEO 1 – Headings**  
   - Prompt: optimize H1–H3 structure for target KW while keeping meaning.
2. **SEO 2 – KW density**  
   - Prompt: analyze & adjust keyword density within N–M%.
3. **SEO 3 – Word count checker**  
   - Simple **Code** node or **OpenAI** summary to enforce min/max word counts.
4. **SEO 4 – Internal links**  
   - Prompt: based on a list of internal URLs (from Supabase or static config), insert 3–5 contextually relevant internal links.
5. **SEO 5 – Schema adder**  
   - Generate JSON‑LD (review schema) as a separate field.
6. **SEO 6 – Table of contents**  
   - Insert / fix TOC at top.
7. **SEO 7 – Meta optimizer**  
   - Output `metaTitle`, `metaDescription`.
8. **SEO 8 – Slug optimizer**  
   - Output `slug`.

Use **Merge** node(s) to consolidate SEO outputs back into one content object. [[Merge combine](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/#node-parameters)]

**Sticky note:** “SEO sub‑agents refine the article: headings, keyword balance, word count, internal links, schema, meta, slug.”

---

## Step 6 – Approval Agent (HITL via Slack)

**Nodes:**

- **Slack – Send and Wait for Approval**  
  - Message: preview of title + summary, link to a draft (e.g. WordPress preview or internal URL).
  - If **approved**, branch continues.
  - If **rejected**, loop back to Content Manager / relevant SEO nodes. [[Slack approval op](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/#slack-node); [Looping until approved](https://docs.n8n.io/flow-logic/looping/#loop-until-a-condition-is-met)]

This implements a human‑in‑the‑loop approval like in the docs’ HITL examples. [[Human fallback example](https://docs.n8n.io/advanced-ai/examples/human-fallback/)]

---

## Step 7 – Image creation

Two options: pure AI (OpenAI Image) + template editing.

**Nodes:**

- **OpenAI – Image → Generate an Image** for:
  - Logo variant
  - Featured image
  - Infographic base. [[OpenAI image ops](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/image-operations/#openai-image-operations)]
- **Edit Image** node
  - Add watermark / border / resize to platform‑specific sizes. [[Edit Image](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage/#edit-image)]

**Sticky note:** “Image agents: generate on‑brand images and then apply standard watermark / sizing with Edit Image.”

---

## Step 8–9 – WP draft & human posting

**Nodes:**

- **WordPress node – Post → Create a post**
  - Title, content HTML, meta, slug from SEO step.
  - Status: `draft`. [[WordPress node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/#wordpress-node)]
- **Slack – Send message**  
  - “New WP draft for {{ brandName }} is ready for review: {{ wp_preview_url }}.”
- Manual human step: “Content posted on WP by human” (represented as a **Sticky note** only).

**Sticky note:** “WP Distribution Agent: auto‑creates drafts, but human retains final publish control.”

---

## Step 10 – Distribution agent (SoMe text + imagery + video)

Here you won’t use specific social nodes from the docs (not covered above), but you can show the logic clearly.

### 10.1 Understand the offer & decide where to post

- **OpenAI node**  
  - Prompt: “From this article, extract the core offer and choose which channels to post on (Twitter/X, Instagram, Facebook, Discord, Telegram, WhatsApp groups) based on these rules. Return JSON: { channels: [...], key_messages: [...], compliance_notes: ... }.”

### 10.2 SoMe image sizes creation

- **Edit Image** node
  - Use **Resize** to create variants per platform: IG feed, story, Twitter, Facebook, etc. [[Edit Image resize](https://blog.n8n.io/ai-workflow-automation/#how-to-use-ai-to-automate-workflows-in-n8n)]

### 10.3 SoMe video creator agent

You can model it conceptually with:

- **Google Gemini node – Video → Generate a Video** or **Image + Text → Video** if you’re using Gemini for that; or simply a **Sticky note** + **Google Gemini node** to generate script and storyboard. [[Google Gemini node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini/#google-gemini-node)]

### 10.4 SoMe writer for captions

- **OpenAI node**
  - Prompt: “Write channel‑specific captions for: Twitter, Instagram feed, Instagram story, Facebook post, Discord announcement, Telegram group, WhatsApp group. Respect character limits and tone.”

Store all of this in Supabase or a dedicated table.

---

## Steps 11–12 – SoMe approval loop

Same HITL pattern as Step 6:

- **Slack – Send and Wait for Approval**
  - Show a summary of channels + captions + sample image/video.
- IF rejected → loop back to SoMe writer / distribution logic. [[Loop until condition](https://docs.n8n.io/flow-logic/looping/#loop-until-a-condition-is-met)]

Then:

- **Slack – Send message** for final “Human check OK; ready to publish”.

Sticky notes: “Second Approval Agent for social content; loops until approved.”

---

## Step 13 – Post agent (final posting)

Because social‑media‑specific posting nodes aren’t in the provided docs, you can model this abstractly:

- **HTTP Request** node(s) for generic API posting (Twitter, Facebook, etc.), with a note: “Can be wired to platform APIs once we agree scope and credentials.” [[HTTP Request usage example](https://blog.n8n.io/build-a-fast-deep-research-automation-flow-with-oxylabs-and-n8n/#building-a-fast-deep-research-flow-in-n8n)]

**Sticky note:** “Post Agent: publishes only content that has passed research, writing, fact‑checking, SEO, and dual human approvals.”

---
