# Birthday‑Cake Loading (BCL) — EVA Website

## Abstract

Modern marketing sites often assume “fast device + fast network”. EVA intentionally does **not**: it should remain readable and usable on old phones, slow mobile networks, and unusual aspect ratios. At the same time, EVA’s “cinematic” experience (animations, parallax, audio, telemetry opt‑in UX, high‑fidelity imagery) should still shine on capable hardware.

**Birthday‑Cake Loading (BCL)** is EVA’s progressive enhancement method:

- **Base layer**: a stable, low‑dependency, fast‑loading experience that works everywhere.
- **Optional layers**: loaded **only** when the runtime has enough “budget” (CPU/memory/network/user‑prefs) and only after the base is already usable.

In practice, BCL combines:

- A **capability detector** (signals → tier → feature flags)
- **Code‑splitting** (lazy rich bundles)
- **Progressive enhancement bridges** (server markup becomes interactive only when the rich layer exists)

---

## 1. Design goals (what BCL optimizes for)

- **Correctness first**: content and navigation work without relying on animations, smooth scrolling, or audio.
- **Fast time‑to‑content**: base layer must render quickly with minimal JS/CPU load.
- **Graceful degradation**: if APIs are missing (older Safari / Android WebView), default to safe behavior.
- **User preference respect**: especially `prefers-reduced-motion` and data saver.
- **Non‑invasive upgrades**: rich features are optional and should not block the base experience.

---

## 2. The model: signals → tier → features

### 2.1 Signals (runtime observations)

EVA reads a small set of **best‑effort signals** in the browser:

- **Network**
  - `navigator.connection.saveData` (data saver)
  - `navigator.connection.effectiveType` (2g/3g/4g)
  - `navigator.connection.downlink`, `navigator.connection.rtt`
- **Device**
  - `navigator.deviceMemory` (approx GB)
  - `navigator.hardwareConcurrency` (cores)
  - `devicePixelRatio`, `screen.width/height`
- **User preference**
  - `prefers-reduced-motion`

These are captured as a `CakeSignals` struct.

### 2.2 Tiering

EVA assigns a tier:

- **`base`**: strongly constrained (save‑data, 2g, very low memory, etc.)
- **`lite`**: constrained; stays mostly static (reserved for future incremental upgrades)
- **`rich`**: capable enough for the cinematic experience
- **`ultra`**: “4k / gaming laptop” class (currently same feature set as `rich`, but reserved)

The tier decision is conservative:

- Unknown/missing APIs → **do not** penalize.
- Strong evidence (Save‑Data / 2g / <2GB memory) → downgrade.

Implementation: `lib/birthday-cake/detect.ts`

### 2.3 Feature flags

From tier + signals, EVA derives a set of feature toggles (`CakeFeatures`):

- **motion**: framer-motion based animation layers
- **smoothScroll**: Lenis + scroll CSS variables
- **audio**: audio engine, SFX/VO gatekeeper + controls
- **privacyBanner**: consent UI (otherwise treated as “declined by default”)
- **richImages**: allow high-fidelity imagery defaults (browser still picks via `srcset`)

Implementation: `lib/birthday-cake/features.ts`

---

## 3. Layering in EVA (what loads when)

### 3.1 Layer 0 — Base (always)

This is the “broad cake base”:

- **Static marketing layout** (no Lenis, no parallax code, no framer-motion)
- **JS-light homepage** with:
  - anchor navigation
  - systems + FAQ expand/collapse (simple React state; no motion libs)
  - minimal visuals (CSS-only gradients)

Files:

- `components/layout/marketing-layout.tsx`
- `components/pages/marketing-home-cake.tsx` (renders the base immediately)
- `app/(marketing)/page.tsx` (now points to the cake page)

### 3.2 Layer 1 — Rich providers (lazy, tier-gated)

On `tier ∈ {rich, ultra}`, EVA loads a separate provider bundle:

- **Framer Motion** `MotionConfig`
- **Lenis** smooth scrolling + CSS scroll variables
- **Audio** (`AudioProvider` + `AudioLayer`) if allowed
- **Privacy** (`PrivacyLayer`) if allowed

Key property: this “rich layer” is a **lazy chunk**, so it does not impact low‑tier devices.

Files:

- `app/provider.tsx` (lightweight, always)
- `app/provider.rich.tsx` (lazy loaded only for rich tiers)

### 3.3 Layer 2 — Rich homepage (lazy, tier-gated)

On `tier ∈ {rich, ultra}`, EVA also loads the full cinematic homepage:

- existing animated sections
- framer-motion reveals
- audio triggers

Files:

- `components/pages/marketing-home-enhanced.tsx` (the previous homepage composition)
- `components/pages/marketing-home-cake.tsx` (loads enhanced via `React.lazy`)

---

## 4. Privacy & cookies: “auto-decline by default” on low tiers

Principle:

- If the **privacy banner** is not mounted, EVA behaves as:
  - analytics: **off**
  - no opt-in UI
  - no banner annoyance on old devices / slow connections

Implementation details:

