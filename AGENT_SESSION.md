# Bounded Go SDK agent sessions

This optional module runs natively in the Go application consuming this exact SDK.
It requires no JavaScript/Python bridge, hosted model or model vendor. Implement
`AgentProvider` with `Identity`, `Version` (`omniwrap.agent-provider/v1`), `Propose`
and context-aware `Close`. `agent-session.json` describes actual resource methods,
SDK parameter names, argument schemas and declared effects. Its contract descriptions
are untrusted data, never permissions or provider instructions.

Create a client with caller-controlled credentials and base URL, then construct a
checkpoint using `NewAgentCheckpoint(path, externalKey)` (at least 32 key bytes).
Call `NewAgentSession(client, provider, AgentSessionOptions{...})` with the verified
SHA-256 of the exact installed module ZIP, an explicit operation allowlist, all nine
limits, `ApproveWrite` and a tested `PersistResult` projection. Call `Run(ctx, input)`
with JSON input. `AgentProposal` contains usage plus either `Calls` or `Final`;
`json.RawMessage("null")` represents an explicit final null. The generated
`New<SDKParams>AgentCall` helpers construct proposals using the SDK's native types.

```go
checkpoint, err := sdk.NewAgentCheckpoint("session.json", checkpointKey)
if err != nil { return err }
session, err := sdk.NewAgentSession(client, provider, sdk.AgentSessionOptions{
    ArtifactHash: installedArchiveSHA256,
    AllowedTools: []string{"getItem"}, // use an operation ID from agent-session.json
    Limits: sdk.AgentLimits{"turns": 5, "model_calls": 5, "tool_calls": 5,
        "tokens": 1000, "cost": 1, "elapsed_seconds": 30,
        "concurrency": 2, "input_bytes": 100000, "output_bytes": 100000},
    Checkpoint: checkpoint,
    ApproveWrite: func(ctx context.Context, tool, argumentsHash string) (bool, error) {
        return false, nil // replace with caller-owned explicit approval when writes are allowed
    },
    PersistResult: projectApprovedFields, // mandatory: remove secrets before persistence/reinjection
})
if err != nil { return err }
result, err := session.Run(ctx, json.RawMessage(`{"goal":"inspect the item"}`))
```

A complete batch validates before effects. Calls cannot set destination, authentication,
headers, cookies or retry options. Unsafe HTTP methods and source-declared writes both
require approval. Retries are disabled for session calls. Results are reinjected in
proposal order. Native SDK errors remain available through `errors.As`/`errors.Is`;
unresolved mutations report `agent_session_in_doubt` with `Retryable=false`.

Checkpoints authenticate source, module archive, registry, provider, permissions and
budgets with an external-key HMAC; receipts are hash chained. Synced file replacement,
writer locks and revision checks refuse stale/concurrent saves. Original input,
argument values, credentials and provider reasoning are not persisted. The caller's
projection must remove response secrets. State and JSON have bounded size/depth.
A durable deadline prevents resetting elapsed budgets on resume. Checkpoints are
Go-lane-specific; they are not interchangeable with Python/JavaScript checkpoints.

After a crash, any pending call remains in doubt. Inspect the external system, then
call `Reconcile(callID, knownResult)` before resuming; it records a projected result
without sending another request. An abandoned `.lock` requires operator recovery
after proving that the writer stopped. A permanent `.started` guard makes a missing
checkpoint fail closed instead of becoming a fresh session. Preserve this guard with
the checkpoint; never delete it to bypass recovery. Never blindly replay a mutation.

Provider, approval and projection code are caller-controlled, not sandboxed. Provider
usage is reported by the adapter. Cancellation bounds waiting; an uncooperative callback
may keep a goroutine or external work alive. Cleanup errors are returned. Go response
buffers are bounded per call (at most 64 MiB); parallel calls can hold concurrency times
that bound. Filesystem permissions and power-loss durability depend on the host platform.
Go's [os.Rename](https://pkg.go.dev/os#Rename) does not guarantee atomic replacement on
non-Unix platforms. A missing, torn or unauthenticated checkpoint requires explicit
operator recovery; the guard addresses missing state, not a loss of the entire directory.
Regex constraints, unqualified schema formats/keywords, model-supplied header/cookie
arguments and stream/upload operations are refused before effects; native SDK calls
remain available separately. No live hosted-model or universal secret-filtering claim.

Verify this module with a Go toolchain meeting go.mod and the trusted OmniWrap engine's
gofmt. Package admission regenerates all Go sources from the embedded inert contract
and settings, comparing source bodies and provenance; an independent source comparison
also requires caller-owned expected API evidence. This checks engine-produced bytes,
not the trustworthiness of an arbitrary contract, toolchain or host. Module ZIP installation
and runtime tests are separate from the package gate's race-detector check, which requires
a compatible C compiler. Generation does not publish a module or run a hosted agent.
