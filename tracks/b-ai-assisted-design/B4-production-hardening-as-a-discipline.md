# B4 — Production Hardening as a Discipline

**Track:** AI-Assisted Design  
**Source repos:** [SuperInstance/edge-compiler](https://github.com/SuperInstance/edge-compiler), [SuperInstance/nexus-git-agent](https://github.com/SuperInstance/nexus-git-agent)  
**Training Port phases served:** Phase 2 (Assistant), Phase 3 (Technician)

## Why this module exists

AI-generated code that runs in a chat window is not the same as code that runs in production. This module teaches the difference through two real repos: one where strict mode caught deploy-breaking errors, and one where the `master` branch could not build at all.

## Case 1: edge-compiler — strict mode as a lie detector

[edge-compiler](https://github.com/SuperInstance/edge-compiler) is a Cloudflare Worker that compiles and quantizes models for specific hardware targets. Before the 2026 hardening round, its TypeScript had never been checked with `strict: true`. The round found and fixed five real strict-mode errors.

The most dangerous one: `compileModel()` referenced `env` without receiving it. The function signature was:

```ts
async function compileModel(model: R2Object, options: CompileRequest): Promise<ArrayBuffer>
```

Inside the function, it called:

```ts
const result = await env.AI.run("@cf/onnx", {
  model: new Uint8Array(modelData),
  options: compilationOptions
});
```

`env` was not in scope. In loose mode, TypeScript treated `env` as `any`, and the code might have deployed. In strict mode, the build failed, preventing a runtime error in production.

A second fix was cultural, not just mechanical. The code called Cloudflare AI model IDs `@cf/onnx` and `@cf/quantization`. Neither model exists in Cloudflare's real Workers AI catalog. The fix was not to invent plausible model IDs. The fix was to mark them honestly — to say, "these are placeholders, and the real catalog must be checked before deployment."

## Case 2: nexus-git-agent — committed is not the same as working

[nexus-git-agent](https://github.com/SuperInstance/nexus-git-agent) is a Cloudflare Worker for fleet coordination and trust scoring. Before the hardening round, its `master` branch could not build. The failure was a mangled Content-Security-Policy header in the source.

The lesson is simple and easy to forget: **a commit is a claim; a build is proof.** Code sitting on `master` looks authoritative. If it does not compile, it is not authoritative. The only way to know is to run the build on a clean machine.

## The discipline

Production hardening is not a single task. It is a set of habits:

1. **Turn on strict mode.** Whether it is TypeScript's `strict`, Rust's `#![deny(warnings)]`, or C's `-Wall -Wextra -Werror`, the compiler is a free auditor.
2. **Run the build in CI, not just locally.** Local environments hide sins. A clean checkout on a different machine exposes them.
3. **Distinguish placeholder from real.** If a model ID, API endpoint, or hardware spec is not verified, label it as such. Do not let a placeholder ride into production because it looks plausible.
4. **Treat build failure as high-priority signal.** A broken `master` is a stop-the-line event, not a backlog item.

## Comprehension questions

1. In `edge-compiler`, `compileModel()` used `env.AI.run()` but `env` was not in the function signature. Why would this code have passed a loose TypeScript build but failed in strict mode? What would have happened if it had deployed?

2. The `@cf/onnx` and `@cf/quantization` model IDs do not exist in Cloudflare's Workers AI catalog. Why is deleting them not the only honest option? What is the alternative, and why does it preserve useful design intent?

3. The `nexus-git-agent` `master` branch could not build because of a mangled CSP header. Explain why "but it worked on my machine" is not a valid defense, and what workflow change would prevent this from reaching `master`.

4. A Phase 3 technician is handed an AI-generated provisioning script for the Field Kit. The script references a model file at `/opt/captain/models/llm/phi-3-mini-4k-instruct-q4.gguf` and an API endpoint at `https://api.fieldkit.tech/v1/status`. Using the discipline from this module, what four checks should the technician perform before approving the script for an install?
