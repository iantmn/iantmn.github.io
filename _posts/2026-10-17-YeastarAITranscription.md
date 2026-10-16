---
title: "☎️ AI call transcription for Yeastar with n8n and Azure"
description: "How to transcribe and summarize every recorded call on a Yeastar P-Series PBX with n8n, Azure Speech and Azure OpenAI, for a fraction of the price of Yeastar's own AI add-on."
date: 2026-10-17
permalink: /posts/2026/10/YeastarAITranscription/
categories: [Guides, 🤖 Automation]
tags: [n8n 🔁, Yeastar ☎️, Azure ☁️, AI 🤖, Automation ⚙️]
pin: false
published: false
---

# AI call transcription for Yeastar with n8n and Azure

## Introduction

At NETW we run our telephony on a Yeastar P-Series cloud PBX, and we record our calls. Recordings are useful in theory and almost useless in practice: nobody listens back to a twelve minute call to find out what was agreed. What you actually want afterwards is a few lines of text. What was the question, what was done, and does anyone still need to act.

Yeastar sells exactly that as an AI add-on, and the honest reason we did not buy it is that we could not. Our distributor was unable to supply it, because the product was not ready on Yeastar's side yet. So the choice was to wait for a release date nobody could give us, or to build it. We already had an Azure AI Foundry environment running for other projects, which made the second option look a lot smaller than it sounds.

It was. Every recorded call now arrives as an email with a Dutch summary, a problem and solution breakdown, a list of follow-up actions, and the full transcript with the speakers separated. It runs on a webhook and three API calls.

The first version was a small Python Azure Function, and it worked fine. Moving it to n8n was a cost decision: the Function carried its own running cost for compute, storage and monitoring, while n8n was already running in our stack doing other work. Rebuilding the pipeline there made the orchestration layer effectively free and left only the AI calls as a real expense. That it also suits this kind of job, which is mostly "call an API, take one field, call the next API", was a bonus rather than the reason.

## How it works

```
 Call ends on Yeastar P-Series
          │
          │  webhook, event 30012 "New CDR"
          ▼
 ┌──────────────────────────────────────────────┐
 │ n8n                                          │
 │                                              │
 │  1. Parse CDR, skip calls without recording  │
 │  2. Get a cached Yeastar API token           │
 │  3. Download the .wav recording              │
 │  4. Azure Speech Fast Transcription          │──► Azure Speech (West Europe)
 │     nl-NL + en-US, diarization on            │
 │  5. Format the transcript per speaker        │
 │  6. Summarize into structured JSON           │──► Azure OpenAI, gpt-4o
 │  7. Send the summary email                   │──► Microsoft 365
 └──────────────────────────────────────────────┘
```

There are four moving parts. **Yeastar** pushes a call detail record to a webhook the moment a call ends, and if the call was recorded that record contains the recording's file name. **n8n** receives the webhook and orchestrates everything after it. **Azure Speech** turns the audio into text with the Fast Transcription API, which handles Dutch and English in a single request and separates the speakers. **Azure OpenAI** reads the transcript and returns a structured summary as JSON.

The important thing about that last step is the word *structured*. The email is the obvious output, but because the model returns fixed JSON fields rather than a paragraph, everything downstream becomes easy later.

## Privacy and compliance

Worth settling before you build anything, because it changes where things are allowed to run.

Recording calls and sending them to an AI service is processing of personal data. Under the AVG/GDPR your colleagues and the people who call you must be informed that calls are recorded and analyzed, and you need a processing agreement with your provider. For Azure that is covered by the standard Online Services DPA, but you should keep the Speech and OpenAI resources in an EU region regardless. West Europe is the obvious choice from the Netherlands and is also the lowest latency.

The second decision is retention. A transcript that lands in a mailbox inherits that mailbox's retention policy, which is probably not what you would choose for a verbatim record of every customer conversation. Decide where summaries go and how long they live before you switch the workflow on, not after someone asks.

## Prerequisites

- A Yeastar P-Series PBX (cloud, software or appliance edition) with call recording enabled
- An n8n instance reachable from the PBX over HTTPS, for example behind [Traefik](/posts/2025/2/traefik/)
- An Azure subscription with a **Speech** resource (Standard S0, West Europe) and an **Azure OpenAI** resource with a `gpt-4o` or `gpt-4o-mini` deployment
- A way to send mail from n8n. This guide uses the Microsoft Outlook node against our M365 tenant

## Step-by-Step Setup

### Step 1: Prepare the Azure resources

