# Technical Architecture & Stack Standards

**Version**: 1.0  
**Last Updated**: [Date when you customize this]

## Philosophy

- Boring technology that works > shiny new frameworks that break
- Convention over configuration
- Security by default
- Everything in a container so it can be ran locally in docker and then used in production with k8s or similar

## Core Stack

### Backend

** The most lightweigh approporiate solution for an MVP

### Frontend

**Styling**: Tailwind CSS  

### Testing

** nothing required for testing for an MVP

### Infrastructure

** AWS perferably severless, containers, basic "as a service"

## Code Standards

- just ship it, it's an mvp

## What We Don't Use

- ❌ Stored Procedures (business logic in code)
- ❌ Microservices (monolith until proven otherwise)
- ❌ Redux (React Query is sufficient)
- ❌ localStorage for tokens (security risk)

## More Details


**Note**: This file is referenced by all agents. Keep it current as your stack evolves.
