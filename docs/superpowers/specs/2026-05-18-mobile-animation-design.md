# Mobile Scroll Animation — Design Spec
**Date:** 2026-05-18

## Goal

Bring the desktop scroll animation on `index.html` to mobile. Currently, mobile skips the animation entirely and shows a static header. The target experience: the Skystar logo + flower bouquet start large and centered in the full-screen splash, then animate into the header as the user scrolls — identical timing and structure to desktop, with a stacked (logo above nav) header layout suited to narrow screens.

---

## Files Changed

| File | Type of change |
|---|---|
| `index.html` | Primary — CSS + JS |
| `shared.css` | Already done — stacked header for inner pages (≤600px) |

---

## CSS Changes in `index.html`

### 1. Strip the `@media (max-width: 768px)` block

Remove the overrides that disabled the animation:
- `header { position: static !important; ... }` → **removed**
- `.splash-brand-wrap { position: static !important; ... }` → **removed**
- `header .logo-wrap { visibility: visible !important; }` → **removed**
- `header nav { opacity: 1 !important; transform: none !important; margin-left: 0 !important; }` → **removed**

Keep only:
- `.splash-brand { font-size: clamp(3.5rem, 16vw, 6rem); }` (smaller logo on small screens)
- `.splash-flowers` size adjustment (keep as-is — flowers remain visible and animate with the logo)

### 2. Add `@media (max-width: 600px)` block inside `index.html`'s `<style>` tag

```css
@media (max-width: 600px) {
  header {
    flex-direction: column;
    justify-content: center;
    align-items: center;
    padding: 14px 20px 10px;
    min-height: unset;
    gap: 8px;
  }
  nav {
    margin-left: 0; /* override desktop margin-left: auto */
  }
}
```

> Note: shared.css has a matching ≤600px block for inner pages (interest, inventory, etc.). That block does not override index.html's inline `<style>` due to CSS cascade order (inline style tag comes after the linked stylesheet). This separate block in index.html handles it for the home page.

---

## JS Changes in `index.html`

### 1. Remove the mobile bypass

Delete the `isMobile` check that immediately showed the header and skipped all scroll listeners:

```js
// REMOVE:
const isMobile = window.innerWidth < 768;
if (isMobile) {
  header.classList.add('bg-in', 'nav-in');
}
```

### 2. Update `measureAll()` — mobile-aware final position

Add a branch so the brand lands centered on narrow screens instead of at the top-left corner:

```js
if (window.innerWidth < 600) {
  // Center the brand (logo + flowers) horizontally in the header
  FINAL_LEFT = (window.innerWidth - brand.offsetWidth * FINAL_SCALE) / 2;
  FINAL_TOP  = 14; // matches mobile header padding-top
} else {
  FINAL_LEFT = 40;  // desktop: top-left corner
  FINAL_TOP  = 22;
}
```

### 3. Remove the `>= 768` guard from the scroll/resize listeners

```js
// BEFORE:
if (!isMobile) {
  window.addEventListener('scroll', onScroll, { passive: true });
  window.addEventListener('resize', function() {
    if (window.innerWidth >= 768) { ... }
  });
  document.fonts.ready.then(...);
}

// AFTER: always register listeners, no width gates
window.addEventListener('scroll', onScroll, { passive: true });
window.addEventListener('resize', function() {
  locked = false;
  measureAll();
  onScroll();
});
document.fonts.ready.then(function() {
  measureAll();
  onScroll();
});
```

---

## Animation Flow (mobile)

| Scroll progress | What happens |
|---|---|
| 0% | Splash fills screen. Logo + flowers centered and large. Header transparent. Nav invisible. |
| 0–50% | Brand moves toward header. Header background fades in. |
| 50–85% | Header fully opaque. Brand still moving and scaling. |
| 85–100% | Nav fades in, centered below logo. |
| 100% | Brand snaps to final centered position. Page snaps to `siteFloor` (content section, just below header). Scroll locked — splash never reappears. |

---

## Header End-State (≤600px)

```
┌─────────────────────────────┐
│         Skystar 🌸          │  ← animated brand, centered
│  LOOKBOOK · EVENTS · REQUEST│  ← nav, centered, wraps if needed
└─────────────────────────────┘
```

Matches the look and feel of the inner pages (interest, inventory, etc.) which already show logo + flower centered above nav on narrow screens.

---

## Edge Cases

| Scenario | Handling |
|---|---|
| Orientation change (portrait → landscape) | Resize handler re-measures `FINAL_LEFT`, `FINAL_SCALE`, replays animation at current scroll. Resets `locked`. Layout switches between mobile/desktop thresholds automatically. |
| Back navigation / reload | `history.scrollRestoration = 'manual'` + `window.scrollTo(0, 0)` force scroll to top on every load. Animation always plays fresh. |
| iOS Safari viewport height | `100vh` is slightly taller than visible area on iOS (address bar). Splash is a hair taller; user scrolls marginally further before animation completes. Acceptable — consistent with existing desktop Safari behavior. No change needed. |

---

## Out of Scope

- Inner pages (interest, inventory, popups, contact) — already handled by the shared.css ≤600px fix
- Any changes to the desktop animation behavior
- Auto-play or skip-button animation variants (considered and ruled out)
