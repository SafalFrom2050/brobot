# Jev AI evaluation plan

Status: research and future experiment, 2026-09-24. No Jev integration or
Brobot firmware exists in this repository yet.

## Decision

Evaluate TypeSafe AI's Jev on a companion computer after Brobot has proven
safe manual driving and has a stable, authenticated companion API. Jev is a
candidate for narrow decisions from text or structured state. It is not part
of the Pico W safety loop, and its availability must never be required to stop
the motors or keep the existing smart-home functions working.

The immediate milestone remains the one in [PROJECT_PLAN.md](PROJECT_PLAN.md):
drive safely over the LAN while preserving room-controller behavior.

## What Jev offers and its current limits

Jev accepts a text string or JSON state and answers predefined Choice, Score,
or yes/no (Noul) questions. Choice and Score return distributions and a
confidence value. This fits classification, routing, and selection from a
closed set of actions. It does not generate conversation, plans, or motor
trajectories. Its current API does not accept images, audio, or video; another
component must first transcribe speech or interpret camera data. Jev is a
hosted API in early access, so published latency is not a local, worst-case
control deadline. TypeSafe documents weaknesses in numeric reasoning,
indirection, irrelevant context, and adversarial input. A typed answer can
still be the wrong decision.

## First experiment: smart-home request routing

Build a small companion-side prototype using recorded text requests. Ask Jev
for narrow choices such as request domain (`ac`, `tv`, `lights`, `expression`,
`unclear`) and action within the selected domain. Include `unclear` or an
equivalent abstain option. Keep device availability, allowed actions,
authentication, and any confirmation policy in ordinary code. Reuse the
existing smart-home REST API; do not change its local behavior to depend on
Jev. If the response is missing, late, uncertain, or invalid, do nothing and
ask for clarification. Thresholds must be selected from measured Brobot
examples rather than copied from a vendor demo.

Potential later low-stakes use: choose one of Brobot's semantic matrix emotions
or decorative LED moods from an approved list. The Pico W retains its emotion
priority rules; fault, battery, disconnection, and movement indicators always
override a model request. Keep interaction rules outside the BLE transport,
as described in [LOTUS_LANTERN_LED_STRIP_RESEARCH.md](LOTUS_LANTERN_LED_STRIP_RESEARCH.md).

## Could this support self-driving later?

Potentially, at the **high-level behavior** layer. A companion could convert
camera and navigation data into a compact state such as detected doorway,
person nearby, localization quality, route progress, and whether a passage is
clear. Jev could then select a bounded behavior such as `wait`, `ask_user`,
`continue_to_waypoint`, or `abort_task`. The companion would validate that
choice against its route and sensor state before sending short-lived commands
through the normal Brobot API.

Jev alone cannot make Brobot self-driving. Its current text-only input needs a
separate perception stack; Brobot would also need localization, mapping or
route following, motion feedback, local obstacle sensing, and extensive
physical testing. The Pico W must always retain motor limits, command expiry,
disconnect handling, emergency stop, and fault response. A local obstacle
stop should work even if the companion, Wi-Fi, cloud API, or Jev fails. Do not
send raw `linear`/`angular` values from Jev directly to motors or rely on a
model confidence score as proof that a path is safe.

The practical autonomy sequence is:

1. Prove Phase 0-1 manual control and all stop paths in the project plan.
2. Add the authenticated companion API and local collision sensing.
3. Build perception, localization, and motion feedback without Jev in the loop.
4. Test Jev on recorded structured states, then in shadow mode while a human
   drives. Log model version, inputs, outputs, latency, and disagreements.
5. Only consider supervised, low-speed, bounded behavior selection after the
   deterministic navigation and safety layers pass their own tests.

## Evaluation gates

- Obtain API access and keep the key on the companion, never in Pico W firmware
  or browser code. Review whether cloud processing of room observations is
  acceptable before sending any personal data.
- Create a representative labeled set of Brobot requests or structured
  observations, including ambiguous wording, malformed input, injected
  instructions, and missing sensor fields.
- Measure accuracy, false actions, abstentions, latency distribution, timeout
  behavior, and cost. Check results separately for any non-English language
  used in the robot; TypeSafe says English is currently strongest.
- Pin the model version for a serious trial and re-evaluate on upgrades.
- Demonstrate that API errors, rate limits, network loss, and companion crashes
  leave the Pico W in its existing safe state.

## Sources checked 2026-09-24

- [TypeSafe announcement](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
- [State and supported modalities](https://docs.typesafe.ai/concepts/state)
- [Choice response and confidence](https://docs.typesafe.ai/primitives/choice)
- [Confidence guidance](https://docs.typesafe.ai/confidence)
- [Model versions, pricing, and limits](https://docs.typesafe.ai/models)
- [Known Jev 1.13 limitations](https://docs.typesafe.ai/model-jaggedness/jev-1.13)
- [TypeSafe smart-home demo](https://docs.typesafe.ai/demos/smart-home)
