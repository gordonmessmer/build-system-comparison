# Fedora as a developer-first build platform

*A proposal to make `mock` and `dist-git` tools developers want to use — and to
turn Fedora's build system into a shared component registry for the Linux
ecosystem.*

Status: draft for discussion.

## Vision

Fedora should grow from *a project that provides users with software* into *a
project that also provides developers with the tools to build software*.
Concretely: enable developers to **publish** software of their own and to
**consume** the software they need with as little friction as PyPI, crates.io, or
Go's decentralized publishing offer today — but with Fedora's strengths
(human-maintained streams, build provenance, a coherent base) underneath.

The vehicle is a **language-agnostic, developer-first shared component registry**:
by carrying upstream-release branches in dist-git, Fedora gains a general-purpose
build backend with the version and compatibility semantics developers already
expect from language registries — a service the Linux ecosystem lacks today and
that Fedora is well positioned to provide. The rest of this document works out the
build-and-delivery model that makes that registry real.

## Problem statement

The near-term goal is **usability**: make `mock` and `dist-git` tools that
developers actually want to reach for — not just tools that distro packagers
tolerate. This is the concrete first step toward the vision above.

**Mechanism: decouple applications from the OS.** The deepest source of friction
today is that applications are tightly coupled to the OS — they build against, and
must move in lockstep with, whatever the distribution currently ships. That
coupling is what makes Rawhide painful to build against, and it is also a security
liability: an application that cannot advance its own components independently of
the OS is an application that is hard to update, and hard-to-update software is
insecure software. Decoupling applications from the OS is the mechanism that
delivers the vision — it makes application builds easier, makes applications easier
to keep patched, and turns dist-git into the shared component registry described
above.

**Scope: this does not replace the distribution.** The base OS and its shared
libraries still come from normal Fedora. What we are adding is an *application*
build-and-delivery layer that sits on top of a classic RPM base: it consumes that
base as a floor, builds applications and their private dependency closures from
source, and ships them as self-contained images. The target artifact is a leaf
application, so self-contained delivery is the natural fit.

Security is a first-class requirement throughout: any design here must keep
Fedora's security posture at least as strong as today's, and we expect it to
*improve* it — consuming upstream security patches as they're published,
recording exact component combinations for CVE discoverability, and tracking the
maintenance status of every version in the graph. Usability and security are
pursued together, not traded off.

## The lens: software is incomplete

Our organizing idea: source code is incomplete. Parts of it must be *completed*
by external components — at build time (our focus here) and at runtime (out of
scope for now).

Every build system answers the same questions about how that completion happens.
The differences between a system developers love (cargo, npm, Go modules) and one
they merely endure (mock + dist-git) live in those answers. So the lens for the
whole comparison is: **how does each system name the needed components, declare
and use compatibility, decide when/where completion happens, and who bears the
work?**

## The questions every system answers

1. **Who names the needed components?** Upstream author vs. distro packager.
2. **Who declares version compatibility, and how expressively?** Machine-readable
   ranges that travel with the source (`^1.2`, `>=1.2,<2`, `!=1.4.0`, yanked
   releases) vs. a human-maintained `Requires:`/`BuildRequires:` against distro
   names. *Reframed: the single-version values argument is not absolute — it is
   conditional on upstream maintenance posture.*
3. **When/where does completion happen?** Fetched from a registry at build time
   vs. pre-staged into a sealed, network-isolated buildroot from one coherent set.
4. **Is coexistence of multiple versions the default or the exception, and at what
   friction?** (Not a yes/no — every system here *can* coexist. See note below.)
5. **What gets pinned?** Lockfile + content hash vs. content-addressed derivation
   vs. an *implicit* buildroot tag.
6. **Who bears the integration/compat work, and is it tested?** Upstream CI
   exercising real version ranges vs. a packager translating and patching after
   the fact.
7. **(Maintenance topology)** Is each version in the graph backed by a
   security-maintained upstream *stream*, and does the system track stream
   support / EOL as first-class metadata?
8. **(Discoverability mechanism)** Is "who is affected by this CVE?" answered by
   *invariant* (a single shared version) or by *manifest* (build-time-recorded
   combinations / SBOM)?

