## [2026-04-04] Spec Compliance Update

### Changes
- Added missing SKILL.md sections per ocas-skill-authoring-rules.md
- Updated skill.json with required metadata fields
- Ensured all storage layouts and journal paths are properly declared
- Aligned ontology and background task declarations with spec-ocas-ontology.md

### Validation
- ✓ All required SKILL.md sections present
- ✓ All skill.json fields complete
- ✓ Storage layout properly declared
- ✓ Journal output paths configured
- ✓ Version: 1.3.0 → 1.3.1

## [1.3.0] - 2026-04-02

### Added
- Structured entity observations in journal payloads (`entities_observed`, `relationships_observed`, `preferences_observed`)
- `user_relevance` tagging on journal observations (default `user` for task-related entities)
- Entity/Person, Concept/Event, Concept/Action added to ontology types
- Elephas journal cooperation in skill cooperation section

## [1.2.3] - 2026-03-31

### Added
- Required SKILL.md sections for OCAS specification compliance
- Filesystem field in skill.json

### Changed
- Documentation improvements for better maintainability

# Changelog

## [1.2.1] - 2026-03-30

### Added
- `## Responsibility boundary` and `## Ontology types` sections per authoring rules v2.4.0

## [1.2.0] - Initial versioned release
