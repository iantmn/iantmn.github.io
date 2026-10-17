---
title: "☎️ AI call transcription for Yeastar with n8n and Azure"
description: "How to transcribe and summarize every recorded call on a Yeastar P-Series PBX with n8n, Azure AI Speech and Azure OpenAI, including the codec problem that makes Whisper reject PBX recordings outright."
date: 2026-10-17
permalink: /posts/2026/10/YeastarAITranscription/
categories: [Guides, 🤖 Automation]
tags: [n8n 🔁, Yeastar ☎️, Azure ☁️, AI 🤖, Automation ⚙️]
pin: false
published: true
---

# AI call transcription for Yeastar with n8n and Azure

## Introduction

At NETW we run our telephony on a Yeastar P-Series cloud PBX, and we record our calls. Recordings are useful in theory and almost useless in practice: nobody listens back to a twelve minute call to find out what was agreed. What you actually want afterwards is a few lines of text. What was the question, what was done, and does anyone still need to act.

Yeastar sells exactly that as an AI add-on, and the honest reason we did not buy it is that we could not. Our distributor was unable to supply it, because the product was not ready on Yeastar's side yet. So the choice was to wait for a release date nobody could give us, or to build it. We already had an Azure AI Foundry environment running for other projects, which made the second option look a lot smaller than it sounds.

It was mostly. Every recorded call now arrives as an email with a summary written in the language of the call, followed by the full transcript with the speakers separated. The pipeline is a webhook, two filters, four API calls and an email.

The first version was a small Python Azure Function, and it worked fine. Moving it to n8n was a cost decision: the Function carried its own running cost for compute, storage and monitoring, while n8n was already running in our stack doing other work. Rebuilding the pipeline there made the orchestration layer effectively free and left only the AI calls as a real expense.

One part was not small, and it is the reason this guide exists in the form it does. The obvious implementation, sending the recording to Whisper, does not work at all with a PBX. More on that below, because it cost me three attempts to understand why.

## How it works

```
 Call ends on Yeastar P-Series
          │
          │  webhook, event 30012 "New CDR"
          ▼
 ┌──────────────────────────────────────────────┐
 │ n8n                                          │
 │                                              │
 │  1. Parse the CDR out of the envelope        │
 │  2. Drop ring group calls (they fire twice)  │
 │  3. Skip calls with no recording             │
 │  4. Get a Yeastar API token                  │
 │  5. Ask for a download URL, then fetch .wav  │
 │  6. Relabel audio/x-wav to audio/wav         │
 │  7. Azure AI Speech Fast Transcription       │──► Azure AI Speech
 │     nl-NL + en-US, diarization on            │
 │  8. Build a readable transcript document     │
 │  9. Summarize it                             │──► Azure OpenAI, gpt-4.1
 │ 10. Send summary plus transcript by SMTP     │──► shared mailbox
 └──────────────────────────────────────────────┘
```

**Yeastar** pushes a call detail record to a webhook the moment a call ends, and if the call was recorded that record names the recording file. **n8n** orchestrates everything after it. **Azure AI Speech** turns the audio into text with the Fast Transcription API, which handles Dutch and English in one request and separates the speakers. **Azure OpenAI** reads the transcript and writes the summary.

Both Azure calls go to the **same resource**. That is worth knowing up front, because it is not obvious: an Azure AI Foundry project provisions a multi-service Cognitive Services resource, so Fast Transcription and your chat deployment live on one host with one key. Only the path, the header name and the body shape differ between them.

## Privacy and compliance

Worth settling before you build anything, because it changes where things are allowed to run.

Recording calls and sending them to an AI service is processing of personal data. Under the AVG/GDPR your colleagues and the people who call you must be informed that calls are recorded and analyzed, and you need a processing agreement with your provider. For Azure that is covered by the standard Online Services DPA, but keep the resource in an EU region regardless. We use Sweden Central; West Europe works equally well from the Netherlands.

