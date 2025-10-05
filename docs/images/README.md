# Images & Assets

This folder contains visual assets for GitHub documentation.

## Files

### Screenshots

| File | Size | Purpose | Used In |
|------|------|---------|---------|
| **sync-progress-hero.png** | 3.8KB | Hero image for README fork notice | README.md |
| **dark-progress-detail.png** | 3.8KB | Progress indicator detail shot | FORK.md |
| **progress-indicator-hero.png** | 260KB | Full interface screenshot | FORK.md |
| **countdown-1.png** | 3.8KB | Frame 1 of countdown | Future animated GIF |
| **countdown-2.png** | 3.7KB | Frame 2 of countdown | Future animated GIF |

### Diagrams

| File | Size | Purpose | Used In |
|------|------|---------|---------|
| **architecture.svg** | 2.8KB | System architecture diagram | FORK.md |

## Image Guidelines

### For Future Contributors

When adding new images to this folder:

1. **Optimize for web**
   - Target: <100KB per PNG
   - Use compression tools (TinyPNG, Squoosh, etc.)
   - SVG preferred for diagrams

2. **Naming convention**
   - Use kebab-case: `feature-name-description.png`
   - Be descriptive: `sync-progress-hero.png` not `image1.png`

3. **Documentation**
   - Add entry to this README
   - Include size and purpose
   - Note where it's used

4. **Alt text**
   - Always provide descriptive alt text in markdown
   - Example: `![Sync Progress Indicator](sync-progress-hero.png)`

## Screenshots Captured

All screenshots were captured from production deployment:
- **Environment**: Proxmox LXC container (10.10.20.8:8080)
- **Date**: October 5, 2025
- **Theme**: Dark mode (Bootswatch Darkly 5.3)
- **Browser**: Chromium via Playwright

## License

These images are part of the AdGuard Home Sync Enhanced Fork project.

**Original Work**: Copyright 2021 bakito
**Fork Enhancements**: Copyright 2025 Eddy Kawira

Licensed under Apache License 2.0. See [LICENSE](../../LICENSE) for details.
