# OKX portfolio-margin options decision log

This log records every material implementation, safety, compatibility, and delivery decision made
for moving BTC-collateralized OKX options into the live trading stack. It is append-only during the
program: superseded decisions remain visible and point to the replacing entry.

## D-001: Pin the program to the current `develop` head

- **Status:** Accepted
- **Decision:** Base all work on commit `73d4686f816c893cda24d61fe6b67bdbf0746131`, which was
  independently verified as the current `develop` head on 2026-08-24.
- **Rationale:** The fork and local checkout agree exactly, so code findings and test vectors refer
  to one reproducible tree.
- **Alternatives:** Rebase opportunistically during implementation; work from the repository's
  default branch without pinning.
- **Cost if wrong:** A later upstream change may need to be rebased before integration.
- **Reversibility:** Fully reversible by rebasing each focused branch.

## D-002: Deliver the stack as focused, ordered change sets

- **Status:** Accepted
- **Decision:** Separate the program into core option economics, OKX option execution bootstrap,
  dynamic instrument lifecycle, and Portfolio Margin operational safety/account reporting. Do not
  mix an account-model redesign into an adapter change.
- **Rationale:** These surfaces have different failure modes and reviewer ownership. Each change
  must remain independently testable and revertible.
- **Alternatives:** One large end-to-end branch; an OKX-only parser workaround.
- **Cost if wrong:** Stacked dependencies may require coordinated rebases or temporarily prevent a
  later branch from running without an earlier one.
- **Reversibility:** Fully reversible; commits can be reordered or split before any push.

## D-003: Fix option premium valuation in the core model

- **Status:** Accepted
- **Decision:** Preserve `is_inverse` as settlement/exercise metadata while deriving a separate
  inverse-price-valuation semantic. `CryptoOption` and `CryptoOptionSpread` use direct premium
  valuation; inverse futures retain reciprocal valuation.
- **Rationale:** OKX BTC options and Deribit coin options quote their premium directly in coin even
  though their expiry payoff is inverse. Changing only the OKX parser would corrupt strike/index
  currency semantics and leave the same defect on Deribit.
- **Alternatives:** Mark OKX options linear and change their quote currency to BTC; add a stored
  valuation enum and migrate constructors and serialization.
- **Cost if wrong:** A venue that genuinely quotes an option premium reciprocally would require a
  stored valuation convention later.
- **Reversibility:** The derived method can be replaced by a persisted enum without changing the
  existing instrument data in this change.

## D-004: Avoid a persisted-schema change in the first core fix

- **Status:** Accepted
- **Decision:** Do not add fields to `CryptoOption`, `CryptoOptionSpread`, or `Position` for this
  correction.
- **Rationale:** A field would force Rust and Python constructor changes plus Serde, Arrow, and
  position-persistence migrations. Current instrument classes are sufficient to derive the
  behavior for all supported venues.
- **Alternatives:** Add `price_currency` and a valuation enum to every serialized instrument and
  position.
- **Cost if wrong:** A future exception cannot be represented per instrument until a schema field
  is introduced.
- **Reversibility:** Fully reversible through a later additive schema migration.

## D-005: Keep public GitHub interaction under human control

- **Status:** Accepted
- **Decision:** Implement, test, review, and commit locally, but do not push, open an issue, or open
  a pull request without an explicit final authorization.
- **Rationale:** Repository policy requires human authorization for each public interaction, and a
  push or PR is an external side effect. The current instruction authorizes implementation but not
  legal attestations or upstream communication.
- **Alternatives:** Open an RFC and PR automatically as part of execution.
- **Cost if wrong:** Delivery pauses at a review-ready local branch rather than appearing on GitHub
  immediately.
- **Reversibility:** The user can authorize a push and PR at the final integration checkpoint.

## D-006: Use a local-only ignore for worktree artifacts

- **Status:** Accepted
- **Decision:** Add `.worktrees/` and `.superpowers/` to `.git/info/exclude` instead of changing the
  repository `.gitignore`.
- **Rationale:** These directories are harness-local state, not project artifacts. A product commit
  solely for local worktree bookkeeping would add unrelated review noise.
- **Alternatives:** Commit `.worktrees/` to `.gitignore`; edit directly in the detached checkout.
- **Cost if wrong:** Another clone will need its own local exclude entry.
- **Reversibility:** Delete the two lines from `.git/info/exclude` after the worktrees are removed.

## D-007: Treat missing local build tooling as a verification blocker

- **Status:** Accepted
- **Decision:** Continue specification and code inspection after recording that `cargo` is absent,
  but do not claim any Rust change is buildable or complete until the pinned Rust 1.98.0 toolchain
  is available and the required tests and checks run.
- **Rationale:** The untouched baseline command `cargo build -p nautilus-model` failed because the
  executable is not installed. Static review cannot substitute for compilation in live-trading
  code.
- **Alternatives:** Proceed and rely on GitHub CI; install an unpinned distribution Rust package.
- **Cost if wrong:** Work may require correction once a compiler is available.
- **Reversibility:** Install the pinned toolchain and run the complete verification matrix before
  integration.

## D-008: Install the pinned Rust toolchain inside the workspace

- **Status:** Accepted; resolves the tooling condition recorded in D-007
- **Decision:** Install the official, checksum-verified Rust 1.98.0 minimal toolchain plus `rustfmt`
  and `clippy` under `/workspace/scratch/3284c9bc1274/toolchains`, without changing the host PATH or
  global toolchain state.