The second decision is retention. A transcript that lands in a mailbox inherits that mailbox's retention policy, which is probably not what you would choose for a verbatim record of every customer conversation. Decide where summaries go and how long they live before you switch the workflow on, not after someone asks.

## Prerequisites

- A Yeastar P-Series PBX (cloud, software or appliance edition) with call recording enabled
- An n8n instance reachable from the PBX over HTTPS, for example behind [Traefik](/posts/2025/2/traefik/)
- An Azure AI Foundry project in an EU region, which gives you a multi-service Cognitive Services resource
- A chat model deployed in that project. We use `gpt-4.1`
- An SMTP credential in n8n for sending the summary

## Step-by-Step Setup

### Step 1: One Azure resource, two APIs

In Azure AI Foundry, create a project in an EU region and deploy a chat model. Name the deployment something you will remember, because the deployment name goes straight into a URL. Then open the resource's **Keys and Endpoint** page and copy `KEY 1` and the endpoint host, which looks like `https://<resource>.cognitiveservices.azure.com`.

You do not deploy anything for transcription. Fast Transcription is not a model you deploy, it is an API available on the resource as soon as the resource exists. The two surfaces differ in three ways and nothing else:

| | Fast Transcription | Chat completions |
|---|---|---|
| Path | `/speechtotext/transcriptions:transcribe` | `/openai/deployments/<name>/chat/completions` |
| Auth header | `Ocp-Apim-Subscription-Key` | `api-key` |
| Body | multipart form data | JSON |

Store the key as an n8n **credential**, not as a value typed into the node. An exported workflow carries node parameters with it in plain text but leaves credentials behind, so a key pasted into a header field travels with every export you ever share. Ask me how I know.

### Step 2: Enable the Yeastar API and webhook

In the Yeastar web portal:

1. Go to **Integrations > API** and enable the API.
2. Note the **Client ID** and **Client Secret**, which act as the username and password for the token endpoint.
3. Enable **Webhook Event Push**, add your n8n webhook URL and subscribe to event **(30012) New CDR**.
4. Under **Call Features > Recording**, confirm recording is enabled for the extensions and trunks you want summarized.

While you are in the API settings, restrict access to n8n's public IP address. The API hands out tokens that can download any recording on the PBX, so it should not be reachable from the open internet just because the credentials are long. For the same reason, treat the webhook path as a secret: n8n generates a UUID for it, and that UUID is the only thing standing between the internet and your pipeline.

### Step 3: Receive and parse the CDR

Add a **Webhook** node with method `POST`. Yeastar wraps the record as a JSON *string* inside `body.msg`, so a **Code** node unpacks it and flattens it next to the envelope fields:

```javascript
// Yeastar sends body.msg as a JSON-encoded STRING, not an object.
// Parse it and flatten it alongside the outer envelope (type, sn).
const body = $input.first().json.body;
const msg = JSON.parse(body.msg);

return [{
  json: {
    event_type: body.type,
    pbx_sn: body.sn,
    ...msg
  }
}];
```

That gives you `call_id`, `call_from`, `call_to`, `type`, `time_start`, `call_duration`, `talk_duration`, `status` and `recording` on one flat object.

### Step 4: Drop ring group calls

This filter is not obvious and you will not miss it until your inbox tells you. A call into a ring group produces **two** CDR webhooks: one for the group and one for the agent who actually answered. Process both and every incoming call gets transcribed twice, summarized twice and mailed twice, at double the cost.

An **If** node solves it. Our ring groups live in the 6000 to 6999 extension range, so:

{% raw %}
- `{{ $json.call_to }}` is greater than or equal to `6000`
- and `{{ $json.call_to }}` is less than or equal to `6999`
{% endraw %}

The **false** branch continues the workflow. The true branch goes nowhere, which is the point. An agent always picks up, so the agent's CDR carries the recording and the group's record is redundant.

