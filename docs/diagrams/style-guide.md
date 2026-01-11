# Diagram Style Guide

## General Principles
- Prefer clarity over completeness
- Avoid visual clutter
- Every diagram must include a title and legend

## Naming Conventions
- Services: PascalCase
- External systems: UPPER_SNAKE_CASE
- Clients: "Client", "Player", "User"

## Colors (logical meaning, not mandatory in Mermaid)
- Blue: Control Plane
- Green: Data Plane
- Red: Failure paths
- Yellow: Security boundaries

## Diagram Types
- Sequence: request lifecycles
- Flow: high-level system movement
- Architecture: components and dependencies
- Data Flow: data boundaries and classification

## Annotation Rules
- Mark trust boundaries
- Label network hops
- Show failure points explicitly