In the Azure Portal, create a **Speech service** resource in `West Europe` on the `Standard S0` tier. Under **Keys and Endpoint**, copy *KEY 1* and note the region as `westeurope`, lowercase and without a space, because that string goes straight into a URL later.

Then in Azure AI Foundry, open your project, go to **Deployments** and deploy `gpt-4o`. Name the deployment `gpt-4o` so there is one less name to remember. A quota of 10K tokens per minute is plenty for this workload. Copy the endpoint, which looks like `https://<resource>.openai.azure.com/`, and a key.

`gpt-4o-mini` handles call summaries perfectly well and is far cheaper per token, although as the Challenges section shows, that barely affects the total because transcription dominates the bill. Start on `gpt-4o`, read a week of summaries, then try mini and see whether you can tell the difference. Treat it as a quality comparison rather than a saving.

In n8n, store both keys as **Header Auth** credentials so they never sit in a node parameter where they end up in an export:

| Credential | Header name | Value |
|---|---|---|
| Azure Speech | `Ocp-Apim-Subscription-Key` | Speech KEY 1 |
| Azure OpenAI | `api-key` | Azure OpenAI key |

### Step 2: Enable the Yeastar API and webhook

In the Yeastar web portal:

1. Go to **Integrations > API** and enable the API.
2. Note the **Client ID** and **Client Secret**. These act as the username and password for the token endpoint.
3. Enable **Webhook Event Push**, add your n8n webhook URL (created in the next step) and subscribe to event **(30012) New CDR**.
4. Under **Call Features > Recording**, confirm recording is enabled for the extensions and trunks you want summarized, and that the file format is `.wav`.

While you are in the API settings, restrict access to n8n's public IP address. The API hands out tokens that can download any recording on the PBX, so it should not be reachable from the open internet just because the credentials are long.

### Step 3: Receive and filter the webhook

Create a workflow and add a **Webhook** node with method `POST`, an unguessable path such as `yeastar-cdr-<random string>`, and **Respond: Immediately**.

That response setting is not a detail. Transcription and summarization together take twenty to sixty seconds. If n8n only answers once the run finishes, the PBX sits on an open request for a minute, and a webhook sender whose request times out is entitled to conclude the event never arrived and send it again. Answer first, work afterwards.

Yeastar sends the record as a JSON *string* inside the `msg` field, so a **Code** node (mode *Run Once for Each Item*) unpacks it:

```javascript
// Yeastar wraps the CDR as a JSON string in body.msg
const body = $json.body;
const msg = typeof body.msg === 'string' ? JSON.parse(body.msg) : body.msg;

return {
  json: {
    event_type: body.type,              // 30012 for New CDR
    call_id: msg.call_id,
    call_type: msg.type,                // Inbound / Outbound / Internal
    call_from: msg.call_from,
    call_to: msg.call_to,
    time_start: msg.time_start,
    talk_duration: Number(msg.talk_duration ?? 0),
    recording: (msg.recording ?? '').trim(),
  },
};
```

Follow it with an **If** node that only lets real recordings through:

{% raw %}
- `{{ $json.event_type }}` is equal to `30012`
- and `{{ $json.recording }}` is not empty
- and `{{ $json.talk_duration }}` is larger than `10`
{% endraw %}

The duration filter skips misdials and calls that hang up during the greeting. There is nothing to summarize in eight seconds of audio, and every one you skip is an API call you do not pay for.

### Step 4: Get a Yeastar access token, and cache it

The Yeastar API uses short-lived tokens. You exchange the Client ID and Secret at `POST /openapi/v1.0/get_token` and get a token valid for thirty minutes.

There is a catch that will not show up in testing: Yeastar allows only **eight valid tokens per application at once**. Fetching a fresh token per call is the obvious implementation and it works fine until a busy morning puts nine recorded calls inside the same half hour.

Before the token logic, put a **Set** node named `Config` right after the webhook filter, holding three values: `yeastar_base_url`, `yeastar_client_id` and `yeastar_client_secret`. It is tempting to reach for `$env` here instead, but n8n blocks environment access inside Code nodes by default, and if you run external task runners it is blocked in the runner as well. A Set node works everywhere and survives an n8n upgrade without surprises. The trade-off is that the secret then lives in the workflow, so treat workflow exports as sensitive and keep n8n itself behind authentication.

Then cache the token in the workflow's static data and only refresh when it is nearly expired:

```javascript
// Reuse a cached token until 2 minutes before expiry.
// Yeastar allows max 8 valid tokens per app, so never fetch one per call.
const cfg = $('Config').first().json;
const cache = $getWorkflowStaticData('global');
const now = Date.now();

if (!cache.token || cache.expiresAt < now + 2 * 60 * 1000) {
  const res = await this.helpers.httpRequest({
    method: 'POST',
    url: `${cfg.yeastar_base_url}/openapi/v1.0/get_token`,
    headers: { 'User-Agent': 'OpenAPI' },   // Yeastar rejects requests without a User-Agent
    body: {
      username: cfg.yeastar_client_id,
      password: cfg.yeastar_client_secret,
    },
    json: true,
  });

  if (res.errcode !== 0) {
    throw new Error(`Yeastar get_token failed: ${res.errmsg}`);
  }

  cache.token = res.access_token;
  cache.expiresAt = now + res.access_token_expire_time * 1000; // seconds -> ms
}

return { json: { ...$json, access_token: cache.token } };
```

One thing to know about that cache: static data is only persisted for **production** executions of an **active** workflow. During manual test runs it is empty every time, so every test fetches a new token and you will not see the caching work until the workflow is live.

### Step 5: Download the recording

Downloading happens in two calls. First you ask the PBX for a temporary download URL, then you fetch the file from it.

An **HTTP Request** node for the URL:

{% raw %}
- Method `GET`, URL `{{ $('Config').first().json.yeastar_base_url }}/openapi/v1.0/recording/download`
- Query parameters: `file` = `{{ $json.recording }}` and `access_token` = `{{ $json.access_token }}`
- Header `User-Agent: OpenAPI`
{% endraw %}

The response contains `download_resource_url`, a path relative to the PBX. A second **HTTP Request** node fetches it:

{% raw %}
- Method `GET`, URL `{{ $('Config').first().json.yeastar_base_url }}{{ $json.download_resource_url }}`
- Query parameter `access_token` = `{{ $('Yeastar token').item.json.access_token }}`
- Header `User-Agent: OpenAPI`
- Options > Response > Response Format: **File**, output field `audio`
{% endraw %}

The `.wav` is now binary data on the item.

### Step 6: Transcribe with Azure Speech

Azure has several speech-to-text APIs and the right one here is **Fast Transcription**. It is synchronous, so you send the file and get the transcript back in the same request, it runs faster than real time, and it supports diarization. Batch transcription is cheaper per audio hour but wants the audio in blob storage and needs polling for the result, which is a lot of plumbing for one call at a time.

An **HTTP Request** node:

- Method `POST`
- URL `https://westeurope.api.cognitive.microsoft.com/speechtotext/transcriptions:transcribe?api-version=2024-11-15`
- Authentication: the *Azure Speech* Header Auth credential
- Body Content Type **Form-Data**, with `audio` as an *n8n Binary File* from field `audio`, and `definition` as a *Form Data* field containing:

```json
{
  "locales": ["nl-NL", "en-US"],
  "diarization": { "enabled": true, "maxSpeakers": 2 },
  "profanityFilterMode": "None"
}
```

Passing two locales turns on language identification, so Azure works out per call whether it is listening to Dutch or English. The response comes back as a `phrases` array, where each phrase carries a `speaker` number, an `offsetMilliseconds` and the recognized `text`.

### Step 7: Turn phrases into a readable transcript

A **Code** node merges consecutive phrases from the same speaker, so the transcript reads as a conversation rather than as a list of fragments:

```javascript
const phrases = ($json.phrases ?? [])
  .filter(p => p.text?.trim())
  .sort((a, b) => a.offsetMilliseconds - b.offsetMilliseconds);

const lines = [];
let current = null;

for (const p of phrases) {
  const speaker = `Spreker ${p.speaker ?? '?'}`;
  if (current && current.speaker === speaker) {
    current.text += ' ' + p.text.trim();
  } else {
    current = { speaker, text: p.text.trim() };
    lines.push(current);
  }
}

const transcript = lines.map(l => `${l.speaker}: ${l.text}`).join('\n\n');

// Carry the call metadata from the Parse CDR node along
return {
  json: {
    ...$('Parse CDR').item.json,
    transcript: transcript || '(geen transcript beschikbaar)',
  },
};
```

The speakers are labelled `Spreker 1` and `Spreker 2`, not "agent" and "caller". That is deliberate, and the Challenges section explains why.

### Step 8: Summarize with Azure OpenAI

An **HTTP Request** node against the deployment, authenticated with the *Azure OpenAI* Header Auth credential:

{% raw %}
```json
{
  "temperature": 0.2,
  "max_tokens": 800,
  "response_format": { "type": "json_object" },
  "messages": [
    {
      "role": "system",
      "content": "Je bent een assistent voor de servicedesk van een MSP. Je analyseert telefoongesprekken en antwoordt uitsluitend met JSON in het Nederlands, met deze velden: samenvatting (2-3 zinnen), medewerker (welke spreker de MSP-medewerker is, of 'onbekend'), probleem (de vraag of het probleem van de beller, of 'n.v.t.' bij interne gesprekken), oplossing (wat er is gedaan of afgesproken), actie_vereist (vervolgacties met eigenaar, of 'geen'), sentiment (Positief, Neutraal of Negatief), categorie (bijv. Netwerk, Hardware, Software, Account, Telefonie, Algemeen)."
    },
    {
      "role": "user",
      "content": "Van: {{ $json.call_from }}\nNaar: {{ $json.call_to }}\nType: {{ $json.call_type }}\nDuur: {{ Math.floor($json.talk_duration / 60) }}m {{ $json.talk_duration % 60 }}s\n\nTranscript:\n{{ JSON.stringify($json.transcript).slice(1, -1) }}"
    }
  ]
}
```
{% endraw %}

The URL is `https://<resource>.openai.azure.com/openai/deployments/gpt-4o/chat/completions?api-version=2024-10-21`.

Two details earn their place. `response_format: json_object` forces valid JSON out of the model; the first version stripped Markdown code fences off the answer with string replacement, which works right up until the day it does not. And the transcript is passed through `JSON.stringify(...).slice(1, -1)` so that quotes and newlines in what people actually said cannot break the request body.

A small **Code** node parses the result:

```javascript
const summary = JSON.parse($json.choices[0].message.content);
return { json: { ...$('Format transcript').item.json, summary } };
```

### Step 9: Send the summary

Finally a **Microsoft Outlook** node set to *Send a message*, addressed to the shared mailbox that should receive summaries, with an HTML body:

{% raw %}
```
Subject: Gesprek {{ $json.call_from }} → {{ $json.call_to }} | {{ $json.summary.categorie }}
```
{% endraw %}

The body puts the summary fields at the top, highlights the follow-up actions, and keeps the full transcript at the bottom for anyone who wants to check what was really said.

### Step 10: Catch the failures

