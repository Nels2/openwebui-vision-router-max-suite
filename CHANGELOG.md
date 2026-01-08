# Changelog
All notable changes to this project will be documented in this file.

## [Unreleased]
### Added
- (add new features here)

### Changed
- (add behavior changes here)

### Fixed
- (add bug fixes here)

---

## [0.3.9] - 2026-01-08
### Added
- Low-hardware safeguards: token cap, injection trimming, concurrency limiting.
- EU language support via language preset (auto/en/pl/de/fr/es/it).
- Stable marker contract for suite interoperability:
  - VISION_META
  - VISION_NEEDS_FOLLOWUP
  - VERIFIER_MISSING

### Changed
- Default `ollama_base_url` to `http://localhost:11434` for portability.
- Verifier behavior tuned for limited hardware (recommended: conditional).

### Fixed
- Multi-image stability fixes.
- Async safety (avoid blocking the event loop).
- Follow-up pipes compatibility with injected markers.
