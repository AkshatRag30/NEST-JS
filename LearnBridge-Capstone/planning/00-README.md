# LearnBridge Planning Folder

This folder is the complete plan for a project called LearnBridge, a course marketplace and learning platform API built with NestJS. It exists for one reason: every concept spread across the eight folders you have already read notes for, the full NestJS course, MongoDB, PostgreSQL, Prisma with Neon, GraphQL, plain JWT concepts, JWT with MongoDB, and rate limiting with Throttler, gets pulled into one real, coherent product instead of eight disconnected toy examples. By the time LearnBridge is finished, you will have written real, working code for everything those notes only let you read about.

Nothing in this folder is code yet. This is the thinking that has to happen before the first line of `src` gets written, the same way a real engineering team writes a plan before opening an editor. Treat this folder the way you would treat a real product's planning documents, because that is exactly what it is trying to teach you to do.

## Why a single project instead of eight small ones

Small isolated examples, which is exactly what the eight source folders are, are excellent for learning one idea in isolation. They are bad at teaching you the thing that actually makes someone a strong backend engineer: how ten different concepts interact and constrain each other inside one real system. A course marketplace needs real user accounts, so it needs authentication. It needs instructors and courses, which is a one to many relationship. It needs students enrolling in many courses while a course has many students, which is many to many, but with extra data attached to the relationship itself, something none of the eight reference projects actually showed you. It needs payments, which means transactions that must not half succeed. It needs a place to log high volume, throwaway events like notifications, which is exactly the kind of data MongoDB is good at, sitting next to the relational data in PostgreSQL that is exactly what Prisma is good at. It benefits from exposing part of itself over GraphQL as well as REST. And every public endpoint, especially login, needs rate limiting. One project, built deliberately, forces all of this to fit together, which is the actual skill.

## How to read this folder

Read the files in this folder in the order below the first time through. After that, treat it as a reference you return to at the start of each phase of actually building the thing.

1. [01-PRD-master.md](01-PRD-master.md), the master product requirements document. What LearnBridge is, who it is for, and the full feature list at a glance.
2. [02-tech-stack-and-architecture.md](02-tech-stack-and-architecture.md), every technology decision explained, and why, with a direct line back to the source folder that taught you that technology.
3. [03-concept-coverage-map.md](03-concept-coverage-map.md), the most important file in this folder. A complete table of every single concept from all eight notes folders, and exactly where in LearnBridge you will practice it for real.
4. [04-data-model-and-relationships.md](04-data-model-and-relationships.md), every entity in the system, every field, and every relationship, labeled by its exact relationship shape.
5. [05-phase-1-PRD-foundations-and-config.md](05-phase-1-PRD-foundations-and-config.md) through [14-phase-10-PRD-typeorm-comparison-appendix.md](14-phase-10-PRD-typeorm-comparison-appendix.md), one small PRD per build phase, in the order you should actually build them.
6. [15-roadmap-and-milestones.md](15-roadmap-and-milestones.md), a suggested pace and a checklist of what "done" looks like at each stage.
7. [16-definition-of-pro-checklist.md](16-definition-of-pro-checklist.md), a closing self test. If you can do everything on that list without looking anything up, this project did its job.

## Turning this plan into actual code

The `prompts` folder sitting next to this file holds one complete, ready to paste prompt per phase, meant for Claude Code, each one telling it exactly which of the files above to read before writing anything, so the actual build never drifts from this plan. Start at [prompts/00-README.md](prompts/00-README.md) before using any of them, it is honest about the real tradeoff between using them and building each phase by hand yourself.

## A promise this plan makes to you

Every one of the eight reference projects you already have notes on contained at least one real bug or a real gap, a missing environment variable, a test that fails the moment you run it, a guard that is defined but never actually applied, an import that only works by accident. Those were not mistakes to laugh at, they were the most honest and useful part of those notes, because production code always has problems like that, and learning to spot them is most of the job. LearnBridge's plan is written the same way, it calls out on purpose, ahead of time, exactly where a shortcut would be easy to take and what the disciplined version costs instead, so that when you build it, you are building the fixed version from the start, not repeating the same mistakes and discovering them later.
