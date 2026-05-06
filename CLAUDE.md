# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Botanica** is a single-file HTML plant identifier app for serious home growers and plant collectors. Users upload plant photos, receive AI-powered botanical identification backed by Wikipedia citations, and maintain a personal plant library with care notes and growing observations.

**Key constraint**: Single self-contained HTML file with zero dependencies—no build process, no server, no frameworks. The entire app (~1500 lines of HTML, CSS, and JavaScript) is in one file that runs in any browser, including iPhone Safari as a PWA.

### Why This Architecture

- **No build step required**: Users open the HTML file directly in their browser
- **Offline capable**: Works without internet after initial load (except API calls)
- **Easy to share**: Single file can be emailed or hosted anywhere
- **Low friction for iteration**: Claude can modify and test changes instantly

---

## Core Technical Stack

| Component | Choice | Notes |
|-----------|--------|-------|
| **Platform** | Single HTML file | ~1500 lines: HTML + CSS + inline JavaScript |
| **AI Model** | claude-sonnet-4-20250514 | Vision API for image analysis, structured JSON responses |
| **Storage** | Browser localStorage | Personal plant collection persisted across sessions |
| **Image Handling** | Client-side compression | Auto-compress to 1200px max dimension before API call |
| **UI Framework** | Plain HTML + CSS | No external dependencies; responsive design for mobile |
| **Target Devices** | Desktop browsers + iPhone Safari PWA | Tested and confirmed working |

---

## Architecture Overview

The single HTML file is organized into distinct sections (though not literally separate files):

### 1. **Identification Engine** (Core Logic)
- **Confidence threshold**: 75%. Below this, the app triggers follow-up dialogue instead of guessing
- **Follow-up flow**: Maximum 2 rounds. After 2 rounds, admits uncertainty honestly with best hypothesis
- **AI prompt injection**: Includes a 26-species reference database embedded in every identification prompt (Bay Area natives, Mediterranean, succulents, Australian plants, etc.)
- **Wikipedia citation**: Auto-generated from scientific name; shown on all results

**Key prompt rules**:
- Reduce confidence on distance shots of fine-textured shrubs → request close-up of leaf/phyllode
- Reduce confidence on low-light photos → request daytime photo instead
- Auto-inject phosphorus sensitivity warning for any Proteaceae identification
- Australian natives suspected at distance → ask for leaf/phyllode close-up specifically

### 2. **Image Processing**
- Accept 1–5 images per identification session
- Display thumbnail grid with remove/add controls
- **Compression**: Silently compress each image to max 1200px dimension, maintaining aspect ratio
- **Storage**: Save as 400px thumbnails in localStorage to conserve storage quota
- Handle multiple formats (JPEG, HEIF, PNG, WebP)

### 3. **Personal Plant Collection**
- Save identified plants to localStorage with full metadata
- Fields: scientific name, common name, family, origin, light, care level, growth rate, temperature
- **Care guide**: 8 fields (watering, soil, humidity, fertilizer, temperature, tolerance, pruning, propagation)
- **Personal notes**: location acquired, date, condition, custom notes
- **Inline editing**: Edit location, date, condition, notes directly on plant detail view
- Search and filter by common/scientific name, condition chips
- Photo carousel for multi-photo plants

### 4. **UI Components**
- **Main flow**: Photo upload → AI identification → Results (two tabs: Overview, Care Guide)
- **History tab**: Records all identification sessions with full conversation threads
- **Collection tab**: Grid view of saved plants with search/filter
- **Detail view**: Bottom sheet modal for plant details, editable notes, photo carousel