Enable **loose type validation** on this node, because `call_to` arrives as a string and you are comparing it numerically.

### Step 5: Skip calls with no recording

A second **If** node, checking that {% raw %}`{{ $json.recording }}`{% endraw %} is not empty. Internal transfers, missed calls and anything on an extension without recording enabled all stop here.

### Step 6: Get an access token

An **HTTP Request** node posting to `https://<your-pbx>/openapi/v1.0/get_token` with the client ID as `username` and the client secret as `password` in a JSON body. Set a `User-Agent` header, because Yeastar rejects requests without one.

Put the client ID and secret in an n8n credential rather than in the JSON body, for the same export reason as the Azure key.

A note on what this does **not** do. We request a fresh token for every call, and Yeastar allows only **eight valid tokens per application at a time**, each living thirty minutes. The arithmetic is uncomfortable: nine recorded calls inside half an hour and the ninth fails. It has not bitten us yet because our call volume sits below that, but it is a latent bug rather than a design. The fix is to cache the token in the workflow's static data and refresh it two minutes before expiry, which is on my list. Note that static data only persists for production executions of an active workflow, so caching is invisible during manual test runs.

### Step 7: Download the recording

Two calls. First ask the PBX where the file is:

{% raw %}
- Method `GET`, URL `https://<your-pbx>/openapi/v1.0/recording/download`
- Query: `file` = `{{ $('Parse Yeastar CDR').item.json.recording }}`, `access_token` = `{{ $json.access_token }}`
{% endraw %}

The response is JSON containing `errcode`, `errmsg`, `file` and `download_resource_url`. That URL is a **relative** path and is valid for thirty minutes. It does not contain the audio.

Put an **If** node here checking {% raw %}`{{ $json.errcode }}`{% endraw %} equals `0`, so a failed lookup stops cleanly instead of sending nonsense to the next node.

Then fetch the bytes:

{% raw %}
- Method `GET`, URL `https://<your-pbx>{{ $json.download_resource_url }}`
- Query: `access_token` = `{{ $('Get Yeastar Access Token').item.json.access_token }}`
- Options > Response > Response Format: **File**, output property `recording_audio`
{% endraw %}

That `access_token` on the second call is required even though Yeastar's documentation does not mention it. Omit it and you get `errcode 10004 TOKEN EXPIRED`, which is a confusing way to say "you forgot the token entirely".

### Step 8: Relabel the audio

A three line **Code** node that looks pointless and is not:

```javascript
// Yeastar's download server declares Content-Type: audio/x-wav, a non-standard
// legacy MIME string. n8n carries that straight through as the multipart
// Content-Type when this binary is uploaded to Azure. Relabel before sending,
// no re-encoding.
const item = $input.first();
item.binary.recording_audio.mimeType = 'audio/wav';
item.binary.recording_audio.fileName = 'recording.wav';
return [item];
```

Nothing is transcoded here. The bytes are identical; only the label changes.

### Step 9: Transcribe with Fast Transcription

An **HTTP Request** node:

- Method `POST`
- URL `https://<resource>.cognitiveservices.azure.com/speechtotext/transcriptions:transcribe?api-version=2025-10-15`
- Header `Ocp-Apim-Subscription-Key` from your credential
- Body content type **multipart form data**, with `audio` as binary data from field `recording_audio`, and a `definition` form field containing:

```json
{"locales":["nl-NL","en-US"],"diarization":{"enabled":true,"maxSpeakers":2}}
```

Listing two locales turns on language identification, so Azure decides per call whether it is hearing Dutch or English. `maxSpeakers: 2` matches a normal two-party call.

Fast Transcription is the right API here for three reasons: it is synchronous, so you send the file and get the transcript in the same request; it is faster than real time; and it supports diarization. Batch transcription is cheaper per audio hour but wants the audio in blob storage and needs polling, which is a lot of plumbing for one call at a time.