- **Rationale:** Rust 1.98.0 is the exact version pinned by `rust-toolchain.toml`; a distribution
  package or a different toolchain would not provide valid build evidence.
- **Alternatives:** Rely on CI; install a global toolchain; continue with static review only.
- **Cost if wrong:** The workspace consumes additional disk and download time.
- **Reversibility:** Remove the task-local `toolchains` directory after delivery.

## D-009: Do not overload quote-order sizing with option exposure

- **Status:** Accepted
- **Decision:** Leave `Instrument::try_calculate_base_quantity` unchanged. Do not represent an
  option's `Q * M` underlying amount through this method.
- **Rationale:** Production callers use this method to convert quote-denominated order amounts into
  traded order quantity. It is not an exposure API, existing inverse callers bypass it, and an
  option spread does not generally have one scalar underlying exposure.
- **Alternatives:** Return `Q * M` for every crypto option; introduce a new exposure API now.
- **Cost if wrong:** A later consumer that needs single-leg option exposure will require a separate
  additive API.
- **Reversibility:** Fully reversible through a focused API design once a concrete consumer exists.

## D-010: Use the deterministic CI Cargo profile for local verification

- **Status:** Accepted
- **Decision:** Run Rust tests in this environment with `--profile ci-pr -j1` and the pinned Rust
  1.98.0 toolchain.
- **Rationale:** The untouched baseline intermittently produced a zero-length codegen object under
  the normal incremental, multi-codegen-unit profiles. The repository's `ci-pr` profile disables
  incremental compilation and uses one codegen unit; under that profile the full model baseline
  completed with 3,057 passed, 0 failed, and 1 ignored tests.
- **Alternatives:** Treat a retry under the flaky profile as evidence; rely only on hosted CI;
  change repository Cargo profiles for this task.
- **Cost if wrong:** Local builds are slower and use a separate target profile.
- **Reversibility:** Remove the command-line profile override in an environment where normal
  profiles are reliable. No repository build configuration is changed.

## D-011: Track the decision log outside ignored agent artifacts

- **Status:** Accepted
- **Decision:** Keep implementation plan and scratch specification files under the repository's
  ignored `docs/superpowers/` tree, but version this user-requested log at
  `docs/developer_guide/decisions/2026-08-24-okx-pm-options-stack.md`.
- **Rationale:** The repository deliberately ignores `superpowers/` directories. Forcing ignored
  harness artifacts into Git would defeat that policy, while an untracked decision log would not
  satisfy the requirement to preserve every material ruling with the work.
- **Alternatives:** Force-add all harness artifacts; leave the log untracked; add a new global
  ignore exception.
- **Cost if wrong:** Maintainers may prefer the log to be squashed out of an upstream PR even though
  it remains useful on the implementation branch.
- **Reversibility:** Move or omit the log during final branch preparation with explicit user
  approval; its history remains in local commits.

## D-012: Propagate notional currency through wallet reservations

- **Status:** Accepted
- **Decision:** Make derivative buy reservations in `WalletAccount` select the observed balance by
  the currency returned from notional valuation, rather than independently inferring it from
  `is_inverse` and `use_quote_for_inverse`.
- **Rationale:** Once inverse option price valuation is direct, `use_quote_for_inverse=true` is
  intentionally inapplicable and still returns BTC. The current wallet path would preselect the USD
  balance and either fail or relabel BTC without conversion. Inverse-future quote mode must continue
  to select its actual quote-currency notional.
- **Alternatives:** Declare derivative wallet reservations unsupported; leave an inconsistent
  generic `Account` implementation; special-case crypto options.
- **Cost if wrong:** The wallet path performs the derivative notional calculation before looking up
  its observed balance, while its exact linear-pair reservation path remains unchanged.
- **Reversibility:** Restore source-currency inference if the account interface later prohibits all
  derivative instruments explicitly.

## D-013: Preserve legacy quanto reservation and margin currency

- **Status:** Accepted
- **Decision:** Correct currency propagation for direct-priced inverse options without changing the
  existing contract that cash-account reservations and generic margin models label non-inverse
  quanto requirements in quote currency.
- **Rationale:** Trusting `notional.currency` for every instrument would also change quanto locks
  from quote to settlement currency. That may warrant a separate economics correction, but it is
  unrelated to BTC-collateralized options and would widen this live-trading change beyond its
  reviewed scope. For non-quanto instruments, returned notional currency remains authoritative.
- **Alternatives:** Change quanto reservations and margins to settlement currency in this branch;
  special-case the two crypto-option concrete types.
- **Cost if wrong:** The pre-existing quanto currency behavior remains even if a future analysis
  concludes settlement currency is economically preferable.
- **Reversibility:** A focused quanto change can update the preserved controls and currency policy
  independently.

## D-014: Pin all local verification tools without changing host state

- **Status:** Accepted
- **Decision:** Use task-local Rust 1.98.0, nightly Rust 1.100.0-nightly
  (`fb6531d55`, 2026-08-23), uv 0.12.5, and prek 0.4.14. Install binaries and caches only under
  `/workspace/scratch/3284c9bc1274/toolchains`, then create the ignored project `.venv` with
  `make sync`.
- **Rationale:** The repository pins uv and prek, requires `cargo +nightly fmt`, and rejects the
  system uv 0.11.33. Exact local versions make Python builds, formatting, and pre-commit executable
  without mutating the host runtime or the repository manifests.