Questions 1, 2, 6 drive usability. Questions 7 and 8 are where the design earns
its security properties — consuming patches as they're published and answering
"who is affected?" by construction. Both sets matter; a proposal that improves
one while regressing the other is not acceptable.

## Reference systems

| System | Role in the comparison |
|---|---|
| **Cargo / crates.io** | Cleanest specimen of declared compatibility: author declares ranges in-tree, resolver (PubGrub) concretizes, lockfile pins exact + hash, multiple versions free & default, fetched at build. The "float-within-a-range" sub-model. |
| **Go modules** | "Name-a-stream" sub-model: import path *is* the repo URL, major version lives in the path (semantic import versioning), can reference a branch. **Guardrail:** default resolution is Minimal Version Selection — conservative, pinned in `go.mod`/`go.sum`; security fixes come via `go get -u` / `govulncheck`, not auto-floating. |
| **Fedora (mock + dist-git)** | The photographic negative. dist-git stores recipe + source artifact, not upstream constraints. mock/Koji build in a **network-isolated** buildroot; build-time third-party fetch is forbidden by design. Compatibility flattened to "whatever Fedora N ships." |
| **Nix / Guix** | Dissolves single-vs-multi: a derivation *is* "incomplete software + exact content-hashed inputs"; `flake.lock` = whole-graph lockfile; multiple versions coexist in the store. Keeps a coherent set (nixpkgs) but with lockfile discipline. Still curator-declared compat (closer to Fedora on Q1/2/6). |
| **Portage / Gentoo** | Richest *distro-native* constraint language: versioned deps, **SLOTs** (parallel versions), **USE flags** (conditional completion), and it fetches + builds from upstream at build time. Exposes the friction of wrapping cargo/npm (historic giant `CRATES=` lists). |
| **Spack** (HPC) | "Declare version-dependent constraints, then *concretize a spec*" into any of many coexisting variants. Sits deliberately between distro and language ecosystem on every question — a non-distro fourth data point. |

## Key framing points

- **Q4 is a friction gradient, not a binary.** Fedora *can* carry multiple
  versions (compat packages with version-bearing names like `rust-pyo3_0.25` vs
  `rust-pyo3`, `compat-openssl11`, parallel `python3.x`) via virtual Provides so
  dependents resolve correctly. It's possible but exceptional and high-friction.
  Stating this plainly matters: Fedora already supports multi-version coexistence —
  just at high, manual cost.

- **Name-encoded versioning is a shared technique.** Fedora's versioned package
  names and Go's semantic import versioning (`.../v2`) are the *same move*: push
  the compatibility boundary into the name. Fedora does manually what Go
  automates.

- **Two language-ecosystem sub-models.** *Float-within-a-range* (cargo, npm, pip)
  vs. *name-a-stream* (Go). Both beat Fedora's flattening, for different reasons,
  and are worth keeping distinct.

- **The "one place to fix a CVE" justification conflates two things:**
  *topology* (how many copies exist) vs. *maintenance* (whether each copy is
  actively security-patched upstream). Single-version is really insurance against
  *unmaintained duplication*. If upstream maintains multiple release branches, N
  versions are N maintained streams, not N liabilities — the distro's job shifts
  from **patcher** to **tracker** (which streams are supported + their EOL).

- **Fedora already concedes the single-copy invariant.** Compat packages break
  it; the `bundled(foo) = 1.2.3` virtual Provide exists precisely to *record*
  embedded copies for CVE discoverability. So the invariant is already
  aspirational, with a weak, manual fallback mechanism.

