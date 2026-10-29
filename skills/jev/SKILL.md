---
name: jev
description: Write and improve programs that call Jev, TypeSafe's System One model. Use when designing TypeSafe questions (Choice, Score, Noul, Box), structuring state, sending images, composing answers in code, setting confidence thresholds, or diagnosing a Jev question that answers wrong or with low confidence.
---

# Writing and improving Jev programs

Jev is a judgment model. It reads one `state`, answers every question in the request independently and in parallel, and returns a probability distribution over the answers you defined. It does not reason in steps and it does not generate text. Code owns the control flow, the arithmetic, and the policy. Jev owns the snap judgments.

A good Jev question is one a knowledgeable person answers in a second, given the right context. "Does this message convey urgency?" fits. "Analyze this message and decide what to do" does not fit. Break that task into small questions and compose the answers in code.

## Models

This guidance applies to `jev-1.14` and `jev-1.13`. The two models share the same price and context limits.

| Name | Points to | Use it when |
| --- | --- | --- |
| `jev-latest` | `jev-1.14.0` | You want the newest stable release. The SDKs use it by default. |
| `jev-preview` | `jev-1.14.0` | You want to try a preview build. It moves ahead of `jev-latest` when one exists. |
| `jev-1.14.0`, `jev-1.13.0` | itself | You have tuned thresholds and want answers that do not change. |

- An alias moves when a new release ships, so answers can change with no change to your code. Pin a versioned ID once you have tuned thresholds on labeled data.
- Pass `model=` on every call, or set it on the client, so the code states which model it expects.
- Log `response.model`. It holds the versioned ID that answered.
- Retest your labeled examples before you move to a new version. TypeSafe publishes a jaggedness page per version. The current page covers `jev-1.13`, and many of its limits may change in later versions.
- English gets the best accuracy. Jev accepts other languages, including CJK scripts, at lower accuracy. Test on your own content and lean more on confidence for non-English input.

## Workflow

1. List the decisions your code must make. Write each as a branch, a threshold, or a ranking.
2. Write one question per judgment. Split any question that weighs two properties.
3. Pick the primitive whose answer your code acts on directly.
4. Build the smallest state that answers every question. Compute in code whatever code can compute.
5. Put every question that shares the state into one request, including questions that matter for only some inputs.
6. Combine the answers in code: branches, weights, and confidence gates.
7. Test against labeled examples. Read `probabilities` on the misses, then revise one or two questions at a time.

Keep the questions and the threshold constants together in one file. People review those first, and agents write weak questions, so expect to edit them by hand.

## Choose the primitive

| Primitive | Use it when | Returns | Code acts on it with |
| --- | --- | --- | --- |
| Choice | The answer is one of a known set with no order | `choice`, `probabilities`, `confidence` | a branch per option |
| Score | The answer is a position on a spectrum you can describe in steps | `score`, `probabilities`, `confidence`, `legend` | a threshold, a rank, or a weight |
| Noul | The answer is a clean yes or no and the probability is the signal | `noul`, from 0 to 1 | an `if` on a threshold, or a sort |
| Box | You need where something is in an image | `box` with four edges, `confidence` | a crop, a drawing, or an overlap test |