- **Alternatives:** Skip Python and pre-commit verification; update global tools; rely entirely on
  hosted CI.
- **Cost if wrong:** The task-local toolchain and Python environment consume additional disk and
  must be included explicitly on `PATH` for every worker shell.
- **Reversibility:** Remove the task-local `toolchains` directory and ignored `.venv` after delivery.

## D-015: Apply the quanto compatibility exception only to non-inverse contracts

- **Status:** Accepted; refines D-013
- **Decision:** Preserve quote-currency locks and generic margins only when an instrument is both
  non-inverse and quanto. For inverse-quanto contracts, retain existing inverse flag behavior:
  base/underlying currency when quote mode is false and quote currency when it is true.
- **Rationale:** `Instrument::is_quanto()` can also be true for an inverse contract. A condition on
  `is_quanto()` alone would silently override the inverse currency convention this branch promises
  to preserve.
- **Alternatives:** Treat every quanto instrument alike; declare inverse-quanto unsupported and
  remove its existing behavior.
- **Cost if wrong:** The compatibility predicate is more explicit and carries extra control tests
  through cash, wallet, and margin paths.
- **Reversibility:** A later unified valuation-convention model can replace both derived predicates.

## D-016: Append the missing option-family input to the Python execution config

- **Status:** Accepted
- **Decision:** Expose the existing Rust `instrument_families` field as a final, optional Python
  constructor argument on `OKXExecClientConfig`, after every existing positional argument.
- **Rationale:** The Rust execution client already resolves and loads configured option families,
  but the Python constructor hardcodes the field to `None`. Appending the argument unlocks the
  existing path without changing positional caller meanings or the serialized Rust config.
- **Alternatives:** Insert the argument beside `instrument_types`; add a second options-only config;
  infer `BTC-USD` when OPTION is selected.
- **Cost if wrong:** The constructor gains one more public input and generated-stub surface even for
  callers that do not trade options.
- **Reversibility:** The default remains `None`; the additive input can be deprecated without
  changing existing calls.

## D-017: Fail closed on unusable OPTION family configuration

- **Status:** Accepted
- **Decision:** Treat a missing, empty, blank-only, or mixed valid/blank family list as invalid when
  OPTION is requested. Skip OPTION loading locally with a warning; return a configured vector
  unchanged only when it is non-empty and every member is non-blank.
- **Rationale:** OKX requires an explicit option family. Sending an ambiguous or blank request risks
  broad discovery, venue rejection, or an execution client that appears configured but has no
  tradable option cache.
- **Alternatives:** Trim and discard blank elements; load all option families; silently default to
  `BTC-USD`; rely only on the HTTP endpoint rejection.
- **Cost if wrong:** A configuration containing an accidental blank alongside valid families stops
  all OPTION startup rather than partially loading.
- **Reversibility:** Validation can later return structured per-entry diagnostics or accept a safer
  typed family value without changing the explicit-family requirement.

## D-018: Expose diagnostics without enabling generic mass cancel

- **Status:** Accepted
- **Decision:** Add read-only Python getters for `instrument_families`, `use_spot_margin`, and
  `use_mm_mass_cancel`, but do not make either `use_*` flag constructor-settable in the bootstrap
  change. Keep generic mass cancel disabled for the option stack until a family-scoped MMP safety
  design is implemented and reviewed.
- **Rationale:** Operators need to verify effective native configuration, while the existing
  `use_mm_mass_cancel` switch changes generic cancel-all routing and is not equivalent to OKX
  option Market Maker Protection.
- **Alternatives:** Expose both flags as constructor inputs immediately; reuse generic mass cancel
  as MMP; expose every internal config field.
- **Cost if wrong:** Advanced callers cannot opt into those existing flags through the public
  Python constructor in the bootstrap slice.
- **Reversibility:** Constructor inputs can be added later after their routing and safety contracts
  have dedicated tests.

## D-019: Satisfy verification prerequisites with exact task-local dependencies

- **Status:** Accepted
- **Decision:** Install PyArrow 25.0.0 into the ignored project environment and
  `cargo-machete` 0.9.2 into the task-local Cargo home, matching the repository's wheel-test and
  pre-commit requirements.
- **Rationale:** The complete pre-commit run proved that Python test collection and dependency
  analysis require those exact tools. Installing them locally closes an environmental verification
  gap without changing project manifests or host-global state.
- **Alternatives:** Skip the affected hooks; rely on hosted CI; install unpinned global versions.
- **Cost if wrong:** The task-local environment consumes additional disk and must remain on the
  verification `PATH`.
- **Reversibility:** Remove the ignored environment and task-local toolchain after delivery.

## D-020: Reclaim only reproducible Cargo artifacts under disk pressure

- **Status:** Accepted
- **Decision:** When repeated full-workspace verification exhausted the workspace volume, reclaim
  only the generated `ci-pr` Cargo profile with `cargo clean --profile ci-pr`; preserve all source,
  Git state, the Python environment, and other verification profiles.
- **Rationale:** The removed 26.9 GiB consisted entirely of rebuildable compiler output. Continuing
  with an almost-full volume risked false compiler and documentation failures.
- **Alternatives:** Delete broad directories manually; remove the Python environment; stop before
  an authoritative pre-commit result.
- **Cost if wrong:** Any later `ci-pr` test command must rebuild its artifacts.
- **Reversibility:** Cargo recreates the profile deterministically on the next build.

## D-021: Apply repository-specific account formatting after semantic approval