- **Manifest substitutes for invariant.** The surviving real benefit of
  single-version is *discoverability* ("which artifacts embed the vulnerable
  thing?"). A build that records its exact component combination (lockfile / SBOM)
  delivers that by construction — which is what single-version tried to guarantee
  by fiat. So single-version can be relaxed *if* (a) upstreams maintain streams
  and (b) builds record combinations. Fedora has neither as a first-class default
  today — that's the opening for the proposal.

- **Fedora has already started bridging.** `%pyproject_buildrequires` reads PEP
  621 / setuptools metadata and generates BuildRequires from upstream's declared
  constraints; `rust2rpm` + cargo macros translate `Cargo.toml`; nodejs tooling
  similarly. The machinery to *ingest upstream machine-readable compatibility*
  exists — but it's per-ecosystem and lossy (can't represent "A needs foo<2 while
  B needs foo>=2"). This is the strongest concrete illustration of the gap.

## Target workflow — principles (implementation left open)

The following are the design principles we want the application build-and-delivery
workflow to satisfy. They are deliberately stated as principles; concrete formats,
command surfaces, and mechanisms are open questions to be worked out with Fedora
developers.

1. **Decouple applications from the OS.** An application builds against a chosen
   base but is not locked to the distribution's current version of everything; it
   can advance its own components independently of the OS. This is the organizing
   principle — the rest follow from it.

2. **Declare dependencies by source + stream, not by binary or fixed version.** A
   build manifest lists the components an application needs as *(component,
   branch)*, where the branch names an upstream release stream (or `main`/`rawhide`
   for mainline). The declaration travels with the application source.

3. **Source is the authoritative input; binaries are a private, per-application
   cache.** A binary built in a different or earlier buildroot may link against
   libraries that are not in the set we need — and RPM's automatic dependency
   generation means a built package's requirements depend on whatever was present
   in the buildroot. So dependency declarations refer to source, never to pre-built
   binaries, and **binaries are not shared between applications**: reliable reuse
   would require an identical base *and* an identical leading set of manifest
   components, which is rarely true and not worth engineering for. Each application
   keeps its own private RPM repo for the components named in its build manifest;
   the "cache" is simply the builds already produced from that manifest. This is
   also what makes multi-version safe for C/C++.

4. **Follow streams; do not pin.** There is no developer-committed lockfile; it is
   enough to follow a stream as long as it stays compatible. The *stream branch
   itself is the compatibility contract*, maintained by people: a Fedora maintainer
   holds back or reverts a bad upstream patch, which is exactly their strength.
   Picking up security patches simply means following the branch. Reproducibility is
   handled exactly as the distribution handles it today — via retained source RPMs —
   so this proposal neither weakens nor re-invents it.

5. **The platform is a floor, not a pin.** The base (a Fedora release) supplies a
   coherent set of inputs and sets a *floor* on versions. Going newer than the
   floor is easy — name the component+branch you need. Going older is hard and
   discouraged. So the developer workflow is: choose a platform at or below your
   compatibility ceiling, then advance selectively upward only the components you
   need newer.

6. **Automate the upstream-release branches.** Creating dist-git branches that
   track upstream releases is largely automatable, so the existing spec+tarball+
   patch model is sufficient and branch proliferation is an automation task, not a
   human burden. source-git is a good fit and should be *enabled*, but it is not
   required.

7. **The application rebuild is the integration gate.** Following a stream does not
   push untested updates at users. Stream patches are picked up when the
   application is rebuilt, and that rebuild is where compatibility is verified
   before anyone receives the result. Implication: rebuilds should be cheap and
   automatable, so security-patch consumption (which happens at rebuild) is easy.

8. **Discoverability comes from build provenance, not a committed lockfile.** The
   build records exactly which component/branch/commit went into each image, and
   images carry or reference that record (an SBOM). "Which images are affected by
   this CVE?" is answered from provenance (Q8) — so dropping the lockfile does not
   drop discoverability.

9. **Ship leaf applications as self-contained images.** Results are not added to
   shared Fedora repos (that would reintroduce the file-conflict problem); an
   application and its private dependency closure are kept in a private repo and
   delivered as a container image. Coexistence then lives *across* images; within
   any one image there is a single coherent resolved set, so no conflicts arise.

10. **Bring registry ergonomics to ecosystems that have no registry.** For C, C++,
    and anything else without a language registry, the distribution's
    stream-branched source collection plays the role crates.io / npm play — the
    same declare-and-resolve experience, with no external registry required.

11. **Preserve the build sandbox; add logic around it, not inside it.** The
    isolated, distro-populated buildroot contract stays intact. New capability
    lives in the resolve/stage steps that feed the build, minimizing risk to the
    trusted build path.

12. **Shrink the translation boundary.** Where upstream already encodes dependency
    knowledge, ingest it directly rather than require a packager to re-express it
    by hand; packager effort shifts to handling genuine exceptions.

13. **Hold security equal or better throughout.** No principle above may weaken
    Fedora's security posture; several (stream-following patch flow, build
    provenance, independent application updates) are intended to strengthen it.

