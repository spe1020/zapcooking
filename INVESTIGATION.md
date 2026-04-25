# Sign-in Modal Refresh — Investigation

Branch: `investigate/signin-modal-refresh`. No production code touched. Stopping here for review per the brief.

---

## Blockers before implementation

These two need answers before the `feat/signin-modal-refresh` branch starts.

### B1. The visual spec doesn't exist in the repo

The brief points at `docs/design/signin-modal-v2.html` as the mockup. **There is no `docs/design/` directory at all** in the working tree (`origin/main` HEAD `dcc3141`), and no file matching that name anywhere in the repo. I can only build from the inline UI requirements in the brief, which describe the layout but not pixel-level styling, exact ramps for the gradient, the bolt-flicker animation curve, etc.

Two ways forward:
- (preferred) drop the HTML mockup into `docs/design/signin-modal-v2.html` and I'll match it
- or accept that I'll build from the inline brief alone and we iterate visually in PR review

### B2. The brief says "four sign-in methods" but the modal has five

The opening paragraph says "four Nostr sign-in methods (NIP-07 browser extension, 'Create Profile', QR/universal pairing, paste bunker URI, import key)" — that's five things in the parenthetical. The hierarchy section then maps them as:

- Primary: extension
- Secondary: create new
- Disclosure: QR/pair, bunker URI, import key

That arithmetic works. Going to assume the "four" was a typo and treat all five as in-scope. **Confirm.**

---

## 1. Where the modal mounts

- `/login` is a SvelteKit route at `src/routes/login/+page.svelte:1-1223`. There is no global overlay or header-triggered modal — the Header "Sign in" button at `src/components/Header.svelte:255` is a plain `href="/login"` link. Every other "sign in to ..." prompt across the app (PostComposer:495, NoteActionBar, ThreadCommentActions, etc.) navigates to `/login?redirect=...`.
- iOS uses a separate component: `src/components/LoginFormIOS.svelte:1-956`. The route swaps based on `$platformIsIOS` at `src/routes/login/+page.svelte:499`. The two flows are **structurally different**, not just style-different — see §5.
- **Implication for the refresh:** the new modal is really one route page. Calling it a "modal" in product copy is fine, but the markup is a full-page card today, not an overlay over content.

## 2. Sign-in method handlers

All five exist on the page; all five route through `authManager` (`src/lib/authManager.ts`). The new markup needs to wire to the same five.

| Method | Page handler | authManager call | Side effects |
|---|---|---|---|
| NIP-07 extension | `loginWithNIP07()` `+page.svelte:141-158` | `authenticateWithNIP07()` `authManager.ts:140-180` | `localStorage.nostrcooking_loggedInPublicKey`, NDK signer set |
| Create Profile (generate keys) | `useGeneratedKeys(skipProfile)` `+page.svelte:246-312` | indirectly via `authenticateWithPrivateKey` after key gen | `localStorage.nostrcooking_loggedInPublicKey` + `nostrcooking_privateKey`; publishes kind:0 metadata; multi-step backup→profile guard |
| QR / universal NIP-46 pairing | `startUniversalPairing()` `+page.svelte:320-341` | `startNip46PairingUniversal()` `authManager.ts:639-718` | `localStorage.nostrcooking_nip46_pending`; opens response listener; on Android, `window.open(uri, '_system')` deeplinks to signer |
| Paste bunker URI (NIP-46) | `loginWithBunker()` `+page.svelte:178-199` | `authenticateWithNIP46(connectionString)` `authManager.ts:332-471` | `localStorage.nostrcooking_nip46`, `nostrcooking_loggedInPublicKey`, `nostrcooking_authMethod='nip46'` |
| Import private key (nsec) | `loginWithPrivateKey()` `+page.svelte:160-176` | `authenticateWithPrivateKey(pk)` `authManager.ts:186-246` | `localStorage.nostrcooking_loggedInPublicKey` + `nostrcooking_privateKey` |

