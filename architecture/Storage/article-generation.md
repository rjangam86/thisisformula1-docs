Here’s your complete, production-ready architecture MD file — clean, structured, and aligned with your existing system, including your strict editorial rules and single-line prompt requirement.

⸻

:::writing{variant=“standard” id=“48291”}

🏎️ HotLapHQ Article Rendering Architecture (v2)

🎯 Objective

Build a scalable, deterministic, and clean article system that:
	•	Produces high-quality editorial articles (not AI-looking)
	•	Eliminates fragile parsing logic
	•	Supports dynamic embeds (tables, sessions, etc.)
	•	Maintains backward compatibility with existing 1600+ articles
	•	Uses structured content blocks instead of raw text parsing

⸻

🧠 Core Principle

❌ Never parse content at render time
✅ Always generate structured content at creation time

⸻

🏗️ SYSTEM ARCHITECTURE OVERVIEW

Scraped Data
     ↓
ChatGPT (Structured Output)
     ↓
Database (blocks[])
     ↓
Frontend Renderer (simple switch-case)
     ↓
Final Article UI


⸻

✍️ CHATGPT GENERATION (CRITICAL)

✅ SYSTEM PROMPT (SINGLE LINE — FINAL)

Using the following scraped Formula 1 article text, write a complete, professional editorial article suitable for publication on a motorsport media website while preserving all factual and technical accuracy; generate a strong original headline conveying the same meaning as the source title but not copying it; structure the article with a clear introduction, logically flowing sections, and a concise conclusion; maintain fluent journalistic English with natural readability and coherence so the article reads as a unified narrative, not disjointed blocks; avoid redundancy and meta references such as “the article states”; include only quotes that exist in the source text exactly as written using proper quotation marks (“ and ”) and do not paraphrase or fabricate quotes; never mix narration inside quotes and never include partial quotes; ensure paragraphs remain clean and readable; then convert the final article into structured JSON format with "title" and "blocks" where blocks is an ordered array using only these types: paragraph, heading, quote, table; use paragraph blocks for all normal content including introduction and conclusion, heading blocks for section titles, quote blocks only for exact standalone quotes, and table blocks when session results like FP1, FP2, FP3, Sprint, Qualifying or Race are present using fields season, round and session instead of writing raw results; ensure all blocks flow logically as a continuous article and not as independent fragments; output strictly valid JSON only with no markdown or explanation.


⸻

📦 OUTPUT STRUCTURE (FROM CHATGPT)

✅ Example

{
  "title": "Piastri Leads McLaren Charge in Suzuka FP2 Statement",
  "blocks": [
    {
      "type": "paragraph",
      "text": "Oscar Piastri delivered a standout performance in second practice at Suzuka..."
    },
    {
      "type": "heading",
      "text": "Session narrative"
    },
    {
      "type": "paragraph",
      "text": "Mercedes had dominated FP1, but the balance shifted in FP2..."
    },
    {
      "type": "quote",
      "text": "My understeer is unreal. We need to address this before qualifying."
    },
    {
      "type": "paragraph",
      "text": "Team-mate Isack Hadjar also struggled during the session..."
    },
    {
      "type": "table",
      "season": 2026,
      "round": 3,
      "session": "fp2"
    },
    {
      "type": "paragraph",
      "text": "The session confirmed McLaren’s growing competitiveness..."
    }
  ]
}


⸻

🗄️ DATABASE STRUCTURE

✅ Store BOTH (for migration safety)

{
  "title": "...",
  "content": "...",  // legacy
  "blocks": [...],   // new system
  "quote_blocks": [...]
}


⸻

🖥️ FRONTEND RENDERING

✅ MAIN RENDERER

function renderArticle(blocks) {

  const output = [];

  blocks.forEach(block => {

    switch (block.type) {

      case "paragraph":
        output.push(`<p>${escapeHTML(block.text)}</p>`);
        break;

      case "heading":
        output.push(`<h3>${escapeHTML(block.text)}</h3>`);
        break;

      case "quote":
        output.push(`<blockquote>${escapeHTML(block.text)}</blockquote>`);
        break;

      case "table":
        output.push(`
          <div class="embedded-table"
               data-season="${block.season}"
               data-round="${block.round}"
               data-session="${block.session}">
            <div class="table-loader">Loading results...</div>
          </div>
        `);
        break;
    }

  });

  return output.join("\n");
}


⸻

🔁 BACKWARD COMPATIBILITY (CRITICAL)

let html;

if (article.blocks && article.blocks.length) {
  html = renderArticle(article.blocks);
} else {
  html = renderParagraphHTML(article.content, article.quote_blocks);
}

$("#single-post-content").html(html);
initEmbeddedTables();


⸻

📊 EMBEDDED TABLE SYSTEM (UNCHANGED)

Your existing system remains valid:

function initEmbeddedTables() {
  $(".embedded-table").each(function () {

    const $container = $(this);

    const round = Number($container.data("round"));
    const session = $container.data("session");

    DataStore.getRoundResults(round, function (res, err) {

      if (err || !res) {
        $container.html("<p>Failed to load results</p>");
        return;
      }

      const data = res.results;
      const items = data[session];

      const $table = $("<table></table>");
      $table.addClass("standings-table results-table");

      if (session.includes("qualifying")) {
        renderQualifyingTable($table, items);
      } else if (session.startsWith("fp")) {
        renderPracticeTable($table, items);
      } else {
        renderRaceTable($table, items);
      }

      const title = `
        <div class="embedded-table-title">
          <strong>${formatSessionName(session)} Results</strong>
        </div>
      `;

      $container.html(title).append($table);
    });

  });
}


⸻

🎨 QUOTE STYLING (FINAL)

blockquote {
  border-left: 4px solid #339967; /* brand green */
  padding-left: 12px;
  margin: 20px 0;
  font-style: italic;
  color: #222;
  background: transparent;
}


⸻

🚫 RULES ENFORCED BY SYSTEM

Quotes
	•	Must be exact matches
	•	Never partial
	•	Never mixed with narration

Paragraphs
	•	Clean, readable, editorial tone
	•	No broken quote fragments

Headings
	•	Clear section labels
	•	No formatting noise

Tables
	•	No manual results text
	•	Always structured block

⸻

🔄 MIGRATION STRATEGY

Phase 1 (Immediate)
	•	Start generating blocks[] for new articles
	•	Keep old parser

Phase 2
	•	Gradually migrate high-traffic articles

Phase 3
	•	Remove old parser entirely

⸻

⚠️ WHAT TO STOP DOING
	•	❌ Regex-based quote detection
	•	❌ Partial quote matching
	•	❌ Parsing raw text into structure
	•	❌ Mixing HTML inside content strings

⸻

✅ FINAL RESULT

✔ Clean architecture
✔ Predictable rendering
✔ No broken quotes
✔ No layout inconsistencies
✔ Fully scalable

⸻

🧠 FINAL PRINCIPLE

Generate structure → Store structure → Render structure
Never guess structure from text again

⸻

:::

⸻

If you want next, I can:
	•	Convert one of your real articles into this blocks format automatically
	•	Or give you a Lambda-side transformer to migrate old content progressively