It is also, as the Challenges section explains, the only one of these APIs that accepts what a PBX actually produces.

The response is not the OpenAI shape. You get `combinedPhrases` with the full text, and `phrases` with per-segment entries carrying `speaker`, timestamps and confidence. The next node reads `phrases`, not `text`.

### Step 10: Build the transcript document

A **Code** node merges consecutive phrases from the same speaker into paragraphs and wraps the result in a header of call metadata, so the model gets context and the human at the end gets something readable:

```javascript
const data = $input.first().json;
const cdr = $('Parse Yeastar CDR').item.json;

function fmtDuration(totalSeconds) {
  const s = Number(totalSeconds) || 0;
  return `${Math.floor(s / 60)}m ${s % 60}s`;
}

const lines = [];
let current = null;
for (const p of (data.phrases || [])) {
  if (current && current.speaker === p.speaker) {
    current.text += ' ' + p.text;
  } else {
    if (current) lines.push(current);
    current = { speaker: p.speaker, text: p.text };
  }
}
if (current) lines.push(current);

const transcriptBody = lines
  .map(l => `Speaker ${l.speaker}: ${l.text}`)
  .join('\n\n');

const document = `CALL TRANSCRIPT
===============

From:      ${cdr.call_from}
To:        ${cdr.call_to}
Direction: ${cdr.type}
Date/Time: ${cdr.time_start}
Duration:  ${fmtDuration(cdr.call_duration)} (talk time: ${fmtDuration(cdr.talk_duration)})
Status:    ${cdr.status}
Call ID:   ${cdr.call_id}

TRANSCRIPT
----------

${transcriptBody}
`;

return [{ json: { ...cdr, transcript_document: document } }];
```

Note that it reads the CDR fields from the named node rather than from `$json`, because at this point `$json` holds the transcription response. Rename that node and this breaks. The output carries every CDR field forward alongside `transcript_document`, so the email node needs nothing else.

### Step 11: Summarize

An **HTTP Request** node to `https://<resource>.cognitiveservices.azure.com/openai/deployments/<deployment>/chat/completions?api-version=2025-01-01-preview`, authenticated with the `api-key` header, `temperature` at `0.2` and `max_tokens` at `1500`.

The user message is simply the document, JSON-encoded so that quotes and newlines in what people said cannot break the request body:

{% raw %}
```
{{ JSON.stringify($json.transcript_document) }}
```
{% endraw %}

The system prompt is where all the actual work is. It went through a lot of iterations, and almost every rule in it exists because of a specific bad summary:

```text
You summarize phone call transcripts for a Dutch IT managed service provider
(MSP). Not every call is a support call: some are sales, internal, personal, or
administrative. Read the whole transcript first and let its content dictate the
shape of the summary; forcing a problem/solution shape onto a call that has none
produces nonsense.

Output format:
- Plain text, no markdown, no bullet points.
- If a person's name is clearly stated or self-introduced in the conversation,
  start with a line "User: <name> (client)" or "User: <name> (NETW)". If both a
  client-side contact and a NETW employee are named, list both, comma-separated,
  each tagged with their role. Never guess a name from a phone number, extension,
  or the call metadata block alone, only from what is actually said in the
  conversation.
- If an external company or organization is clearly named or identifiable from
  what was said, add a line "Company: <name>". Omit this line for purely internal
  calls, or when no company is mentioned.
- Omit the User and Company lines entirely when nothing is discoverable this way;
  never write "not applicable" for them, they are best-effort, not required fields.
- Next, on its own line: call type (support / sales / internal / personal /
  administrative / other) and one short phrase naming the subject.
- Only for call type personal or administrative, and only when the call truly
  contains no problem, status, decision, system, or action item, however brief:
  stop here with at most one added sentence of context, and skip everything below.
  An informal or joking tone is never by itself a reason to use this shortcut. A
  casual-sounding internal, support, or sales call that discusses a real piece of
  work (a tool, a vendor, a task, a pending request, a technical choice) still
  goes through the full field structure below, even if the participants joke
  around or the language is very informal.

Otherwise, write the following fields as separate labeled lines, in this order,
one to two sentences each.

Core fields. Always include these three. If the transcript gives a field nothing,
write "not applicable" rather than omitting the line or inventing content.
- Problem: the issue or question raised, if any. If a root cause was identified
  (as opposed to just the symptom reported), note it here.
- Solution: what was resolved, proposed, or the current status, if any.
- Decisions: what was explicitly agreed or decided, if any.

Situational fields. Include a line only when the transcript actually contains
that content; omit it entirely otherwise, do not write "not applicable" for these.
- Action items: concrete next steps, each tagged with who owns it (NETW or client)
  and a deadline if one was stated.
- Urgency: only if the call signals something time-critical, an outage, or a
  contractual response-time commitment. Skip for routine calls.
- System: the specific server, service, site, or asset involved, if named.
- Client sentiment: only when the transcript contains a clear, explicit expression
  of frustration, dissatisfaction, satisfaction, or relationship risk (e.g. a
  complaint, a threat to switch providers). Never infer mood from tone or general
  politeness alone.
- Opportunity: only when the client explicitly mentions wanting more licenses,
  additional services, a hardware refresh, or similar, even in passing. Never
  infer an opportunity from an unrelated technical need.

Client sentiment and Opportunity are judgment calls in a way the others aren't.
When the evidence is thin, leave them out rather than stretching to fill them.

Language: write in the same language the call was conducted in. If the call mixes
Dutch and English, write in whichever language dominates, and keep quoted phrases,
product names, or technical terms in their original wording.

Fidelity: preserve concrete details exactly as stated (names, dates, ticket or
case numbers, amounts, versions). Never fabricate or infer content the transcript
doesn't support. If the transcript covers more than one distinct issue, cover each
briefly rather than merging them into one vague line.

Keep it compact: one or two sentences per field, most calls should total well
under 250 words even when several situational fields apply.

NETW employees include: <first names of your team>.
```

Four things in there are worth lifting even if you summarize something entirely different.

**Letting the content choose the shape.** The first version demanded Problem, Solution and Action items on every call. Run that over a thirty second call where a colleague asks whether you want coffee and you get an invented problem, an invented solution and a confident action item. Telling the model to read first and decide the shape afterwards fixed more bad output than any other single change.

**Closing the loophole the escape hatch opened.** Once the model could skip the full structure for "personal" calls, it started classifying any informally phrased call as personal, and jokey internal calls about real work vanished into one-liners. The paragraph spelling out that tone is not the test, content is, exists entirely because of that.

**Distinguishing required from best-effort fields.** Core fields always appear, and say "not applicable" when empty. Situational fields are omitted entirely when empty. Without that distinction you either get a wall of "not applicable" lines or you get fields quietly dropped when they did contain something.

**Telling it explicitly not to guess.** `Never guess a name from a phone number, extension, or the call metadata block alone` is there because it did exactly that, confidently and wrongly.

### Step 12: Send it

An n8n **Send Email** node over SMTP, to a shared mailbox. Subject line:

{% raw %}
```
Call summary: {{ $('Build Transcript Document').item.json.call_from }} -> {{ $('Build Transcript Document').item.json.call_to }} ({{ $('Build Transcript Document').item.json.time_start }})
```
{% endraw %}

The body is the summary first, then the full transcript underneath, so whoever reads it can check what was actually said without opening anything else:

{% raw %}
```
Samenvatting:

{{ $json.choices[0].message.content }}

Transcriptie:

{{ $('Build Transcript Document').item.json.transcript_document }}
```
{% endraw %}

Plain SMTP against your existing mail host avoids an app registration purely to send one message. Turn off n8n's "append attribution" option unless you enjoy explaining the footer.

### Step 13: Catch the failures

