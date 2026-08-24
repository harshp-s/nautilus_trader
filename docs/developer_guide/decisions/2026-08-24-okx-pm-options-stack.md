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