- **Status:** Accepted
- **Decision:** Add the blank-line separations required by the repository formatting hook in the
  two modified account methods, as a dedicated style-only commit after the semantic Task 4 review.
- **Rationale:** The authoritative pre-commit hook rejected the otherwise reviewed code for two
  local formatting conventions. Isolating the two-line correction preserves review provenance and
  makes the non-semantic change explicit.
- **Alternatives:** Fold the correction into the prior semantic commit; ignore the repository hook.
- **Cost if wrong:** One additional local commit must be carried or squashed during integration.
- **Reversibility:** The style-only commit can be squashed without altering behavior.

## D-022: Gate the core branch on the complete repository policy suite

- **Status:** Accepted
- **Decision:** Treat the core option-economics branch as a valid stack base only after
  `make pre-commit` passes with `CHANGED_BASE_SHA` pinned to D-001, selecting the entire branch
  range rather than only the most recent commit.
- **Rationale:** Focused tests and the earlier clean-tree run supplied strong evidence, but the
  explicit base SHA proves that all changed model, execution, risk, portfolio, Python, and
  documentation surfaces satisfy the repository-wide policy suite together.
- **Alternatives:** Rely only on per-task tests; let the clean-worktree fallback infer changed
  crates; defer the branch-wide gate to hosted CI.
- **Cost if wrong:** The local gate repeats some expensive Clippy and documentation work.
- **Reversibility:** Later stacked branches can use their own exact base SHA while retaining this
  core result as historical evidence.

## D-023: Isolate option price PnL from commission currency in tests

- **Status:** Accepted
- **Decision:** Construct option-position valuation tests with explicit zero-valued BTC
  commissions rather than the generic fill stub's default USD commission.
- **Rationale:** The cases are intended to prove direct premium PnL and notional arithmetic. An
  unrelated USD fee either changes the independently derived PnL literal or exercises currency
  conversion that is outside this correction.
- **Alternatives:** Reuse the generic USD commission; subtract an observed fee from expected PnL;
  special-case commission handling in production.
- **Cost if wrong:** The tests would not cover a nonzero option commission, which remains covered by
  the fee-model tests rather than the position-price tests.
- **Reversibility:** Add a separate nonzero BTC commission case without changing these isolated
  arithmetic controls.

## D-024: Preserve honest integration evidence after the owning production fix

- **Status:** Accepted
- **Decision:** Treat downstream MarginAccount and Python Position cases added after the owning
  core valuation changes as post-change integration evidence. Do not revert correct production code
  merely to manufacture a second RED phase.
- **Rationale:** Tasks 1 and 2 already captured behavioral RED failures at the production boundary.
  Reverting them would weaken provenance and could make unrelated downstream expectations fail for
  the wrong reason.
- **Alternatives:** Temporarily revert production for every downstream test; omit downstream
  integration coverage; label a test RED without observing it.
- **Cost if wrong:** Those individual downstream tests have no isolated mutation run, although the
  owning production behavior has observed RED/GREEN evidence and full-suite coverage.
- **Reversibility:** A future mutation-testing job can prove each downstream assertion independently.

## D-025: Stage conditional downstream changes only by reviewed exact path

- **Status:** Accepted
- **Decision:** If verification had required a risk or portfolio correction, stage only the exact
  reviewed files and include them in the owning downstream contract commit; never stage either
  directory broadly or leave reviewed changes merely staged.
- **Rationale:** The worktree may contain user-owned changes, and conditional scope must not expand
  silently. In the observed run, both full suites passed and no such production file changed.
- **Alternatives:** Stage whole directories; make an unreviewed cleanup commit; omit failing
  downstream coverage.
- **Cost if wrong:** Exact-path staging requires more bookkeeping when several files legitimately
  change.
- **Reversibility:** None was exercised because the conditional production changes were unnecessary.

## D-026: Accept the observed negative-spread RED failure mode

- **Status:** Accepted
- **Decision:** Count the negative option-spread notional case as a valid RED even though the
  existing `Some(true)` quote-mode path bypassed the anticipated positive-price guard and returned
  the wrong reciprocal USD value instead of raising that guard.
- **Rationale:** The independently derived direct signed BTC expectation failed before production
  changed and passed afterward. The exact pre-change failure mechanism differed from the planning
  prediction, but the required behavior was still demonstrably absent.
- **Alternatives:** Rewrite the test to force the predicted error; discard the observed failure;
  change the expectation to match reciprocal output.
- **Cost if wrong:** Only the plan's failure-mode narration changes; the product behavior and
  independent expected value remain tested.
- **Reversibility:** The preserved test output can be reclassified without changing implementation.

## D-027: Record the core branch verification evidence and limitations

- **Status:** Accepted
- **Decision:** Accept the core valuation stack after the following exact local verification
  commands all exited zero:

  - `cargo test --profile ci-pr -p nautilus-model --lib -j1`: 3,087 passed, 0 failed,
    1 suite-marked ignored test;
  - `cargo test --profile ci-pr -p nautilus-execution --lib -j1`: 596 passed, 0 failed;
  - `cargo test --profile ci-pr -p nautilus-risk --lib -j1`: 26 passed, 0 failed;
  - `cargo test --profile ci-pr -p nautilus-portfolio --lib -j1`: 64 passed, 0 failed;
  - `VIRTUAL_ENV= uv run --no-sync pytest -q tests/unit/model/test_instruments.py
    tests/unit/model/test_position.py`, from `python/`: 117 passed;
  - `make format`: Rust and Python formatting passed, with 739 Python files unchanged;
  - `make pre-commit` with
    `CHANGED_BASE_SHA=73d4686f816c893cda24d61fe6b67bdbf0746131`: every hook passed,
    including Python collection, Clippy, Cargo docs, cargo-machete, formatting, security, and
    repository conventions;
  - `git diff --check` and committed/staged/working-tree inspections: passed and clean.