Create a second workflow with an **Error Trigger** node and set it as the main workflow's error workflow under **Settings**. Mine pushes a short message to [Pushover](https://pushover.net/) containing the execution URL, so a failed call arrives as a notification I can tap, land on the failed execution, see which node broke and retry it.

That retry button is most of the reason a visual workflow beats a Function for this job. When a call fails you want to see the step that broke and run it again, not read application logs.

## Challenges & Learnings

🎙️ **Whisper cannot read a telephone call**: This is the one that cost real time, and the failure mode is genuinely misleading. The obvious design is to send the recording to the OpenAI-compatible `/audio/transcriptions` endpoint. I tried `whisper-1`, which returned `unsupported_format`. I relabelled the MIME type and renamed the file, same error. I switched to `gpt-4o-transcribe-diarize`, a different model entirely, and got a byte-identical error message. Three attempts with the same result on two different models ruled out labelling and pointed at the audio itself. PBX recordings are telephony-encoded, typically G.711 A-law or mu-law in a WAV container, and the whisper family simply does not accept that codec. Azure's Fast Transcription API explicitly documents A-law and mu-law in WAV as supported inputs, so the fix was not transcoding, it was using the right API. If you are wiring up any PBX to any speech service, check the codec list first and save yourself the afternoon.

🏷️ **`audio/x-wav` is a real MIME type and nothing likes it**: Yeastar's download server declares `Content-Type: audio/x-wav`, a legacy string that predates the standard `audio/wav`. n8n faithfully carries it through as the multipart content type on the next request, and an allowlist that only knows `audio/wav` rejects the upload. Three lines of JavaScript relabel it without touching a byte. Worth remembering generally: when a file upload is rejected and the bytes are definitely fine, look at what you claimed the bytes were.

☎️ **A ring group fires the webhook twice**: One inbound call into a ring group produces two CDR events, one for the group and one for the agent who answered. Without a filter every incoming call is transcribed, summarized and mailed twice, which is both annoying and exactly double the Azure bill. Dropping calls whose destination falls in the ring group extension range fixes it, because the agent's record is the one carrying the recording anyway.

🔑 **The undocumented token on the download call**: Yeastar's recording download is two calls, one for a temporary URL and one for the file. The documentation does not say the second call needs the access token. It does. Leave it off and you get `errcode 10004 TOKEN EXPIRED`, which sends you off debugging token lifetimes when the actual problem is that you never sent one.

🗣️ **Diarization knows there are two voices, not who they are**: The recording is a single mixed stream, so Azure separates the speakers but has no idea which one works for you. It labels them `Speaker 0` and `Speaker 1` and that is all it will ever tell you. Assuming the first speaker is the employee is roughly right for inbound calls, because we answer with a greeting, and exactly wrong for outbound. The judgement has to be made from context, which is why the prompt asks the model to work it out from who introduces themselves, and forbids it from inferring anything from the metadata block.

🧍 **The pipeline has no idea who your customers are**: This is the real limitation and it is not a transcription problem. The workflow sees a phone number and an audio file. It has no connection to our client base, so when the summary names a company it is working purely from what was said out loud, and company names it has never encountered come out as phonetic guesses. The prompt mitigates this by forbidding guesses from phone numbers, but mitigation is not a fix. The fix is better input: look the caller ID up against the client database first and hand the model the real company name as a fact rather than asking it to infer one. That is the next thing to build.

📝 **We went with free text, and that is a real trade-off**: An earlier version returned strict JSON with fixed fields. Free text won because the summaries read better and because forcing a schema onto calls of wildly different kinds produced filler. The cost is that the summary is now prose, so the obvious next steps, opening a ticket or alerting on negative sentiment, need parsing or a second model call. If I were starting again knowing a ticket integration was coming, I would use structured output with optional fields and render the email from it. The labeled line format in the prompt is a deliberate hedge: it is prose a human reads easily and a parser can still cope with.

