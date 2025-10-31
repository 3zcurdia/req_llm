# Fixture System Public API Implementation Plan

**Goal:** Promote ReqLLM's internal fixture system from `test/support/` to `lib/req_llm/fixtures/` as a stable public API for consuming applications.

**Issue:** [#154](https://github.com/agentjido/req_llm/issues/154)  
**Effort:** L (1-2 days)  
**Target:** Next minor release

---

## Current State Analysis

### Modules in `test/support/` (to be moved)

```
test/support/
├── chunk_collector.ex       → ReqLLM.Test.ChunkCollector
├── env.ex                   → ReqLLM.Test.Env
├── fixture_path.ex          → ReqLLM.Test.FixturePath
├── fixture.ex               → ReqLLM.Step.Fixture.Backend
├── fixtures.ex              → ReqLLM.Test.Fixtures
├── transcript.ex            → ReqLLM.Test.Transcript
└── vcr.ex                   → ReqLLM.Test.VCR
```

### Modules in `lib/` (already public)

```
lib/req_llm/
├── step/fixture.ex          → ReqLLM.Step.Fixture (entry point)
└── streaming/fixtures.ex    → ReqLLM.Streaming.Fixtures.HTTPContext
```

### Current Dependencies

**Files in `lib/` that reference `ReqLLM.Test.*` modules:**
- `lib/req_llm/streaming.ex` - References `ReqLLM.Test.Fixtures`
- `lib/req_llm/streaming/finch_client.ex` - References `ReqLLM.Test.VCR`
- `lib/req_llm/stream_server.ex` - References `ReqLLM.Step.Fixture.Backend`
- `lib/req_llm/step/fixture.ex` - References `ReqLLM.Step.Fixture.Backend`

**Files in `test/support/` that cross-reference:**
- `fixture.ex` - Uses `ReqLLM.Test.{Env, FixturePath, VCR, Fixtures}`
- `fixtures.ex` - Uses `ReqLLM.Test.{Env, FixturePath}`
- `fake_keys.ex` - Uses `ReqLLM.Test.Env`

---

## Proposed Architecture

### New Structure in `lib/`

```
lib/req_llm/fixtures/
├── backend.ex              → ReqLLM.Fixtures.Backend (was Step.Fixture.Backend)
├── chunk_collector.ex      → ReqLLM.Fixtures.ChunkCollector
├── env.ex                  → ReqLLM.Fixtures.Env
├── http_context.ex         → ReqLLM.Fixtures.HTTPContext (move from streaming/)
├── path.ex                 → ReqLLM.Fixtures.Path
├── transcript.ex           → ReqLLM.Fixtures.Transcript
└── vcr.ex                  → ReqLLM.Fixtures.VCR

lib/req_llm/step/
└── fixture.ex              → ReqLLM.Step.Fixture (update to use Fixtures.Backend)

lib/req_llm/streaming/
└── fixtures.ex             → DELETE (HTTPContext moves to fixtures/)
```

### Backward-Compatible Shims in `test/support/`

```
test/support/
├── chunk_collector_shim.ex     → defdelegate to ReqLLM.Fixtures.ChunkCollector
├── env_shim.ex                 → defdelegate to ReqLLM.Fixtures.Env
├── fixture_path_shim.ex        → defdelegate to ReqLLM.Fixtures.Path
├── fixtures_shim.ex            → defdelegate to ReqLLM.Fixtures (facade)
├── transcript_shim.ex          → defdelegate to ReqLLM.Fixtures.Transcript
└── vcr_shim.ex                 → defdelegate to ReqLLM.Fixtures.VCR
```

**Note:** `fixture.ex` in test/support can be deleted since Backend moves to lib.

---

## Implementation Steps

### Phase 1: Create New Public Modules (No Breaking Changes)

#### Step 1.1: Move HTTPContext
- [ ] Move `lib/req_llm/streaming/fixtures.ex` → `lib/req_llm/fixtures/http_context.ex`
- [ ] Rename module: `ReqLLM.Streaming.Fixtures.HTTPContext` → `ReqLLM.Fixtures.HTTPContext`
- [ ] Update `@moduledoc` to indicate this is now public
- [ ] Keep backward-compatible alias in `streaming/fixtures.ex` temporarily

#### Step 1.2: Create `ReqLLM.Fixtures.Env`
- [ ] Copy `test/support/env.ex` → `lib/req_llm/fixtures/env.ex`
- [ ] Rename: `ReqLLM.Test.Env` → `ReqLLM.Fixtures.Env`
- [ ] Update `@moduledoc` - mark as public API
- [ ] Add configuration support:
  ```elixir
  @doc """
  Returns current fixture mode (:record or :replay).
  
  Priority:
  1. Application config: `config :req_llm, fixtures_mode: :replay`
  2. Environment variable: `REQ_LLM_FIXTURES_MODE=record`
  3. Default: `:replay`
  """
  @spec mode() :: :record | :replay
  def mode do
    Application.get_env(:req_llm, :fixtures_mode) ||
      mode_from_env() ||
      :replay
  end
  
  defp mode_from_env do
    case System.get_env("REQ_LLM_FIXTURES_MODE") do
      "record" -> :record
      "replay" -> :replay
      nil -> nil
      other -> raise "Invalid REQ_LLM_FIXTURES_MODE: #{inspect(other)}"
    end
  end
  ```

#### Step 1.3: Create `ReqLLM.Fixtures.Path`
- [ ] Copy `test/support/fixture_path.ex` → `lib/req_llm/fixtures/path.ex`
- [ ] Rename: `ReqLLM.Test.FixturePath` → `ReqLLM.Fixtures.Path`
- [ ] Update `@moduledoc` - mark as public API
- [ ] Add configurable root:
  ```elixir
  @doc """
  Returns the fixtures root directory.
  
  Priority:
  1. Application config: `config :req_llm, fixtures_root: "path"`
  2. Default: "test/support/fixtures"
  """
  @spec root() :: String.t()
  def root do
    Application.get_env(:req_llm, :fixtures_root, "test/support/fixtures")
    |> Path.expand()
  end
  ```

#### Step 1.4: Create `ReqLLM.Fixtures.Transcript`
- [ ] Copy `test/support/transcript.ex` → `lib/req_llm/fixtures/transcript.ex`
- [ ] Rename: `ReqLLM.Test.Transcript` → `ReqLLM.Fixtures.Transcript`
- [ ] Update `@moduledoc` - mark as public API
- [ ] No logic changes needed

#### Step 1.5: Create `ReqLLM.Fixtures.ChunkCollector`
- [ ] Copy `test/support/chunk_collector.ex` → `lib/req_llm/fixtures/chunk_collector.ex`
- [ ] Rename: `ReqLLM.Test.ChunkCollector` → `ReqLLM.Fixtures.ChunkCollector`
- [ ] Update `@moduledoc` - mark as public API
- [ ] No logic changes needed

#### Step 1.6: Create `ReqLLM.Fixtures.VCR`
- [ ] Copy `test/support/vcr.ex` → `lib/req_llm/fixtures/vcr.ex`
- [ ] Rename: `ReqLLM.Test.VCR` → `ReqLLM.Fixtures.VCR`
- [ ] Update all internal references:
  - `ReqLLM.Test.Transcript` → `ReqLLM.Fixtures.Transcript`
  - `ReqLLM.Test.ChunkCollector` → `ReqLLM.Fixtures.ChunkCollector`
  - `ReqLLM.Test.FixturePath` → `ReqLLM.Fixtures.Path`
- [ ] Update `@moduledoc` - mark as public API

#### Step 1.7: Create `ReqLLM.Fixtures.Backend`
- [ ] Copy `test/support/fixture.ex` → `lib/req_llm/fixtures/backend.ex`
- [ ] Rename: `ReqLLM.Step.Fixture.Backend` → `ReqLLM.Fixtures.Backend`
- [ ] Update all internal references:
  - `ReqLLM.Test.Env.fixtures_mode()` → `ReqLLM.Fixtures.Env.mode()`
  - `ReqLLM.Test.FixturePath` → `ReqLLM.Fixtures.Path`
  - `ReqLLM.Test.VCR` → `ReqLLM.Fixtures.VCR`
  - `ReqLLM.Test.Fixtures` → `ReqLLM.Fixtures` (facade)
  - `ReqLLM.Streaming.Fixtures.HTTPContext` → `ReqLLM.Fixtures.HTTPContext`
- [ ] Update `@moduledoc` - mark as public API

#### Step 1.8: Create `ReqLLM.Fixtures` Facade
- [ ] Copy `test/support/fixtures.ex` → `lib/req_llm/fixtures.ex`
- [ ] Rename: `ReqLLM.Test.Fixtures` → `ReqLLM.Fixtures`
- [ ] Update all internal references:
  - `ReqLLM.Test.Env` → `ReqLLM.Fixtures.Env`
  - `ReqLLM.Test.FixturePath` → `ReqLLM.Fixtures.Path`
- [ ] Update `@moduledoc`:
  ```elixir
  @moduledoc """
  Public facade for fixture recording and replay.
  
  Provides unified interface for both streaming and non-streaming fixtures.
  
  ## Configuration
  
      # config/test.exs
      config :req_llm,
        fixtures_mode: :replay,              # :record | :replay
        fixtures_root: "test/fixtures/llm"   # Custom fixture directory
  
  ## Usage in Tests
  
      # Non-streaming
      {:ok, response} = ReqLLM.generate_text(
        "anthropic:claude-sonnet-4-20250514",
        "Explain recursion",
        fixture: "recursion_test"
      )
      
      # Streaming
      {:ok, stream_response} = ReqLLM.stream_text(
        "openai:gpt-4o",
        "Count to 10",
        fixture: "streaming_count"
      )
      
      # Custom path (overrides everything)
      {:ok, response} = ReqLLM.generate_text(
        model,
        prompt,
        fixture_path: "test/custom/my_fixture.json"
      )
  
  ## Recording Fixtures
  
      # Option 1: Environment variable
      REQ_LLM_FIXTURES_MODE=record mix test
      
      # Option 2: Application config
      config :req_llm, fixtures_mode: :record
      
  ## Fixture File Structure
  
      test/support/fixtures/
      ├── anthropic/
      │   ├── claude_3_5_sonnet_20241022/
      │   │   ├── basic.json
      │   │   └── streaming.json
      │   └── claude_3_5_haiku_20241022/
      │       └── tool_calling.json
      └── openai/
          └── gpt_4o/
              ├── basic.json
              └── reasoning.json
  
  Note: Model names are "slugged" (lowercased, non-alphanumerics → "_")
  """
  ```

### Phase 2: Update Library Code to Use New Modules

#### Step 2.1: Update `lib/req_llm/step/fixture.ex`
- [ ] Change `Code.ensure_loaded(ReqLLM.Step.Fixture.Backend)` → `Code.ensure_loaded(ReqLLM.Fixtures.Backend)`
- [ ] Update apply call: `apply(ReqLLM.Fixtures.Backend, :step, [provider, name])`
- [ ] Update `@moduledoc` with new usage examples
- [ ] Add `@since "1.1.0"` tag to document when fixtures became public

#### Step 2.2: Update `lib/req_llm/stream_server.ex`
- [ ] Line 750: Change `Code.ensure_loaded(ReqLLM.Step.Fixture.Backend)` → `Code.ensure_loaded(ReqLLM.Fixtures.Backend)`
- [ ] Line 751: Update pattern match to `{:module, ReqLLM.Fixtures.Backend}`
- [ ] Line 760: Update apply to `apply(ReqLLM.Fixtures.Backend, :save_streaming_fixture, [...])`
- [ ] Line 771: Update debug message

#### Step 2.3: Update `lib/req_llm/streaming.ex`
- [ ] Line 221: Change `Code.ensure_loaded(ReqLLM.Test.Fixtures)` → `Code.ensure_loaded(ReqLLM.Fixtures)`
- [ ] Line 222: Update apply to `apply(ReqLLM.Fixtures, :capture_path, [model, opts])`

#### Step 2.4: Update `lib/req_llm/streaming/finch_client.ex`
- [ ] Update all `ReqLLM.Test.VCR` references → `ReqLLM.Fixtures.VCR`
- [ ] Update all `ReqLLM.Test.Fixtures` references → `ReqLLM.Fixtures`
- [ ] Update `ReqLLM.Streaming.Fixtures.HTTPContext` → `ReqLLM.Fixtures.HTTPContext`

### Phase 3: Create Backward-Compatible Shims

#### Step 3.1: Create shims in `test/support/`
Create these files to maintain backward compatibility for internal tests:

**`test/support/chunk_collector_shim.ex`:**
```elixir
defmodule ReqLLM.Test.ChunkCollector do
  @moduledoc false
  defdelegate start_link(), to: ReqLLM.Fixtures.ChunkCollector
  defdelegate add_chunk(collector, chunk), to: ReqLLM.Fixtures.ChunkCollector
  defdelegate get_chunks(collector), to: ReqLLM.Fixtures.ChunkCollector
end
```

**`test/support/env_shim.ex`:**
```elixir
defmodule ReqLLM.Test.Env do
  @moduledoc false
  defdelegate fixtures_mode(), to: ReqLLM.Fixtures.Env, as: :mode
end
```

**`test/support/fixture_path_shim.ex`:**
```elixir
defmodule ReqLLM.Test.FixturePath do
  @moduledoc false
  defdelegate root(), to: ReqLLM.Fixtures.Path
  defdelegate file(model_or_spec, name), to: ReqLLM.Fixtures.Path
  defdelegate slug(name), to: ReqLLM.Fixtures.Path
end
```

**`test/support/fixtures_shim.ex`:**
```elixir
defmodule ReqLLM.Test.Fixtures do
  @moduledoc false
  defdelegate mode(), to: ReqLLM.Fixtures
  defdelegate replay_path(model_or_spec, opts), to: ReqLLM.Fixtures
  defdelegate capture_path(model_or_spec, opts), to: ReqLLM.Fixtures
end
```

**`test/support/transcript_shim.ex`:**
```elixir
defmodule ReqLLM.Test.Transcript do
  @moduledoc false
  defdelegate read!(path), to: ReqLLM.Fixtures.Transcript
  defdelegate write!(path, data), to: ReqLLM.Fixtures.Transcript
  defdelegate validate(data), to: ReqLLM.Fixtures.Transcript
end
```

**`test/support/vcr_shim.ex`:**
```elixir
defmodule ReqLLM.Test.VCR do
  @moduledoc false
  
  defdelegate record(path, opts), to: ReqLLM.Fixtures.VCR
  defdelegate load(path), to: ReqLLM.Fixtures.VCR
  defdelegate load!(path), to: ReqLLM.Fixtures.VCR
  defdelegate streaming?(transcript), to: ReqLLM.Fixtures.VCR
  defdelegate status(transcript), to: ReqLLM.Fixtures.VCR
  defdelegate headers(transcript), to: ReqLLM.Fixtures.VCR
  defdelegate replay_response_body(transcript), to: ReqLLM.Fixtures.VCR
  defdelegate replay_as_stream(transcript, provider_mod, model), to: ReqLLM.Fixtures.VCR
  defdelegate replay_into_stream_server(path, stream_server_pid), to: ReqLLM.Fixtures.VCR
end
```

#### Step 3.2: Delete old modules
- [ ] Delete `test/support/chunk_collector.ex` (replaced by shim)
- [ ] Delete `test/support/env.ex` (replaced by shim)
- [ ] Delete `test/support/fixture_path.ex` (replaced by shim)
- [ ] Delete `test/support/fixture.ex` (moved to lib, no shim needed)
- [ ] Delete `test/support/fixtures.ex` (replaced by shim)
- [ ] Delete `test/support/transcript.ex` (replaced by shim)
- [ ] Delete `test/support/vcr.ex` (replaced by shim)
- [ ] Delete `lib/req_llm/streaming/fixtures.ex` (HTTPContext moved)

### Phase 4: Update Tests

#### Step 4.1: Update test files that directly reference modules
- [ ] `test/req_llm/test/chunk_collector_test.exs` - Update alias
- [ ] `test/req_llm/test/env_test.exs` - Update alias
- [ ] `test/req_llm/test/transcript_test.exs` - Update alias
- [ ] `test/req_llm/test/vcr_test.exs` - Update alias (if exists)

#### Step 4.2: Update test helpers
- [ ] `test/support/fake_keys.ex` - Update `ReqLLM.Test.Env.fixtures_mode()` references
- [ ] `test/support/streaming_case.ex` - Update `REQ_LLM_FIXTURES_MODE` references if needed

### Phase 5: Documentation

#### Step 5.1: Update main docs
- [ ] Add `lib/req_llm/fixtures.ex` to main module documentation
- [ ] Update `README.md` with fixtures section:
  ```markdown
  ## Testing with Fixtures
  
  ReqLLM includes built-in fixture support for both streaming and non-streaming tests:
  
  ```elixir
  # config/test.exs
  config :req_llm,
    fixtures_mode: :replay,
    fixtures_root: "test/fixtures/llm"
  
  # In your tests
  test "LLM response parsing" do
    {:ok, response} = ReqLLM.generate_text(
      "anthropic:claude-sonnet-4-20250514",
      "Explain recursion",
      fixture: "recursion_explanation"
    )
    
    assert response.choices[0].message.content =~ "function calls itself"
  end
  ```
  
  See `ReqLLM.Fixtures` for full documentation.
  ```

#### Step 5.2: Create guides
- [ ] Create `guides/testing_with_fixtures.md`:
  - Overview of fixture system
  - Recording fixtures
  - Replaying fixtures
  - Custom fixture paths
  - Streaming vs non-streaming
  - Best practices
  - Troubleshooting

#### Step 5.3: Update CHANGELOG.md
```markdown
## [Unreleased]

### Added

- **Public Fixtures API** - Fixture system promoted from internal testing infrastructure
  to public API in `lib/req_llm/fixtures/` (#154)
  - `ReqLLM.Fixtures` - Main facade for fixture operations
  - `ReqLLM.Fixtures.Env` - Mode configuration (:record/:replay)
  - `ReqLLM.Fixtures.Path` - Fixture path resolution with configurable root
  - `ReqLLM.Fixtures.VCR` - Recording and replay engine
  - `ReqLLM.Fixtures.Transcript` - Fixture file format handling
  - `ReqLLM.Fixtures.Backend` - Req step implementation
  - `ReqLLM.Fixtures.HTTPContext` - HTTP metadata for streaming fixtures
  - `ReqLLM.Fixtures.ChunkCollector` - Streaming chunk collection
  - Configuration via `config :req_llm, fixtures_mode:` and `fixtures_root:`
  - Environment variable support: `REQ_LLM_FIXTURES_MODE=record`
  - Works with both `generate_text/3`, `generate_object/4`, and `stream_text/3`
  - No longer requires adding `deps/req_llm/test/support` to `elixirc_paths`

### Changed

- Moved `ReqLLM.Streaming.Fixtures.HTTPContext` to `ReqLLM.Fixtures.HTTPContext`
- Internal `ReqLLM.Test.*` modules now delegate to public `ReqLLM.Fixtures.*` API

### Deprecated

- `ReqLLM.Test.*` modules in `test/support/` are now shims (will be removed in 2.0)
  - Use `ReqLLM.Fixtures.*` instead for public API access
```

### Phase 6: Testing & Validation

#### Step 6.1: Run existing tests
- [ ] `mix test` - Ensure all tests pass with shims
- [ ] `mix test test/req_llm/test/` - Test the test infrastructure
- [ ] `REQ_LLM_FIXTURES_MODE=record mix test --only "provider:anthropic" --only "scenario:basic"` - Test recording
- [ ] Verify fixtures still work in replay mode

#### Step 6.2: Test consumer usage
Create a test consumer app to verify the public API:

**`test_consumer/mix.exs`:**
```elixir
defp deps do
  [
    {:req_llm, path: "../"}
  ]
end

defp elixirc_paths(:test), do: ["lib", "test/support"]
defp elixirc_paths(_), do: ["lib"]
```

**`test_consumer/config/test.exs`:**
```elixir
import Config

config :req_llm,
  fixtures_mode: :replay,
  fixtures_root: "test/fixtures/llm"
```

**`test_consumer/test/llm_test.exs`:**
```elixir
defmodule LLMTest do
  use ExUnit.Case

  test "fixtures work without elixirc_paths hacks" do
    # This should work without adding deps/req_llm/test/support
    assert Code.ensure_loaded?(ReqLLM.Fixtures.Backend)
    
    # Record a fixture (run once with REQ_LLM_FIXTURES_MODE=record)
    {:ok, response} = ReqLLM.generate_text(
      "anthropic:claude-sonnet-4-20250514",
      "Say hello",
      max_tokens: 50,
      fixture: "hello_test"
    )
    
    assert response.choices[0].message.content =~ "hello"
  end
end
```

- [ ] Verify test passes in replay mode
- [ ] Verify test records in record mode
- [ ] Verify no `elixirc_paths` hacks needed

#### Step 6.3: Validate documentation
- [ ] Run `mix docs` and verify all new modules appear
- [ ] Check that examples in docs are correct
- [ ] Verify guides are accessible

### Phase 7: Quality Checks

#### Step 7.1: Run quality tools
- [ ] `mix format` - Format all new/modified files
- [ ] `mix compile --warnings-as-errors` - No warnings
- [ ] `mix dialyzer` - Type checking passes
- [ ] `mix credo --strict` - Linting passes

#### Step 7.2: Coverage validation
- [ ] `mix mc --sample` - Validate sample models still work
- [ ] Check that fixture system works for all provider types

---

## Configuration API

### Application Config

```elixir
# config/test.exs
config :req_llm,
  fixtures_mode: :replay,              # :record | :replay (default: :replay)
  fixtures_root: "test/fixtures/llm"   # default: "test/support/fixtures"
```

### Environment Variables

- `REQ_LLM_FIXTURES_MODE=record` - Override config mode
- `REQ_LLM_FIXTURES_MODE=replay` - Explicit replay mode

### Options

```elixir
# Use fixture with model's provider
ReqLLM.generate_text(model, prompt, fixture: "test_name")

# Explicit provider
ReqLLM.generate_text(model, prompt, fixture: {:anthropic, "test_name"})

# Custom path (overrides everything)
ReqLLM.generate_text(model, prompt, fixture_path: "test/custom/my_fixture.json")
```

---

## Migration Path for Consumers

### Before (Broken in consuming apps)
```elixir
# mix.exs
defp elixirc_paths(:test), do: ["lib", "test/support", "deps/req_llm/test/support"]

# Tests - doesn't work because Backend not compiled
{:ok, response} = ReqLLM.generate_text(model, prompt, fixture: "test")
```

### After (Works out of the box)
```elixir
# config/test.exs
config :req_llm,
  fixtures_mode: :replay,
  fixtures_root: "test/fixtures/llm"

# Tests - just works!
{:ok, response} = ReqLLM.generate_text(model, prompt, fixture: "test")
```

---

## Rollback Plan

If issues are discovered:

1. **Revert shims** - Change shims back to original implementations
2. **Keep public modules** - Leave new modules in `lib/` but don't document
3. **Mark as experimental** - Add `@moduledoc "Experimental - use at own risk"`
4. **Document workaround** - Update issue with temporary solution

---

## Success Criteria

- [ ] All existing tests pass
- [ ] Test consumer app works without `elixirc_paths` hacks
- [ ] Both streaming and non-streaming fixtures work
- [ ] Documentation is complete and accurate
- [ ] Quality checks pass (format, dialyzer, credo)
- [ ] CHANGELOG updated
- [ ] No breaking changes for internal tests
- [ ] GitHub issue #154 resolved

---

## Post-Release Tasks

- [ ] Update issue #154 with resolution
- [ ] Monitor for feedback from users
- [ ] Consider deprecation warnings for `ReqLLM.Test.*` shims in next major version
- [ ] Evaluate adding more convenience features based on feedback:
  - Fixture validation CLI tool (`mix req_llm.fixtures.validate`)
  - Fixture cleanup tool (`mix req_llm.fixtures.clean`)
  - Fixture comparison tool (diff recorded vs current)

---

## Timeline

**Day 1:**
- Morning: Phase 1 (Create new public modules)
- Afternoon: Phase 2 (Update library code)
- Evening: Phase 3 (Create shims)

**Day 2:**
- Morning: Phase 4 (Update tests), Phase 5 (Documentation)
- Afternoon: Phase 6 (Testing & validation), Phase 7 (Quality checks)
- Evening: Final review and PR creation

---

## Notes

- Keep shims in `test/support/` indefinitely for backward compatibility
- Consider adding deprecation warnings in 2.0 for the shims
- HTTPContext move is clean since it's in `lib/` already
- All fixture file formats remain unchanged (backward compatible)
- No changes to fixture file structure or location by default