- **Rationale:** This is the durable evidence required before later OKX adapter branches depend on
  the core semantics. The single ignored model test is an expected suite marker, not a failure, and
  was unchanged by this branch.
- **Alternatives:** Preserve results only in ignored harness files; rely on hosted CI; report only
  focused tests.
- **Cost if wrong:** These results are environment-specific and do not substitute for demo/live OKX
  qualification of later adapter changes.
- **Limitations:** No live venue request was made. No Arrow/SQL schema, FFI header, generated
  wrapper, or `.pyi` file changed. Python linking required the D-014/D-019 task-local environment,
  and generated Cargo profiles may need rebuilding after D-020 cleanup.
- **Reversibility:** Append a superseding evidence entry if a later final-HEAD rerun changes any
  result; never rewrite the recorded observation.

## D-028: Count only a loaded local extension as Python behavioral evidence

- **Status:** Accepted
- **Decision:** Do not count the first Python factory invocation as a behavioral RED when Python
  could not import the branch's native extension. Build and install the branch-local debug extension
  into the ignored project environment first, then count only tests which execute the changed Rust
  constructor through that extension.
- **Rationale:** An import/link failure proves an incomplete test environment, not the absence of
  the requested configuration behavior. The later test exercised a real `LiveNode` construction
  with `instrument_families` crossing the Python-to-Rust boundary.
- **Alternatives:** Report the import error as RED; mock the Rust configuration object; defer the
  Python boundary to hosted CI.
- **Cost if wrong:** Building the extension adds substantial local compile time and disk use.
- **Reversibility:** Future wheel-based verification can replace the editable debug build while
  retaining the rule that behavioral evidence must reach the production boundary.

## D-029: Clear only stale package artifacts when a focused Rust test reuses a RED diagnostic

- **Status:** Accepted
- **Decision:** When the focused PyO3 configuration test continued to report the already-fixed
  constructor arity, run `cargo clean --profile ci-pr -p nautilus-okx` and rebuild that exact test.
  Do not delete source, Git state, the Python environment, or unrelated Cargo profiles.
- **Rationale:** The source and compiler invocation showed the new argument, while the diagnostic
  was byte-for-byte the earlier RED result. A package/profile-scoped rebuild distinguished stale
  generated output from a current compiler failure and then passed the test.
- **Alternatives:** Edit correct source to accommodate the stale diagnostic; delete the entire
  target tree; accept the test without a clean rebuild.
- **Cost if wrong:** The adapter and its dependencies must be rebuilt for the `ci-pr` profile.
- **Reversibility:** Cargo recreates the removed package artifacts deterministically.

## D-030: Validate the generated Python stub as part of the candidate change

- **Status:** Accepted
- **Decision:** Generate and stage `python/nautilus_trader/adapters/okx/__init__.pyi` before running
  the repository's generated-drift check. Require the stub to expose the appended constructor input
  and the three new read-only getters, and verify its runtime declarations with the existing stub
  tests.
- **Rationale:** The drift checker compares regenerated output with the Git index. Leaving an
  intended generated change only in the worktree creates an expected false failure and does not
  model the candidate commit that CI will inspect.
- **Alternatives:** Hand-edit the stub; ignore generated drift; change the checker to compare the
  worktree.
- **Cost if wrong:** Exact-path staging must be kept synchronized with its Rust source during
  iteration.
- **Reversibility:** Regenerate the stub from the final Rust declarations and restage the exact
  file.

## D-031: Use the repository-pinned Markdown hooks after standalone npm integrity failure

- **Status:** Accepted
- **Decision:** Treat `npm ECOMPROMISED: Lock compromised` from the standalone `make
  check-markdown` download path as an environmental failure. Run the already-pinned repository
  `markdownlint` and Markdown-table pre-commit hooks over the exact changed documentation instead,
  and require both to pass.
- **Rationale:** The pinned hooks apply the repository's authoritative versions and configuration
  without trusting the failed transient npm resolver. This preserves the intended formatting gate
  rather than weakening or skipping it.
- **Alternatives:** Retry an unpinned npm install; skip Markdown validation; change repository
  dependencies during the feature.
- **Cost if wrong:** The standalone Make target itself remains unproven in this environment even
  though its two substantive checks pass through their canonical hooks.
- **Reversibility:** Rerun `make check-markdown` when npm's integrity state is healthy and append
  the result without rewriting this observed failure.

## D-032: Accept the strict option-family bootstrap verification evidence

