# Trackfy Design System

**Status:** Draft
**Version:** 0.1
**Reference:** Raycast-inspired
**Purpose:** Single source of truth for Trackfy's visual language and interaction design.

Trackfy takes inspiration from Raycast's product philosophy and interface discipline — fast, simple, focused, dense where useful, and polished in small details — while remaining a distinct product for tracking, attribution, revenue intelligence and operational observability.

Raycast publicly describes its design principles as **fast, simple, and delightful**, and its interface emphasizes a prominent search/action surface, contextual actions, simple icons, compact presentation and keyboard-first workflows. Trackfy adopts these principles rather than copying Raycast's brand, artwork, proprietary assets or exact UI. citeturn233607search0turn233607search2

## Overview

Trackfy should feel like a precision instrument for understanding a marketing system.

The interface is dark, calm, compact and information-dense without becoming visually noisy. Surfaces should feel close to the background. Separation comes primarily from tonal elevation, subtle borders and spacing instead of large cards, gradients or decorative effects.

The product should communicate three qualities immediately:

- **Fast:** interactions feel immediate; navigation and filtering never feel heavy.
- **Clear:** the user can understand the state of tracking, attribution and revenue without decoding the UI.
- **Operational:** errors, missing data and failed integrations are visible and actionable instead of being hidden behind decorative dashboards.

Trackfy is not a generic dark SaaS dashboard. Avoid excessive rounded cards, giant hero sections, gradient backgrounds, oversized metric tiles and visually competing colors.

## Colors

The palette is a near-black neutral foundation with restrained elevation and a single Trackfy brand accent. Raycast's visual language is a useful reference for this restrained approach, but Trackfy must maintain its own brand identity. citeturn158305search0turn158305search1

### Base palette

```css
--color-bg: #090A0C;
--color-surface-1: #101114;
--color-surface-2: #15171A;
--color-surface-3: #1B1E22;

--color-border: rgba(255, 255, 255, 0.08);
--color-border-strong: rgba(255, 255, 255, 0.12);

--color-text: #F5F5F5;
--color-text-secondary: #B5B7BC;
--color-text-muted: #777B83;
--color-text-disabled: #555960;

--color-accent: #8B5CF6;
--color-accent-soft: rgba(139, 92, 246, 0.14);

--color-success: #34D399;
--color-warning: #FBBF24;
--color-danger: #FB7185;
--color-info: #60A5FA;
```

The accent is used as a signal, not as wallpaper. It should identify active navigation, focused controls, important links and selected analytical states.

Semantic colors are reserved for semantic meaning. A green number means a positive/healthy state; it must not merely be decoration.

Do not use gradients as a primary surface treatment. Do not use pure `#000000` for the entire interface. Do not use pure white as a large surface.

## Typography

Use a modern system-oriented sans-serif as the primary UI typeface.

Preferred stack:

```css
font-family:
  Inter,
  ui-sans-serif,
  -apple-system,
  BlinkMacSystemFont,
  "Segoe UI",
  sans-serif;
```

For event IDs, API keys, timestamps, raw values and technical identifiers:

```css
font-family:
  "Geist Mono",
  "SFMono-Regular",
  Consolas,
  monospace;
```

Typography should be compact and hierarchical rather than dramatic.

| Role | Size | Weight | Line height |
|---|---:|---:|---:|
| Display | 32px | 600 | 1.1 |
| Page title | 24px | 600 | 1.2 |
| Section title | 16px | 600 | 1.3 |
| Body | 14px | 400 | 1.45 |
| Body emphasized | 14px | 500 | 1.45 |
| Caption | 12px | 500 | 1.35 |
| Micro / metadata | 11px | 500 | 1.3 |
| Technical value | 12px | 500 | 1.35 |

Use weight and contrast before introducing additional font sizes.

Avoid huge typography for dashboard metrics. A metric should be readable in context, not dominate the entire screen.

## Layout & Spacing

Trackfy uses a compact 4px base spacing system.

```css
--space-1: 4px;
--space-2: 8px;
--space-3: 12px;
--space-4: 16px;
--space-5: 20px;
--space-6: 24px;
--space-8: 32px;
--space-10: 40px;
--space-12: 48px;
```

The default desktop application layout is:

```text
┌───────────────────────────────────────────────────────┐
│ Top bar / workspace context                           │
├───────────────┬───────────────────────────────────────┤
│               │                                       │
│ Sidebar       │ Main content                          │
│               │                                       │
│ navigation    │ tables / charts / debugger           │
│               │                                       │
│               │                                       │
└───────────────┴───────────────────────────────────────┘
```

The interface prioritizes the working surface. Navigation is visually quiet. Content gets the highest contrast.

Use a consistent content grid and optical alignment. Avoid arbitrary absolute positioning when normal layout primitives can solve the problem.

### Density

