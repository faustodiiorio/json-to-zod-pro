# Security Policy

## Scope

JSON to Zod Pro is a **fully client-side tool** with no server component, no external API calls, and no data transmission of any kind. All JSON input is processed locally in the browser and never leaves the user's machine.

## Reporting a vulnerability

If you discover a security issue (e.g., XSS via crafted JSON input), please open a GitHub issue tagged `security`. There is no sensitive infrastructure at risk, so public disclosure is acceptable.

## Dependencies

This tool has **zero runtime dependencies**. The only external resource loaded is the JetBrains Mono web font from Google Fonts, which can be removed by deleting the `<link>` tag in `index.html` without affecting functionality.
