 Inbound Lead Qualification Agent

**Built on:** n8n
**Type:** Automation / AI Agent Workflow
**Author:** Big Boy Recruits (A recruitment firm)

---

## Table of Contents
1. [Purpose](#purpose)
2. [Value](#value)
3. [How It Works (Design)](#how-it-works-design)
4. [Agent Prompt](#agent-prompt)
5. [Workflow 1: Lead Qualification & Form Submission](#workflow-1-lead-qualification--form-submission)
6. [Workflow 2: Qualified Lead Classification & Notification](#workflow-2-qualified-lead-classification--notification)
7. [Tool Description & Payload Schema](#tool-description--payload-schema)
8. [Email Templates](#email-templates)
9. [Usage Flow](#usage-flow)
10. [Setup Instructions](#setup-instructions)
11. [Downloads](#downloads)

---

## Purpose

Companies that market themselves well eventually face a common problem: too many inbound leads, many of which aren't a good fit ("qualified") for what they actually sell. For Big Boy Recruits, that means filtering out companies that are too small, outside their target industries, or otherwise not a match for their recruitment services.

The process of researching a new lead and deciding whether to take a call is known as **qualification**. This agent automates that entire research-and-decision process, so no lead is manually screened by a human before the sales team gets involved.

## Value

Instead of paying a human to manually qualify every inbound lead, or relying on rigid rule-based filters that risk cutting out potentially valuable leads, this automation:

- Immediately and consistently qualifies leads using AI judgment (not brittle if/then rules)
- Researches each company automatically before making a decision
- Triggers the appropriate next step (sales handoff or a same-day rejection) without any manual review
- Frees up the sales team to focus only on leads worth their time

## How It Works (Design)

The system is split into **two connected n8n workflows**:

### Workflow 1 — Intake, Research & Qualification
1. **On Form Submission** — triggers when a lead fills out the inbound form (name, company website, message/request).
2. **HTTP Request → Relevance AI** — sends the submitted company website to a Relevance AI tool that scrapes and researches the company.
3. **AI Agent (OpenAI Chat Model + Memory)** — receives the form data *and* the company research, then evaluates the lead against Big Boy Recruits' qualification criteria using the Agent Prompt (below).
4. **Tool Call: "Call n8n Workflow Tool"** — if the AI Agent determines the lead is qualified, it invokes Workflow 2 and passes along a structured JSON payload (name, email, message, and company information).
5. **Email (Gmail)** — if the lead is *not* qualified, the agent immediately responds to the lead directly, letting them know Big Boy Recruits isn't the right fit and offering to connect them with a partner instead.

### Workflow 2 — Lead Classification & Routing
1. **Executed by Workflow 1** — receives the qualified lead's data as input.
2. **AI Agent (OpenAI)** — classifies the lead into one of two categories: **SaaS** or **Development Agency**, based on the company information provided.
3. **If Node** — branches based on the classification output.
4. **Agency Lead Notification** vs **SaaS Lead Notification** — sends a Gmail notification to the appropriate internal recruiter/team, routing the lead to the rep who specializes in that category.

---

## Agent Prompt

This is the system prompt used by the Workflow 1 AI Agent to determine qualification:

```
You are an inbound lead qualification agent. Your job is to
analyze the form submission and company research provided
and decide whether or not they are qualified to work with
Big Boy Recruits, a Dallas based recruitment firm.

Big Boy Recruits specializes in IT and tech talent
placements. We are specialists in capturing talent post
liquidation and can therefore provide talent to clients
as well as the market.

We only work with Software based businesses, e.g. SaaS
or development agencies. These companies are willing to
pay much more developers than your average marketing
company or local business, therefore we only work with
them.

Your job is to determine if the lead is provided with a
good fit for you, and if so call the 'lead_is_qualified'
tool and send the lead information to it. If the lead is
not qualified, you must trigger the send email tool to
let them know that we are unable to help them.

Here is the lead information for you to analyze:
Name: {{ $('On form submission').item.json['What is your
name?'] }}
Company Website: {{ $('On form submission').item.json['What
is your company website?'] }}
Message/Request: {{ $('On form submission').item.json['What
can we help you with?'] }}
Company Research (scraped from their website):
{{ $json.output.company_information_answer }}
```

> **Note:** This prompt tells the agent exactly *who* the business is, *who* it wants as customers, and *what data* it has available to make the call — this specificity is what makes the qualification reliable.

---

## Workflow 1: Lead Qualification & Form Submission

| Step | Node | Function |
|---|---|---|
| 1 | On Form Submission | Captures name, company website, and request/message from the inbound form |
| 2 | HTTP Request (Relevance AI) | Sends the company URL to Relevance AI to scrape/research the company |
| 3 | AI Agent (OpenAI Chat Model + Memory) | Reasons over form data + research to decide qualification |
| 4a | Tool: Call n8n Workflow Tool | If qualified → triggers Workflow 2 with structured lead data |
| 4b | Send Email | If not qualified → auto-replies to the lead directly |

## Workflow 2: Qualified Lead Classification & Notification

| Step | Node | Function |
|---|---|---|
| 1 | Executed by Another Workflow (trigger) | Receives the qualified lead payload from Workflow 1 |
| 2 | AI Agent (OpenAI) | Classifies lead as **SaaS** or **Development Agency** |
| 3 | If Node | Routes based on classification |
| 4a | Agency Lead Notification (Gmail) | Notifies the agency-focused rep |
| 4b | SaaS Lead Notification (Gmail) | Notifies the SaaS-focused rep |

---

## Tool Description & Payload Schema

When the lead is qualified in Workflow 1, the following tool description and JSON payload structure are used to hand off to Workflow 2:

**Tool Description:**
> If the lead is qualified to work with Big Boy Recruits, e.g. they are a software based business like SaaS or development agencies, then trigger this tool and send the lead data in the following format (the data in this is dummy and you should replace it with the correct data):

**Payload Schema:**
```json
{
  "name": "string — lead's full name",
  "email": "string — lead's email address",
  "message": "string — what the lead is asking for help with",
  "qualified": true,
  "company_information": "string — summary of scraped company research, e.g. 'this company is Miranda...a bit of info here'"
}
```

**Important build note from the design:** the Relevance AI tool node needs to be reconfigured before reuse — clone it and update the request body to match the **v2 endpoint format**, not v1. Sending the wrong payload shape to the wrong endpoint is a common cause of empty output (see the endpoint URL differences below).

---

## Email Templates

### 1. Rejection / Non-Qualified Lead Response
Sent automatically by Workflow 1 when a lead does *not* meet the qualification bar:

```
Hi {{ $('On form submission').item.json['What is your name?'] }},

Thanks for your interest in Big Boy Recruitment services!
As we specialize in recruitment for software businesses
such as SaaS and development agencies, we're not a good
fit for you based on your company's industry.

Please let me know if you'd like to connect with one of
our partners who specializes in dealing with your needs.

Cheers,

Huge Jackman
Head of Sales, Big Boy Recruits (BBR)
Dallas, TX
```

### 2. Internal Classification Notification (Workflow 2)
Sent to the internal team once a lead is routed into SaaS or Agency:

```
We have a new inbound lead and we need to classify it
into either SaaS or development agency category.

Here is the lead information:
Name: {{ $json['Lead Name'] }}
Request: {{ $json.Message }}
Company Information: {{ $json['Company Information'] }}

If the lead is a SaaS company, output 'SaaS'
If the lead is a development agency, output 'Agency'
```

---

## Usage Flow

1. **Lead submits the form** with their name, company website, and request.
2. **Relevance AI scrapes** the company's website to gather research/context.
3. **The AI Agent evaluates** the lead against the qualification criteria in the system prompt.
4. **If qualified:** the agent calls the second workflow, which classifies the lead as SaaS or Development Agency and sends an email notification to the correct internal rep.
5. **If not qualified:** the agent immediately emails the lead back, letting them know Big Boy Recruits isn't the right fit and offering a referral to a partner instead.

---

## Setup Instructions

1. **Form node:** Set up an n8n Form Trigger with three fields — Name, Company Website, and Message/Request.
2. **Relevance AI tool:** Duplicate the existing Relevance AI studio/tool and update the trigger endpoint to the **v2 webhook URL** (the one that accepts `{"company_url": ""}` as the body, with `project` passed as a query parameter — *not* the `trigger_limited` v1 endpoint, which expects a nested `params` object).
3. **AI Agent node:** Paste in the Agent Prompt above, connect an OpenAI Chat Model, and attach a Memory node so context persists across the interaction.
4. **Sub-workflow tool:** Add the "Call n8n Workflow Tool" and point it at Workflow 2, using the JSON payload schema above.
5. **Email nodes:** Connect Gmail (or your email provider of choice) for both the rejection email and the internal classification notifications, and paste in the templates above.
6. **Workflow 2:** Set the trigger to "Executed by Another Workflow," add the classification AI Agent with the SaaS/Agency prompt, and wire the If node to the two notification branches.
7. **Test end-to-end** by submitting a real form entry and confirming the correct email fires — either the rejection email or the internal routing notification.

---

## Downloads

- `Lead_Qualification_Agent_Form_Submissions.json` — Workflow 1 automation template
- `Qualified_Lead_Classifier_AND_Notifier.json` — Workflow 2 automation template

*(Import both JSON files into n8n via Workflows → Import from File)*
