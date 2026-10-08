# AltLens Feature Specification

This document outlines the specifications for the AltLens accessibility auditing tool based on interviews and decisions made during planning.

## Feature 1: URL Input Validation, Design, and Failure State Behavior

### 1.1 URL Validation
- **Enhanced validation**: Check for proper URL format (http:// or https:// prefix) AND perform domain reachability check (DNS lookup) to warn about unreachable domains
- **Blocked URLs**: Prevent analysis of localhost, private IP addresses (192.168.x.x, 10.x.x.x, etc.), and internal domains for security

### 1.2 Input Design
- **Card-based design**: URL input field placed within a visually distinct card with appropriate spacing and typography
- **Input group**: Text input and submit button visually grouped together
- **Loading state**: Show skeleton loader during validation process
- **Error presentation**: Inline error messages displayed below the input field with red border on invalid input

### 1.3 Failure State Handling
- **Invalid URL format**: Show inline error "Please enter a valid URL (including http:// or https://)"
- **Network error**: Show inline error "Unable to reach the domain. Please check your internet connection and the URL."
- **Timeout**: Show inline error "Request timed out. Please try again or check if the website is accessible."
- **Blocked URL**: Show inline error "URL blocked for security reasons. Cannot analyze localhost or internal domains."
- **PageSpeed API issues**: Show toast notification for API-related errors (quota exceeded, invalid key, etc.)
- **Page fetch failed**: Show inline error "Unable to fetch the webpage. The site may be down or blocking requests."
- **Page too large**: Show inline error "Webpage exceeds maximum size limit (10MB) and cannot be processed."

## Feature 2: PageSpeed API Data Mapping and Dashboard Display

### 2.1 Display Approach
- **Detailed breakdown**: Show overall accessibility score with expandable sections for each failing audit
- **Only failing audits**: Focus exclusively on audits with scores < 1 (failing) to keep interface actionable

### 2.2 Score Presentation
- **Prominent score**: Large, prominent display of accessibility score (0-100 scale)
- **Color coding**: 
  - Red: < 50 (Poor)
  - Yellow: 50-89 (Needs Improvement)
  - Green: 90-100 (Good)
- **Score label**: Show both numeric score and qualitative label (e.g., "87 / 100 - Needs Improvement")

### 2.3 Audit Details
- **Grouped by category**: Organize failing audits by type (Images, Multimedia, ARIA, Color Contrast, etc.)
- **Action indicators**: For each failing audit, show what AltLens will automatically fix:
  - Image alt → "Will be fixed by AI image captioning"
  - Video/audio captions → "Will be fixed by AI transcription"
  - ARIA labels → "Requires manual fix (AI cannot determine appropriate labels)"
  - Color contrast → "Requires manual fix (design adjustment needed)"
- **Each audit shows**:
  - Audit title and description from Lighthouse
  - Count of affected elements
  - Icon representing the audit category
  - Fixability indicator (Auto-fixable vs Manual fix required)

### 2.4 Additional Information
- **Lighthouse version/fetch time**: Displayed as small footer text (e.g., "Lighthouse v11.3.0 • Fetched 2s ago")
- **Historical data**: Current analysis only (historical comparison to be implemented in future versions)

## Feature 3: Worker Instantiation Rules and Result Streaming

### 3.1 Worker Instantiation
- **Automatic instantiation on URL submit**: All three Workers (vision, whisper, translation) are created when user submits a URL for analysis
- **Lazy instantiation**: Workers are only created if they don't already exist (singleton pattern per worker type)
- **Singleton pattern**: Each worker file maintains a module-level pipeline instance to avoid reloading models

### 3.2 Result Streaming
- **Real-time streaming**: Image captions, audio transcriptions, and translations are displayed in the UI as soon as each individual item completes
- **Progress indication**: 
  - Overall progress bar showing "Processing X/Y items" (e.g., "Processing 12/50 images")
  - Per-item status indicators shown ONLY for failed items (error icons with tooltips)
- **Error handling**: 
  - Show error and continue processing: Display which specific item failed but keep processing remaining items
  - Individual item errors do not halt the entire analysis process
- **UI updates**: Immediate React state updates as each Worker response is received for responsive feedback

### 3.3 Specific Streaming Behaviors
- **Image captions**: Each caption appears below its corresponding image thumbnail as soon as generated
- **Audio transcriptions**: Transcribed text displayed in audio component UI upon completion
- **Translation**: English alt text translated to Arabic and displayed alongside original when both are available
- **Progress updates**: Model download/progress events update the Model Sidebar in real-time

## Definition of Done

A feature is considered "Done" and ready to commit when all of the following criteria are met:

### 4.1 Accessibility Compliance
- **WCAG AA compliance**: The feature itself must meet WCAG AA accessibility standards
- **Keyboard navigable**: All interactive elements reachable and operable via keyboard
- **Screen reader friendly**: Proper ARIA labels, roles, and live regions where appropriate
- **Color contrast**: Minimum WCAG AA contrast ratios (4.5:1 normal text, 3:1 large text)
- **Focus management**: Logical tab order and appropriate focus trapping/movement

### 4.2 Architectural Compliance
- **CLAUDE.md adherence**: Follows all rules outlined in the project's CLAUDE.md file
- **AGENTS.md adherence**: Follows all rules outlined in the project's AGENTS.md file
- **Server/Client boundaries**: 
  - No `cheerio`, `better-sqlite3`, or Node.js built-ins in Client Components or Web Workers
  - No `@xenova/transformers` in Server Components or Route Handlers
  - PageSpeed API key accessed only in Route Handlers
- **Web Worker rules**: 
  - All transformers pipeline calls run inside Web Workers
  - Lazy instantiation of Workers
  - Singleton pattern inside each Worker
  - Typed message passing via `lib/worker-protocol.ts`

### 4.3 Technical Quality
- **TypeScript strict**: Zero TypeScript compilation errors with `strict: true`
- **Linting passes**: ESLint reports no errors
- **No console errors**: Feature does not produce errors or warnings in browser developer tools during normal operation
- **Responsive design**: Feature works correctly on mobile, tablet, and desktop screen sizes

### 4.4 Verification
- **Manual testing**: Feature tested with real URLs to verify end-to-end functionality
- **User flow testing**: Complete analysis flow tested from URL input to results display
- **Error case testing**: Various failure scenarios tested to ensure proper error handling

## Notes
- This specification captures the decisions made during the planning interview
- Implementation should follow the specific patterns and examples provided in the skill files:
  - `.agents/skills/accessibility-audit/SKILL.md`
  - `.agents/skills/transformers-js/SKILL.md`
  - `.agents/skills/web-worker-bridge/SKILL.md`
- When in doubt, refer to CLAUDE.md and AGENTS.md for architectural guidance