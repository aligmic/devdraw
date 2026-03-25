# DevStash — Project Overview

> **One fast, searchable, AI-enhanced hub for all your developer knowledge & resources.**

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Target Users](#target-users)
- [Features](#features)
- [Data Models](#data-models)
- [Tech Stack](#tech-stack)
- [UI / UX](#ui--ux)
- [Monetization](#monetization)
- [Architecture Diagram](#architecture-diagram)

---

## Problem Statement

Developers keep their essentials scattered across too many places:

| What                  | Where it ends up              |
|-----------------------|-------------------------------|
| Code snippets         | VS Code, Notion, Gists       |
| AI prompts            | Chat histories                |
| Context files         | Buried in project directories |
| Useful links          | Browser bookmarks             |
| Docs & notes          | Random folders                |
| Terminal commands      | `.bash_history`, `.txt` files |
| Project templates     | GitHub Gists                  |

This causes **context switching**, **lost knowledge**, and **inconsistent workflows**. DevStash solves this by providing a single, unified hub.

---

## Target Users

| Persona                       | Primary Need                                             |
|-------------------------------|----------------------------------------------------------|
| **Everyday Developer**        | Fast access to snippets, prompts, commands, and links    |
| **AI-first Developer**        | Save prompts, contexts, workflows, and system messages   |
| **Content Creator / Educator**| Store code blocks, explanations, and course notes        |
| **Full-stack Builder**        | Collect patterns, boilerplates, and API examples         |

---

## Features

### A. Items & Item Types

Items are the core unit of DevStash. Each item has a **type** that determines its behavior and appearance. Users start with system types (non-editable) and can create custom types on Pro.

| Type        | Icon          | Color                    | Content Model | Access |
|-------------|---------------|--------------------------|---------------|--------|
| `snippet`   | `<Code />`       | `#3b82f6` 🔵 Blue       | Text          | Free   |
| `prompt`    | `<Sparkles />`   | `#8b5cf6` 🟣 Purple     | Text          | Free   |
| `command`   | `<Terminal />`   | `#f97316` 🟠 Orange     | Text          | Free   |
| `note`      | `<StickyNote />` | `#fde047` 🟡 Yellow     | Text          | Free   |
| `link`      | `<Link />`       | `#10b981` 🟢 Emerald    | URL           | Free   |
| `file`      | `<File />`       | `#6b7280` ⚪ Gray       | File upload   | Pro    |
| `image`     | `<Image />`      | `#ec4899` 🩷 Pink       | File upload   | Pro    |

> All icon names reference [Lucide Icons](https://lucide.dev/icons/).

**Content models** determine the input UI:

- **Text** — Markdown editor (snippets, notes, prompts, commands)
- **URL** — URL input field (links)
- **File** — File upload via Cloudflare R2 (files, images)

**Routes:** `/items/snippets`, `/items/prompts`, `/items/commands`, etc.

Items open in a **slide-out drawer** for quick access and creation.

### B. Collections

Collections are user-created groups that hold items of **any type**. An item can belong to **multiple collections**.

Example collections:

- *React Patterns* — snippets, notes
- *Context Files* — files
- *Python Snippets* — snippets
- *Interview Prep* — snippets, prompts, notes

### C. Search

Full search across:

- Item content
- Item titles
- Tags
- Item types

### D. Authentication

Powered by **NextAuth v5**:

- Email / Password
- GitHub OAuth

### E. Core Features

- Favorite collections and items
- Pin items to top
- Recently used items
- Import code from file
- Markdown editor for text-based types
- File upload for file/image types
- Export data (JSON / ZIP) — *Pro*
- Dark mode by default, light mode optional
- Add/remove items to/from multiple collections
- View which collections an item belongs to

### F. AI Features (Pro Only)

| Feature                | Description                                    |
|------------------------|------------------------------------------------|
| AI Auto-Tag            | Suggest tags based on item content             |
| AI Summaries           | Generate concise summaries of notes/snippets   |
| AI Explain This Code   | Plain-English explanation of a code snippet    |
| Prompt Optimizer       | Improve and refine AI prompts                  |

Powered by **OpenAI `gpt-5-nano`**.

---

## Data Models

### Entity Relationship Overview

```
USER 1──* ITEM *──* COLLECTION
  │         │            │
  │         *            │
  │       TAG (M2M)      │
  │                      │
  └──* ITEMTYPE          │
      (system + custom)  │
                         │
ITEM *──* COLLECTION  (via ITEMCOLLECTION join table)
```

### Prisma Schema

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ─── User ────────────────────────────────────────────

model User {
  id                   String    @id @default(cuid())
  name                 String?
  email                String?   @unique
  emailVerified        DateTime?
  image                String?
  isPro                Boolean   @default(false)
  stripeCustomerId     String?   @unique
  stripeSubscriptionId String?   @unique
  createdAt            DateTime  @default(now())
  updatedAt            DateTime  @updatedAt

  accounts   Account[]
  sessions   Session[]
  items      Item[]
  itemTypes  ItemType[]
  collections Collection[]

  @@map("users")
}

// NextAuth required models
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
  @@map("accounts")
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("sessions")
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
  @@map("verification_tokens")
}

// ─── Item Types ──────────────────────────────────────

model ItemType {
  id       String  @id @default(cuid())
  name     String             // "snippet", "prompt", etc.
  icon     String             // Lucide icon name
  color    String             // Hex color
  isSystem Boolean @default(false)
  userId   String?            // null for system types

  user  User?  @relation(fields: [userId], references: [id], onDelete: Cascade)
  items Item[]

  @@unique([name, userId])
  @@map("item_types")
}

// ─── Items ───────────────────────────────────────────

enum ContentType {
  text
  url
  file
}

model Item {
  id          String      @id @default(cuid())
  title       String
  contentType ContentType
  content     String?     // text content (null if file)
  fileUrl     String?     // R2 URL (null if text)
  fileName    String?     // original filename
  fileSize    Int?        // bytes
  url         String?     // for link types
  description String?
  language    String?     // programming language (optional)
  isFavorite  Boolean     @default(false)
  isPinned    Boolean     @default(false)
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt

  userId     String
  itemTypeId String

  user       User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  itemType   ItemType        @relation(fields: [itemTypeId], references: [id])
  tags       TagsOnItems[]
  collections ItemCollection[]

  @@index([userId, itemTypeId])
  @@index([userId, isFavorite])
  @@index([userId, isPinned])
  @@map("items")
}

// ─── Collections ─────────────────────────────────────

model Collection {
  id            String   @id @default(cuid())
  name          String
  description   String?
  isFavorite    Boolean  @default(false)
  defaultTypeId String?  // default item type for new items
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt

  userId String

  user  User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  items ItemCollection[]

  @@index([userId])
  @@map("collections")
}

// ─── Join Tables ─────────────────────────────────────

model ItemCollection {
  itemId       String
  collectionId String
  addedAt      DateTime @default(now())

  item       Item       @relation(fields: [itemId], references: [id], onDelete: Cascade)
  collection Collection @relation(fields: [collectionId], references: [id], onDelete: Cascade)

  @@id([itemId, collectionId])
  @@map("item_collections")
}

// ─── Tags ────────────────────────────────────────────

model Tag {
  id    String        @id @default(cuid())
  name  String        @unique
  items TagsOnItems[]

  @@map("tags")
}

model TagsOnItems {
  itemId String
  tagId  String

  item Item @relation(fields: [itemId], references: [id], onDelete: Cascade)
  tag  Tag  @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([itemId, tagId])
  @@map("tags_on_items")
}
```

> **Important:** Never use `db push` to update database structure. Always create and run migrations with `npx prisma migrate dev` in development and `npx prisma migrate deploy` in production.

---

## Tech Stack

| Layer            | Technology                                                         |
|------------------|--------------------------------------------------------------------|
| **Framework**    | [Next.js 16](https://nextjs.org/) / React 19                      |
| **Language**     | TypeScript                                                         |
| **Database**     | [Neon](https://neon.tech/) (Serverless PostgreSQL)                 |
| **ORM**          | [Prisma 7](https://www.prisma.io/) (latest)                       |
| **Auth**         | [NextAuth v5](https://authjs.dev/) (Email/Password + GitHub OAuth) |
| **File Storage** | [Cloudflare R2](https://www.cloudflare.com/products/r2/)          |
| **AI**           | [OpenAI](https://platform.openai.com/) — `gpt-5-nano`             |
| **Styling**      | [Tailwind CSS v4](https://tailwindcss.com/) + [shadcn/ui](https://ui.shadcn.com/) |
| **Caching**      | Redis (under consideration)                                        |

### Key Architecture Decisions

- **Single codebase** — Next.js handles frontend SSR pages and API routes (items CRUD, file uploads, AI calls) in one repo.
- **SSR pages with dynamic components** — Server-rendered shells with client-side interactive elements (drawers, search, editors).
- **Prisma migrations only** — No `db push`. All schema changes go through `prisma migrate dev` → `prisma migrate deploy`.

---

## UI / UX

### Design Principles

- Modern, minimal, developer-focused
- Dark mode by default, light mode optional
- Clean typography with generous whitespace
- Subtle borders and shadows
- Syntax highlighting for code blocks

**Design references:** [Notion](https://notion.so), [Linear](https://linear.app), [Raycast](https://raycast.com)

### Layout

```
┌──────────────────────────────────────────────────────┐
│  DevStash                              [Search] [+]  │
├────────────┬─────────────────────────────────────────┤
│            │                                         │
│  SIDEBAR   │   MAIN CONTENT                          │
│            │                                         │
│  Types     │   Collections (color-coded grid cards)  │
│  ├ Snippets│   ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  ├ Prompts │   │ React   │ │ Python  │ │ Context │  │
│  ├ Commands│   │ Patterns│ │ Scripts │ │ Files   │  │
│  ├ Notes   │   └─────────┘ └─────────┘ └─────────┘  │
│  ├ Links   │                                         │
│  ├ Files 🔒│   Items (color-coded border cards)      │
│  └ Images🔒│   ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│            │   │ useState │ │ git log │ │ API     │  │
│  Recent    │   │ Hook  🔵│ │ tips  🟠│ │ notes 🟡│  │
│  Collect.  │   └─────────┘ └─────────┘ └─────────┘  │
│  ├ React.. │                                         │
│  ├ Python..│   → Click item opens slide-out drawer   │
│  └ Context.│                                         │
│            │                                         │
└────────────┴─────────────────────────────────────────┘
```

- **Sidebar:** Item type links, recent collections (collapsible, becomes drawer on mobile)
- **Main:** Grid of collection cards (background color based on dominant item type) and item cards (border color matches type)
- **Item detail:** Slide-out drawer for quick view/edit

### Responsive Behavior

- Desktop-first design
- Sidebar collapses to a drawer on mobile
- Cards reflow to single column on small screens

### Micro-interactions

- Smooth transitions on navigation and drawer open/close
- Hover states on cards (subtle elevation/border change)
- Toast notifications for CRUD actions
- Loading skeletons during data fetches

---

## Monetization

### Free vs Pro Comparison

| Feature                        | Free          | Pro ($8/mo · $72/yr) |
|--------------------------------|:-------------:|:--------------------:|
| Items                          | 50            | Unlimited            |
| Collections                    | 3             | Unlimited            |
| System types (snippet–link)    | ✅            | ✅                   |
| File & Image uploads           | ❌            | ✅                   |
| Custom types                   | ❌            | ✅ (coming later)    |
| Basic search                   | ✅            | ✅                   |
| AI auto-tagging                | ❌            | ✅                   |
| AI code explanation            | ❌            | ✅                   |
| AI prompt optimizer            | ❌            | ✅                   |
| AI summaries                   | ❌            | ✅                   |
| Export (JSON/ZIP)              | ❌            | ✅                   |
| Priority support               | ❌            | ✅                   |

Payments handled via **Stripe** (stored as `stripeCustomerId` and `stripeSubscriptionId` on User).

> **Development note:** During development, all users have access to all features. Pro gating will be enforced before launch.

---

## Architecture Diagram

```
                          ┌──────────────────┐
                          │    Client (SSR)   │
                          │   Next.js 16 App  │
                          │   React 19 + TW4  │
                          └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │  Next.js API      │
                          │  Routes           │
                          │  /api/*           │
                          └──┬────┬────┬─────┘
                             │    │    │
              ┌──────────────┘    │    └──────────────┐
              ▼                   ▼                    ▼
   ┌──────────────────┐ ┌────────────────┐ ┌──────────────────┐
   │  Neon PostgreSQL  │ │ Cloudflare R2  │ │  OpenAI API      │
   │  (via Prisma 7)   │ │ (File Storage) │ │  gpt-5-nano      │
   │                    │ │                │ │                  │
   │  Users, Items,     │ │  Files, Images │ │  Auto-tag        │
   │  Collections,      │ │                │ │  Summarize       │
   │  Tags, Types       │ │                │ │  Explain Code    │
   └──────────────────┘ └────────────────┘ │  Optimize Prompt │
                                            └──────────────────┘
              ┌──────────────────┐
              │   NextAuth v5    │
              │  Email/Password  │
              │  GitHub OAuth    │
              └──────────────────┘

              ┌──────────────────┐
              │  Stripe          │
              │  Subscriptions   │
              │  & Billing       │
              └──────────────────┘
```

---

## Quick Reference: Routes

| Route                      | Description                    |
|----------------------------|--------------------------------|
| `/`                        | Dashboard / Home               |
| `/items/snippets`          | All snippets                   |
| `/items/prompts`           | All prompts                    |
| `/items/commands`          | All commands                   |
| `/items/notes`             | All notes                      |
| `/items/links`             | All links                      |
| `/items/files`             | All files (Pro)                |
| `/items/images`            | All images (Pro)               |
| `/collections`             | All collections                |
| `/collections/[id]`        | Single collection view         |
| `/search`                  | Global search                  |
| `/settings`                | User settings & account        |
| `/settings/billing`        | Stripe billing portal          |

---

*Last updated: March 2026*
