# Lillibot
A Telegram-integrated AI assistant that summarizes online articles using scraped markdown content. Built with n8n, Firecrawl, and OpenAI, it supports follow-up questions via memory and provides clean, structured summaries in real time.

![image](https://github.com/user-attachments/assets/65f33f5c-1c8f-4000-bc0c-640686ea2fe6)

---

## How to Build Lillibot Using n8n

This guide helps you build an end-to-end Telegram chatbot using **n8n**, **OpenAI**, and **Firecrawl**. The bot accepts a URL, scrapes the article, summarizes it using GPT-4o, and replies on Telegram. It can also handle follow-up questions using memory.

You can either **import the full workflow** using the provided JSON file, or **build it manually** from scratch by following the steps below.

---

## Option 1: Import the Prebuilt Workflow

1. Download [`telegram-summarizer-workflow.json`](https://github.com/saiganeshvenk00/Lillibot/blob/main/Summary_Gen_Bot.json) from this repository.
2. Open your [n8n editor](https://n8n.io) (cloud or self-hosted).
3. Click “Import” in the top menu, then select “From File.”
4. Upload the JSON file and click “Import.”
5. Set up the required credentials:
   - [How to create a Telegram Bot Token](https://core.telegram.org/bots#botfather)
   - [How to get your OpenAI API key](https://help.openai.com/en/articles/4936850-where-do-i-find-my-openai-api-key)
   - [Firecrawl API setup guide (cURL example)](https://docs.firecrawl.dev/reference/quickstart)

**Video Placeholder:** [How to import workflows into n8n](https://www.youtube.com/watch?v=MD4_RgcyCNk)

---

## Option 2: Build the Workflow from Scratch

### Step 1: Telegram Trigger

- **Node:** Telegram Trigger  
- **Purpose:** Listens for new messages from a Telegram bot.  
- **Trigger On:** `["message"]`

---

### Step 2: Identify Message Type (URL vs Follow-up)

- **Node:** If  
- **Condition:**  
  ```javascript
  {{ $json.message.text }} starts with https://
  ```
- **Purpose:** Distinguishes whether the input is an article link or a follow-up question.

---

### Step 3A: Firecrawl Scraping (for URLs)

- **Node:** HTTP Request (rename as `Firecrawl Scraping`)
- **Method:** POST
- **URL:** `https://api.firecrawl.dev/v1/search`
- **Auth Type:** Header Auth
- **Header Key:** `Authorization`
- **Header Value:** `Bearer YOUR_FIRECRAWL_API_KEY`
- **Send Body:** true
- **Body (JSON):**
```json
{
  "query": "{{ $json.message.text }}",
  "limit": 5,
  "scrapeOptions": {
    "format": "markdown"
  }
}
```

**Video Placeholder:** _Using Firecrawl with n8n_

---

### Step 3B: Follow-up Mapping (for non-URLs)

- **Node:** Set (rename as `Follow-up Message`)
- **Mode:** Manual Mapping
- **Field:**
  - **Name:** `Follow up`
  - **Value:** `{{ $json.message.text }}`

---


### Step 4: Summarizing Agent

- **Node:** Tools Agent (rename as `Summarizing Agent`)
- **User Prompt:**
  You are an AI-powered research assistant integrated into a Telegram chatbot. Your purpose is to summarize online articles and help answer follow-ups for the user. Your inputs will be in the form of markdown ({{ $json.data[0].markdown }}) or a follow up message from the user ({{ $json["Follow up"] }}). Answer {{ $json["Follow up"] }} using memory saved.

- **System Prompt:**
  You are an AI-powered research assistant integrated into a Telegram chatbot. Your purpose is to summarize online articles and answer follow-up questions clearly and accurately. Your inputs will be in the form of markdown or in the form of a message sent by the user. You have a simple memory buffer attached for your use. Here are the two cases you will deal with.

  Case 1 (input is markdown: {{ $json.data[0].markdown }}):
  You are given an article in markdown and you will do the following:
  - When handling a new markdown input, call the memory tool to clear all previous memory.
  - This indicates the start of a new article session.
  - Extract the article title and body from the markdown.
  - Summarize the article clearly and neutrally.
  - After generating the article summary, store it in memory under the key "summary".

  Case 2 (input is user follow-up: {{ $json['Follow up'] }}):
  - You are given a regular message (i.e., not a scraped article in markdown). Do the following:
  - Retrieve the value stored under the key "summary" using the memory tool.
  - If found, use the stored summary to guide your response and answer the user's question clearly and factually.
  - If the message goes beyond the article's scope, use the LLM to generate a response and clearly indicate it is not sourced from the article.

  Keep answers simple, within 200 words and preferably in bullet format. For all such messages, respond using a mix of memory + LLM. No need to use any summary formatting.

  Style & Constraints:
  Stay neutral and fact-based.
  Never include metadata like dates, author names, categories, social links, or navigation prompts.
  Use memory + LLM for all follow-up questions.
  Use a clean, conversational tone unless summarizing scraped article content.

---

### Step 5: OpenAI Chat Model

- **Node:** OpenAI Chat Model
- **Model:** `gpt-4o-mini`
- **Credential:** Connect your OpenAI API key

[OpenAI Credential Setup](https://docs.n8n.io/integrations/builtin/credentials/OpenAI/)

---

### Step 6: Add Memory for Follow-ups

- **Node:** Simple Memory
- **Session Key:**
  ```javascript
  {{ $('Telegram Trigger').item.json.message.chat }}
  ```
- **Context Window Length:** 5

---

### Step 7: Send Output Back to Telegram

- **Node:** Telegram (Send Message)
- **Purpose:** Sends the AI-generated response back to the user.
- **Chat ID:**
  ```javascript
  {{ $('Telegram Trigger').item.json.message.chat.id }}
  ```
- **Text:**
  ```javascript
  {{ $json.output }}
  ```

```
Telegram Trigger
   ↓
If Node (check if URL)
   ├── True  → Firecrawl Scraping → Summarizing Agent
   └── False → Follow-up Message  → Summarizing Agent
                           ↓
                Chat Model, Memory
                           ↓
                 Telegram Send Message
```

---

### Required Credentials

| Tool      | Required Info   | Learn More                                                   |
|-----------|------------------|---------------------------------------------------------------|
| Telegram  | Bot Token        | [How to create a Telegram Bot](https://core.telegram.org/bots) |
| OpenAI    | API Key          | [Getting Started with OpenAI](https://platform.openai.com/docs/guides/authentication) |
| Firecrawl | Bearer Token     | [Firecrawl Quickstart Guide](https://docs.firecrawl.dev/reference/quickstart) |


---

## Final Workflow Layout

This is the complete structure of the **Lillibot** Telegram summarization bot.

**Workflow Steps:**

1. **Telegram Trigger**  
   - Listens for incoming messages from the Telegram bot.

2. **If | URL or Follow-up**  
   - Branches based on whether the message is a link or not using:
     ```javascript
     {{ $json.message.text.startsWith("https://") }}
     ```

3. **Firecrawl Scraping (for URLs)**  
   - Sends a POST request to Firecrawl to scrape the article in markdown format.

4. **Follow-up Message (for non-URLs)**  
   - Maps user message as a "Follow up" input.

5. **Summarizing Agent (Tools Agent)**  
   - Takes either scraped markdown or user question and generates a response.
   - Connected to:
     - **OpenAI Chat Model (gpt-4o-mini)**
     - **Simple Memory**

6. **Simple Memory Node Configuration**
   - **Session Key:**  
     ```javascript
     {{ $('Telegram Trigger').item.json.message.chat }}
     ```
   - **Context Window Length:** `5`
   - This maintains memory across a few messages to enable contextual follow-ups.

7. **Output to User**  
   - Sends the final summarized or follow-up response back to Telegram.

---






