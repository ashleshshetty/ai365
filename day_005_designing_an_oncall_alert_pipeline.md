# Day 005: Designing an On-Call Alert Pipeline for a Time-Critical Feed
**Date:** 2026-10-06 | **Category:** #Alerting #Telegram #MTProto #EdgeVLM #SystemDesign | **Environment:** Apple M1 Pro macOS (design), Lenovo ThinkPad T470 Windows 10 (intended host)

## Day Map

1. Started from a personal problem: a handful of busy public Telegram groups occasionally carry a time-critical availability sighting, and missing one while asleep has a real cost. Wanted a pager, not a notification.
2. Researching the delivery end first turned out to be the right order — discovering that one notification service can *retract* an alert changed the whole architecture, because it meant classification no longer had to happen before notification.
3. Reading actual screenshots of the groups then reframed the filtering problem: the discriminator is not keywords but grammatical intent, and the dominant message class is trivially removable without any model.
4. Checked whether the intended host could run a local vision model. It can, but slowly, which the architecture from step 2 already tolerates.
5. Closed by walking the design one layer at a time and asking what money buys at each. The answer was "nothing" at five layers out of six, which is what made the sixth easy to justify.

No code written today. This was a design day, so the evidence below is research findings and hardware readings rather than a build.

---

## Topic 1 — Alert-First-Then-Retract as an Alerting Architecture

**Bottom line:** when a missed event is catastrophic and a false alarm is merely annoying, the conventional classify-then-notify pipeline is backwards; notify immediately and use a retraction API to cancel the ones that turn out to be noise.

**What I learned**

* **An alerting API that supports cancellation inverts the pipeline.** Pushover accepts an arbitrary `tags` value on an emergency notification and exposes a `cancel_by_tag` call that kills all outstanding receipts carrying that tag. That one feature is what makes "fire first, judge later" viable: a slow classifier running *after* the page can still stop the repeats, so the worst case for a false positive is roughly one retry interval of noise instead of a missed event.
* **Asymmetric error costs should change pipeline order, not just thresholds.** The instinct when facing a noisy feed is to tune the filter harder. But when false-negative cost vastly exceeds false-positive cost, the better move is to take the classifier off the critical path entirely and let it act as an after-the-fact retractor. Threshold tuning trades one error for the other; reordering reduces latency without trading anything.
* **In a high-volume status feed, the negative filter does almost all the work.** The dominant message in these groups is a terse "NA" status posted roughly once a minute. Matching *that* class is a plain regex with no model involved, and it removes the overwhelming majority of traffic for free — which is what keeps a slow local LLM viable, since it only ever sees the small ambiguous residue.
* **The real signal was an intent distinction, not a keyword one.** "I saw a slot available but when I logged in it was gone" is actionable (it means availability is moving right now); "when will slots be available?" is worthless. Both contain every positive keyword. The discriminator is grammatical mood — a first-person report of an observation versus an interrogative — which is why keyword matching alone cannot work on discussion-style groups and an intent classifier can.
* **A home-hosted always-on job needs an externally-held dead-man's switch.** The failure mode of a machine in your house is not complexity, it is silent death: an unattended Windows reboot, a dropped wifi link, a power blip. Monitoring that from the same machine is circular. A heartbeat pushed outward to a free external checker every few minutes inverts the dependency, so the thing that notices the watcher died is not itself the watcher.

**Gotchas**

* Pushover emergency priority has hard limits worth knowing before designing around it: `retry` minimum 30 seconds, `expire` maximum 10,800 seconds, and a cap of 50 retries regardless of what `expire` says.

**Detail**