Trackfy is an operational application. Tables, event timelines and campaign analysis should support high information density.

Prefer:

- compact rows;
- aligned numeric columns;
- sticky contextual headers where useful;
- progressive disclosure for secondary information;
- dense-but-readable event timelines.

Do not compress everything. Density should increase information per viewport, not reduce usability.

## Elevation & Depth

Depth should come primarily from very subtle surface changes, borders and layered shadows.

```css
--shadow-panel:
  0 12px 32px rgba(0, 0, 0, 0.32),
  0 0 0 1px rgba(255, 255, 255, 0.04);

--shadow-popover:
  0 20px 48px rgba(0, 0, 0, 0.44),
  0 0 0 1px rgba(255, 255, 255, 0.06);
```

Normal dashboard surfaces should not look like floating cards. Elevation is for popovers, dialogs, command surfaces and transient layers.

For ordinary containment, prefer:

```css
border: 1px solid rgba(255, 255, 255, 0.08);
```

rather than a heavy shadow.

## Shapes

Trackfy uses restrained radii.

```css
--radius-sm: 6px;
--radius-md: 8px;
--radius-lg: 10px;
--radius-xl: 12px;
```

Controls should generally use 8px or less. Larger radii are reserved for major floating surfaces.

Avoid making every component a pill.

Pills are appropriate for compact statuses, filters or tags. Primary application controls should remain conventional and precise.

## Components

### Command / Search

Search is a first-class interaction, inspired by Raycast's emphasis on a central search/action surface. Trackfy should provide global search and contextual command access where they materially reduce navigation friction. citeturn233607search0turn233607search7

The user should be able to quickly search for:

- Orders;
- Visitors;
- Sessions;
- Clicks;
- Campaigns;
- Sites;
- Integrations;
- Events;
- settings and actions.

Suggested shortcut:

```text
⌘K / Ctrl+K
```

Search surfaces should feel immediate and focused, not like a full-page form.

### Action Bar

Contextual actions should live close to the current object.

Example:

```text
Event #evt_123

[ Inspect ] [ Replay ] [ Copy ID ] [ Open Order ]
```

Secondary actions can be keyboard accessible and grouped into an action menu. Raycast's action bar is a useful reference for discoverable contextual commands. citeturn233607search0

### Sidebar

The sidebar should be narrow, quiet and persistent.

Primary information architecture:

```text
Overview
Tracking
  Events
  Live Debugger
  Sites
  Sources
Attribution
  Models
  Paths
Revenue
  Orders
  Revenue
  Costs
  Profit
Analytics
  Campaigns
  Funnels
  Reports
Integrations
  Ad Platforms
  Sales Platforms
  Webhooks
Settings
```

Do not expose every feature simultaneously. Secondary navigation can appear contextually.

### Tables

Tables are a major Trackfy surface.

Rules:

- align numbers by their natural numeric edge;
- keep row height compact;
- use muted metadata rather than extra columns when possible;
- use semantic coloring only for meaningful status;
- support keyboard navigation for dense operational workflows;
- make row actions contextual.

Avoid card-based replacements for data that is inherently tabular.

### Event Timeline

The Event Debugger is a signature Trackfy experience.

The timeline should read like an execution trace:

```text
10:31:02  click
10:31:03  session_started
10:31:05  page_view
10:31:18  lead
10:32:11  checkout_started
10:33:07  purchase
10:33:09  server_purchase
10:33:09  duplicate_detected
10:33:10  order_created
```

Use a monospace timestamp/ID layer and a clear semantic event marker. The selected event reveals deeper payload and processing information without forcing the entire timeline to expand.

### Status

Status should be explicit and compact.

Examples:

```text
● Healthy
● Processing
● Pending
● Failed
● Unmatched
● Duplicate
```

Status indicators must have accessible text; color alone is insufficient.

### Metrics

Metrics should answer operational questions:

```text
Revenue
R$ 128,430

Profit
R$ 42,190

ROAS
4.72x

Unmatched Events
1.8%
```

Avoid decorative chart junk. Every chart must have a question it answers.

### Charts

Charts should prioritize signal over presentation.

Use subtle grids, strong axis hierarchy and minimal ornamentation. Tooltips should expose exact values. Color should encode meaningful series, not decoration.

### Dialogs / Sheets

Dialogs should be focused and short.

Use sheets or contextual panels when a task benefits from preserving the underlying page context.

Never use a modal for information that could be shown inline without interrupting the workflow.

### Keyboard Shortcuts

Keyboard shortcuts are first-class affordances for frequent actions, inspired by Raycast's command model. citeturn233607search7

Show shortcuts where they meaningfully help users discover faster paths.

Suggested core shortcuts:

```text
⌘K / Ctrl+K   Global command/search
G then O      Overview
G then T      Tracking
G then A      Attribution
G then R      Revenue
G then I      Integrations
```