14. **Independent upgrade schedules at every layer.** The base-platform cadence is
    a *fixed input*, not a design variable: Fedora continues to ship a new release
    about every six months and maintain it for about thirteen (roughly two
    generations supported at once). On top of that fixed floor, upgrade schedules
    are independent:
    - *Application developers* choose when to rebase onto a newer base generation,
      driven by their application's compatibility, not the OS calendar.
    - *Users/endpoints* upgrade applications independently of their machine's OS
      version, because applications are delivered as container images (including
      Fedora Flatpaks on the Fedora runtime). Upgrading — or not upgrading — the
      endpoint does not gate access to applications.

    Consequence: because rebases happen per-application rather than synchronized to
    an OS release, there is **no coordinated mass-rebuild event**. Rebuild cost is
    opt-in and spread across time — the opposite of today's Rawhide, where a base
    change can force everyone to rebuild at once.

## The component collection (the "store")

In this model the shared collection applications build against is not a set of
binaries — it is the set of **stream-branched source repositories**. Binaries are
*derived and private to each application* (principle 3), never a shared pool. That
reframes the earlier "store population" strategies:

- **Inventory = stream branches.** A component's available versions are its
  upstream-release branches (plus `main`/`rawhide`). Creating and updating these
  is largely automated (principle 6).
- **Demand-driven creation.** Most applications want the default stream; a
  non-default stream branch need only exist when some application actually needs it.
- **Coherence by rebuild, into a private per-application repo.** A chosen version
  is built from source against the selected base, into the application's own repo.
  Binaries are never shared between applications — RPM's automatic buildroot
  dependencies make cross-application reuse unreliable (principle 3). This is what
  retired the "reuse old Koji binaries" trap.
- **Federation is just recursive platform-as-floor.** See the next subsection.

### Federation — base images as floors

Federation needs no separate subsystem. The platform floor is simply a container
image, and our system already builds images from manifests — so an intermediate
**base image** (e.g. a desktop platform, or a language + stack base) is just
another floor that applications build `FROM`. A Fedora release image is one such
floor; a curated base image is another. Federation = *platform-as-floor applied
recursively.*

- **Richer base = more convenience, tighter version coupling.** The more a base
  includes, the more it fixes the versions of what it ships for its consumers. The
  minimal Fedora-release floor is maximally flexible precisely because it ships
  little above the core.
- **Overriding a base library is normally fine.** Because the output is a plain OCI
  image (not a Flatpak-style read-only shared runtime), an application can upgrade a
  base-provided library in its own layer. As long as the update is
  backward-compatible — which is exactly what *following a stream* gives you by
  construction — this is safe and requires no fork of the base. ABI-incompatible
  bumps usually coexist too (distinct sonames are distinct files); the only real
  hazard is *removing* an soname other base binaries still need, which is ordinary
  shared-library hygiene (handled with compat sonames), not a federation-specific
  problem.
- **Two-level patch flow.** A base image is itself rebuilt to pick up stream patches
  (principle 7), and applications then rebuild on the new base to inherit them. So
  bases need an owner and a signal to consumers that a rebuild is warranted — the
  same pattern as any container base image.
- **Trust is distributed but auditable.** Consumers inherit whatever is in a base,
  but build provenance / per-image SBOM (Q8) makes a base's exact stream/commit
  contents inspectable. What remains open is *policy*, not mechanism: signing and
  who may publish an "official" base.

## The matrix — how each system answers Q1–Q8

The proposed system is the final row, so its position relative to the references
is visible at a glance.