Emergency-priority send shape, per the [Pushover API docs](https://pushover.net/api/):

```
priority=2&retry=30&expire=1800&tags=<correlation-id>
```

Repeats every 30 seconds for up to 30 minutes until acknowledged. The `tags` value is what a later `cancel_by_tag` call uses to stop it.

---

## Topic 2 — Telegram Automation via MTProto (User Session, Not Bot API)

**Bottom line:** reading ordinary groups you belong to requires a user-account MTProto session rather than a bot, and passive update delivery is unreliable enough that polling has to be the primary mechanism.

**What I learned**

* **The Bot API is the wrong tool for reading groups you merely belong to.** A bot can only read a chat it administers, which you will not be for a public community group. Reading requires MTProto with a user session — the same credentials path as the official clients, via an `api_id`/`api_hash` registered at my.telegram.org.
* **Telegram does not reliably push updates for chats you have not joined, and is flaky even for some you have.** `updates.getChannelDifference` returns `CHANNEL_PRIVATE` with the explicit message "You haven't joined this channel/supergroup". Separately, a long-running Telethon issue collects reports of passive new-message events simply not firing for particular channels, with one practitioner putting the trigger rate around 20%. The practical consequence: an event-driven watcher can go silently deaf, which is the exact failure this project cannot tolerate.
* **The robust pattern is polling with a tracked message ID, with events as an opportunistic fast path.** Poll each chat on a short interval, fetch by `offset_id`, keep the last-seen ID per chat, and deduplicate against any events that do arrive. The same issue thread reports this running at 30-second intervals across five clients for a month with zero missed messages and no complaints from Telegram. The cost is bounded added latency; the gain is that it degrades visibly rather than silently.
* **Forum topics are implemented as message threads, not as separate chats.** There is no "topic" field. For a top-level post in a topic the topic ID is `reply_to.reply_to_msg_id`; for a reply within it, the topic ID moves to `reply_to.reply_to_top_id` and `reply_to_msg_id` points at the parent message instead. Filtering one topic out of a busy forum means checking both.

**Gotchas**

* A `web.telegram.org` link pasted into any agent or web tool returns nothing — the content exists only inside an authenticated session, so fetching the URL yields an empty page rather than the conversation.
* Telegram Desktop has a per-chat "Export chat history" that writes JSON including images and requires no API credentials at all. For building a corpus to write and test rules against, it is far faster than standing up API access first.

---

## Topic 3 — Vision-Language Models on 2017-Era Laptop Silicon

**Bottom line:** a small VLM will run on an eight-year-old dual-core with no GPU, but at minutes per image — fine as enrichment, disqualifying as a gate.

**What I learned**

* **Small-VLM CPU inference is measured in minutes, not seconds.** Published edge benchmarks put Qwen2.5-VL-3B OCR at 145 seconds on a Raspberry Pi 5 and 302 seconds on a Pi 4. An i7-7500U is in that general class for this workload, so a 3B vision model on the intended host means minutes per image. This is the number that retroactively justified the alert-first architecture from Topic 1 — any design placing this model before the notification would have been unusable.
* **There is a real size ladder below 3B worth knowing.** Moondream ships a 0.5B int8 build at roughly 593 MB on disk and under 1 GB resident, trading OCR accuracy for speed; the 2B int8 build is about 1.7 GB on disk and 2.6 GB resident. When the model's job is "confirm and extract" rather than "decide", the small end of that ladder is often sufficient.

**Evidence**

Intended host, read from System Information:

*Output Verified:* Lenovo ThinkPad T470 (20HDCTO1US), Intel Core i7-7500U @ 2.70 GHz, 16.0 GB installed / 15.9 GB total physical memory, Windows 10 Pro 10.0.18363, no discrete GPU, Hyper-V virtualization not enabled.

Toolchain check on the Mac used for design:

```bash
which ollama && ollama list
python3 -V
which tesseract
```

*Output Verified:* `ollama not found`, `Python 3.14.3`, `tesseract not found` — neither the runtime nor an OCR fallback is installed anywhere yet.

---

## Topic 4 — The System Design, and the Free-vs-Paid Choice at Each Layer

**Bottom line:** every layer of this system has a working free option, and the total cost of the chosen design is five dollars once — but that five dollars buys the one capability the architecture actually depends on, and nothing else was worth paying for.

**What I learned**

* **Free tiers here are genuinely sufficient almost everywhere, so the paid decision should be isolated to one layer.** Ingest, hosting, text filtering, image understanding and liveness monitoring all have free options good enough for this workload. Going down the list layer by layer and asking "what does money buy here?" produced "nothing" five times out of six — which is the useful outcome of the exercise, because it concentrates the one real spend somewhere it can be justified.
* **"Bypasses Do Not Disturb" is free; "repeats until acknowledged, and can be retracted" is not.** Apple's Critical Alerts is an entitlement granted per-app, and both Pushover and the free open-source ntfy now hold it, so breaking through silent mode costs nothing. What ntfy has no equivalent for is the repeat-until-acknowledged loop and `cancel_by_tag`. Since the whole design in Topic 1 rests on retraction, that is the specific thing the five dollars buys.
* **A free tier's limit should be read against your actual event rate, not its headline.** PagerDuty's free plan is the strongest-looking option on paper — 5 users, on-call schedules, escalation policies, unlimited push, and 100 phone or SMS notifications a month. But 100 phone notifications a month is about three a day, and a burst period could exhaust that in a single night, after which the block does not clear on its own. A limit that is generous for an unlucky week can be the wrong shape for a bursty one.
* **The weakest-looking host won because the hosting layer is not where the risk lives.** An eight-year-old laptop on home wifi looks worse than a cloud VM, but the thing that makes a home host dangerous is undetected death, and that is fixed by the external heartbeat from Topic 1 rather than by better hardware. Once the failure is detectable, the remaining difference between a free home machine and a paid VM is small enough not to pay for.

**Detail**

The layer-by-layer option space, with what was chosen and why:

| Layer | Free options | Paid options | Chosen | Why |
|---|---|---|---|---|
| Ingest | Bot API; MTProto user session; Desktop export | Third-party monitoring services | MTProto user session, plus Desktop export for the corpus | Bot API cannot read a group it does not administer; export needs no credentials and unblocks rule-writing immediately |
| Host | Mac; home laptop; Raspberry Pi; Oracle Cloud free tier | Hetzner ~$4-5/mo; Fly.io | Home laptop | Mac sleeps; cloud free tier has signup friction; the home machine's real risk is handled at the liveness layer instead |
| Text filtering | Regex; local LLM via Ollama | Cloud LLM API, fractions of a cent per call | Regex first, local LLM on the residue | Regex removes the dominant message class outright, so the slow local model sees little enough traffic to keep up |
| Image understanding | Alert on any image; Tesseract OCR; local small VLM | Cloud vision API, cents per day | Alert on any image, local VLM as enrichment | The alert does not wait on the model, so the model being slow stops mattering |
| Delivery | ntfy, incl. iOS Critical Alerts; PagerDuty free tier | Pushover $5 one-time; Twilio voice ~$1/mo plus per-call | Pushover | Only option combining repeat-until-acknowledged with a retraction API |
| Liveness | healthchecks.io free tier; UptimeRobot | Paid monitoring | healthchecks.io free tier | A few pings a minute sits far inside the free tier |

Deferred rather than rejected: a Twilio voice call as a second delivery path. It is worth adding only if a push notification ever fails to arrive, because its value is that it rides the cellular voice network instead of push infrastructure — an independent road, not a better one.

---

## Review Questions

1. Why does an alerting API that supports cancellation let you remove the classifier from the critical path, and when is that worth doing?
2. What does Telegram return when you request updates for a channel you can read but have not joined, and what does that force in the watcher design?
3. Given a post containing the words "slot" and "available", what property of the sentence determines whether it is worth being woken for?
4. Where does a forum topic's ID live on a message, and why are there two different fields to check?
5. What are the three hard numeric limits on a Pushover emergency notification?
6. Roughly how long does a 3B vision model take per image on CPU-only laptop silicon, and what does that rule out architecturally?
7. Given that a free notification service can also bypass Do Not Disturb on iOS, what exactly is the paid one buying?
8. Why was an eight-year-old laptop on home wifi an acceptable host, when a cloud VM was available for a few dollars a month?

## Next Steps

* **Highest priority:** export chat history to JSON from Telegram Desktop for all four groups, covering the last seven days plus the April window when availability was actually moving. April is the only period containing real positives, so it is the only way to backtest the rules rather than guess at them.
* Settle the one open design question: during a burst of sightings, whether the first page should open a quiet window for subsequent ones or whether every sighting should page independently. Deferred until the April data shows the actual event rate.
* Confirm from the exported corpus what a genuine positive post looks like in the terse status feeds — the "NA" negative convention is confirmed, its opposite is not.
* Install Ollama on the ThinkPad and time a small VLM on a real screenshot from the corpus, to replace the Pi-derived estimate with a measured number on the actual host.
