Good — this is the right moment to fix it properly. You’re not far off, you just need to shift responsibility upstream (generation stage) instead of trying to “fix” things downstream (rendering stage).

Let’s design this cleanly, without breaking your 1600 existing articles.

⸻

🚀 WHAT YOU SHOULD DO (CLEAR PLAN)

🔁 High-level strategy

Layer	Current	New
ChatGPT Output	❌ raw text blob	✅ structured JSON (blocks)
Storage	❌ content string	✅ blocks[] + fallback content
Frontend	❌ parser-heavy	✅ simple renderer
Legacy Articles	❌ fragile parsing	✅ fallback mode


⸻

✅ STEP 1 — CHANGE CHATGPT OUTPUT (MOST IMPORTANT)

You don’t need to rewrite everything, just change your prompt slightly.

⸻

🔥 NEW SYSTEM PROMPT (CRITICAL)

Replace your current prompt with this:

const systemInstruction = `
Using the following scraped Formula 1 article text, generate a structured article in JSON format.

Return ONLY valid JSON. Do not include explanations.

Structure the output as:

{
  "title": "...",
  "blocks": [
    { "type": "paragraph", "text": "..." },
    { "type": "heading", "text": "..." },
    { "type": "quote", "text": "..." },
    { "type": "table", "season": 2026, "round": 3, "session": "fp2" }
  ]
}

Rules:

- Preserve all factual and technical accuracy.
- DO NOT invent quotes.
- Only include quotes that exist in the source.
- Quotes must be EXACT matches from input.
- NEVER mix narration inside quote blocks.
- Paragraphs must NOT contain quotation marks unless they are full quotes.
- Headings must be clean (no colon, no formatting).
- Keep introduction and conclusion as normal paragraphs (no special type needed).
- If article contains session results (FP1/FP2/etc), insert a "table" block instead of listing results manually.
- Do NOT output markdown or HTML — JSON only.
`;


⸻

💥 THIS FIXES YOUR BIGGEST PROBLEMS

❌ Today
	•	Quote detection = broken
	•	Partial matching = broken
	•	Green block = inconsistent
	•	Tables = hacky embed

✅ After this
	•	Quotes are explicit → no parsing
	•	Tables are explicit → no regex
	•	Structure is explicit → no guessing

⸻

✅ STEP 2 — FRONTEND (SUPER SIMPLE)

Replace your giant parser with:

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

👉 That’s it. No loops. No madness.

⸻

✅ STEP 3 — KEEP LEGACY SUPPORT (IMPORTANT)

You already have 1600 articles.

So:

if (article.blocks) {
  // NEW SYSTEM
  html = renderArticle(article.blocks);
} else {
  // OLD SYSTEM (your current parser)
  html = renderParagraphHTML(article.content, article.quote_blocks);
}

👉 Zero risk migration.

⸻

✅ STEP 4 — OPTIONAL (AUTO-MIGRATION LATER)

Later, you can run a script:
	•	Convert old content → blocks
	•	Store back into DB
	•	Slowly kill old parser

But don’t do it now.

⸻

🎯 ABOUT QUOTES (IMPORTANT DECISION)

You asked a very real question:

Should we even keep these green blocks?

My honest answer:

❌ Current system
	•	Looks inconsistent
	•	Breaks often
	•	Feels “generated”, not editorial

⸻

✅ Best approach

Use clean editorial quote style

blockquote {
  border-left: 4px solid #339967; /* your green */
  padding-left: 12px;
  margin: 20px 0;
  font-style: italic;
  color: #222;
}

👉 Keep green, but remove background.

⸻

🔥 KEY RULE (VERY IMPORTANT)

A quote block must ALWAYS be:
	•	Exact quote
	•	Nothing else
	•	No narration mixed

⸻

❌ WHY YOUR CURRENT SYSTEM BREAKS

Example:

“my understeer is unreal”. Team-mate Isack Hadjar also struggled...

Your parser:
	•	sees quote start
	•	grabs everything
	•	fails to split

👉 This problem disappears completely with structured blocks.

⸻

🧠 FINAL ARCHITECTURE (CLEAN)

Backend (ChatGPT output)

{
  "title": "...",
  "blocks": [...]
}

Frontend
	•	Simple renderer
	•	No parsing
	•	No regex

DataStore
	•	Already compatible (tables fetch same way)

⸻

⚠️ MOST IMPORTANT SHIFT

Right now you are doing:

“Let me fix broken text after generation”

You need to move to:

“Let me generate correct structure from the start”

⸻

🚀 WHAT I RECOMMEND YOU DO NEXT
	1.	✅ Update ChatGPT prompt (biggest win)
	2.	✅ Add blocks field to DB
	3.	✅ Add new renderer (simple switch-case)
	4.	✅ Keep old parser as fallback
	5.	❌ Stop improving current parser (dead end)

⸻

💬 FINAL STRAIGHT TALK

You built a very advanced system already.

But:

👉 You hit the ceiling of “text parsing architecture”

To scale:

👉 You MUST move to “structured content architecture”

⸻

If you want, next step I can:

✅ Convert one of your real articles → blocks format
✅ Show exactly how ChatGPT output should look
✅ Help you plug it into your Lambda pipeline

This is a one-time shift — after this, your whole platform becomes clean, predictable, and scalable.