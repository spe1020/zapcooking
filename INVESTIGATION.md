# Sign-in Modal Refresh — Investigation (revised brief)

Branch: `investigate/signin-modal-refresh`. Investigation only — no production code touched. Stopping here for review per the brief.

The previous attempt at this work (PR #345 on `feat/signin-modal-refresh`) was rejected. This pass is against the revised brief — different visual direction (wordmark + tile row instead of brand mark + disclosure), different typography (Albert Sans instead of Fraunces), and an explicit color palette.

---

## Blockers before implementation

### B1. The visual spec is still missing from the repo

The brief points at `docs/design/signin-modal-final.html` as the mockup. **There is no `docs/design/` directory at all** in the working tree. Same blocker as yesterday with a new filename — `signin-modal-v2.html` was the previous miss.

I can build from the inline brief description (palette is now spelled out with hex codes, hierarchy is explicit, typography is named), so the gap is narrower than before. But pixel-level details — exact drop-shadow ramp on the primary CTA glow, tile button hover states, divider styling, exact wordmark sizing on each breakpoint, modal entrance curve nuance — won't match without the HTML.

Two options:
- (preferred) drop `signin-modal-final.html` into `docs/design/` and I'll match it
- or accept inline-brief fidelity and iterate visually in PR review

### B2. Albert Sans isn't in the project

Not in `package.json`, not in `app.html`, not in `app.css`. The repo currently loads only Orbitron via Google Fonts (`app.html:71-74`). Adding Albert Sans the same way (extending the existing `<link>` to include it) is zero-dep and consistent with how Orbitron is wired — recommend that. Confirm before I go.

### B3. Logo asset — the situation is better than the brief implies, but worth a call

The brief says: "The current logo asset is black-on-white and won't read on the dark modal surface." That's not quite right. The repo has **two** wordmark variants:

- `static/zap_cooking_logo_black.svg` — black wordmark + flame icon for light surfaces
- `static/zap_cooking_logo_white.svg` — **white wordmark** + flame icon (gradient `#FFAA00 → #FF0C1C`) for dark surfaces

Both are 1080×290 vectors. Today's `/login` route already swaps between them via `$: isDarkMode` (`+page.svelte:28`) and the `<img src={isDarkMode ? '/zap_cooking_logo_white.svg' : '/zap_cooking_logo_black.svg'}>` at `:1018`.

So a working dark-surface wordmark exists. The brief's desired tint is **cream `#F5EBDD`** — not pure white. Two paths:

- **(A) Use the existing white SVG.** Pure white reads cleanly on the dark `#13171F` surface. Fast to ship; no new asset. The mockup uses cream as a brand-warmth choice, not a contrast requirement.
- **(B) Add a cream-tinted variant.** Either a new SVG file (`zap_cooking_logo_cream.svg`) or — simpler — load the white SVG inline and recolor via `currentColor` / `fill` override at the use site. The flame icon's gradient stays untouched in either case (it's the brand mark and reads fine on dark).

Going to default to (A) for the implementation pass unless told otherwise. Asset is in the repo, swap is one line.

---

## 1. Where the modal mounts

- `/login` is a SvelteKit route at `src/routes/login/+page.svelte:1-1223`. Not a global overlay. The Header "Sign in" button at `src/components/Header.svelte:255` is a plain `href="/login"`. Every "sign in to ..." prompt across the app (PostComposer:495, NoteActionBar, ThreadCommentActions, etc.) navigates to `/login?redirect=...`.
- iOS uses a separate component: `src/components/LoginFormIOS.svelte:1-955`. The route swaps on `$platformIsIOS` at `+page.svelte:499`.
- **The brief calls this surface a "modal" but it's a route page rendering a centered card.** Real ESC-closes / click-outside-closes behavior on the route only makes sense as "navigate back" (history.back) or "navigate to /explore". Sub-modals (nsec / bunker / generate / NIP-46 universal) keep their existing modal-on-modal behavior.

## 2. Sign-in method handlers

Five methods, all on `authManager` (`src/lib/authManager.ts`). All five can be invoked unchanged from the new markup — handlers don't reach into the DOM.

