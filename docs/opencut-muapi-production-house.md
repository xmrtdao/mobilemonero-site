# OpenCut + MUAPI Production House

## Recommendation

Use **MUAPI as the generative upstream** and **OpenCut as the editorial, assembly, review, and export layer**. They should remain separate services with a small XMRT orchestration service between them. This keeps expensive model calls, media editing, and publication permissions independently controllable.

OpenCut’s current rewrite is MIT-licensed and is being rebuilt around a Rust core with planned editor APIs, plugins, an MCP server, headless mode, and scripting. The project itself says the rewrite is not yet ready for general outside contributions and points users who need a working editor to the archived `opencut-classic` repository. Therefore the initial XMRT integration should target a **fork or pinned classic deployment**, while tracking the rewrite behind an adapter boundary.

MUAPI provides image, video, and audio generation, multi-node workflows, storyboarding, webhooks, SDKs, n8n nodes, ComfyUI nodes, and agent/CLI integration. It is a strong fit for shot generation and variation, but it should not own the editorial timeline or final publication decision.

## Service boundaries

```text
XMRT Relay / Production Orchestrator
├── Brief + source ingestion
├── Shot list, continuity, prompt and provenance records
├── MUAPI task submission and webhook/poll handling
├── Asset manifest + object storage
├── OpenCut project/package generation
├── Human review and approval state
└── Paragraph / social / archive publication

MUAPI (generation)
├── Concept frames and PFPs
├── Image-to-video and text-to-video shots
├── Music, narration, SFX, and voice work
├── Talking-avatar or lipsync assets
└── Upscale, translation, and variation passes

OpenCut (editing)
├── Timeline, tracks, transitions, captions, and overlays
├── Scene ordering and continuity review
├── Editorial trims and brand-safe composition
├── Preview renders and final export
└── Future headless/MCP automation when stable
```

## Golden workflow

1. **Brief:** An agent creates a production brief with audience, format, runtime, aspect ratios, safety constraints, and a definition of done.
2. **Storyboard:** MUAPI generates a small set of concept frames. A human or trusted agent selects the visual direction before generating a full batch.
3. **Shot generation:** The orchestrator submits MUAPI tasks with idempotency keys, records request IDs, and waits for webhooks or bounded polling. Failed tasks are retried only when the provider marks them retryable.
4. **Asset normalization:** Store source media, generated media, prompt/model metadata, duration, frame rate, dimensions, license notes, and checksum in an asset manifest. Do not place provider secrets in OpenCut projects.
5. **Editorial assembly:** Build an OpenCut project from the manifest. Use stable asset IDs and timeline timecodes rather than provider URLs so a regenerated shot can be swapped without rewriting the whole project.
6. **Review:** Require preview approval for continuity, likeness, copyright, captions, audio levels, accessibility, and factual claims. Keep draft, approved, and published states separate.
7. **Export:** OpenCut renders the approved master and platform derivatives. Generate a caption file, thumbnail, alt text, content warning when needed, and a provenance sidecar.
8. **Publish:** Paragraph, social, and newsletter delivery are downstream actions. Publication should be an explicit approved transition, not a side effect of generation or export.
9. **Archive:** Preserve the brief, manifest, project package, final outputs, approvals, and publication links. Record corrections and takedowns as new events rather than mutating history.

## Initial implementation plan

### Phase 1 — dependable, low-risk integration

Deploy `opencut-classic` in a private environment as the browser editor and use an XMRT adapter to produce importable project assets. Keep the adapter’s interface provider-neutral:

```json
{
  "project_id": "mtv-2026-001",
  "assets": [
    {
      "asset_id": "shot-003-v2",
      "uri": "s3://media/shot-003-v2.mp4",
      "kind": "video",
      "duration_ms": 6200,
      "fps": 30,
      "width": 1920,
      "height": 1080,
      "source": "muapi",
      "provider_task_id": "muapi-task-id",
      "prompt_hash": "sha256:..."
    }
  ],
  "timeline": [
    {"asset_id": "shot-003-v2", "track": "video-1", "start_ms": 12000, "trim_in_ms": 0, "trim_out_ms": 6200}
  ]
}
```

Use object storage with signed URLs, short-lived access, and lifecycle cleanup for abandoned drafts. Do not expose raw MUAPI credentials to the browser.

### Phase 2 — agent-assisted production

Add an MCP-facing production skill with safe, narrow tools such as `create_brief`, `generate_storyboard`, `submit_shot_batch`, `get_generation_status`, `build_opencut_project`, `render_preview`, and `request_approval`. Keep `publish-paragraph` and social posting in a separate approval scope.

### Phase 3 — headless automation

When the OpenCut rewrite’s Editor API and headless mode are stable, replace the classic adapter with the new API behind the same interface. Use deterministic rendering in CI for a small regression suite: title cards, captions, music ducking, aspect-ratio variants, and one multi-scene sequence.

## Production-house roles

| Role | Responsibility | Recommended authority |
| --- | --- | --- |
| Brief agent | Converts a request into a bounded production brief | Public/trusted read and draft write |
| Storyboard agent | Generates and ranks concept frames | MUAPI generation only |
| Shot agent | Creates and monitors media tasks | MUAPI generation and asset write |
| Editor agent | Assembles timeline and captions | OpenCut project write; no publish |
| Review agent | Checks continuity, safety, factuality, and accessibility | Read plus approval recommendation |
| Publisher agent | Publishes approved artifacts | Explicit publish scope only |
| Archivist | Stores manifests, outputs, and links | Append-only archive write |

## Security and reliability rules

- Keep MUAPI keys and provider webhooks server-side.
- Verify webhook signatures where supported and reject replayed task events.
- Use idempotency keys for every generation request and publication attempt.
- Cap fan-out, duration, resolution, and cost per brief.
- Store content hashes and prompt/model metadata for provenance.
- Never infer consent for faces, voices, brands, or copyrighted source material.
- Run captions and audio loudness checks before approval.
- Make publication opt-in and auditable; drafts must never auto-send newsletters.
- Add cost estimates before large batches and a kill switch for runaway workflows.
- Record provider outages separately from editorial failures.

## Why this completes the stack

MUAPI supplies breadth and speed across generation modalities. OpenCut supplies a creator-facing timeline and a future programmable editing surface. XMRT’s relay, certification, storage, and publication controls supply identity, coordination, and governance. Together they form a production house without pretending that generation, editing, and distribution are the same problem.

## Sources

- [OpenCut rewrite repository](https://github.com/OpenCut-app/OpenCut)
- [OpenCut classic repository](https://github.com/OpenCut-app/opencut-classic)
- [MUAPI documentation](https://muapi.ai/docs)
- [MUAPI homepage](https://muapi.ai/)
- [XMRT MUAPI comparison and workflow notes](https://github.com/xmrtdao/xmrtdao.github.io/blob/main/docs/muapi-vs-notebooklm.md)