- **Status:** Accepted
- **Decision:** Accept the bootstrap slice after all of the following final-HEAD checks exited zero:

  - focused option-family configuration/constants tests: 32 passed;
  - focused PyO3 configuration-constructor test: 1 passed through the branch-local extension;
  - Python factory and public-module-name tests: 41 passed;
  - generated Python stub drift and runtime declaration checks: passed;
  - `cargo test --profile ci-pr -p nautilus-okx --all-targets --features python -j1`: passed across
    every adapter target, including 732 library tests, 120 execution integration tests, 105 HTTP
    tests, 46 WebSocket tests, the Python boundary test, all remaining integration targets, and
    benchmark smoke targets;
  - `make format`: Rust and Python formatting passed, with 739 Python files unchanged;
  - `CHANGED_BASE_SHA=d539fb88cbb8738327a7b024fe80fd340a1f9707 make pre-commit`: every
    exact-base hook passed, including Python collection, Rust/Python formatting, Clippy, Cargo docs,
    cargo-machete, generated/convention checks, security checks, and Markdown validation; and
  - `git diff --check` plus tracked working-tree inspection: passed and clean before this evidence
    entry.

- **Rationale:** Strict family selection is the startup boundary on which every later registry,
  preflight, reconciliation, settlement, and risk slice depends. Full adapter and exact-base gates
  are required in addition to the focused constructor tests.
- **Alternatives:** Rely only on focused tests; defer full adapter verification to hosted CI; leave
  the result only in the ignored planning harness.
- **Cost if wrong:** These results prove local offline behavior, not OKX demo/live account mode,
  permissions, endpoint availability, or option-family enrollment.
- **Limitations:** No venue request was made, no credential was used, and no GitHub-side action was
  taken. The later production stack still requires its own full integration and demo qualification.
- **Reversibility:** Append a superseding final-HEAD evidence entry if a later dependency or
  integration rerun changes the result; never rewrite this observation.

## D-033: Describe bootstrap family validation without inventing a venue grammar

- **Status:** Accepted
- **Decision:** Describe the bootstrap resolver as requiring a non-empty list of nonblank,
  unpadded, unique strings. Do not call that check a complete canonical OKX family grammar. The
  strict Portfolio Margin stack separately requires the exact configured family `BTC-USD`.
- **Rationale:** The resolver deliberately prevents missing or broad OPTION discovery without
  guessing an exchange grammar that is not represented by a typed, versioned model. Calling every
  other malformed spelling "invalid" overstates what the code proves.
- **Alternatives:** Add an ad hoc regular expression in the bootstrap; accept trimming or
  deduplication; leave the documentation broader than the implementation.
- **Cost if wrong:** A nonblank, unpadded, unique but venue-invalid string can reach the ordinary
  discovery request and be rejected there. It cannot make the later strict PM client
  order-capable, which accepts only `BTC-USD`.
- **Reversibility:** A future typed family parser can strengthen the general resolver with explicit
  compatibility tests and supersede this entry.

## D-034: Freeze the reviewed stack contracts by content hash

- **Status:** Accepted
- **Decision:** Use the following SHA-256 digests as the exact reviewed design checkpoint:

  - instrument registry: `f7a7ccb41b141deed2d675b14c26dc600b96b7fd8a2e5cf3a8021d993eed6876`;
  - order preflight: `5c6e9229f6c521f14073866b90a1fab2ddd65e5548f005466328c573c93e26b9`;
  - private reconciliation: `849c9135bd0716fe5705780e2c6d5a5b3ea59b4e71af2da9166f15cd0baea130`;
  - risk and MMP: `975ab076b619a8d792e1f2e94aed84cf4da6dfc8ef3953a0cfc12a2065169848`;
  - expiry settlement: `cbdfe2b9928f3724244aff1fe8d09bbb075b8a2def0613e481a5fc20f7737207`;
    and
  - ordered implementation plan: `9d6b4b6589f9be5cc908cae76bb279f3356a87d81c29c98582f64830b1828ca2`.

  All five contracts received a final adversarial review with no remaining Critical or Important
  finding. The five contracts and plan passed the repository-pinned Markdown and table checks.
  They remain local ignored planning artifacts under `docs/superpowers`; do not force-add them.
- **Rationale:** Hashes bind this tracked record to the exact locally reviewed inputs without
  overriding the repository's ignore policy for agent planning artifacts.
- **Alternatives:** Force-add ignored files; commit an unreviewed summary; begin implementation
  without freezing the contract revision.
- **Cost if wrong:** A future checkout contains the accepted decisions but not the ignored source
  text; resumption requires the local artifacts or independently supplied byte-identical copies.
- **Reversibility:** Append new hashes and a superseding decision after any reviewed contract edit.

## D-035: Keep V1 feasibility conditional and deliberately narrow

- **Status:** Accepted
- **Decision:** The intended first qualified lane is OKX Portfolio Margin, Cross margin, Net position
  mode, family `BTC-USD`, BTC collateral, coin-margined options, and individually submitted limit,
  GTC, or post-only orders. Initial production qualification is LongOnly. Market orders, shorts,
  place-order lists, batch amend, and production MMP remain disabled until their separate contracts
  and venue capabilities are proved. Strict order placement remains blocked by default.
- **Rationale:** The core premium-valuation and bootstrap commits make the project technically
  feasible, but correctness still depends on durable reconciliation, PM risk, expiry, account, and
  recovery properties which do not yet exist in production code.
- **Alternatives:** Enable every OKX option feature together; treat bootstrap discovery as trading
  readiness; permit generic adapter behavior to bypass the strict lane.
- **Cost if wrong:** The first release supports fewer strategies and may require later capability
  migrations for shorts, market execution, and batch operations.
- **Reversibility:** Expand only through explicit versioned capabilities, tests, and demo/live
  qualification; never widen an existing strict capability implicitly.

## D-036: Make the locked option store a mandatory safety boundary