🔁 **The token limit we have not hit yet**: We fetch a fresh Yeastar token per call, and the platform allows eight valid tokens at a time. Our volume stays under that, so it works, but it is a latent failure rather than a design. Writing this guide is what made me notice it properly, which is an argument for writing guides.

💶 **Almost all of the cost is transcription, not the AI**: I have not run this long enough to have a real invoice, so what follows is an estimate from list prices rather than a measurement. Azure bills speech per hour of audio, in the order of €1, while the chat model bills per token. A five minute Dutch call is roughly 1,300 tokens of transcript plus about 300 tokens of summary coming back, which is well under a cent on `gpt-4.1`. So the call costs roughly **€0.08 for the transcription and under €0.01 for the summary**, around €0.09 for five minutes, or a bit under two cents a minute. A thousand call minutes a month lands near €17. The useful part of that arithmetic is the ratio: speech is around nine tenths of the bill, so switching to a cheaper chat model barely moves the total. If this ever needs to be cheaper, the levers are filtering more calls out before transcription and Azure's commitment tiers, not the model.

⚖️ **Privacy is a design decision, not a footnote**: The transcript ends up in a mailbox, which means a verbatim record of every customer conversation now lives under whatever retention policy that mailbox happens to have. That is a decision worth making deliberately, in advance, with the people whose calls these are. Building it is the easy part.

## Troubleshooting

If nothing arrives in n8n at all, check that the workflow is **activated**. While the editor is open only the `/webhook-test/` path responds, which catches everyone at least once. After that, verify the URL in Yeastar and confirm event 30012 is subscribed.

If every call is processed twice, your ring group filter is missing or its extension range is wrong. Look at `call_to` on both executions and you will see the group and the agent.

If the download returns `errcode 10004 TOKEN EXPIRED` immediately after a successful token request, you are missing the `access_token` query parameter on the second download call, not on the first.

If the recording downloads but Azure rejects it with `unsupported_format`, you are almost certainly on a whisper-family model rather than Fast Transcription. Relabelling the MIME type will not save you; the codec is the problem.

If Azure Speech returns a 401, the key is wrong or the host in the URL does not match the resource. A 400 instead usually means `definition` was not sent as its own form field, or the audio went up as text rather than binary.

If the chat call returns a 404, the deployment name in the URL does not match the deployment in Foundry. It is almost never the key. If it returns an API version error, open the deployment's code sample in Foundry and copy whatever version it shows rather than trusting a value from a guide, including this one.

If the transcript is empty, the recording is silent or extremely short. Check the recording settings on the extension.

## Next Steps

- Cache the Yeastar token in workflow static data, before call volume makes the eight-token limit a real outage
- Enrich the call before summarizing by looking the caller ID up against the client database, so the model is told which company called instead of guessing it from audio
- Create or update a HaloPSA ticket straight from the summary, with the transcript attached
- Send a Pushover or Teams alert for any call where the summary reports an open action or explicit dissatisfaction
- Move the Azure key and Yeastar secret into n8n credentials if they are still sitting in node parameters, and make rotating them part of the routine

## Conclusion

Yeastar already does the genuinely hard part. It records the call and tells you, through a webhook, exactly when it finished and where the audio lives. Everything after that is four API calls in a row, which is precisely what n8n is good at.

The result is a written summary in the right inbox about a minute after the call ends, for something in the order of two cents per call. We started building because we could not buy the vendor's version, which is a slightly embarrassing origin story, and it turned out to be the better outcome: the pipeline does what we want rather than what the add-on offers.

If you take one thing from this, make it the codec. Everything else here is plumbing you would have worked out eventually. The fact that a PBX recording is a format most speech models silently refuse is the kind of detail that costs an afternoon and appears in nobody's tutorial.

💡 Want to know more about building AI pipelines on top of your existing telephony, or need help working out whether to build or buy? Feel free to reach out!