| | Q1 Names deps | Q2 Declares compat | Q3 Completion site | Q4 Coexistence | Q5 Pinning | Q6 Integration work | Q7 Stream maint. | Q8 Discoverability |
|---|---|---|---|---|---|---|---|---|
| **Cargo** | upstream author | semver ranges, in-tree | registry fetch at build | free, default (multi) | Cargo.lock + hashes | upstream, CI-tested | yank + RustSec advisories | manifest (lock/SBOM) |
| **Go** | upstream; path = repo | min-version + major-in-path | proxy/VCS fetch at build | majors coexist; MVS within | go.mod + go.sum | upstream, CI; MVS deterministic | Go vuln DB (govulncheck) | manifest (go.sum/SBOM) |
| **Fedora (today)** | distro packager | hand Requires; single version | sealed dnf buildroot | exception; hand-built compat | implicit buildroot tag | packager translates + patches | per-pkg CVE; `bundled()` | invariant (+ weak `bundled()`) |
| **Nix / Guix** | expression author | curator-pinned version | hermetic source/substituter | free, default (store hashes) | CAS derivation; flake.lock | nixpkgs curators; Hydra CI | per-pkg; pin any version | manifest (closure, strong) |
| **Portage** | ebuild maintainer | versioned deps, SLOT, USE | source fetch + build on host | SLOTs (blessed); else one | @world; no hash lock | maintainer; USE combinatorics | GLSA; slots keep old | mixed (installed DB) |
| **Spack** | author; user concretizes | version ranges, variants | source build; binary mirror | free, default (hashed prefixes) | concrete spec hash; spack.lock | maintainer + user variants | per-pkg; many versions kept | manifest (spec hash/env) |
| **Fedora (proposed)** | **application developer** | **by stream/branch; maintainer-kept contract** | **source build in sealed buildroot, from stream-branched dist-git** | **free across apps/images; one per image** | **none committed; provenance recorded** | **dev picks streams; maintainers keep branches; rebuild = gate** | **explicit: branches maintained; follow = patch; EOL = retire branch** | **manifest/provenance (per-image SBOM)** |

### What's novel vs. borrowed

- **Borrowed — Q1 (who names deps):** the proposal moves dependency naming to the
  *application developer*, matching the language-ecosystem rows. It is the only
  distro-side row to do so; this is the usability leap.
- **Borrowed with a twist — Q2 (compatibility model):** we adopt Go's
  *name-a-stream* model over cargo/Spack's numeric ranges, but the stream branch is
  a **human-maintained** contract (Fedora maintainers). That maintained-ness is
  what lets us drop pinning.
- **Novel — Q5 + Q7 together:** every other "modern" row buys reproducibility with
  a lockfile. The proposal is the only row that **drops committed pinning** (Q5 =
  none) and instead relies on maintained streams (Q7) plus build provenance (Q8).
  This is the most novel and most scrutinized cell: the claim is that *maintained
  stream + provenance* substitutes for *lockfile*. The next section examines it.

## Anticipated challenges from Fedora maintainers (the Q5+Q7 bet)

The design makes one genuinely novel bet: drop the committed lockfile (Q5 = none)
and rely instead on human-maintained streams (Q7) plus build provenance (Q8).
Below, each anticipated objection is stated at full strength, followed by our
response.

1. **"You've multiplied my maintenance surface, not reduced it."** *(the central
   objection)* Many concurrent stream branches per component each need continuous
   **gatekeeping** — simultaneous CVE triage/backport across streams and
   compatibility-vetting every upstream point release before it lands on a branch.
   Branch *creation* is automated; this *gatekeeping* is not. The model shifts a
   maintainer from patcher to N-streams-wide compatibility-and-security gatekeeper,
   and they will question whether that is sustainable.
   - **Response:** Automation helps more than it first appears, because branches
     frequently *coincide* — a Fedora release branch (`f44`) and an upstream
     `release-x.y` branch often point at the same release and can be trivially
     reconciled, so an extra stream is often near-zero extra work. More
     fundamentally, a large share of today's Fedora maintenance is the effort of
     making Fedora's *specific, upstream-untested* combinations of component
     versions work together. By letting applications build against combinations the
     upstream actually tested, this system should produce *fewer untested
     combinations* and therefore *less* patch management, not more.

2. **"Bundling makes remediation O(apps)."** In the shared-library
   world a CVE is patched once and everything relinks. With private per-application
   closures, the same CVE requires **every affected image to be rebuilt, reshipped,
   and re-pulled** — remediation effort scales with the number of embedding apps,
   the historical reason Fedora forbids bundling.
   - **Response:** Fedora *already* ships applications as container images (e.g.
     Flatpaks) and already rebuilds images to remediate — this is not a new burden
     in kind. Bundling into containers does mean more builds, but it reduces
     *maintenance* overhead, and the binding constraint for Fedora is **human
     labor, not compute cycles**. Compute for security is comparatively easy to
     provide; meanwhile many Fedora packages lag upstream today purely for lack of
     labor, which is the more serious security problem. Trading compute for labor is
     the right trade.

