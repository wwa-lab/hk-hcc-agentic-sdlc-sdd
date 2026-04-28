# Design System Document: Operational Intelligence
 
## 1. Overview & Creative North Star: "The Tactical Command"
The Creative North Star for this design system is **The Tactical Command**. We are moving away from the "friendly SaaS" trend of rounded bubbles and vast white space. Instead, we embrace the high-density, high-stakes environment of a NASA Mission Control center. 
 
This system is designed for power users who require maximum data density without cognitive overload. We achieve this through **intentional asymmetry**, where the layout mimics a customized dashboard rather than a rigid template. By utilizing overlapping modular cards and a sophisticated dark-mode hierarchy, we create an experience that feels engineered, authoritative, and precise. This isn't just a tool; it’s a cockpit.
 
---
 
## 2. Colors: Tonal Depth & The "No-Line" Rule
The palette is rooted in deep obsidian and navy tones to reduce eye strain during long-duration monitoring.
 
### The "No-Line" Rule
Traditional 1px solid borders are strictly prohibited for layout sectioning. We define space through **Background Shifts**. A `surface-container-low` panel sitting on a `surface` background creates a natural, sophisticated boundary that feels integrated into the hardware.
 
### Surface Hierarchy & Nesting
Depth is achieved through the "Physical Stack" method. Use the following tiers to define importance:
*   **Base Level:** `surface` (#0b1326) – The infinite void of the application.
*   **Secondary Layer:** `surface-container-low` (#131b2e) – Used for sidebars and navigation backgrounds.
*   **Active Modules:** `surface-container-high` (#222a3d) – The primary tier for interactive cards and data tables.
*   **Floating/Emphasis:** `surface-container-highest` (#2d3449) – Used for modals or high-priority popovers.
 
### The "Glass & Gradient" Rule
To elevate the "Control Tower" aesthetic, use **Glassmorphism** for floating overlays. Apply `surface_variant` at 60% opacity with a `20px` backdrop blur. This ensures the underlying data is still "sensed" by the user, maintaining the feeling of a continuous, living system. 
 
**Signature Texture:** Main Action buttons or AI-driven insights should utilize a subtle linear gradient from `secondary` (#89ceff) to `on_secondary_container` (#00344e) at a 135-degree angle to provide a machined, "lit from within" glow.
 
---
 
## 3. Typography: Technical Authority
We use a dual-font strategy to separate human-readable interface elements from machine-generated data.
 
*   **UI Interface (Inter):** Used for all labels, headings, and navigation. Inter provides the modern SaaS clarity required for speed.
*   **Technical Specs (JetBrains Mono):** Used for IDs, timestamps, coordinates, and raw data values. This imparts an "engineering-first" vibe.
 
**Hierarchy Strategy:**
*   **Display/Headline:** High-contrast `on_surface` (#dae2fd). Use `display-sm` for page headers to maintain a compact, editorial feel.
*   **Body:** Use `body-sm` (0.75rem) as the primary density standard. Information density is a feature, not a bug.
*   **Labels:** Use `label-md` in uppercase with 0.05em letter spacing for a "HUD" (Heads-Up Display) effect.
 
---
 
## 4. Elevation & Depth: Tonal Layering
In this system, light is the architect. We do not use "shadows" in the traditional sense; we use **Ambient Glows**.
 
*   **The Layering Principle:** Stack `surface-container-lowest` cards on top of `surface-container-low` sections. The subtle shift in hex value creates enough contrast to define the container without the visual clutter of a line.
*   **Ambient Shadows:** For floating elements, use a shadow color of `#060e20` (Surface Lowest) at 40% opacity with a 32px blur. It should feel like a soft occlusion, not a drop shadow.
*   **The "Ghost Border":** Where containment is required (e.g., input fields), use `outline_variant` (#45464d) at 20% opacity. This creates a "etched" look that feels premium and intentional.
 
---
 
## 5. Components: The Modular Cockpit
 
### Cards & Modules
*   **Execution:** Card dividers are strictly forbidden. Use `16px` of vertical whitespace or a transition from `surface-container-high` to `surface-container-low` to separate content. 
*   **Radius:** Maintain a strict `4px` (sm) radius across all components to reinforce the "engineered" aesthetic.
 
### Buttons
*   **Primary:** A "Machined" look. Background: `primary` (#bec6e0), Text: `on_primary` (#283044). 
*   **Active/AI:** Utilize `secondary` (Electric Cyan) with a subtle outer glow (0px 0px 8px).
*   **Tertiary:** Transparent background with `outline` text. Hover state shifts background to `surface-variant` at 10% opacity.
 
### Inputs & Fields
*   **Status:** Background `surface-container-lowest`. Bottom-only border using the Ghost Border rule (20% opacity `outline-variant`).
*   **Data IDs:** Always rendered in JetBrains Mono.
 
### Status Indicators (The Pulse)
*   **Health (Emerald):** `tertiary` (#4edea3)
*   **Incident (Crimson):** `error` (#ffb4ab)
*   **Approval (Amber):** Use custom hex #F59E0B.
Indicators should be small, 6px circular "LED" pips with a 2px outer diffusion of the same color.
 
### New Component: The "Data Ribbon"
A high-density horizontal strip placed at the top of cards containing technical metadata (system ID, uptime, latency) in `label-sm` JetBrains Mono, separated by pipes `|`.
 
---
 
## 6. Do's and Don'ts
 
### Do
*   **Do** embrace high-density layouts. Group related data points closely to allow for "at-a-glance" monitoring.
*   **Do** use JetBrains Mono for any value that is calculated or technical.
*   **Do** use intentional asymmetry—allow a side panel to be narrower than a standard grid to emphasize its utility.
 
### Don't
*   **Don't** use 100% opaque borders to separate sections. Use tonal shifts.
*   **Don't** use standard "Blue" for links. Use `secondary` (Electric Cyan) for active interactions.
*   **Don't** use large border radii. Anything above `8px` breaks the professional, "NASA" aesthetic.
*   **Don't** use pure black (#000000). Always use the `surface` palette to maintain depth and color-bleed.