| Method | Page handler | authManager call | Side effects |
|---|---|---|---|
| Sign in with Browser Signer (NIP-07) | `loginWithNIP07()` `+page.svelte:141-158` | `authenticateWithNIP07()` `authManager.ts:140-180` | `localStorage.nostrcooking_loggedInPublicKey`, NDK signer set |
| Create Profile (generate keys) | `useGeneratedKeys(skipProfile)` `+page.svelte:246-312` | `authenticateWithPrivateKey` after key gen | `localStorage.nostrcooking_loggedInPublicKey` + `nostrcooking_privateKey`; publishes kind:0 metadata; multi-step backup→profile guard |
| Scan QR / Universal Pairing | `startUniversalPairing()` `+page.svelte:320-341` | `startNip46PairingUniversal()` `authManager.ts:639-718` | `localStorage.nostrcooking_nip46_pending`; opens response listener; on Android, `window.open(uri, '_system')` deeplinks to signer |
| Paste Bunker URI (NIP-46) | `loginWithBunker()` `+page.svelte:178-199` | `authenticateWithNIP46(connectionString)` `authManager.ts:332-471` | `localStorage.nostrcooking_nip46`, `nostrcooking_loggedInPublicKey`, `nostrcooking_authMethod='nip46'` |
| Import Key (nsec) | `loginWithPrivateKey()` `+page.svelte:160-176` | `authenticateWithPrivateKey(pk)` `authManager.ts:186-246` | `localStorage.nostrcooking_loggedInPublicKey` + `nostrcooking_privateKey` |

## 3. Modal management primitive

