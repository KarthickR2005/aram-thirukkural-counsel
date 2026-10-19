# Aram — Thirukkural counsel agent

**a Thirukkural counsel agent that judges whether your life dilemma aligns with Aram**

An agent on AWS that hands one decision to **Strands Decider 2B** and the conversation to **Amazon Bedrock**. Decision pattern: `#policy`. Tags: `#strands-decider-2b` `#policy`.

- Video: (add the YouTube/Vimeo link)
- Article: (add the AWS Builder Center link)
- Deployed: run `./kit deploy`

## The project

Aram — a Thirukkural counsel agent. These answers drive the code, benchmark, README, article and
video, and the project leads all three.

- **Pitch:** a Thirukkural counsel agent that judges whether your life dilemma aligns with Aram
  (righteous living) and guides you with the right couplet.
- **Who it is for:** anyone with a personal life question, doubt, or moral dilemma who wants
  counsel grounded in the Thirukkural's Aram, in Tamil or English.
- **Features:**
  - take a life question or dilemma in plain language, and decide if it is a real moral question
    worth guiding;
  - Strands Decider 2B scores Aram-alignment and the virtue at stake in one fast pass, shown live
    in the demo's decision panel;
  - retrieve the most fitting Thirukkural couplet (Tamil and English) and compose clear guidance
    via Amazon Bedrock;
  - low-confidence or ambiguous cases ask you one clarifying question instead of guessing;
  - deployed on Amazon Bedrock AgentCore Runtime.
- **The decision handed to Strands Decider 2B:** before composing guidance, does this situation
  align with Aram, is it a real dilemma, and which virtue is at stake? (`#policy`)
- **Why a decision model:** the judgement needs reading language — what the person meant, whether
  their action upholds a virtue written in prose — and it runs on every turn, so it must be fast,
  cheap and private, and return numbers the code can branch on.
- **Amazon Bedrock:** the conversation and the composed guidance (Amazon Nova by default).



## How it is built

Strands Agents with Amazon Bedrock for the generative part, the tools in `agent/tools.py`, deployed to Amazon Bedrock AgentCore Runtime, and one decision handed to Strands Decider 2B at a clear seam in the agent's path.

Every Amazon Bedrock model id is set in `.env` (`BEDROCK_MODEL_ID`, and `BEDROCK_SMALL/MEDIUM/LARGE_MODEL_ID` for routing), Amazon Nova by default.



## Reusable decision pattern: Aram alignment  (#policy)

The pattern lives in [`agent/decision.py`](agent/decision.py), labelled `REUSABLE DECISION PATTERN`. It contains the typed questions and their options, the wiring that puts the decision in the agent's path, and a note on how to reuse it:

```text
REUSABLE DECISION PATTERN: Aram alignment  (#policy)

What it does
    Before a Thirukkural counsel agent composes any guidance, Strands Decider 2B reads the
    person's situation and judges, in one call:
    - is this a real moral dilemma worth guiding, or just chat / a factual question?
    - does the described course of action align with Aram (righteous living)?
    - which virtue is most at stake?
    Ordinary `if` statements then branch: chat (no dilemma), affirm (aligned), caution (against),
    or ask a clarifying question (ambiguous or low confidence). The model classifies; the code
    decides. The decider does not pick the couplet — retrieval is a deterministic lookup over a
    curated Thirukkural dataset, so guidance is always grounded in a real couplet.

Typed questions and options
    `is_dilemma` (noul), `aram_alignment` (score: against / ambiguous / aligned),
    `theme` (choice: honesty / non_harm / self_control / forgiveness / duty / compassion).
    See QUESTIONS below for the option descriptions.

Wiring
    `AramPolicy` runs as a pre-agent step (outside the Strands `Agent`). In `agent/agent.py`:
        rec, verdict = AramPolicy(decider).evaluate(build_state(message, history))
        if verdict == "chat":   reply = ...           # converse, no couplet
        else:                   reply = compose(...)  # affirm / caution / clarify, with a couplet
    The record (probabilities in 0..1, a `verdict`, `decider_ms`) is appended to `AramPolicy.log`
    and returned in the agent's `decisions`, so the demo panel and the benchmark show what was
    decided. `build_state(message, history)` builds the exact state text the benchmark's cases use.

How to reuse it
    Copy this file, keep `PATTERN_TAG` and the `REUSABLE DECISION PATTERN` header, and reword the
    three questions for your own virtue set or policy. The thresholds that turn probabilities into
    the verdict live in `decide()` (ALIGN_CONF): change them there, in code you own.
```

Typed questions:

```python
QUESTIONS = {
    "is_dilemma": Decider.noul(
        "Is the person describing a real moral dilemma, decision, or doubt they personally face "
        "(something they must choose or are troubled by), rather than a greeting, a thank-you, "
        "small talk, or a factual question with no moral stakes?",
        {"true": "a personal situation requiring a moral choice, doubt, or counsel",
         "false": "a greeting, a thank-you or casual acknowledgement, small talk, a factual or "
                  "general question, or a request with no moral stakes"},
    ),
    "aram_alignment": Decider.score(
        "Does the described course of action align with Aram (righteous living)? "
        "Levels (lowest to highest alignment): against = the action is harmful, deceitful, unjust, "
        "or unrestrained; ambiguous = mixed, unclear, or depends on context not stated; "
        "aligned = the action upholds truth, compassion, self-control, justice, or duty.",
        ["against", "ambiguous", "aligned"],
    ),
    "theme": Decider.choice(
        "Which virtue (Aram) is most at stake in this situation?",
        {
            "honesty": "truthfulness, integrity in words, not lying",
            "non_harm": "not causing injury or death to living beings; non-violence",
            "self_control": "restraint of the senses, appetite, anger, and desire",
            "forgiveness": "forbearance, patience, returning good for harm, letting go of anger",
            "duty": "doing one's rightful work, responsibility, justice in conduct",
            "compassion": "kindness, charity, concern for others, giving",
        },
    ),
}
```

## Measured

Decider: strands-decider-2b-hobson-v19 (ugmdu). 14 labelled cases, 38 decisions.

| | Strands Decider 2B | qwen3.5-4b |
|---|---|---|
| Accuracy | 84.2% | 79.0% |
| Latency p50 per decision | 3914 ms | 22464 ms |
| Latency p95 | 4196 ms | 23862 ms |
| Cost per 1,000 decisions | $0.0350 | $0.0000 |
| Answered alone at confidence >= 0.7 | 42% | 100% |
| Accuracy when confident | 93.8% | 79.0% |

At the median the decider was 5.7x faster than the LLM.

Reproduce with `./kit bench` (cases in `agent/eval_cases.jsonl`; charts in `submission/bench/`).

## Run it

```bash
./kit setup        # checks your machine and AWS, sets up Strands Decider 2B (local model or hosted API)
./kit demo         # web UI at http://localhost:8501
./kit bench        # the decision vs an Amazon Bedrock LLM
./kit deploy       # Amazon Bedrock AgentCore Runtime
```

Requirements: Python 3.10+, AWS credentials with Amazon Bedrock access, and for deploying, Node 20+ with the AgentCore CLI (`./kit agentcore-cli`).

## Credits

Built with the AWS User Group Madurai Strands Decider hackathon kit (https://github.com/awsugmdu). The kit's reusable library (`decider_kit/`, the demo UI, the video and article tooling, and the Kiro steering, skills and hooks) is the user group's; the agent in `agent/` is mine. Strands Decider: https://github.com/strands-labs/strands-decider (Apache-2.0).

Licence: Apache-2.0 (see LICENSE).