- Privacy UI is only mounted in the rich provider when `features.privacyBanner === true`.
- The footer contains a static **server-rendered** “Datenschutz‑Einstellungen” button.
- In rich mode, a tiny bridge (`PrivacyOpenBridge`) wires this button to the dialog without importing the heavier `PrivacySettingsTrigger` component.

Files:

- `components/privacy/privacy-layer.tsx`
- `components/privacy/privacy-open-bridge.tsx`
- `components/layout/marketing-layout.tsx`

---

## 5. Audio: hard opt-in, and *not even shipped* on constrained tiers

Principle:

- Audio is a “nice‑to‑have”. If the device/network is constrained, audio should not load, prompt, or consume memory.

Implementation details:

- Audio layer is only mounted in the rich provider when `features.audio === true`.
- **Safety**: `useAudio()` no longer throws when audio isn’t mounted. It returns a no‑op API, so UI can still render without runtime crashes.

Files:

- `app/provider.rich.tsx`
- `lib/audio/audio-provider.tsx` (no-op default context)

---

## 6. Debugging & forcing tiers

### 6.1 HTML dataset

At runtime, EVA writes these attributes on `<html>`:

- `data-eva-cake-tier`
- `data-eva-cake-ready`
- `data-eva-cake-motion`
- `data-eva-cake-audio`
- `data-eva-cake-privacy`
- `data-eva-cake-save-data`

This is useful for:

- CSS conditionals
- QA screenshots
- diagnostics in DevTools

Implementation: `components/birthday-cake/cake-context.tsx`

### 6.2 Tier override (session)

Set a session override:

- sessionStorage key: `eva_cake_tier_override`
- allowed values: `base | lite | rich | ultra`

Clear it to return to auto-detection.

---

## 7. Methodology (how to extend BCL)

When adding a new “nice-to-have” feature:

1. **Identify the baseline fallback**
   - What is the simplest representation that still preserves meaning and navigation?
2. **Assign the feature to a budget**
   - CPU heavy? network heavy? memory heavy? interaction critical?
3. **Gate by tier and/or a specific signal**
   - example: audio depends on `!saveData` and not `2g/3g`
4. **Code-split the heavy part**
   - put the heavy implementation into a `*.rich.tsx` file or a lazily imported module
5. **Ensure it fails safe**
   - missing APIs should never crash the base experience
6. **Write a test expectation for both modes**
   - base behavior: content present, navigation works
   - rich behavior: enhancements present

---

## 8. Research paper style writeup (condensed)

### Title

**Birthday‑Cake Loading: Capability‑Adaptive Progressive Enhancement for Modern Marketing Sites**

### Introduction

Marketing sites often maximize visual fidelity, but this can exclude users on older hardware, data-saving connections, or accessibility settings. Birthday‑Cake Loading (BCL) proposes a structured way to deliver a reliable baseline first and then conditionally load enhancements as device/network budgets allow.

### Related work

- Progressive Enhancement (PE) and “baseline first” web design
- Adaptive loading strategies (network-aware resource selection)
- Code-splitting and islands architectures
- User preference media queries (`prefers-reduced-motion`)

### Method

BCL uses a 3-step pipeline:

1. **Signal acquisition**: read best-effort device/network/user-preference signals.
2. **Tier classification**: map signals into a small number of tiers with conservative downgrade rules.
3. **Layer activation**: lazy-load and activate feature bundles strictly in ascending optionality order.

### Implementation (EVA case study)

EVA implements:

- a low-dependency baseline layout and homepage
- a lazily loaded rich provider layer enabling motion, smooth scroll, audio, and privacy UX
- progressive enhancement bridges (server buttons become interactive only in rich mode)

### Evaluation plan

Quantitative metrics to capture:

- **Performance**: JS bytes shipped to base tier, FCP/LCP/CLS, main-thread blocking time
- **Energy**: CPU time during scroll + interactions (proxy for battery cost)
- **User experience**: conversion events (Discord clicks), opt-in rates, bounce rate

Experimental approach:

- A/B compare “always rich” vs BCL
- Segment by network class + device memory
- Use RUM for real-world distribution

### Limitations

- Some signals are unavailable or obfuscated (privacy, browser differences)
- Tiering is heuristic and must be tuned with real telemetry
- Content swaps can cause momentary UX shifts; minimize with consistent layouts and careful fallbacks

### Future work

- Add a true `lite` layer (e.g., static images but no motion)
- Add server-side hints (`Save-Data` header, Client Hints) for better first-render decisions
- Expand “bridges” to progressively enhance existing DOM without swapping entire components

---

## 9. Code map (quick links)

- Detection + model: `lib/birthday-cake/*`
- Context + dataset: `components/birthday-cake/cake-context.tsx`
- Base providers: `app/provider.tsx`
- Rich providers (lazy): `app/provider.rich.tsx`
- Base homepage + gate: `components/pages/marketing-home-cake.tsx`
- Rich homepage: `components/pages/marketing-home-enhanced.tsx`
- Privacy bridge: `components/privacy/privacy-open-bridge.tsx`