- `<Modal>` exists at `src/components/Modal.svelte:1-122`. Sub-modals already use it via `bind:open={...} on:close={modalCleanup}`.
- Already provides: `aria-modal="true"`, `aria-labelledby="title"`, ESC close, click-outside on backdrop, body scroll lock (inline `<style>` block), portal-to-body, blur+scale entrance transition.
- Does **not** provide: focus trap, initial focus management. No focus-trap library in `package.json`.
- **Recommendation:** stay self-contained. Implement focus trap with a small inline handler (~25-30 LOC, no dep). The previous attempt (PR #345) did this — code can be re-derived or cherry-picked.

## 4. Logo asset

Covered in B3 above. Summary: dark-mode wordmark exists at `static/zap_cooking_logo_white.svg`. Cream tint per the mockup palette is a small additional ask if we want exact fidelity; otherwise the white variant ships.

The flame mark inside the wordmark (left of the "ZAP COOKING" text) uses a `#FFAA00 → #FF0C1C` linear gradient — close to but not identical to the brief's `flame #F7931A / ember #FF5F1F` palette. Worth a call: do we update the source SVG to match the new palette, or accept the existing gradient as visually-aligned-enough?

## 5. Analytics / event tracking

**None.** Confirmed yesterday and re-confirmed: no `track(`, `analytics.`, `posthog.`, `mixpanel.`, `gtag(`, no custom `dispatchEvent` calls in the login route or `authManager.ts`. "Preserve every existing event" simplifies to "no events to preserve."

If we want to add basic instrumentation as part of this refresh — `signin_method_clicked` per CTA, `signin_succeeded`, `signin_failed` — that's net-new infrastructure and is **out of scope** per the brief unless explicitly added. Flagging.

## 6. Mobile / Capacitor considerations

- iOS uses `LoginFormIOS.svelte`, structurally different (full-page scrolling card, not viewport-centered modal with backdrop blur). Drops NIP-07 (no extension on iOS) and drops QR/universal pairing (App Store cross-app handoff policy). Today's iOS hierarchy: Create Profile (primary), import private key + bunker paste as secondaries.
- **The brief's tile-row (Scan QR / Bunker URI / Import key) doesn't map cleanly to iOS.** QR and "Pair phone" can't ship on iOS per platform policy. Two paths:
  - **(A) iOS shows two tiles** — Bunker URI + Import key — with the QR tile omitted entirely. Cleanest UX.
  - **(B) iOS shows three tiles** with the QR tile disabled and a small note explaining the App Store restriction. Honest but cluttered.
  - Recommend (A). The existing `supportsNIP46QRPairing()` helper at `src/lib/platform.ts:134` already returns `false` on iOS — wire the UI off it.
- **Safe-area:** `src/app.css:18-23` defines `--safe-area-inset-*` from `env(safe-area-inset-*)`. The new card needs `padding-top/bottom: max(1.5rem, env(safe-area-inset-*))` on mobile, especially for iOS notched devices. Not handled today on the LoginFormIOS form.
- **Keyboard overlap:** no `visualViewport` workarounds in the login flow today. Capacitor's keyboard plugin presumably handles it implicitly. Worth a manual smoke during implementation, particularly for the bunker-paste textarea since it's the only form input on the iOS card.
- **Platform helpers exist but aren't called by the login route today.** The platform-gating on iOS is implicit (different file). The refresh should call `supportsNIP46QRPairing()` and `supportsNIP46BunkerPaste()` from `src/lib/platform.ts` so the policy lives in one place.

## 7. Risks / surprises

- **Form state lives on the page, not in sub-modal components.** `nsecInput`, `bunkerConnectionString`, `generatedKeys`, profile fields are all page-level `let`s. Existing pattern; the refresh should keep it.
- **Multi-step Create Profile flow.** `useGeneratedKeys()` is a multi-step modal (backup → profile). Next button at `:835` is `disabled={!backupDownloaded}` — a load-bearing guard. Brief shows only the entry-point hierarchy; **the inner Create Profile multi-step is unchanged in this refresh** unless told otherwise.
- **NIP-46 universal pairing has a side-effect listener** that outlives the modal. Triggers `authState.isAuthenticated → true` async via relay listener. ESC/backdrop close on the QR sub-modal does NOT abort the listener today. Existing behavior, not changed.
- **`bind:open` on every sub-modal is load-bearing** (`nsecModal`, `bunkerModal`, `generateModal`, `nip46UniversalModal` are all booleans driving `<Modal bind:open={...}>`). The new tile buttons set those booleans to `true`; existing pattern works fine.
- **Redirect param.** `+page.svelte:109` reads `$page.url.searchParams.get('redirect') || '/explore'`. Stays at the page level.
- **`SuggestedFollowsModal`** (`+page.svelte:973-984`) renders post-Create-Profile, gated by `showSuggestedFollows`. Refresh keeps that handoff working.
- **Background visual layer.** Lines 990-1003 of the current `+page.svelte` have animated emoji bouncing in the background. The new dark-surface card with the brief's palette will look strange behind those — they should either be removed or wholly hidden behind a darker scrim. Flag.
- **Tailwind config is minimal** (`tailwind.config.cjs`: just plugins, no custom palette). The brief's palette tokens (ink/surface/elevated/line/flame/ember/cream/wheat/mute) can be added either to the Tailwind config or scoped as CSS custom properties on the login surface. Existing convention is CSS custom properties (see `src/app.css:49-86` for `--color-*` tokens). Recommend matching that pattern — adds the new tokens to the existing system rather than introducing a parallel one.

---

## Open questions answered (from the brief)

- **Modal used elsewhere?** No. All sign-in flows funnel to `/login`.
- **Tailwind palette tokens?** Tailwind config is bare; existing tokens live in `src/app.css:49-86` as CSS custom properties. Recommend adding the new flame/ember palette as additional `--color-*` tokens scoped to the login surface (or globally if we want them available app-wide).
- **i18n layer?** None. No `svelte-i18n` / `i18next` / `t()` / `$_`. Strings are hard-coded English.
- **Dark-mode wordmark SVG?** Exists at `static/zap_cooking_logo_white.svg`. Pure white, not the brief's cream tint. See B3.

## Open questions back to you

1. **B1 — drop `docs/design/signin-modal-final.html` into the repo, or implement from inline brief?**
2. **B2 — confirm Albert Sans gets added via Google Fonts (zero-dep, consistent with Orbitron loading)?**
3. **B3 — use existing white wordmark, or add a cream-tinted variant?**
4. **iOS tile row** — drop the QR tile (option A) or show it disabled (option B)?
5. **Background animated emojis** (`+page.svelte:990-1003`) — keep them behind the new scrim, or remove for the cleaner dark surface?
6. **Wordmark flame gradient** — keep existing `#FFAA00 → #FF0C1C` or update source SVG to the new `flame #F7931A / ember #FF5F1F` ramp?
7. **Add basic auth analytics** alongside the refresh (out of scope per the brief; flagging because the funnel is currently un-instrumented)?

Stopping here per the brief — no implementation until these are addressed. PR #345 stays as-is on `feat/signin-modal-refresh` for reference; once direction is confirmed, the next pass will start fresh on a new implementation branch.