3. **"No pin means you can't reproduce or review a change."** Without a
   committed lockfile, inputs drift implicitly: a build that worked last week may
   differ today, with no committed artifact to deterministically recreate it, and no
   explicit diff to review/gate a dependency change.
   - **Response:** Fedora's reproducible-builds work is about build *determinism*,
     not the ability to recreate a build later. The system still produces **source
     RPMs**; to rebuild an application later, select the source RPMs that were used
     or created during that specific build rather than re-resolving the manifest.
     Lockfiles are not used in Fedora's current systems, and a new container-image
     build system need not require features the existing distribution lacks.

4. **"Discoverability is only as good as the SBOM — and that's hard."**
   Provenance-based discoverability requires every image's SBOM to be complete,
   accurate, retained, and queryable at fleet scale.
   - **Response:** The **private RPM repo** (everything above the floor) together
     with the **base platform SBOM** (the floor) is an adequate reference for SBOM
     processing. Because the closure is RPM-based end to end, discoverability stays
     a query over authoritative data (base SBOM ∪ private repo), not a
     reverse-engineering problem.

5. **"Who enforces EOL, and what's the bifurcation cost?"** Apps that
   keep following an EOL'd stream become a silent hole; and two dependency worlds
   (stream-following closures vs. single-version shared libs) mean two security
   processes and two mental models for responders.
   - **Response (EOL):** EOL processing is an *important feature Fedora lacks
     today* — 15–30% of packages are outdated, with no system tracking whether the
     version in use belongs to a still-maintained upstream release series. This work
     is worth doing regardless of whether this proposal is adopted: provide
     information about when release series go EOL upstream and how Fedora should
     react.
   - **Response (bifurcation):** The cost is worth it. It lets applications move
     ahead without disrupting the stability of the underlying platform, and
     asynchronous application development is a large productivity benefit. More
     importantly, the bifurcation is what makes the **shared component registry**
     possible — see the *Vision* section: providing upstream-release branches in
     dist-git yields a language-agnostic build backend and a developer-first
     registry, moving Fedora from *providing users with software* to *providing
     developers with the tools to build software*. The second dependency world is
     not a regrettable side effect; it is the deliverable.

Three ideas carry most of the weight across these responses: Fedora already
rebuilds containers for remediation, human labor is scarcer than compute, and
source RPMs provide both reproduction and provenance without a lockfile. The
deepest point is #5 — the second dependency world is not a cost to be excused but
the deliverable: the shared component registry described in the Vision.

## Design decisions so far

- **Spine:** decouple applications from the OS (usability and security together); an
  application layer on a classic base, not a replacement for the distribution.
- **Workflow:** declare dependencies by source + stream; follow streams rather than
  pin; treat the platform as a floor; ship applications as self-contained images.
- **Component collection:** the shared collection is stream-branched *source*;
  binaries are derived and private per application; branch creation is automated;
  stream branches are the human-maintained compatibility contract.
- **Discoverability:** build provenance / per-image SBOM (base SBOM ∪ private repo);
  no committed lockfile required.
- **Base cadence:** a fixed input — Fedora's existing ~6-month release / ~13-month
  maintenance model (≈2 concurrent generations). Upgrade schedules are independent
  per layer, so there is no coordinated mass-rebuild event.
- **Binary cache:** no cross-application sharing (RPM's automatic buildroot
  dependencies make it unreliable); each application has a private RPM repo scoped
  to its own manifest.
- **Federation:** recursive platform-as-floor via base images; overriding a
  base-provided library is fine for backward-compatible / stream updates on a plain
  OCI image, with no fork of the base needed.
- **Comparison:** the full Q1–Q8 matrix across cargo, Go, Fedora (today), Nix,
  Portage, Spack, and the proposed system, with a novel-vs-borrowed reading.
- **Maintainer challenges:** five anticipated objections to the Q5+Q7 bet, each with
  a response.

## Open questions

- **Base-publisher trust policy** — signing, and who may publish an "official" base.
  The mechanism and visibility are already covered by provenance/SBOM, so this is a
  policy question rather than a blocker. (Deferred unless Fedora asks.)