- **Status:** Accepted
- **Decision:** Require one configured absolute option-store path, an account/environment/family
  fingerprint, exclusive writer lock and term, and non-destructive schema migration before strict
  startup. A brand-new store uses `BootstrapPending`: install without replacement, fsync the parent,
  reopen and fully verify bytes, then atomically assign the first writer term and clear pending.
  Restart from term zero/pending repeats parent sync and verification. Missing, corrupt,
  incompatible, or migration-incomplete stores fail closed; recovery uses a separately fenced
  transport identity and never converts uncertainty into absence.
- **Rationale:** Orders, publications, source watermarks, risk leases, and settlement cannot be made
  crash-safe from in-memory state or a silently recreated database.
- **Alternatives:** Relative/default paths; destructive reset on mismatch; process-local locks;
  inference from a missing store.
- **Cost if wrong:** Operators must provision and retain an additional durable store and explicitly
  resolve identity or migration failures.
- **Reversibility:** A later store version may migrate through shadow-copy verification and atomic
  activation while preserving every old byte until acceptance.

## D-037: Admit lifecycle and command effects through durable publication workflows

- **Status:** Accepted
- **Decision:** Persist normalized source rows, entity predecessor/version allocation, immutable
  prepared publication payloads, backend receipt groups, and per-consumer acknowledgements. Startup
  hydrates unresolved workflows before new work. Strict place first enters its durable
  `AdmissionPending` workflow; strict modify/cancel first enter their durable proposal inbox. Every
  durably accepted modify/cancel proposal eventually produces one receipt-bearing command outcome,
  even when no schema-valid native lifecycle rejection exists. Native lifecycle publications
  complete after their projection-applicable consumer receipts; proposal rejection additionally
  requires the Execution command-outcome and durable Trader/Strategy ticket receipts.
- **Rationale:** A venue update and its Portfolio, analyzer, cache, and strategy consequences must
  not fork across crashes, retries, backend partial failure, or terminal-order races.
- **Alternatives:** Direct WebSocket callbacks; best-effort rejection events; memory-only pending
  commands; treating backend enqueue as application.
- **Cost if wrong:** The workflow adds durable states, receipts, replay logic, and blocked alerts.
- **Reversibility:** Additional consumers or backend adapters may be versioned into later receipt
  groups without weakening existing acknowledgements.

## D-038: Sequence modify and cancel recovery before venue invocation

- **Status:** Accepted
- **Decision:** Core proposal admission is the first durable side effect for modify/cancel. For a
  position-changing modify, the draining actor then acquires the deterministic
  position-changing lease and any separate amend-delta reservation in one option-store transaction
  before HTTP precheck or venue invocation. Modify failure before allocation emits a receipt-bearing
  `NotAllocated` outcome. Definitive modify rejection releases only the exact child reservation and
  lease, with a durable release receipt; it never rewrites the old live-order basis or fill debit.
  Place/cancel ambiguity is resolved by a durable arbiter, and all cancel sources share one target
  claim and attempt receipt. Cancel allocates no position-changing lease or amend child and has risk
  release `NotApplicable`. Batch cancel groups are atomic at the command-outcome layer.
- **Rationale:** Risk state cannot precede an uncertain core proposal, and retries must resume the
  same command/hold instead of double-reserving, double-cancelling, or losing a terminal outcome.
- **Alternatives:** Allocate risk before core admission; invoke cancel independently per source;
  infer release from a transient error.
- **Cost if wrong:** The command path has more serialized durable transitions and can block for
  operator resolution rather than guess after ambiguous outcomes.
- **Reversibility:** Concurrency may be widened only after proving equivalent per-target and
  per-position serialization.

## D-039: Recompute Portfolio Margin risk for every position-changing request

- **Status:** Accepted
- **Decision:** Every strict place or amend, including an apparently reducing order, requires fresh
  venue account/position/order evidence and projects both IMR and MMR. V1 permits one
  position-changing admission lease per policy scope and holds exact BTC premium and fee amounts.
  Release from account evidence requires a strictly newer authoritative account `uTime`. A
  `SentOrAmbiguous` place may be proved absent only when the calibrated lower bound is strictly
  after its immutable `expTime`, the writer generation is fenced and released, all WebSocket,
  pending, and history proofs agree, and the current upper bound is still strictly inside the
  two-hour history-retention window. Equality or stale calibration blocks.
- **Rationale:** Portfolio Margin is account-wide and nonlinear; labels such as reduce-only cannot
  substitute for current venue evidence or causal release.
- **Alternatives:** Static per-order margin; cache-only precheck; immediate timeout release;
  concurrent leases in V1.
- **Cost if wrong:** Safe admission is conservative and may reject or pause trades during stale
  evidence, clock uncertainty, or history outages.
- **Reversibility:** Calibrated model/rate improvements may increase concurrency after replay and
  failure-injection tests prove the same safety invariant.

## D-040: Preserve source lower bounds through reconciliation overflow and restart

- **Status:** Accepted
- **Decision:** Reconciliation uses strict endpoint-specific pagination cursors, frozen generations,
  durable attempts, and a fair/coalesced shared quota scheduler. Before rejecting a frame which
  crosses a configured count or byte bound, perform a bounded single-frame parse sufficient to
  persist `UnjoinedUpdateLowerBoundV1`; if parsing is impossible or prohibited by the resource
  bound, persist `RecoveryGapMarkerV1` with raw digest and length. Neither condition may advance a
  normal watermark or authorize absence. External principal activity remains an operational
  invariant: unexplained account mutation blocks the strict scope instead of being assimilated.
