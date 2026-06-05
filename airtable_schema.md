# Airtable Schema — Influencer Outreach Pipeline

This schema powers an automated influencer outreach system for a D2C brand. It manages the full lifecycle of an influencer contact: from raw lead discovery through AI-powered message generation, human review, email delivery, and response tracking. Every field exists because a workflow reads it, writes it, or a formula depends on it.

---

## Table: Leads

| Field | Type | Written by | Read by | Description |
|---|---|---|---|---|
| `Username` | Text | Apify import | LLM prompt | Instagram handle. Primary field. No @ prefix. |
| `Profile URL` | URL | Apify import | Manual review | Direct link to the influencer's Instagram profile |
| `Full name` | Text | Apify import | LLM prompt | Used for personalization in the AI-generated message |
| `Bio` | Long text | Apify import | LLM prompt | Instagram bio — the primary context that Claude uses to write a personalized outreach message |
| `Followers` | Number | Apify import | Qualification formula | Follower count at time of scraping |
| `Business email` | Text | Apify import | Qualification formula | Populated by Apify as `TRUE` or `FALSE`. Indicates whether a business email was detected on the profile. Acts as the gate in the qualification formula — only leads with a confirmed email enter the pipeline. |
| `Official Business Email` | Email | Apify import | Sending workflow | The actual email address used for outreach delivery |
| `Niche` | Single select | Manual / future LLM | Qualification formula | Options: Beauty, Lifestyle, Fashion, Fragrance, Skincare, Wellness, Other |
| `Qualified` | Formula | Auto-calculated | View filter | Returns `✓` or `✗` based on follower range and email availability. See formula below. |
| `Status` | Single select | Workflows + manual | Triggers + views | Current pipeline stage. Drives all automation triggers. See status flow below. |
| `Generated Message` | Long text | LLM workflow | Sending workflow | The AI-generated outreach email. This field is the handoff point between the two automations. |
| `Date Added` | Created time | Auto | Reporting | When the record first entered the base |
| `Date Sent` | Date | Sending workflow | Reply tracking formula | Timestamp set automatically when the outreach email is dispatched |
| `Reply Received` | Checkbox | Manual | Reply tracking formula | Checked manually when the influencer responds |
| `Date Replied` | Date | Manual | Reply tracking formula | Date the reply was received |
| `Days to reply` | Formula | Auto-calculated | Reporting | Number of days between send and reply. Returns blank until a reply is logged. |
| `Outcome` | Single select | Manual | Reporting | Final result: Response, Declined, Interested, Collaborated, Lost |

---

## Design Decisions

**Why two email fields?**
`Business email` and `Official Business Email` serve different purposes. The first is a detection flag: Apify returns `TRUE` or `FALSE` indicating whether a business email exists on the profile. The second is the actual address. Separating them means the qualification formula can gate on email availability without depending on the email field being non-empty — which would break if the scraper returned the address in a different field or format. The detection flag is stable; the address field may vary.

**Why 1,000–80,000 followers?**
Below 1,000 followers, the influencer likely lacks the reach to justify outreach effort. Above 80,000, they typically have management or are expensive to work with, making a gifting-only proposition less viable. This range targets nano and micro-influencers where brand gifting has the highest conversion rate relative to cost.

**Why is `Days to reply` blank instead of zero?**
The formula returns `BLANK()` for contacts who haven't replied yet. A zero value would skew pipeline reporting and create false signals in average response time calculations. Blank values are excluded from Airtable aggregations by default — preserving metric accuracy.

---

## Qualification Formula

```
IF(
  AND(
    {Followers} >= 1000,
    {Followers} <= 80000,
    {Business email} = "TRUE"
  ),
  "✓",
  "✗"
)
```

A contact passes qualification when both conditions are true: follower count is within the target range, and a business email was confirmed on their profile. Records that fail either condition are marked `✗` and excluded from the automated pipeline.

---

## Days to Reply Formula

```
IF({Reply Received}, DATETIME_DIFF({Date Replied}, {Date Sent}, 'days'), BLANK())
```

Only calculates after `Reply Received` is manually checked. Returns the integer number of days between outreach and response.

---

## Status Pipeline

```
New → Qualified → Ready to Send → Sent → Replied → Converted / Rejected / Skip
```

| Status | Trigger | What happens next |
|---|---|---|
| `New` | Record imported from Apify | Appears in **Inbox** view for manual niche tagging |
| `Qualified` | Passed qualification formula + manual status change | Enters **Queue** view → LLM workflow generates a personalized message |
| `Ready to Send` | LLM workflow writes Generated Message and updates status | Appears in **Review** view for human approval → enters **Ready to Send** view |
| `Sent` | Sending workflow dispatches email and timestamps record | Awaiting influencer response |
| `Replied` | Manual status change when reply is received | Check `Reply Received`, log `Date Replied`, set `Outcome` |
| `Converted` | Collaboration confirmed | Terminal state — success |
| `Rejected` | Declined or no response after follow-up window | Terminal state — closed |
| `Skip` | Manually excluded at any point | Terminal state — removed from pipeline |

---

## Views

| View | Filter | Purpose | Connected workflow |
|---|---|---|---|
| **Inbox** | `Status` = `New` | Newly scraped leads awaiting manual review and niche classification | None — manual processing |
| **Queue** | `Qualified` = `✓` AND `Status` = `Qualified` | Qualified leads waiting for AI message generation | `workflow_enrichment.json` |
| **Review** | `Status` = `Ready to Send` | Generated messages awaiting human approval before send | None — manual review gate |
| **Ready to Send** | `Official Business Email` is not empty AND `Status` = `Ready to Send` | Approved messages with a valid delivery address | `workflow_sending.json` |

---

## Data Flow

```
Apify scrape
  → Airtable import (Status: New)
    → Manual niche tagging (Status: Qualified)
      → [workflow_enrichment.json] Claude generates message (Status: Ready to Send)
        → Human reviews message in Review view
          → [workflow_sending.json] Gmail sends email (Status: Sent)
            → Manual reply tracking (Status: Replied → Outcome logged)
```

Two automation workflows handle the machine steps. Two manual gates (niche classification and message review) keep a human in the loop at the points where judgment matters most.