**Styling**: Earthy botanical aesthetic (moss green #2d4a2d, fern green #4a7c59, cream #f5f0e8, parchment #ede5d4)

---

## Current Status & Known Issues

### ✅ Working
- Photo upload and compression
- AI identification with follow-up dialogue
- Personal collection with notes
- Care guide and Wikipedia citations
- Multi-photo support
- iPhone/Safari PWA (tested)

### ⚠️ Needs Work
- **Git repo sync**: The working app in Google Drive has not been committed to git. Current git repo contains only a partial stub (371 lines). Real app (1500 lines) needs to be committed.
- **localStorage backup/restore**: Personal collections have no export/import backup. One `localStorage.clear()` call loses everything. Build JSON export/import to Collection tab.
- **Australian native accuracy**: Tested at 81–85% accuracy. Australian natives (Acacia, Leptospermum, Callistemon, Grevillea) are a consistent blind spot—documented testing gap.
- **Species database status**: 26-species reference database was rebuilt Apr 29, 2026 from session history and exists in Drive. Needs to be re-embedded into app identification prompt.

---

## How to Develop

### Opening the App
```bash
# From ~/Developer/botanica/
open plant-identifier.html
```
The app runs entirely in the browser—no server needed. Refresh to reload after editing.

### Testing Changes
1. Make code changes directly in `plant-identifier.html`
2. Save the file
3. Refresh the browser tab
4. Test the feature (upload photos, check results, test edge cases)

### Making Commits
After meaningful changes, save to git:
```bash
git add plant-identifier.html
git commit -m "description of what changed"
```

Examples:
```bash
git commit -m "improved Australian native detection in prompt"
git commit -m "added localStorage backup/restore JSON export"
git commit -m "fixed image compression to handle HEIF on iPhone"
```

To see history:
```bash
git log --oneline
```

---

## Key Development Patterns

### Adding Features to the Identification Prompt
The Claude API call is a large, structured prompt that:
1. Defines the task (identify plant from image)
2. Injects the 26-species reference database
3. Sets rules for confidence thresholds and follow-up questions
4. Specifies JSON response structure

When editing the prompt:
- Keep the reference database in sync (Bay Area natives + Mediterranean + succulents + Australian plants)
- Test against 10+ known plants after prompt changes
- Monitor for accuracy regressions—Australian natives especially fragile

### Editing the UI
All CSS is in the `<style>` tag at the top. The DOM structure is simple HTML with minimal nesting. JavaScript event handlers attach to buttons and form inputs by ID.

When adding UI:
- Test on iPhone Safari (the primary constraint)
- Maintain the earthy color palette
- Keep components mobile-first: stacked on small screens, horizontal on desktop
- Test localStorage quota—images consume 50–70% of the 10MB limit

### Handling localStorage Edge Cases
Collection data is stored as JSON in localStorage:
```javascript
localStorage.setItem('botanicaCollection', JSON.stringify(plants));
plants = JSON.parse(localStorage.getItem('botanicaCollection')) || [];
```

When editing storage:
- Always JSON.stringify when saving
- Always JSON.parse when loading, with fallback to empty array
- Test quota errors—when localStorage is full, the app should gracefully strip photos before losing plant data (already implemented)

---

## Testing Approach

### Field Testing Protocol
The app has been tested against real plants in the Bay Area:
- **Session 1** (Mar 2026): 12 plants, 83% accuracy (10/12 correct)
- **Accuracy gap**: Australian natives—Acacia and Leptospermum misidentified at distance

When testing new features or prompt changes:
1. Test against 5–10 known plants first (plants you can verify)
2. Log results: plant name, photo distance/lighting, result, accuracy
3. Document failures with specific details (why did it miss?)
4. Update prompt rules if patterns emerge
5. Re-test the same plants to verify the fix

### Accuracy Benchmarks
- **Target**: 80%+ accuracy across diverse plants
- **Container/indoor plants**: Typically 95%+
- **Australian natives**: Currently 50–60%, target 85%+
- **Distance shots**: Higher risk; request close-up if unclear

---

## Important Constraints & Decisions

### Why 75% Confidence Threshold?
Below 75%, the app asks follow-up questions rather than guessing. This treats the user as a knowledgeable collaborator, not a passive recipient. A serious plant collector would rather be asked "Can I see the petiole attachment?" than receive a confident wrong answer.

### Why Maximum 2 Follow-Up Rounds?
After 2 rounds, the app admits uncertainty honestly with best hypothesis and reason for uncertainty. This balances thoroughness with user patience—endless back-and-forth is frustrating.

### Why Single-File Architecture?
No build process, no deployment step, no framework dependencies. Claude can modify, save, and test in < 5 seconds. This is critical for iterative development and debugging.

### Why localStorage (Not Cloud)?
Phase 1 is offline-first. Phase 2 may add cloud sync with Google Drive, but for now, localStorage keeps the app self-contained. Be aware: one `localStorage.clear()` loses all collections. Build backup/restore first.

---

## Common Tasks

### Re-embed Species Database
The species database (26 plants covering Bay Area natives, Mediterranean, succulents, Australian plants, etc.) should be injected into the Claude prompt as context. Currently exists in Drive but not in app prompt.

**Location in code**: Find the `messages` array in the API call. The system prompt or first user message should include the species data as a formatted table or JSON list.

### Improve Australian Native Accuracy
Acacia and Leptospermum are consistently misidentified. Add these specific rules to the prompt:
1. Request close-up of leaf/phyllode if photo is at distance
2. Ask "Is this plant from Australia or New Zealand?" in follow-up
3. Check the reference database before committing to ID

### Add localStorage Backup/Restore
Users need to export their plant collection as JSON and restore/merge from backup. Add two buttons to the Collection tab:
1. "Export Collection" → downloads `botanica-backup.json`
2. "Import Backup" → file picker, merges or replaces collection

Handle merge logic carefully: allow user to choose "replace" or "merge by date".

---

## Code Style Notes

- **Plain JavaScript**: No frameworks, minimal abstraction. Functions are organized by feature (upload, identification, collection, UI)
- **Event handlers**: Attach via `addEventListener` or inline `onclick` attributes (both present in current code)
- **localStorage**: Always JSON.stringify/parse
- **API calls**: Fetch with appropriate error handling; log failures
- **CSS**: Use CSS variables (defined at `:root`) for colors; mobile-first media queries
- **Comments**: Minimal—code is straightforward. Add comments only for non-obvious logic (API response parsing, threshold logic)

---

## Debugging Tips

### Image Compression Not Working
Check browser console. If compression fails silently:
1. Verify canvas is available (should be, but check)
2. Test with a simple JPEG first (not HEIF or WebP)
3. Log the image dimensions before/after compression

### AI Returns Wrong Plant
1. Check the confidence score—below 75% should trigger follow-up
2. If above 75%, the prompt needs adjustment
3. Check if the species is in the reference database—if not, consider adding it
4. Test with the same photo through Claude API directly (claude.ai) to isolate the issue

### localStorage Quota Error
The app should handle gracefully (strip photos before losing plant data). If data is lost:
1. Check user's localStorage quota: browser DevTools → Application → Storage
2. Implement or improve the backup/restore feature
3. Consider Phase 2 cloud storage migration

### Mobile Layout Issues
Test on iPhone Safari specifically (not desktop Chrome responsive mode). iPhone Safari has quirks:
- Bottom sheet modals may overlap keyboard
- Fixed positioning may behave differently
- Touch events may not fire as expected on certain elements

---

## Git Workflow

### Current State (as of May 6, 2026)
Branch: `claude/add-claude-documentation-AKKBx` (empty, no commits yet)

### Immediate Tasks
1. Sync git with actual app from Drive
2. Commit the working app: `git commit -m "initial: working botanica app from Drive"`
3. Add this CLAUDE.md file
4. Commit CLAUDE.md
5. Push to the feature branch

### Branching Strategy
Work on feature branches:
- `claude/add-claude-documentation-AKKBx` ← CLAUDE.md updates
- `claude/improve-australian-natives` ← prompt refinement
- `claude/add-backup-restore` ← localStorage export/import
- Merge back to main once tested

---

## Resources in Google Drive

- **botanica-v2.html**: The working app (actual source of truth)
- **Botanica-Project-Status.docx**: Feature checklist and test results
- **botanica-project-plan-v2.docx**: Detailed design decisions and task breakdown
- **Botanica-Progress-Plan.md**: Recent status, roadmap, and immediate next steps
- **species-data/bay-area-natives**: 26-species reference database (covers CA natives, Australian/NZ, Mediterranean, South African proteaceae, succulents)

---

## Final Notes

This is a genuine human ↔ AI collaboration project. The human brings domain expertise (what serious collectors need), real-world testing, and curation decisions. Claude brings vision analysis, structured data extraction, and code generation.

When developing:
- Test against real plants, not hypothetical scenarios
- Document failures—they inform prompt improvements
- Iterate: test → observe pattern → refine prompt → re-test
- Prioritize Australian native accuracy—it's the biggest current gap
- Commit frequently; back up to Drive; never rely on working directory alone

Good luck with development! 🌿
