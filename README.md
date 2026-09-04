# Agent Skills

My custom agent skills for extension of agent capabilities.

## Skills

### Modular (Pragmatic — recommended for MVPs & small teams)

#### [Frontend React Modular Architecture](skills/frontend-react-modular/SKILL.md)
Clean, modular React apps with Vertical Slicing and Screaming Architecture. Hooks for logic, presentational components with props only. Supports Vite and Next.js. React Native ready.

#### [Backend NestJS Modular Architecture](skills/backend-nestjs-modular/SKILL.md)
Clean, modular NestJS APIs with Vertical Slicing. Controller → Service → Repository pattern. Supports Prisma. No hexagonal overhead.

### Hexagonal (Enterprise — strict DDD & ports/adapters)

#### [Frontend React Vertical-Hexagonal Architecture](skills/frontend-react-vertical-hexagonal/SKILL.md)
Hexagonal Architecture, Vertical Slicing, and DDD in React frontends (Vite and Next.js).

#### [Frontend Vertical-Hexagonal Architecture (CodelyTV Style)](skills/frontend-vertical-hexagonal/SKILL.md)
Feature-First with Hexagonal Architecture using React, Next.js, Zustand, React Query, and Axios. Validated via `eslint-plugin-hexagonal-architecture`.

#### [Backend NestJS Vertical-Hexagonal Architecture](skills/backend-nestjs-vertical-hexagonal/SKILL.md)
Hexagonal Architecture, Vertical Slicing, and DDD using NestJS and Prisma. Validated via `eslint-plugin-hexagonal-architecture`.

## Installation

Install all skills:
```bash
npx skills add ricardosv46/agent-skills
```

Install a specific skill:
```bash
# Modular (pragmatic)
npx skills add ricardosv46/agent-skills@frontend-react-modular
npx skills add ricardosv46/agent-skills@backend-nestjs-modular

# Hexagonal (enterprise)
npx skills add ricardosv46/agent-skills@frontend-react-vertical-hexagonal
npx skills add ricardosv46/agent-skills@frontend-vertical-hexagonal
npx skills add ricardosv46/agent-skills@backend-nestjs-vertical-hexagonal
```
