# Edge Stack (moved)

The Edge stack (`p4n4-edge`) merged into the AI stack
([ADR-005](../decisions/adr/ADR-005.md)):

- **The inference runner** (Edge Impulse `.eim` · ONNX · mock) is the AI stack's `ei-runner`
  service, unchanged: see [Edge Inference](edge-inference.md).
- **The repository** became `p4n4-extras`, for optional services outside the core: see
  [Extras](extras.md).
- **Projects with an `edge` layer** keep working; `p4n4 migrate edge` moves them: see
  [Migrating from the edge stack](../guides/migrating-from-edge.md).
