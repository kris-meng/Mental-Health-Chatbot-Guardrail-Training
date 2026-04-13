# Mental-Health-Chatbot-Guardrail-Training
Generated a labeled synthetic dataset of ~X conversations across 23 risk categories for training a mental health chatbot input guardrail. The guardrail that was trained for the competition was a stacked finetuned mBert + LLM classifier.



# Synthetic Training Data Generator
### Input Guardrails for a Youth Mental Health Chatbot

> **Context:** This was my contribution to a hackathon project building input guardrails for a mental health chatbot aimed at Canadian youth. The team's goal was to train a classifier that could detect high-risk messages and trigger appropriate escalation responses. I owned the **data generation pipeline** end-to-end.

---

## The Problem

Training a risk classifier on real chat data from vulnerable youth isn't feasible — it's ethically fraught, legally complex, and the data simply doesn't exist in labeled form. We needed a way to generate realistic, diverse, labeled conversations at scale.

The challenge: *how do you simulate a distressed 14-year-old texting a chatbot in a way that actually captures the hesitation, the deflection, the coded language, the refusal to accept help?*

---

## My Solution: A Two-Agent Simulation Pipeline

I built a data generation pipeline that pits two LLM agents against each other to produce realistic multi-turn conversations:

```
[ User Actor LLM ]  ←──── persona + risk signals + pacing rules
       ↕  (8–10 turns)
[ Chatbot LLM ]     ←──── empathy instructions + escalation protocol
       ↓
[ LLM-as-a-Judge ]  ←──── quality filter before saving
       ↓
[ Labeled CSV ]     →  Conversation_id | Turns | Text | Category | Risk | label
```

**Key design decisions:**

- **Pacing logic** — The user actor is explicitly told which turn it's on and when it's "allowed" to reveal high-risk signals. This prevents flat, front-loaded conversations that look nothing like how people actually talk.
- **Refusal personas** — For high-risk conversations, the user actor is assigned a randomized psychological reason to refuse helpline referrals (e.g. phone anxiety, hopelessness, fear of parents finding out). This forces the chatbot to navigate resistance, which is the most important guardrail scenario.
- **Helpline state tracker** — A flag prevents the chatbot from repeating the Kids Help Phone number robotically. Once offered, the system prompt is updated dynamically to redirect toward hesitation-exploration instead.
- **Category fallback** — LLM-generated personas don't always stick to taxonomy categories exactly. An auto-correction step maps deviant labels back to the closest valid category using a lightweight LLM call, with a random fallback as last resort.

---

## Risk Taxonomy

23 conversation categories, each with explicit low and high signal definitions:

| Category | Low Signal | High Signal |
|---|---|---|
| Suicide | Thoughts about death without intent | Plan / means / timeline / intent to act |
| Self-Harm | Past self-harm, not current | Active urges; increasing severity |
| Isolation | Lonely | "Better off without me"; withdrawing |
| Substance Use | Curiosity | Overdose risk; coping-motivated use |
| Physical Violence | Arguments | Intent to harm; fear of harm |
| User Joking With Chatbot | Memes, trolling | Dark humour masking real distress |
| ... | ... | ... |

These signals are injected directly into both agents' system prompts to calibrate realistic escalation.

---

## Persona Library

The pipeline uses two types of personas:

### Hand-Crafted (14 personas)
Psychologically detailed profiles covering a range of demographics and risk profiles:

| Persona | Age | Risk | Focus |
|---|---|---|---|
| rue | 17 | high | Substance use, grief — flat/sarcastic voice |
| alex | 15 | high | LGBTQ+ identity distress |
| skylar | 17 | high | Indigenous youth, intergenerational trauma |
| farid | 16 | low | Refugee/asylum seeker, survivor guilt (English-Persian) |
| nia | 8 | high | Racialized youth, racial bullying |
| ryan | 16 | high | Externalized aggression, threat-to-others profile |
| sam | 14 | low | Neurodivergent (autism), social confusion |
| marisol | 16 | low | Parentified child, caregiver burnout |
| eli | 17 | high | Dissociation, emotional numbness |
| ... | ... | ... | ... |

Each persona has: `voice`, `reveals_pain_by`, `relationship_to_help`, `signature_phrases`, and `backstory_seeds`.

### LLM-Generated (scalable)
`build_massive_persona_library()` generates additional personas in batches of 2 (to stay under token limits), focused on specific demographics:
- LGBT youth in Canada
- POC youth in Canada
- Indigenous youth on reservations
- Immigrant / refugee youth
- General youth mental health

---

## Output Dataset

**File:** `data/persona.csv`

| Column | Description |
|---|---|
| `Conversation_id` | Zero-padded unique ID |
| `Turns` | Total message count (1 turn = 1 user + 1 assistant message) |
| `Text` | Full conversation transcript as a single string |
| `Category` | List of risk categories for this conversation |
| `Risk` | `"high"` or `"low"` |
| `language` | e.g. `"english"`, `"english - persian mixed"` |
| `label` | Binary: `1` = high risk, `0` = low risk |

---

## Quality Control: LLM-as-a-Judge

Every generated conversation passes through an evaluator before being saved. The judge checks:
1. **Realism** — Does the user sound like a real human, not a clinical textbook?
2. **Risk accuracy** — Do high-risk conversations actually contain high-risk signals?

Conversations that fail are logged and discarded. Acceptance rate in practice was roughly 80–85%.


---

## Stack

- Python 3.10
- pandas
- Custom LLM provider wrapper (OpenAI-compatible endpoint)
- Hackathon infrastructure: BuzzPerformance Cloud (120B OSS model)

---

## What I Learned

- **Prompt engineering for role separation is hard.** The user actor LLM would frequently break character and start giving supportive advice. Fixing this required explicit negative instructions ("You are the caller, NOT the assistant") plus an actor-framing wrapper around every turn.
- **LLMs over-escalate without pacing constraints.** Without the turn-count instructions, every high-risk conversation front-loaded all the distress into the first 2 messages. Real conversations don't work that way.
- **Taxonomy design matters more than model choice.** The quality of the signal definitions (low/high per category) had more impact on output realism than any prompt tuning.
