---
name: Healing Touch
colors:
  surface: '#f7f9fb'
  surface-dim: '#d8dadc'
  surface-bright: '#f7f9fb'
  surface-container-lowest: '#ffffff'
  surface-container-low: '#f2f4f6'
  surface-container: '#eceef0'
  surface-container-high: '#e6e8ea'
  surface-container-highest: '#e0e3e5'
  on-surface: '#191c1e'
  on-surface-variant: '#3f4850'
  inverse-surface: '#2d3133'
  inverse-on-surface: '#eff1f3'
  outline: '#6f7881'
  outline-variant: '#bec7d1'
  surface-tint: '#006492'
  primary: '#006492'
  on-primary: '#ffffff'
  primary-container: '#2d9cdb'
  on-primary-container: '#003049'
  inverse-primary: '#8ccdff'
  secondary: '#49607c'
  on-secondary: '#ffffff'
  secondary-container: '#c7dfff'
  on-secondary-container: '#4b637e'
  tertiary: '#4a6364'
  on-tertiary: '#ffffff'
  tertiary-container: '#7f9899'
  on-tertiary-container: '#183031'
  error: '#ba1a1a'
  on-error: '#ffffff'
  error-container: '#ffdad6'
  on-error-container: '#93000a'
  primary-fixed: '#cae6ff'
  primary-fixed-dim: '#8ccdff'
  on-primary-fixed: '#001e2f'
  on-primary-fixed-variant: '#004b6f'
  secondary-fixed: '#d1e4ff'
  secondary-fixed-dim: '#b0c9e8'
  on-secondary-fixed: '#011d35'
  on-secondary-fixed-variant: '#314863'
  tertiary-fixed: '#cde8e9'
  tertiary-fixed-dim: '#b1cbcd'
  on-tertiary-fixed: '#051f20'
  on-tertiary-fixed-variant: '#334b4c'
  background: '#f7f9fb'
  on-background: '#191c1e'
  surface-variant: '#e0e3e5'
typography:
  headline-lg:
    fontFamily: Be Vietnam Pro
    fontSize: 30px
    fontWeight: '700'
    lineHeight: 38px
    letterSpacing: -0.02em
  headline-lg-mobile:
    fontFamily: Be Vietnam Pro
    fontSize: 24px
    fontWeight: '700'
    lineHeight: 30px
    letterSpacing: -0.01em
  headline-md:
    fontFamily: Be Vietnam Pro
    fontSize: 20px
    fontWeight: '600'
    lineHeight: 28px
  body-lg:
    fontFamily: Be Vietnam Pro
    fontSize: 18px
    fontWeight: '400'
    lineHeight: 26px
  body-md:
    fontFamily: Be Vietnam Pro
    fontSize: 16px
    fontWeight: '400'
    lineHeight: 24px
  label-md:
    fontFamily: Be Vietnam Pro
    fontSize: 14px
    fontWeight: '600'
    lineHeight: 20px
    letterSpacing: 0.01em
  label-sm:
    fontFamily: Be Vietnam Pro
    fontSize: 12px
    fontWeight: '500'
    lineHeight: 16px
rounded:
  sm: 0.25rem
  DEFAULT: 0.5rem
  md: 0.75rem
  lg: 1rem
  xl: 1.5rem
  full: 9999px
spacing:
  unit: 4px
  container-padding: 20px
  stack-gap: 16px
  section-gap: 32px
  gutter: 12px
---

## Brand & Style
The design system is built on the pillars of empathy, clarity, and reliability. It aims to reduce the anxiety often associated with medical environments by utilizing a "Warm Minimalist" aesthetic. The interface prioritizes breathability and soft transitions to create a calm user experience for patients and caregivers.

The visual direction avoids the sterile, high-contrast look of traditional clinical software in favor of a soothing, tactile interface that feels more like a wellness companion than a medical tool. This is achieved through generous whitespace, subtle tonal shifts, and a typography-first information hierarchy.

## Colors
The palette is centered around a calming Teal-Blue that serves as the primary anchor for actions and branding. To ensure the interface feels approachable, the background uses a soft off-white (`#F7F9FB`) rather than a pure, harsh white.

- **Primary (#2D9CDB):** Used for primary actions, progress indicators, and active states.
- **Secondary (#102A43):** A deep ink blue for text and high-contrast iconography to ensure legibility.
- **Tertiary (#E0FBFC):** A light wash used for subtle backgrounds and highlighted areas.
- **Surface:** Backgrounds are layered with very light grays and teals to differentiate content blocks without using heavy borders.

## Typography
This design system utilizes **Be Vietnam Pro** for its friendly, contemporary character. The typeface strikes a balance between professional geometry and humanist warmth, making it ideal for clear communication of health data.

Emphasis is placed on a clear hierarchy. Headlines use a slightly tighter letter spacing and heavier weights to feel grounded. Body text is set with generous line heights to ensure maximum readability for users who may be stressed or visually impaired. Labels are used for metadata and small buttons, utilizing a medium or semi-bold weight for distinction.

## Layout & Spacing
The layout follows a fluid model optimized for mobile-first interactions. It relies on a 4px baseline grid to maintain vertical rhythm.

- **Safe Areas:** A minimum of 20px horizontal padding is maintained on all screens to ensure content doesn't feel cramped against the bezel.
- **Negative Space:** Generous vertical margins (32px+) are used between major content sections (e.g., separating "Upcoming Appointments" from "Health Metrics") to reduce cognitive load.
- **Grid:** A 4-column layout is used for cards and buttons, with 12px gutters.

## Elevation & Depth
Depth is created through **Tonal Layering** rather than heavy shadows. This keeps the interface feeling light and modern.

- **Level 0 (Base):** The off-white surface (`#F7F9FB`).
- **Level 1 (Cards):** Pure white containers with a very soft, high-diffusion shadow (4px Blur, 2% Opacity, Secondary color tint).
- **Level 2 (Interactive):** Floating Action Buttons or active modals use a more pronounced shadow to indicate tapability.
- **Overlays:** Full-screen modals utilize a background blur (12px) with a semi-transparent white tint to maintain context while focusing user attention.

## Shapes
The shape language is defined by "Medium Roundness." A 12px corner radius (Rounded-LG) is the standard for cards and primary buttons. This curvature is significant enough to feel soft and safe, but structured enough to remain professional.

Smaller elements like chips or input fields use a 8px radius. Pill-shapes are reserved exclusively for status indicators (e.g., "Confirmed", "Pending") to differentiate them from actionable buttons.

## Components
- **Buttons:** Primary buttons are filled with the primary teal color and use 12px rounded corners. Text is white for high contrast. Secondary buttons use a teal outline or a light teal tint.
- **Cards:** Cards are the primary container. They feature a white background, 12px border radius, and a subtle 1px border in a slightly darker neutral tone than the background.
- **Input Fields:** Fields are tall (56px) to be thumb-friendly. They use a light gray background with a subtle teal focus state.
- **Chips:** Used for filtering or selecting appointment types. They use the `label-md` type and have a high degree of roundness (16px+).
- **Lists:** List items are separated by whitespace and light horizontal dividers (1px, 5% opacity). Each item has a minimum touch target height of 48px.
- **Medical Icons:** Icons are drawn with a 2px stroke width and rounded caps to match the typography and shape language.