- **Rationale:** The update that triggers overflow may be the only durable proof that a venue event
  exists. Dropping it would turn overload into false absence after restart.
- **Alternatives:** Discard the overflow frame; restart pagination from local receipt time; share
  mutable cursors between endpoint generations.
- **Cost if wrong:** Recovery may remain blocked until retention-safe evidence or operator action is
  available.
- **Reversibility:** Bounds and parsers may be raised or optimized while retaining the exact gap and
  lower-bound semantics.

## D-041: Treat MMP as a durable circuit, not an order convenience

- **Status:** Accepted
- **Decision:** MMP enablement requires an exact qualified venue/account/session capability and a
  durable breaker workflow. Trigger, cancellation coverage, acknowledgement, retry, cooldown,
  rearm, and generation fencing survive restart. No order is considered covered merely because its
  local attributes resemble a protected quote.
- **Rationale:** A process-local flag cannot prove venue protection or prevent stale sessions from
  rearming after a crash.
- **Alternatives:** Enable MMP optimistically; model it as a normal cancel-all; infer state from
  current open orders.
- **Cost if wrong:** MMP stays unavailable until demo/live capability evidence and failure-injection
  tests are complete.
- **Reversibility:** Add qualified venue variants behind new exact capability versions.

## D-042: Gate expiry settlement on a frozen lifecycle and authoritative absolute target

- **Status:** Accepted
- **Decision:** Expiry first freezes the family through cancellation plus two complete sweeps and a
  durable cutover. Settlement uses lifecycle generation, birth/source transition, nonblank current
  `cTime`, final entity heads, and an authoritative positions-history absolute BTC `realizedPnl`
  target joined to required bills. The target environment must prove `cTime` distinguishes
  close/reopen; otherwise V1 auto-settlement is blocked. A dispatch token orders late pre-expiry
  lifecycle edges against core commit. Before invocation, an edge invalidates and rebuilds the
  freeze; once the replacement freeze is Frozen, one CAS may requalify the unchanged retained
  source against the unchanged applied head only after proving the old token was revoked and never
  issued. At or after invocation uncertainty, no ordinary append is allowed. Settlement paths use
  the same fallible scratch-projection transition, apply absolute BTC PnL exactly once, flatten
  volume exactly once, and mark price return unavailable instead of fabricating a return. Account
  reconciliation is two-stage: premerge account evidence is comparison-only, and a fresh
  post-lifecycle account request supplies final evidence.
- **Rationale:** Options can expire while late private data is still in flight; source equality,
  flat/reopen reuse, client-response loss, and account ordering must not strand settlement or apply
  money/volume twice.
- **Alternatives:** Settle from bills alone; use receive time; accept blank/reused lifecycle
  metadata; revoke an already-issued call; reuse the premerge account snapshot.
- **Cost if wrong:** Automatic settlement intentionally blocks when the venue cannot supply the
  required lifecycle discriminator, retention window, quota capacity, or authoritative evidence.
- **Reversibility:** A later version may introduce an independently proved lifecycle discriminator
  or operator-authorized full replay/correction workflow without weakening V1.

## D-043: Pause at the reviewed design checkpoint

- **Status:** Accepted
- **Decision:** Stop before the first production implementation slice. Commit this tracked decision
  log as the local checkpoint on `feat/okx-pm-btc-option-stack`, leaving the reviewed contracts and
  plan ignored and hash-anchored. Do not push, open a PR or issue, merge, use credentials, or make a
  demo/live OKX request. Resume with the ordered test-driven configuration/store slice only after a
  new user instruction.
- **Rationale:** The user explicitly requested a pause after finishing and committing the task at
  hand. The completed task is contract closure and adversarial review, not production integration.
- **Alternatives:** Begin implementation before pausing; force-add planning artifacts; publish the
  branch to GitHub.
- **Cost if wrong:** No production option-trading capability is added by this checkpoint commit.
- **Reversibility:** Resume from the hash-bound plan, or supersede any contract through a new review
  and appended decision before implementation.

## D-044: Publish the checkpoint after explicit user authorization

- **Status:** Accepted
- **Decision:** Publish `feat/okx-pm-btc-option-stack` to `origin` only after the user's subsequent
  explicit push instruction. The workspace has no HTTPS Git credential, so the ordinary non-force
  `git push` cannot authenticate. Use the connected GitHub app to replay the complete logical commit
  sequence from pinned `develop` base `73d4686f816c893cda24d61fe6b67bdbf0746131`, preserving every
  commit message and tree state, then create the previously absent remote branch. The app cannot
  preserve local author/committer metadata, so remote commit IDs will differ; require the final
  remote tree hash to equal the local HEAD tree hash exactly. Do not force-update, merge, or open a
  pull request.
- **Rationale:** This publishes the authorized checkpoint through the installed GitHub connection
  without exposing credentials, weakening branch safety, squashing the audit sequence, or claiming
  that different commit objects are identical.
- **Alternatives:** Ask the user to configure a workspace Git credential; publish one squashed
  snapshot; leave the checkpoint local.
- **Cost if wrong:** The local and remote branches contain the same final files but have different
  commit identities. A later conventional push requires an explicit history-reconciliation choice;
  it must not be forced automatically.
- **Reversibility:** A credentialed maintainer may create a new exact-history branch or explicitly
  reconcile the two histories after verifying tree equivalence.