Exact shortcuts can change during usability validation.

## Interaction Principles

### Fast feedback

Every interaction should visibly acknowledge immediately.

Loading states should appear early. Mutations should expose success/failure without forcing a full page reload.

### Progressive disclosure

The default screen shows the most useful information. Advanced details become available through expansion, command panels or drill-downs.

This is particularly important for event payloads and attribution explanations.

### Context preservation

Opening an event, order or campaign should preserve the user's analytical context whenever possible.

Do not repeatedly force users back to a list after inspecting an object.

### Recoverability

Errors must explain what happened and what can be done next.

Example:

```text
Meta delivery failed
HTTP 429 — rate limited
Retry scheduled in 18s
[ View delivery log ] [ Retry now ]
```

Do not display generic “Something went wrong” messages when the system knows the actual failure class.

## Motion

Motion is functional and short.

Preferred properties:

```css
opacity
transform
```

Typical duration range:

```text
80ms — micro feedback
120ms — control transition
160ms — panel transition
200ms — larger surface transition
```

Respect `prefers-reduced-motion`.

Avoid continuous decorative animation.

## Responsive Behavior

Trackfy is desktop-first because its primary users work with dense operational data, but core workflows must remain usable on smaller screens.

```text
≥ 1280px
Full sidebar + dense dashboard

1024–1279px
Reduced sidebar + compressed grids

768–1023px
Collapsible navigation + fewer simultaneous columns

< 768px
Single-column operational views
```

Do not simply shrink desktop tables until they become unreadable. Use responsive column prioritization and drill-down patterns.

## Accessibility

Accessibility is part of the component contract.

Requirements:

- keyboard navigation for all core workflows;
- visible focus states;
- sufficient text contrast;
- semantic HTML;
- status not communicated by color alone;
- reduced-motion support;
- screen-reader labels for icon-only controls;
- logical tab order;
- no keyboard traps.

## Data Visualization Rules

Trackfy is a measurement product. Visualization must preserve analytical honesty.

Never:

- truncate axes to exaggerate differences;
- hide uncertainty when it materially affects interpretation;
- use decorative 3D charts;
- imply attribution certainty that the underlying evidence does not support.

When attribution is partial, recovered or unavailable, the UI must represent that state explicitly.

## Empty, Loading and Error States

Every primary surface needs four deliberate states:

```text
Loading
Empty
Ready
Error
```

An empty event stream, for example, should explain how to configure tracking rather than displaying a blank card.

A failed integration should explain why it failed and expose the relevant diagnostic path.

## Do's and Don'ts

### Do

- Keep the visual system dark, restrained and precise.
- Prioritize information hierarchy over decoration.
- Make search and contextual actions discoverable.
- Use compact tables and timelines for operational data.
- Keep colors semantic.
- Make important state visible immediately.
- Prefer subtle borders and tonal elevation over giant cards.
- Preserve context during drill-downs.
- Make keyboard interaction progressively faster for power users.
- Treat the Event Debugger as a flagship interface, not an admin afterthought.

### Don't

- Don't copy Raycast's logo, proprietary assets or exact interface.
- Don't turn Trackfy into a clone of a command launcher.
- Don't use gradients as a substitute for hierarchy.
- Don't cover the screen with rounded cards.
- Don't use accent color on every button or metric.
- Don't hide failures behind generic success-looking dashboards.
- Don't replace tables with decorative cards when the data is tabular.
- Don't overuse animation.
- Don't sacrifice readability for visual density.
- Don't make the interface look like generic “AI SaaS” or “crypto dashboard” templates.

## Design Quality Bar

A screen is not finished because it technically renders.

It is finished when:

1. The user's primary task is visually obvious.
2. The hierarchy can be understood in seconds.
3. Important states are unambiguous.
4. Dense information remains scannable.
5. Keyboard and pointer paths both work.
6. Loading, empty and error states are designed.
7. The screen remains coherent at smaller widths.
8. Motion is purposeful and accessible.
9. No visual choice exists solely because a UI library supplied it by default.
10. The screen feels like Trackfy, not a generic dashboard.

## Agent Guidance

When generating or modifying Trackfy UI, read this file first.

Do not invent visual tokens locally. Reuse the tokens defined here.

For each new component, determine:

```text
Purpose
User task
Information hierarchy
Interaction model
State model
Keyboard behavior
Responsive behavior
Accessibility behavior
```

The design system is a product constraint, not decoration applied after implementation.

## Reference Note

The Raycast reference is used for principles of speed, simplicity, focused interaction, contextual actions, compact presentation and polished details. Raycast's own public design discussion emphasizes these qualities and its action/search patterns. citeturn233607search0turn233607search7

The Trackfy design system remains independently branded and tailored to tracking, attribution, revenue intelligence and operational debugging.