All can be invoked unchanged from new markup. The handlers don't reach into the DOM.

## 3. Modal management primitive

- A generic `<Modal>` component exists: `src/components/Modal.svelte:1-122`. The current login uses it for nsec/bunker/generate-keys/NIP-46-universal sub-modals via `bind:open={...} on:close={modalCleanup}`.
- It already provides:
  - `aria-modal="true"` (`:73`), `aria-labelledby="title"` (`:72`)
  - ESC close (`:35-51`)
  - Click-outside close on the backdrop (`:65`, `on:click|self={close}`)
  - Body scroll lock via inline style block (`:106-120`, `html { overflow: hidden; touch-action: none }`)
  - Portal-to-body via a custom action (`:3-13`)
  - Svelte `transition:` for the open animation (blur 250ms, scale 250ms — `:67`, `:71`)
- It does **not** provide a focus trap or initial-focus management. No focus-trap library in `package.json`. **Recommendation:** stay self-contained for this PR — implement the trap with a small handler over the modal's tab cycle (~25 lines of plain JS, no dep). If we later want a primitive, extract it then.
- The `<Modal>` API is already aligned with what we need (open binding + close event), so the refresh can keep using it without modifications.

## 4. Analytics / event tracking

**None exist.** Searched the login route, the iOS form, and `authManager.ts` for `track(`, `analytics.`, `posthog.`, `mixpanel.`, `gtag(`, custom event dispatches — zero hits. The current login fires no events.

The brief says "preserve every existing event" — there are zero, so nothing to preserve. Worth noting that this is a small architectural gap (we have no instrumentation on auth funnel) but explicitly out of scope for this UI refresh; flag separately.

## 5. Mobile / Capacitor considerations