Add a second workflow with an **Error Trigger** node and point the main workflow at it under **Settings > Error Workflow**. Mine pushes a short message to [Pushover](https://pushover.net/) containing the execution URL, so a failed call arrives on my phone as a notification I can tap, land on the failed execution, see which node broke and retry it.

That retry button is most of the reason a visual workflow beats a Function for this job. When a call fails to transcribe you want to see which step broke and run it again, not read application logs.

## Challenges & Learnings

🔑 **Eight tokens, and not one more**: The Yeastar API allows eight valid access tokens per application at a time, each living thirty minutes. Requesting a fresh token per call is the obvious implementation, and it behaves perfectly while you are testing with a handful of calls. It then fails on the first busy morning, when the ninth recorded call inside a half hour gets an error instead of a token and the run dies on a customer conversation you actually wanted. Caching the token in static data and refreshing it two minutes before expiry turns a hard platform limit into something you never think about again.

⏱️ **Answer the webhook before doing the work**: A full run takes twenty to sixty seconds. If n8n responds only when everything is finished, the PBX holds an open HTTP request for the duration, and a sender whose request times out is entitled to assume delivery failed and send the event again. You then transcribe and summarize the same call twice and mail it twice. Responding immediately costs nothing and removes the whole class of problem.

🗣️ **Diarization knows there are two voices, not who they are**: The recording is one mixed stream, so Azure can separate the speakers but has no idea which of them works for you. My first version assumed speaker one is the employee, which is roughly right for inbound calls because we answer with a greeting, and exactly wrong for outbound calls because the other side says "hallo" first. Handing that judgement to the language model, which can see which speaker introduces themselves as NETW, works far better than any rule I was going to write. This is the general lesson: use the model for the part that needs context, not for the part that needs arithmetic.

🌍 **Two locales, zero per-call configuration**: Passing both `nl-NL` and `en-US` lets Azure identify the language itself, so Dutch calls and the occasional English supplier both work with nothing configured per extension or per trunk. Dutch technical jargon survives better than I expected.

🏷️ **The model has no idea who anyone is**: This is the real limitation, and it is not a transcription problem. The pipeline sees a phone number and an audio file, and nothing else. It has no connection to our client base, so when the model reports which company called or which of our people handled it, it is working purely from what was said out loud. Names it has never seen come out as phonetic guesses, and company names in particular can end up genuinely strange. The `medewerker` field has the same weakness. The fix is not a better prompt, it is better input: look the caller ID up against the client database first and hand the model the real company name and the extension owner as facts rather than asking it to infer them. That is the next thing I want to build.

🧾 **Structured output beats a nicely written paragraph**: Asking for fixed JSON fields rather than free text makes summaries consistent to read, but the real payoff is downstream. Because the summary is data, adding a HaloPSA ticket, a Teams alert on negative sentiment, or a monthly report of what customers call about is one extra node each. Had the model returned prose, every one of those would have meant parsing English back into fields.

💶 **Almost all of the cost is the transcription, not the AI**: I have not run this long enough to have a real invoice, so the following is an estimate from list prices rather than a measurement. Azure Speech bills per hour of audio, somewhere around €1, while gpt-4o bills per token. A five minute Dutch call is roughly 1,300 tokens of transcript, plus a short system prompt, plus about 300 tokens of JSON coming back. That works out at roughly €0.08 for the transcription and well under a cent for the summary, so about **€0.085 per five minute call, or €0.017 per call minute**. A thousand call minutes a month lands near €17.

The useful part of that arithmetic is the ratio. Speech is about nine tenths of the bill, which means the obvious saving, switching to `gpt-4o-mini`, barely moves the total: it takes a five minute call from roughly €0.085 to €0.080. If this ever needs to be cheaper, the levers are the duration filter, excluding call types that are never worth summarizing, and Azure's commitment tiers for Speech. Changing models is a quality decision, not a cost one.

⚖️ **Privacy is a design decision, not a footnote**: The transcript ends up in a mailbox, which means a verbatim record of every customer conversation now lives under whatever retention policy that mailbox happens to have. That is a decision worth making deliberately, in advance, with the people whose calls these are. Building it is the easy part.

## Troubleshooting

If nothing arrives in n8n at all, check that the workflow is **activated**. While the editor is open only the `/webhook-test/` path responds, which catches everyone at least once. After that, verify the URL in Yeastar and confirm event 30012 is actually subscribed.

If `get_token` starts returning errors after working fine, you have hit the eight-token limit. Confirm the cache is doing its job, remembering that static data only persists for production executions of an active workflow.

If the recording download fails, the usual causes are a missing `User-Agent` header, which Yeastar rejects outright, an expired token, or a recording file name containing characters that need URL encoding.

If Azure Speech returns a 401, the key is wrong or the region in the URL does not match the resource. A 400 instead usually means the `definition` was not sent as its own form field, or the audio went up as text rather than binary.

If the transcript comes back empty, the recording is probably silent or extremely short. Check the duration filter and the recording settings on the extension.

If Azure OpenAI returns a 404, the deployment name in the URL does not match the deployment in Foundry. It is almost never the key.

## Next Steps

- Enrich the call before summarizing by looking the caller ID up against the client database, so the model is told which company called and who owns the extension instead of guessing both from the audio
- Create or update a HaloPSA ticket straight from the summary, using `categorie` and `actie_vereist`, with the transcript attached
- Send a Teams notification for any call with sentiment `Negatief` or an open follow-up action
- Store the JSON per call so you can report on what customers actually call about, which is a question most MSPs answer from memory
- Exclude specific numbers, such as HR or private lines, before the recording is downloaded rather than after
- Compare a week of `gpt-4o` summaries against `gpt-4o-mini` and move down if you cannot tell them apart

## Conclusion

Yeastar already does the genuinely hard part. It records the call and tells you, through a webhook, exactly when it finished and where the audio lives. Everything after that is three API calls in a row, which is precisely what n8n is good at.

The result is a written summary in the right inbox about a minute after the call ends, for something in the order of two cents per call. We started building because we could not buy the vendor's version, which is a slightly embarrassing origin story, but it turned out to be the better outcome: the pipeline does what we want rather than what the add-on offers, and it goes wherever we point it next.

The part worth copying is not the transcription, it is building the output as structured data from the start. The email was finished on day one; the tickets, the alerts and the reporting were all one node away, because nothing has to be parsed back out of a paragraph.

💡 Want to know more about building AI pipelines on top of your existing telephony, or need help working out whether to build or buy? Feel free to reach out!
