# n8n Quora Question AI Comment Generator

An automated n8n workflow that searches for software testing questions on Quora via Google RSS, generates AI-powered answers using Ollama (LLaMA 3.2), and saves the results to Google Sheets.

---

## What It Does

1. Searches Google RSS feed for software testing questions on Quora
2. Parses the XML feed into JSON
3. Extracts question titles using a JavaScript Code node
4. Limits the number of questions to process
5. Sends each question to a local Ollama instance to generate a relevant answer/comment
6. Parses the Ollama response to extract the title and comment
7. Appends the results as rows in a Google Sheet

---

## Workflow Flow

```
[Manual Trigger]
      ↓
[HTTP Request] — Fetch Google News RSS
      ↓
[XML to JSON] — Parse RSS feed
      ↓
[Code in JavaScript] — Extract article titles
      ↓
[Limit] — Limit number of articles
      ↓
[HTTP Request - Ollama] — Generate AI comment for each title
      ↓
[Code in JavaScript1] — Parse Ollama response
      ↓
[Append row in sheet] — Save to Google Sheets
```

---

## Prerequisites

- [n8n](https://n8n.io/) installed and running locally
- [Ollama](https://ollama.com/) installed and running locally
- LLaMA 3.2 model pulled in Ollama:
  ```bash
  ollama pull llama3.2
  ```
- Google Sheets API credentials configured in n8n
- A Google Sheet with columns: `Title` and `Comment`

---

## Setup Instructions

### 1. Clone or Import the Workflow
Import the workflow JSON file into your n8n instance via **Settings > Import Workflow**.

### 2. Configure Google RSS Search HTTP Request Node
- **Method:** GET
- **URL:** `https://news.google.com/rss/search?q=software+testing+site:quora.com&hl=en-IN&gl=IN&ceid=IN:en`
- This searches Google RSS for Quora questions related to **software testing**
- Change `software+testing` to any topic you want to search questions for

### 3. Configure Ollama HTTP Request Node
- **Method:** POST
- **URL:** `http://127.0.0.1:11434/api/generate`
- **Body (JSON):**
  ```json
  {
    "model": "llama3.2:latest",
    "prompt": "Write a comment for this exact question: '{{ $json['title'] }}'. Rules: 1) Do NOT change or rephrase the title. 2) Return ONLY this exact JSON format: {\"title\": \"the exact question\", \"comment\": \"your answer here\"}. No extra text, no markdown, nothing outside the JSON.",
    "stream": false,
    "format": "json"
  }
  ```

### 4. Configure the Code Node (Parse Ollama Response)
```javascript
const items = $input.all();
const limitItems = $('Limit').all();

const results = items.map((item, index) => {
  const rawResponse = item.json.response;
  const parsed = JSON.parse(rawResponse);

  const title = limitItems[index].json.title;

  let comment;
  if (parsed.comment) {
    comment = parsed.comment;
  } else if (parsed.value) {
    comment = parsed.value;
  } else {
    comment = Object.values(parsed)[0];
  }

  return {
    json: {
      title: title,
      comment: comment
    }
  };
});

return results;
```

### 5. Configure Google Sheets Node
- **Operation:** Append Row
- **Column Mapping:**
  - `Title` → `{{ $json['title'] }}`
  - `Comment` → `{{ $json['comment'] }}`

---

## Google Sheet Output Example

| Title | Comment |
|---|---|
| What is your achievements in your job role as a software tester? | As a software tester, my achievements include identifying and reporting bugs... |
| What are some software testing tools that you use and why? | Some popular software testing tools include TestRail, PractiTest, and Zephyr... |

---

## Notes

- Make sure Ollama is running before executing the workflow: `ollama serve`
- The RSS URL searches Google for **Quora questions** on your topic using `site:quora.com`
- The `format: "json"` parameter in the Ollama request forces consistent JSON output
- The Code node handles multiple Ollama response formats as a fallback
- Adjust the **Limit** node to control how many questions are processed per run
- The Quora question title is always taken from your input, not from Ollama, to avoid Ollama rephrasing it

---

## Tech Stack

- [n8n](https://n8n.io/) — Workflow automation - Local n8n
- [Ollama](https://ollama.com/) — Local AI model runner
- [LLaMA 3.2](https://ollama.com/library/llama3.2) — AI language model
- [Google RSS Search](https://news.google.com/rss) — Quora question source
- [Quora](https://www.quora.com/) — Question source platform
- [Google Sheets](https://sheets.google.com/) — Output storage