- **iOS uses a different layout entirely.** `LoginFormIOS.svelte` is a full-page scrolling card (`:837-927`), not a viewport-centered modal with a backdrop blur like web. It also drops NIP-07 (App Store can't deliver an extension) and drops QR/universal pairing (App Store policy on cross-app handoff — see PR #340 / #331 history).
- iOS hierarchy today: Create Profile (primary, `:860-870`), then private-key import + bunker paste as secondary link-style buttons (`:874-889`).
- **Safe-area:** `src/app.css:18-23` defines `--safe-area-inset-*` CSS variables from `env(safe-area-inset-*)`. The current iOS form uses plain `pt-12 md:pt-20` (`:837`), not the safe-area variables. **UNKNOWN:** whether notched devices currently look right — worth a manual smoke during implementation.
- **Keyboard overlap:** no `visualViewport` workarounds, no scroll-on-focus handlers found. Capacitor's keyboard plugin presumably handles it implicitly. **UNKNOWN:** whether the bunker-paste textarea on iOS gets covered by the keyboard.
- **Platform-gated UI:** `supportsNIP46QRPairing()` (`src/lib/platform.ts:134`, `false` on iOS) and `supportsNIP46BunkerPaste()` (`:143`, true everywhere) are defined but **not called by the login route**. The platform gating today is implicit — the iOS form is a different file that omits the QR button. The refresh should call these helpers explicitly so the policy lives in one place.
- **Implication for the refresh:** we have to refresh **both** `+page.svelte` AND `LoginFormIOS.svelte`, OR collapse them into one component that uses the helpers. Collapsing is the cleaner end state but expands scope. Recommend keeping them as two files for this PR and refreshing both — same visual language, same `<Modal>` usage, but iOS continues to mount a stripped subset.

## 6. Risks / surprises

- **Form state lives on the page, not in the modal.** `nsecInput`, `bunkerConnectionString`, `generatedKeys`, profile fields are all page-level `let`s consumed by sub-modals via `bind:`. The refresh can keep this pattern (simplest) or hoist into a small store. Don't extract sub-modals into separate components without prop-drilling — the form/auth coupling is real.
- **Multi-step Create Profile flow.** `useGeneratedKeys()` is a multi-step modal: step 1 download backup → step 2 profile setup. The Next button at `:835` is `disabled={!backupDownloaded}` — that guard is load-bearing for "user actually saved the key." Visual redesign of this flow must keep the guard. The brief doesn't mention multi-step explicitly; flag as design ambiguity.
- **NIP-46 universal pairing has a side-effect listener.** `authManager.startNip46PairingUniversal()` returns immediately with the QR URI but starts a relay-listener subscription that will flip `authState.isAuthenticated` true async. The redirect at `+page.svelte:108-111` runs as a reactive `$:` block tied to that state. If the modal closes mid-pairing (user clicks away, presses ESC), the listener keeps running until either pairing completes or `cleanup()` fires from a different code path. This is existing behavior, not new — but worth confirming the new "click outside / ESC closes the modal" behavior doesn't tear down the pairing listener mid-flight. The cleanest answer: ESC/backdrop close should leave the pairing-pending state intact; only an explicit Cancel should abort.
- **`bind:open` on every sub-modal is load-bearing** — `nsecModal`, `bunkerModal`, `generateModal`, `nip46UniversalModal` are all booleans. The refresh's "More sign-in methods" disclosure is independent of these: clicking a method in the disclosure sets one of the booleans true and opens its sub-modal on top. So we have nesting: outer modal (the login card itself, if it remains a modal) and inner sub-modals (one per method). The brief shows the disclosure inline in the main card, not in its own modal — confirm that's the intent.
- **Redirect param.** `+page.svelte:109` reads `$page.url.searchParams.get('redirect') || '/explore'`. If we restructure the modal into a component, that param has to stay accessible at the calling layer. Easiest: keep handlers and redirect on the page, only the markup moves.
- **`SuggestedFollowsModal`** (`+page.svelte:973-984`) renders as a sibling after generate-keys completes, gated by `showSuggestedFollows`. The refresh should keep that handoff working — after Create Profile finishes, the new modal closes, and SuggestedFollowsModal opens.
- **Brand assets.** Brief calls for "flame/ember gradient circle with the lightning bolt, gentle pulse animation" and "lightning-bolt logo flicker." No SVG/asset path is provided. Do we have a logo asset to use, or do I assemble the gradient circle inline? Flag.

---

## Open questions answered (from the brief)

- **Modal used elsewhere?** No. All sign-in flows funnel to `/login`. Confirmed by exhaustive grep over component imports.
- **Tailwind / design tokens?** Yes. `tailwind.config.cjs` is standard with `@tailwindcss/forms` + `@tailwindcss/typography`. CSS custom properties live in `src/app.css:49-86` — `--color-primary: #ec4700` (orange), full light/dark ramps, `--bc-color-brand` for Bitcoin Connect parity. The modal already uses these tokens; the refresh should too.
- **i18n layer?** None. No `svelte-i18n` / `i18next` / `t()` / `$_`. Strings are hard-coded English throughout. The refresh keeps strings hard-coded; if i18n lands later it's a separate sweep.

## Open questions back to you

1. **B1 — drop the mockup into the repo or accept building from inline brief?** Need before I start implementation.
2. **B2 — confirm "four methods" was a typo for five.**
3. **iOS form — refresh `LoginFormIOS.svelte` in the same PR with parity styling, or collapse iOS + web into one component (bigger scope)?** Recommend the former.
4. **Multi-step Create Profile flow** — is the new design supposed to redesign that as well, or just the entry-point card? The brief shows only the entry hierarchy.
5. **Brand asset for the lightning bolt** — existing SVG path or inline-build the gradient circle?
6. **Disclosure inline or as its own sub-modal?** Brief reads inline; confirming.
7. **NIP-46 pairing-mid-close** — when the user opens "scan QR / pair phone" and then ESCs out, do we want the pairing to continue silently in the background (current behavior would, since the listener isn't tied to the modal), or aborted on close?

Stopping here for review per the brief — no implementation until these are addressed.
