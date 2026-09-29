# Architecture

## System boundary

The pipeline is split into five layers. The public case study describes their
responsibilities without exposing reusable creative prompts, proprietary story
concepts, provider credentials or production source code.

```mermaid
flowchart TB
    CLI[Brief CLI] --> ORCH[Orchestration layer]
    ORCH --> AGENTS[Specialised LLM agents]
    AGENTS --> MODELS[Typed domain models]
    MODELS --> GATES[Validation and fidelity gates]
    GATES --> ARTIFACTS[Immutable run artifacts]
    ARTIFACTS --> MEDIA[Manual media handoff]
    MEDIA --> QA[Media QA and assembly]
    QA --> DELIVERY[Final delivery package]
```

## 1. Orchestration layer

Python coordinates the run lifecycle. LangGraph manages the story review/rewrite
loop and carries approval state explicitly rather than treating model prose as a
decision. A run identifier ties every artifact and QA result to one production run.

**Responsibilities**

- accepts a brief, target age and duration;
- executes each stage in a controlled order;
- enforces bounded retries;
- records timestamps and durable run metadata.

## 2. Creative-agent layer

Specialised roles use local Ollama models rather than one generic prompt:

| Role | Responsibility |
| --- | --- |
| Planner and writer | Produce a story plan and child-appropriate narrative. |
| Reviewer and independent judge | Assess story quality independently. |
| Character-bible creator | Defines stable visual identity and constraints. |
| Style-bible creator | Defines palette, lighting, camera language and exclusions. |
| Production planner | Converts scenes into timed shots. |
| Fidelity reviewers | Compare generated plans against approved source artifacts. |
| Repair agents | Make one targeted correction from recorded findings. |

Models are selected by task: `qwen3:8b` for planning/writing and production
artifacts, `gpt-oss:20b` for independent evaluation, and
`qwen2.5:14b-instruct` for semantic fidelity checks. Prompt templates and runtime
configuration remain private.

## 3. Typed contracts and deterministic validation

Pydantic contracts model stories, characters, visual bibles, production plans and
reviews. Deterministic services validate facts that should never be delegated to a
language model: required fields, scene/shot identifiers, timing budgets, character
roster completeness and artifact shape.

Examples of invariants:

- all recurring named characters have a precise species and profile;
- a production plan has the required scene and shot structure;
- shot duration totals match the requested duration budget;
- character identity changes are treated as blocking fidelity failures;
- a repaired artifact is validated again before it can proceed.

## 4. Media handoff and QA

The system exports a single next prompt and expected file location for a human
operator. This supports external image-to-video tools while preserving continuity
and avoiding uncontrolled bulk generation.

For the reference pilot, approved still frames were created with **OpenAI
ImageGen in ChatGPT**. Each still was then animated manually with **Wan 2.2 14B
Fast** through a Hugging Face ZeroGPU Space. English narration was generated with
**ElevenLabs `eleven_multilingual_v2`** and normalized locally; the final mix used
a licensed track from the **YouTube Audio Library**. These are documented pilot
choices, not hard dependencies: provider credentials, prompts and adapter
configuration remain private, and the architecture keeps the media boundary
replaceable.

After each handoff, scripts check file existence, dimensions, duration, codec,
decodeability, audio properties and completeness. The assembly stage creates scene
masters, a review master and a final delivery manifest.

## 5. Artifact lifecycle

Each successful stage saves a latest artifact for convenient access and an immutable
snapshot under a run-specific archive. Fidelity reviews are retained even when a
gate fails, allowing review criteria to improve from real evidence.

```text
run/<run_id>/
├── story.json
├── character_bible.json
├── character_bible_fidelity_review.json
├── visual_style_bible.json
├── production_plan.json
├── production_fidelity_review.json
└── run.json
```

## Technology choices

| Area | Technology |
| --- | --- |
| Language/runtime | Python |
| Workflow orchestration | LangGraph |
| LLM integration | LangChain + Ollama (`qwen3:8b`, `gpt-oss:20b`, `qwen2.5:14b-instruct`) |
| Data contracts | Pydantic |
| Keyframe generation (pilot) | OpenAI ImageGen in ChatGPT, human-operated |
| Image-to-video (pilot) | Wan 2.2 14B Fast via Hugging Face ZeroGPU, human-operated |
| Narration (pilot) | ElevenLabs `eleven_multilingual_v2` |
| Final music (pilot) | YouTube Audio Library |
| Media processing and QA | deterministic Python/FFmpeg-based scripts |
| Data format | JSON run artifacts |
| Development environment | Windows, virtual environment, local models |

## Trade-offs

- The system prioritises validation and repeatability over one-click generation.
- External visual generation stays manual because creative quality and provider
  access require human judgement.
- The project uses local models for predictable cost and privacy, while retaining
  provider adapters as an extensibility boundary.
