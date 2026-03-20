---
title: Future work
has_children: false
nav_order: 13
---

# Known issues and future work

## Known issues

- **LLM model availability is unstable**: from time to time the selected model becomes unavailable or moves to paid-only access.  
  Current workaround is manual model switch in configuration/code.

- **Need for automatic model fallback**: there is no runtime algorithm to automatically select an available model when the primary one fails.  
  This causes manual intervention and service interruptions.

- **Low-quality image limitations**: dog recognition can fail on blurry, dark, or heavily compressed images.  
  This is mainly an ML quality limitation, not only an API/UI issue.

## Missing features

- **User account functionality** is not implemented yet:
  - registration/login
  - personal account area
  - persistent user profile and history data

- **Mobile-first capture flow** is missing:
  - currently users mainly upload an existing photo
  - direct camera capture from phone should be improved and made first-class

## Future work

- Implement a **model selection strategy** with fallback chain (primary -> secondary -> tertiary), health checks, and automatic switching.
- Improve ML robustness with better preprocessing and quality checks (blur/lighting warnings before inference).
- Extend the product from a detector to a **dog-care service platform** (recommendation tracking, reminders, personalized care plans).
- Add a stronger **mobile UX** with camera-first flow and optimized interaction for small screens.