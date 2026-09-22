# Blood Donor Approval System

Rubot প্রজেক্টের **Verification & Submission Engine (Pillar 1)** এর আওতায় বানানো প্রথম automation। উদ্দেশ্য: RU/IER students-এর মধ্যে blood donor data collect করা, human-verified রাখা, যাতে ভবিষ্যতে guardian প্রয়োজনমতো search করে donor খুঁজে পায়।

## Core Pattern

```
Form Submission → Rule-based Check → Human Approval (Telegram) → Database Update (Supabase)
```

## Data Flow

```
[Google Form: Student Information Form]
        │  (Name, Department, Blood Group, Phone, Consent)
        ▼
[Google Sheet: Form Responses]
        │  (n8n প্রতি ১ মিনিটে নতুন row চেক করে)
        ▼
[n8n Trigger: New Form Response (Google Sheets Trigger)]
        ▼
[IF Node: Validate Submission]
   - Department == "IER" ?
   - Name/Phone/BloodGroup খালি না?
   - Phone: ১১ ডিজিট, "01" দিয়ে শুরু?
        │
   ┌────┴────┐
  Pass       Fail
   │           │
   ▼           ▼
[Supabase   [No Operation
 Insert]     - থেমে যায়]
 status=
 "pending"
   │
   ▼
[Telegram: Notify Admin]
   - নাম, Dept, Blood Group, Phone সহ মেসেজ
   - Inline button: ✅ Approve / ❌ Reject
   - callback_data = "approve_<id>" / "reject_<id>"
   │
   ▼
[Telegram Trigger: Callback Query শোনে]
        ▼
[Code Node: Parse Callback Data]
   - callback_data থেকে action + id আলাদা করে
        ▼
[IF Node: action == approve/reject?]
   │
   ┌────┴────┐
Approve    Reject
   │           │
   ▼           ▼
[Supabase   [Supabase
 Update:     Update:
 status=     status=
 approved]   rejected]
   │           │
   └─────┬─────┘
         ▼
[Telegram: Edit Message]
   - "✅ Approved হয়েছে" / "❌ Rejected হয়েছে" দেখায়
```

## Supabase Table: `blood_donors`

```sql
create table blood_donors (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  department text not null,
  blood_group text not null,
  phone text not null,
  consent boolean not null default false,
  status text not null default 'pending', -- pending / approved / rejected
  telegram_message_id text,
  submitted_at timestamptz default now(),
  decided_at timestamptz
);

alter table blood_donors disable row level security;
```

## Google Form Fields

| Field | Type |
|---|---|
| নাম | Short answer |
| Department | Dropdown (এখন শুধু: IER) |
| Blood Group | Dropdown (A+, A-, B+, B-, O+, O-, AB+, AB-) |
| Phone Number | Short answer |
| Consent | Checkbox — "আমি সম্মত যে আমার তথ্য blood প্রয়োজনে অন্যদের দেখানো হবে" |

## Credentials লাগবে

- **Google Sheets Trigger** (OAuth2) — Form response sheet পড়ার জন্য
- **Supabase API** — Host (`https://<project-id>.supabase.co` ফরম্যাটে পূর্ণ URL) + Secret Key (`service_role` key, Project Settings → API Keys থেকে)
- **Telegram API** — Bot token (@BotFather দিয়ে বানানো bot থেকে)

## Setup করার সিকোয়েন্স (নতুন environment-এ আবার বসাতে হলে)

1. Supabase-এ উপরের SQL চালিয়ে table বানাও, RLS disable করো
2. Google Form বানাও (উপরের field অনুযায়ী), Responses ট্যাব থেকে Sheets আইকনে ক্লিক করে Response Sheet লিংক করো
3. Telegram-এ @BotFather দিয়ে bot বানাও, token সংগ্রহ করো
4. নিজের/approval দেওয়ার লোকের Chat ID বের করো (@get_id_bot দিয়ে, বা group হলে group-এ মেসেজ পাঠিয়ে `https://api.telegram.org/bot<token>/getUpdates` চেক করে — group ID negative number হবে)
5. n8n-এ Credential Manager-এ তিনটা credential (Google Sheets, Supabase, Telegram) যোগ করো
6. n8n-এ নতুন workflow বানাও, AI assistant-কে নিচের prompt দাও
7. প্রতিটা node-এ credential assign করো; callback_data → id parsing (Code node) ঠিক আছে কিনা manually verify করো
8. Workflow Publish + Active করো
9. Test submission দিয়ে end-to-end verify করো

