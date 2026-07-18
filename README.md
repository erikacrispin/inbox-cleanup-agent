
/**
 * INBOX CLEANUP AGENT
 * Runs automatically (set up a twice-daily trigger — see instructions).
 * Sorts inbox clutter into AI-Cleanup labels and archives it.
 *
 * SAFETY RULES (never violated):
 *  - Never deletes anything. Label + archive only.
 *  - Never touches starred emails.
 *  - Never touches emails from real people (only bulk/automated senders).
 *  - Anything uncertain stays in the inbox.
 */

// ---------- configuration ----------
const LABELS = {
  promo: "AI-Cleanup/Promo",
  newsletter: "AI-Cleanup/Newsletter",
  notification: "AI-Cleanup/Notification",
};

// Senders that must NEVER be archived, even if they look automated.
// Add addresses or domains here (lowercase).
const PROTECTED_SENDERS = [
  "yourbank.com",
  "accounts.google.com",
  "yourworkdomain.com",
  // add your bank, lawyer, doctor, work domain, etc.
];

// Bulk senders you want archived by category. Extend freely.
const RULES = {
  promo: {
    senders: ["example-store.com", "example-gym.com"],
    subjectWords: ["% off", "sale", "deal", "discount", "limited time", "book now", "last chance"],
  },
  newsletter: {
    senders: ["substack.com", "mailchimp.com", "beehiiv.com"],
    subjectWords: [],
  },
  notification: {
    senders: [
      "jobs-noreply@linkedin.com",
      "notifications@",
      "no-reply@",
      "noreply@",
      "donotreply@",
    ],
    subjectWords: [],
  },
};

const MAX_THREADS_PER_RUN = 50;

// ---------- main ----------
function runInboxCleanup() {
  const labels = {};
  for (const [key, name] of Object.entries(LABELS)) {
    labels[key] = GmailApp.getUserLabelByName(name) || GmailApp.createLabel(name);
  }

  const threads = GmailApp.search("in:inbox -is:starred", 0, MAX_THREADS_PER_RUN);
  let archived = 0;

  for (const thread of threads) {
    const msg = thread.getMessages()[thread.getMessageCount() - 1];
    const from = msg.getFrom().toLowerCase();
    const subject = (msg.getSubject() || "").toLowerCase();
    const rawHeaders = msg.getHeader("List-Unsubscribe") || "";

    // Rule 0: protected senders are untouchable
    if (PROTECTED_SENDERS.some((s) => from.includes(s))) continue;

    // Rule 1: replies/threads with your own participation stay
    if (thread.getMessageCount() > 1) continue;

    const category = classify(from, subject, rawHeaders);
    if (!category) continue; // uncertain → leave in inbox

    thread.addLabel(labels[category]);
    thread.moveToArchive();
    archived++;
  }

  console.log(`Inbox cleanup complete: ${archived} thread(s) archived.`);
}

// ---------- classification ----------
function classify(from, subject, listUnsubHeader) {
  // explicit sender rules first
  for (const [category, rule] of Object.entries(RULES)) {
    if (rule.senders.some((s) => from.includes(s))) return category;
  }
  // subject keyword rules
  for (const [category, rule] of Object.entries(RULES)) {
    if (rule.subjectWords.some((w) => subject.includes(w))) return category;
  }
  // generic bulk-mail signal: a List-Unsubscribe header means it's a
  // mailing list or marketing blast, never a personal email
  if (listUnsubHeader) {
    if (subject.match(/newsletter|digest|weekly|daily|edition/)) return "newsletter";
    if (subject.match(/off|sale|deal|save|free|offer/)) return "promo";
    return "notification";
  }
  return null; // not confident → stays in inbox
}

/**
 * SETUP INSTRUCTIONS
 *
 * 1. Go to https://script.google.com and click "New project"
 * 2. Delete the placeholder code, paste this entire file, and save (name it
 *    "Inbox Cleanup Agent")
 * 3. In the toolbar dropdown, select "runInboxCleanup" and click Run once.
 *    Google will ask you to authorize the script's Gmail access — approve it.
 *    (This authorization is between you and Google; nothing leaves your account.)
 * 4. Check the execution log — it will report how many threads it archived.
 * 5. Set the twice-daily schedule: click the clock icon ("Triggers") in the
 *    left sidebar → "+ Add Trigger" →
 *       function: runInboxCleanup
 *       event source: Time-driven
 *       type: Hour timer
 *       interval: Every 12 hours
 *    → Save.
 *
 * That's it. It now runs itself twice a day, forever, for free.
 *
 * TUNING OVER TIME
 *  - When a new bulk sender floods you: add its domain to RULES.
 *  - When something gets archived that shouldn't be: add the sender to
 *    PROTECTED_SENDERS and drag the email back to your inbox.
 *  - To pause the agent: Triggers → delete the trigger. To resume, re-add it.
 */

MIT License

Copyright (c) 2026 Erika Crispin

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

# Inbox Cleanup Agent

A zero-cost, self-hosted Gmail agent that runs twice a day and keeps your inbox clean — without deleting a single email.

Built with Google Apps Script. No servers, no API keys, no dependencies.

## What it does

Every 12 hours, the agent scans your inbox and sorts bulk mail into color-coded labels, then archives it out of your inbox:

| Label | Catches |
|---|---|
| `AI-Cleanup/Promo` | Sales, marketing, offers |
| `AI-Cleanup/Newsletter` | Digests, subscriptions, roundups |
| `AI-Cleanup/Notification` | Automated alerts, job alerts, event blasts |

Everything stays searchable and recoverable — archived, never deleted.

## Safety rules

The agent is deliberately conservative:

- **Never deletes.** Label + archive only.
- **Never touches starred emails** or threads you've replied to.
- **Never touches protected senders** (banks, security alerts, appointments — configurable).
- **When uncertain, it does nothing.** Ambiguous emails stay in your inbox.

Bulk mail is detected through explicit sender rules, subject keywords, and the
`List-Unsubscribe` header — a signal that an email is a mailing-list blast
rather than a personal message.

## Setup (~3 minutes)

1. Go to [script.google.com](https://script.google.com) → **New project**
2. Replace the placeholder with the contents of `Code.gs` and save
3. Select `runInboxCleanup` in the toolbar dropdown → **Run** once → approve the
   Gmail authorization (Google will warn the app is unverified — that's expected
   for personal scripts; click *Advanced → Go to project → Allow*)
4. Open **Triggers** (clock icon) → **Add Trigger** →
   `runInboxCleanup` · Time-driven · Hour timer · **Every 12 hours** → Save

Done. It now runs itself twice a day.

## Tuning

All configuration lives at the top of `Code.gs`:

- `PROTECTED_SENDERS` — domains that must never be archived
- `RULES` — sender domains and subject keywords per category
- `MAX_THREADS_PER_RUN` — batch size per run

When something gets archived that shouldn't be: drag it back to your inbox and
add its sender to `PROTECTED_SENDERS`. When a new bulk sender floods you: add
its domain to `RULES`.

## License

MIT