Box and images are experimental. See [Images and Box](#images-and-box).

- A Choice always picks an option, because its probabilities sum to 1. When no option may fit, ask a companion Noul in the same request, e.g., "Does `message` name any of `our_products`?" Read the Choice only when the Noul says yes. The docs say an `other` or `none of the above` option often works poorly. Use one only when it tests well on your data.
- Give a Choice the full list of options. A Choice accepts up to 255 options, and each extra option costs a few tokens. Split larger sets into a taxonomy walk or a shortlist.
- A Noul of 0.5 means Jev is unsure. It does not mean "medium". Use a Score to measure a degree.
- A Noul needs a crisp condition. "Is this candidate strong in Python?" is vague. "Does the resume state that the candidate used Python at work?" is crisp.
- Use a Choice or several Nouls when the answer has no in-between.

## Write the instructions

- State the exact condition. Jev answers the words you wrote. It reads scoping words, negations, and implied conditions at face value.
- Ask one property per question. Hidden second judgments lower accuracy and confidence.
- Name the part of the state the question judges, with a backticked path: `` `ticket.messages[0].text` ``.
- Write directly. Avoid double negatives, a property of a property, and any question that needs several hops.
- Name the narrowest fact that decides the answer. "Does this line pick up mid-sentence?" works better than "Is this line in the same paragraph?"
- Name the idea, not the parameter. "Which resolution?" gives Jev nothing to match. Describe what the option means.
- Keep numerals that stand for levels out of the instructions. "Rate from 0 to 2" gives Jev nothing to match.
- Write the full question in `instructions`. The question ID never reaches the model.
- Keep decision policy out of the question. "A shared address cannot override a name conflict" belongs in code.
- When you explain what you really meant after a wrong answer, that explanation is the missing half of the instruction. Add it.

Instructions accept a string, an object, or an array. Use an object when the question has labeled parts or needs supporting data. Pass a schema, taxonomy, or row as JSON. Do not serialize it into a string template.

```json
"instructions": {
  "question": "Does the `message` ask the recipient to disclose a sensitive credential?",
  "inspect": "message",
  "focus": "Look for a request to send the credential itself, not a request to change or reset it."
}
```

Useful keys from the docs: `question`, `focus`, `inspect`, `note`, `compare` (a list of state paths), and `field` (a `name`, `type`, `unit`, `description` record that several questions share). None of these keys are reserved. Jev reads the key names along with the values, so pick short names that label what follows.

## Write the criteria

Treat criteria as an extension of the instruction. The two must ask for the same thing, in the same direction. A Noul whose `true` side describes "no" performs worse.

Use the same field names on every option or level, so Jev compares like with like.

**Choice.** Map each option to a description. Use `None` (or `null`) when an option name is clear on its own. Make the descriptions contrastive when options sit close together:

```json
"billing": {
  "what": "Charges, invoices, refunds, or subscriptions",
  "not_for": "Order tracking or account access",
  "examples": ["I was charged twice", "Where is my refund?"]
}
```

Jev can lean toward the first option. Reorder the options on a few test inputs and check that the answer stays the same.

**Score.** List the levels from low to high. Use two to ten levels, and only as many as you can describe distinctly. The API rejects more than ten.

- Describe situations. "Broken feature, but a workaround exists" works. "Moderately severe" does not.
- Make each level stand alone. Jev judges each level separately and sees neither its number nor its neighbors. "Worse than the previous level" means nothing to it.
- Keep each Score to one dimension. "Punctual and smart and experienced" measures three things.
- Give a rare extreme its own level when your code must treat it differently.
- A level may be an object: `{"what": "One change, clearly stated", "examples": ["A single fix or feature"]}`.
- Write examples that look like your real inputs. An example unrelated to your inputs changes nothing.

**Noul.** Criteria are optional. Try each Noul with and without criteria and keep the version that does better on your data. Add `true` and `false` sides when the boundary is subtle. Each side can be a string, e.g., `NoulCriteria(true="Mentions a prior ticket", false="No sign of earlier contact")`, or an object with `what` and `examples`. Put the neighboring case in the description of the side it belongs to.

- Phrase the Noul so that a high value means yes. "Does the message contain personal data?" is clear. "Is the message free of personal data?" inverts the meaning, and later code reads it backwards.
- A statement works as well as a question, e.g., "The customer is requesting a refund."

**Examples.** Write short concrete instances, such as "I was charged twice". Do not write a description of an instance, such as "a message about a billing problem".

## Build the state

- Send only the fields the questions need. Unrelated detail lowers accuracy, and a large state hides which input caused a wrong answer.
- Retrieve and filter in code first. When code cannot filter, ask a relevance Noul per passage and keep the passages that pass.
- Keep the state structured so questions can point into it by path. A plain string is fine for a single piece of text.
- To ask about many lines or items, prefix each with a short ID, e.g., `L052| ...`, and use the IDs as Choice options or in question text.
- Convert numeric encodings to words before sending them. Send a color name in place of a hex value. Send a computed number or a named bucket in place of raw figures.
- Compute date order, duration, windows, counts, and sums in code. Send the result.
- Treat text in the state as able to steer the answer. Jev does not treat state as hostile. State in the criteria what counts, and test injected and self-describing content before deployment.

Limits:

- The state and all questions share 64k tokens.
- The state plus any single question must fit in 32k tokens.
- There is no limit on the number of questions. Only the token limits apply.
- The `TypeSafe-Danger-Zone-Long-Context: true` header raises both limits to 262k tokens. Long requests are slower and less accurate, so send the header only on requests that need it.
- The request body can be at most 32 MiB, with at most 64 levels of nesting. Send identifiers and large exact numbers as strings.

## Compose the answers in code

**Speculative fan-out.** Ask every question your code might need in one request, including questions that matter only on some branches. Questions run in parallel, so extra questions add little latency and few tokens. Code ignores the answers it does not need. The docs measured one 13-question request at about 12 times cheaper and 10 times faster than 13 separate calls, with the same answers.

**Second requests.** Make a second request only when code cannot build it without the first answer, e.g., when the answer decides what data to fetch or which options the next question offers. Questions in one request never see each other's answers.

**Confidence-gated routing.** The answer says what. Confidence says whether to act. Set a floor below which no action runs, then set a threshold per action that rises with the cost of a wrong call. The docs use Choice confidence floors of 0.5 to 0.6, and 0.85 to 0.9 for high-stakes actions. Start conservative and tune on your data. The three standard paths are:

- act
- confirm or flag
- hand off

**Noul bands.** Start with 0.7 for yes and 0.3 for no, and send values in between to a person or a slower path. Answers vary a little between runs, so a single cutoff such as 0.5 can flip on a repeat call. Raise the yes threshold when a false yes is costly. Lower it when a missed yes is costly. A screening gate that must catch most cases can sit well below 0.5.

**Composite scoring.** Split a complex judgment into one Score per dimension. Normalize each score by `len(criteria) - 1`, then combine with weights in code. Change a weight when priorities shift. Do not rewrite a question to change policy.

**Any red flag.** When one strong flag should stop the action, combine the flag Nouls with `max`. An average hides a single confident flag.

**Weakest link.** When an answer depends on several questions, e.g., a date built from parts, use the lowest confidence among them as the confidence of the whole.

**Intent routing.** Classify with a Choice, and add a complexity Score beside it. Route each intent to deterministic code, a specialist LLM, or a person. Send low-confidence classifications to a person.

**Taxonomy walk.** Ask one Choice per tree level and walk the tree in code. Give each option its subtree as the criteria value, so Jev sees what lives under a branch. Trim large subtrees to direct children and a sample of leaves. When the probabilities are close, keep the best few paths alive and rank them by the product or geometric mean of their probabilities. When confidence at a deep level is low, report the parent category in place of handing off.

**Shortlist and re-check.** Rank a large set with one Choice. Then send a second request on the top few candidates with their full descriptions, plus one Noul per candidate so code can reject all of them.

**Re-ranking.** Ask one Noul per query and candidate pair in one request, then sort the candidates by `noul`.

**Line search.** Put ID-prefixed lines in the state and ask a Choice over the line IDs. Ask a Noul in the same request that checks whether the document answers the query at all.

**Counting.** Jev does not count. Ask one Noul per item in a single request, then sum the answers that pass your threshold.

**Dates.** Extract each date part with a Choice over enumerated options, including a "not stated" option. Assemble and compare the date in code.

**Extraction.** Generate candidates with a regex or a generative model. Ask Jev to pick among them with a Choice, or to verify one with a Noul. For an optional value, ask a Noul whether the input states it at all, and use the default when it does not.

**Features for a model.** Use Jev's probabilities as input features for a classical model, e.g., gradient-boosted trees. A Score gives two features: the score and its confidence.

```python
from typesafe_sdk import Choice, Noul, Score, TypeSafeClient

MODEL = "jev-1.14.0"
SEVERITY = [
    "Cosmetic; no impact to functionality",
    "Broken or degraded feature; a workaround exists",
    "Blocking issue; no workaround exists",
]
CATEGORY_FLOOR = 0.6
REFUND_YES = 0.7

with TypeSafeClient(model=MODEL) as client:
    response = client.system_one(
        state={"ticket": ticket_text},
        questions={
            "category": Choice(
                instructions="Which category fits the main request in `ticket`?",
                criteria={
                    "bug_report": "Something is broken or producing errors",
                    "billing": "Charges, invoices, refunds, or subscriptions",
                    "other": "Anything else",
                },
            ),
            # Speculative: only used when the ticket is a bug report.
            "severity": Score(instructions="How severe is the issue reported in `ticket`?", criteria=SEVERITY),
            "refund_requested": Noul(instructions="Does `ticket` explicitly ask for a refund or credit?"),
        },
    )

category = response.choices["category"]
severity = response.scores["severity"]
if category.confidence < CATEGORY_FLOOR:
    route_to_human(ticket_id)
elif category.choice == "bug_report":
    normalized = severity.score / (len(SEVERITY) - 1)
    if normalized > 0.75 and severity.confidence > 0.5:
        escalate(ticket_id)
    else:
        add_to_backlog(ticket_id)
elif category.choice == "billing" and response.nouls["refund_requested"].noul > REFUND_YES:
    route_to_billing(ticket_id, refund_likely=True)
```

## SDK notes

- Python: `TypeSafeClient` and `AsyncTypeSafeClient` from `typesafe_sdk`. JavaScript: `TypeSafeClient` with `choice()`, `noul()`, `score()`, and `box()` from `@typesafe-ai/sdk`.
- `response.answers[id]` holds every answer. The typed accessors `response.nouls`, `response.choices`, `response.scores`, and `response.boxes` return answers of one type.
- `response.usage.input_tokens` and `response.usage.output_tokens` report tokens. They can be `None`. You pay only for input tokens.
- Score `probabilities` and `legend` use integer keys in the Python SDK and string keys in raw JSON.
- `Score.criteria` is a list. The SDKs stopped accepting a dict keyed by integers in version 0.6.
- The SDKs retry `429` and `529` responses with backoff. Configure retries with `RetryPolicy`.
- Plain dict questions with a `"type"` key work in place of the typed classes.

## Read the answers

- `score` is the probability-weighted mean of the level numbers. A score of 1.0 can mean certainty on level 1 or an even split between levels 0 and 2. Read `probabilities` with it.
- Threshold a score, rank by it, or round it to the nearest level. Do not interpolate a quantity from it. Jev's levels are weakly calibrated as numbers.
- `confidence` measures how peaked the distribution is. It describes the model's answer and does not guarantee a correct one.
- When code only needs the best option, take `choice` and skip the threshold. Add a confidence gate only where a wrong answer has a cost.
- Two other measures from `probabilities` often work well. The top probability is easy to reason about, but its meaning depends on the number of options, so set it per question. The ratio of the top probability to the second measures how clearly the winner beats the runner-up.
- A Noul has no `confidence`. Its distance from 0.5 plays that role. To gate Nouls and Choices with the same code, use `abs(2 * noul - 1)`.
- Every answer stays inside the options you supplied, so code never parses prose.

## Images and Box

Jev accepts PNG and JPEG images in the `state`, in `instructions`, and in `criteria`. The Box primitive returns where a target sits in an image. TypeSafe calls both features "dreamer-ready", so use them for prototypes and expect the format to change.

- Send the `TypeSafe-Danger-Zone-Images: true` header with every request that has an image or a Box question.
- Box needs `jev-1.14` or later.
- A Box always returns a box, even when the target is absent. Ask a presence Noul in the same request.
- Put an image that several questions use in the `state` once, and refer to it by path.

Read [references/images-and-box.md](references/images-and-box.md) before writing a request with images or a Box question. It covers placement, Box question design, reading values from scans, and image preparation and limits.

## Improve a program

Find the question that fails before changing anything. Collect labeled examples, run them, and compare each question's answers and probabilities against the labels.

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Wrong answers with high confidence | Jev read the instruction literally | State the exact condition. Put the boundary case in the criteria. |
| Low confidence on a Choice | Options overlap, or no option fits | Add `what`, `not_for`, and `examples`. Add a companion Noul that checks whether any option fits. |
| A Choice picks an option when none fits | Choice probabilities always sum to 1 | Ask a presence Noul in the same request and read the Choice only when it passes. |
| The answer changes when options are reordered | Jev leans toward the first option | Test several orders. Make the descriptions more contrastive. |
| Low confidence on a Score | Levels overlap, the question measures two things, or the state says too little | Rewrite levels as distinct situations. Split the question. Add the missing field to the state. |
| Scores cluster in the middle | Levels are degrees or numbers | Describe a concrete situation per level. Remove numerals. |
| Top-of-scale cases look alike | The extreme case has no level | Add a level for the extreme. |
| A Noul hovers near 0.5 | The condition is vague | Define the condition. Add `true` and `false` criteria with examples. |
| The same input routes differently on repeat runs | A single cutoff sits inside run-to-run variation | Use a yes and no band and send the middle to review. |
| Accuracy falls as inputs grow | The state carries irrelevant detail | Filter in code. Send only the needed fields. |
| Errors on counts, sums, dates, or numeric nearness | Jev is doing arithmetic | Move the arithmetic to code. Ask Jev only for extraction or per-item judgments. |
| Errors on nested or negated questions | Too much indirection | Ask the direct question. Name the state path. Split into two literal questions and combine in code. |
| The answer follows text inside the state | The content steers the model | Tighten the criteria. Test adversarial cases. Gate the action on confidence. |
| A Box lands on the wrong object or an absent one | Box always returns a box, or the target name is vague | Add a presence Noul. Name the target. Crop and ask again. |
| Rewording one question trades one error for another | One question weighs several properties | Split it into atomic questions and combine them in code. |
| The final decision is wrong while each answer is right | The policy is wrong | Change the weights or thresholds in code. Leave the questions alone. |
| Results changed with no code change | An alias moved to a new model | Pin a versioned model ID. Retest before you move. |
| The program is slow or costly | Questions are spread over sequential calls | Merge them into one request. Keep a second request only when it truly depends on the first answer. |

Rules for revising:

- Change one or two questions per revision. Jev's probabilities shift in ways that are hard to predict, so leave questions that discriminate well untouched.
- Judge a revision on labeled data. Higher confidence alone does not show a better question, and two wordings of one scale can behave differently on your data.
- Avoid fitting thresholds closely to a handful of test cases.
- Keep the answer space stable once code depends on it. Adding or removing a level or an option changes what every earlier answer meant.
- Write general rules in instructions and criteria. Put specific names and values only in `examples`.

## Checklist

- [ ] Each question asks one property and a person could answer it in a second.
- [ ] The primitive matches how code uses the answer.
- [ ] Instructions state the exact condition and name state paths in backticks.
- [ ] Criteria agree with the instructions and point the same direction.
- [ ] Score levels describe situations, stand alone, and carry no numerals.
- [ ] Nouls are phrased so a high value means yes.
- [ ] Choices that may not cover every input have a companion Noul.
- [ ] Code does all counting, arithmetic, and date comparison.
- [ ] The state holds only what the questions need and fits the token limits.
- [ ] All questions on the same state travel in one request.
- [ ] Every action has a confidence threshold matched to its risk, and low confidence has a fallback.
- [ ] Noul gates use a yes and no band, not a single cutoff.
- [ ] Questions, weights, and thresholds live together in code.
- [ ] The model ID is pinned once thresholds are tuned, and `response.model` is logged.
- [ ] Image and Box requests send the images header, and every Box has a presence Noul.
- [ ] Labeled examples back every revision.

## Sources

- https://docs.typesafe.ai/models and https://docs.typesafe.ai/model-jaggedness/jev-1.13
- https://docs.typesafe.ai/primitives, with `/choice`, `/score`, `/noul`, `/box`, and `/advanced`
- https://docs.typesafe.ai/confidence and https://docs.typesafe.ai/images
- https://docs.typesafe.ai/concepts/how-to-build-with-system-one and https://docs.typesafe.ai/concepts/state
- https://docs.typesafe.ai/patterns, with `/fan-out`, `/confidence-routing`, `/composite-scoring`, and `/intent-routing`
- https://docs.typesafe.ai/cookbooks, especially `skill_suggestion`, `hierarchical_classification`, `classification_using_confidence`, `rerank_typesafe`, `semantic_find`, `sde_cascade`, `consistency_noul_cookbook`, `reading_scanned_documents`, and `ocr_confidence_bounding_boxes`
- https://docs.typesafe.ai/sdk/python and https://docs.typesafe.ai/api/errors
- https://docs.typesafe.ai/llms.txt lists every page. Append `.md` to a docs URL for raw markdown.