## n8n AI Assistant Prompt

```
আমি একটা n8n workflow বানাতে চাই ৫টা ধাপে, blood donor data 
collection এবং approval system-এর জন্য:

1. Trigger: Google Sheets Trigger, "On row added"। এই Sheet একটা 
   Google Form-এর response sheet, columns: Name, Department, 
   BloodGroup, Phone, Consent।

2. IF node "Validate Submission": চেক করবে Department field-এর 
   value exactly "IER", Name/BloodGroup/Phone খালি না, Phone field 
   ঠিক ১১ ডিজিট এবং "01" দিয়ে শুরু। পাস করলে true branch, নাহলে 
   No Operation node দিয়ে থেমে যাবে।

3. true branch-এ Supabase node, operation "Insert", table 
   "blood_donors", field mapping অনুযায়ী, status="pending" (fixed)। 
   Insert-এর response-এর id সেভ রাখো পরের ধাপের জন্য।

4. Telegram node, "Send Message" আমার chat-এ, নাম/Dept/Blood 
   Group/Phone দেখিয়ে, Inline keyboard দুইটা button সহ: 
   "✅ Approve" callback_data="approve_<id>", 
   "❌ Reject" callback_data="reject_<id>"

5. নতুন trigger branch: Telegram Trigger, updates: "callback_query"

6. Code node: callback_data থেকে action আর id আলাদা করো (underscore 
   দিয়ে split)

7. IF node: action == "approve" নাকি "reject"

8. দুইটা আলাদা Supabase Update node (table "blood_donors", filter 
   id দিয়ে): Approve branch status="approved", Reject branch 
   status="rejected", দুটোতেই decided_at=current timestamp

9. সবশেষে প্রতিটা branch-এ Telegram node দিয়ে মূল message edit করে 
   confirmation দেখাও ("✅ Approved হয়েছে" / "❌ Rejected হয়েছে")

সব node-এর নাম স্পষ্ট রাখো। Credentials আমি নিজে পরে select করব।
```

## Common Issues

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| Google Sheets Trigger-এ "No results" | Form-এর সাথে এখনো কোনো Sheet link করা হয়নি | Form → Responses ট্যাব → Sheets আইকন → Create |
| Supabase credential "Couldn't connect" | Host field-এ শুধু Project ID বসানো হয়েছিল, পুরো URL না | `https://<project-id>.supabase.co` ফরম্যাটে বসাতে হবে |
| Telegram-এ approval message আসছে না | (ক) Workflow Active করা হয়নি, (খ) n8n workspace sleep/offline (trial hosting-এ হয়) | Active toggle চেক করো; workspace অফলাইন হলে কিছুক্ষণ অপেক্ষা করে রিফ্রেশ করো |
| Bot personal chat-এ মেসেজ পাঠাতে পারছে না | Telegram-এর নিয়ম: user আগে bot-কে Start না করলে bot মেসেজ পাঠাতে পারে না | নিজের bot খুঁজে আগে Start চাপতে হবে |

## Design Principles

- **Rule-based check** (IF/Switch node) — Department/Blood Group structured data, তাই AI/LLM লাগে না। AI তখনই লাগবে যখন কোনো free-text field বিচার করা প্রয়োজন হবে।
- **Human approval বাধ্যতামূলক** — phone number ও health-related (blood group) তথ্য থাকায়, pilot stage-এ ১০০% submission মানুষ দেখেই approve করছে।
- **Single-category workflow** — এই মুহূর্তে শুধু blood donor handle করে। ভবিষ্যতে একাধিক category (tutor, bashabhara ইত্যাদি) আসলে প্রতিটার নিজস্ব Form + Supabase table + workflow থাকবে, pattern প্রমাণিত ও repetitive হলে তখনই generic/consolidated engine ভাবা হবে।
