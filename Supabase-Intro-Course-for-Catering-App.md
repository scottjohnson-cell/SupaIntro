# Supabase for Your Catering App: An Introductory Course

*Built for someone who's comfortable with GitHub and currently building recipes, production guides, purchase guides, and BEOs in Google Docs/Sheets.*

---

## Module 0: The Mental Model — What Supabase Actually Is

Forget the marketing language for a second. Here's the plain version:

**Supabase = a real Postgres database, sitting in the cloud, with a bunch of pre-built services wrapped around it so you don't have to build a backend from scratch.**

Think about what you're doing today in Google Sheets/Docs:

- A recipe is a "document" with a list of ingredients and steps.
- A BEO is a document that pulls from a recipe, adds event details (date, headcount, timing), and gets shared with staff.
- A purchase guide aggregates ingredient quantities across multiple BEOs to figure out what to buy.

The problem with Google Docs/Sheets for this is that **the data isn't actually connected** — you're copy-pasting or manually re-entering the same ingredient across ten different sheets. If you fix a recipe, nothing downstream updates.

Supabase solves this by giving you an actual **relational database**. A recipe is a row in a `recipes` table. Its ingredients are rows in a `recipe_ingredients` table that *reference* the recipe. A BEO references one or more recipes. A purchase guide is really just a **query** that sums up ingredient needs across every BEO for a given week. Change the recipe once, and everything that references it stays accurate.

On top of that database, Supabase gives you, out of the box:

| Service | What it does | Your use case |
|---|---|---|
| **Database (Postgres)** | Stores all your structured data | Recipes, ingredients, events, vendors |
| **Auto-generated API** | Every table instantly gets a REST + GraphQL API — no backend code required | Your app reads/writes recipes without you writing server code |
| **Auth** | User accounts, logins, roles | Chefs, event managers, and admins log in with different permissions |
| **Storage** | File storage (like S3) integrated with the database | Recipe photos, plated dish references, signed BEO PDFs |
| **Row Level Security (RLS)** | Rules that control who can see/edit which rows | Line cooks see production guides but can't edit pricing |
| **Realtime** | Live updates pushed to connected clients | A BEO update instantly reflects for everyone viewing it |
| **Edge Functions** | Small serverless functions for custom logic | E.g., auto-generate a PDF purchase guide, send an email |

You will mostly live in the **Database** and **Auth** pieces at first. Storage and Edge Functions come later once the basics are solid.

---

## Module 1: Core Concepts You Need Before Touching Anything

### Tables, rows, and relationships
A Postgres database is made of **tables** (like a tab in a spreadsheet), which have **columns** (fields) and **rows** (records). The power move relational databases give you that spreadsheets don't: tables can **reference each other**.

For example:
- `recipes` table has a row for "Braised Short Rib"
- `ingredients` table has a row for "Short Rib, Boneless"
- `recipe_ingredients` table has a row that says "Recipe #12 uses Ingredient #45, quantity 6, unit lb"

This is called a **join table** — it's how you connect a many-to-many relationship (one recipe has many ingredients, one ingredient appears in many recipes) without duplicating data.

### Primary keys and foreign keys
- **Primary key**: a unique ID for each row (Supabase typically uses a `uuid` or auto-incrementing `id`).
- **Foreign key**: a column that points to another table's primary key. This is the "reference" mechanism — it's what makes the database relational instead of just a pile of disconnected spreadsheets.

### The auto-generated API
This is the single biggest difference from "building an app from scratch." The moment you create a table in Supabase, it automatically exposes a REST endpoint and a GraphQL endpoint for that table — <cite index="9-1">you did not write a single endpoint, it is just there</cite>. Your app (built in whatever framework you choose) talks to Supabase using a client library, and the client library calls that auto-generated API for you. You'll rarely write raw HTTP requests.

### Row Level Security (RLS)
This is Postgres's built-in permission system, and Supabase leans on it heavily. Without RLS turned on, anyone with your API key could read or write any row in a table. With RLS, you write policies like "a user can only see BEOs for events they're assigned to" directly as SQL rules attached to the table. This is *the* thing to understand well before you put any real business data in — it's the difference between a secure app and an open database.

### Migrations
Instead of clicking around a UI to change your database structure (which is easy to lose track of), you write your schema changes as SQL files called **migrations**, and store them in your GitHub repo. This is the piece that will feel most natural to you as a GitHub user — schema changes get reviewed and versioned exactly like code changes.

---

## Module 2: Setting Up Your Project

1. Create a free account at supabase.com and start a **New Project**. You'll set a database password (save this somewhere safe — a password manager, not a sticky note) and pick a region close to your team.
2. Once it spins up, you land on the **Dashboard**, which has these key areas:
   - **Table Editor** — a spreadsheet-like UI for viewing/editing tables (this will feel the most familiar coming from Sheets)
   - **SQL Editor** — where you run raw SQL queries and write migrations
   - **Database** — schema view, relationships, indexes
   - **Auth** — manage users and login providers
   - **Storage** — file buckets
   - **Project Settings → API** — this is where you'll get your **Project URL** and **API keys**, which your app needs to connect

You don't need to touch Edge Functions or Realtime yet.

---

## Module 3: Modeling Your Catering Data

This is the most important design work you'll do, and it's worth sketching on paper before building anything. Here's a starting schema for your use case — treat this as a first draft, not gospel:

```sql
-- Core recipe data
create table recipes (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  yield_qty numeric,
  yield_unit text,
  instructions text,
  created_at timestamptz default now()
);

create table ingredients (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  default_unit text,
  current_cost numeric  -- cost per unit, for purchase guide calculations
);

-- Join table: which ingredients belong to which recipe, and how much
create table recipe_ingredients (
  id uuid primary key default gen_random_uuid(),
  recipe_id uuid references recipes(id) on delete cascade,
  ingredient_id uuid references ingredients(id),
  quantity numeric not null,
  unit text not null
);

-- Events / BEOs
create table events (
  id uuid primary key default gen_random_uuid(),
  client_name text not null,
  event_date date not null,
  headcount int,
  venue text,
  notes text
);

-- Which recipes are being served at which event, and at what scale
create table event_menu_items (
  id uuid primary key default gen_random_uuid(),
  event_id uuid references events(id) on delete cascade,
  recipe_id uuid references recipes(id),
  scale_factor numeric default 1  -- e.g. 2.5x the base recipe yield
);
```

Notice what this buys you that Google Docs can't:
- A **production guide** for an event is a query joining `event_menu_items` → `recipes` → `recipe_ingredients`, scaled by headcount.
- A **purchase guide** across a whole week is the same query, summed across every event happening that week, grouped by ingredient.
- A **BEO** is `events` plus `event_menu_items`, rendered into a document — the data view and the "pretty document" become two different outputs of the same source of truth.

This is the conceptual leap: **stop thinking in documents, start thinking in data that documents are generated from.**

---

## Module 4: GitHub + Supabase — How the Integration Actually Works

Since you already know GitHub, here's the workflow that will feel most natural, based on how Supabase's GitHub integration is currently built:

### The `supabase/` folder
When you install the Supabase CLI and run `supabase init` in your repo, it creates a `supabase/` folder. <cite index="10-1">It's safe to commit this folder to version control.</cite> This folder holds:
- `migrations/` — SQL files representing every schema change, in order
- `config.toml` — project configuration
- `seed.sql` — sample/starter data

### Local development
The CLI can run Supabase's entire stack locally in Docker — <cite index="10-1">the CLI includes the entire Supabase stack, and a few additional images useful for local development, like a local SMTP server and a database diff tool</cite>. This means you can build and test schema changes on your laptop before anything touches production.

### Connecting GitHub to Supabase
In your Supabase project: **Project Settings → Integrations → GitHub Integration → Authorize GitHub**, then pick your repo and confirm the working directory (the path to your `supabase/` folder). <cite index="11-1">With this integration, Supabase watches all commits, branches, and pull requests of your GitHub repository.</cite>

Once connected, two things become possible:

1. **Deploy to production**: <cite index="12-1">you can deploy directly from GitHub on any plan — connect your repository and changes pushed to main are automatically deployed to your project.</cite>
2. **Preview branches** (Pro plan): <cite index="12-1">branching adds isolated preview environments for each pull request, so you can safely experiment with changes to your Supabase project without affecting your production setup.</cite> Each branch gets <cite index="12-1">a separate environment with its own Supabase instance and API credentials.</cite> When a PR opens, <cite index="11-1">a comment is added to the PR with the deployment status of your preview branch, and the migrations in the migrations subdirectory are automatically run</cite> — importantly, <cite index="11-1">no production data is copied to your preview branch, to protect sensitive production data.</cite>

**Practically, this means your day-to-day workflow looks like:**

```
1. Create a git branch: git checkout -b add-purchase-guide-view
2. Make a schema change locally (supabase db diff -f add_purchase_view)
3. Test it against your local Docker instance
4. Commit the migration file, push, open a PR
5. Supabase spins up an isolated preview database for that PR
6. Review the change, merge to main
7. Supabase automatically applies the migration to production
```

This is genuinely just Git branching applied to your database schema — which is likely the part that will feel most comfortable to you already.

### A newer, simpler alternative: Dashboard branching
As of the most recent Supabase updates, you don't strictly need Git to branch anymore — <cite index="20-1">branching without Git is now the default for all Supabase projects; you can create a branch directly from the Supabase Dashboard, make schema changes, review the diff, and merge, with no Git configuration required.</cite> <cite index="20-1">Git-based branching remains fully supported for teams that manage migrations in version control, and you can start with dashboard branching and add a Git integration later.</cite>

Given you're already comfortable with GitHub and want your schema versioned alongside your app code, I'd recommend going straight to the Git-based workflow — but it's good to know the no-Git dashboard option exists for quick one-off experiments.

---

## Module 5: Connecting Your Actual App

Once your schema exists, your frontend app (whatever you build it in — the CLI supports quickstarts for Next.js, React, Vue, and others) talks to Supabase using a client library and two values from **Project Settings → API**:
- Your **Project URL**
- Your **Publishable/anon key** (safe to expose in client-side code, because RLS is what actually protects your data — the key alone doesn't grant access to everything)

A basic query, once connected, looks like:

```js
const { data, error } = await supabase
  .from('recipes')
  .select('name, instructions, recipe_ingredients(quantity, unit, ingredients(name))')
  .eq('id', recipeId)
```

That single call pulls a recipe *and* all its ingredients in one go — the kind of join that would take you multiple manual lookups in Sheets.

---

## Module 6: Security — Don't Skip This

Before any real client or pricing data goes in, turn on **Row Level Security** on every table and write explicit policies. A simple starting policy for your team might be:

```sql
alter table events enable row level security;

create policy "Staff can view all events"
on events for select
to authenticated
using (true);

create policy "Only admins can edit events"
on events for update
to authenticated
using (auth.jwt() ->> 'role' = 'admin');
```

Pair this with **Supabase Auth**, which handles logins for your team (email/password, or SSO if you want it later) so every request carries a known user identity that your RLS policies can check against.

---

## Module 7: A Realistic Learning Path

Given where you're starting from, here's a sequence that avoids overwhelm:

1. **Week 1**: Create a project, model just `recipes` and `ingredients` in the Table Editor UI (no code yet). Get comfortable with how rows/relationships work.
2. **Week 2**: Install the CLI, run `supabase init`, move your table creation into migration files, connect to GitHub.
3. **Week 3**: Add `events` and `event_menu_items`, write your first "production guide" query (a join across three tables).
4. **Week 4**: Turn on RLS, set up Auth with at least two roles (staff vs. admin).
5. **Week 5+**: Start wiring up the actual frontend app that reads/writes through Supabase's client library, and consider Storage for recipe photos and signed BEOs.

---

## Quick Glossary

- **Project** — one Supabase instance = one Postgres database + its services
- **Table Editor** — spreadsheet-like UI for your tables
- **SQL Editor** — where you run raw queries/migrations
- **Migration** — a versioned SQL file describing a schema change
- **RLS (Row Level Security)** — Postgres rules controlling per-row access
- **Branch** — an isolated copy of your database for testing changes safely
- **Client library** — the code (JS, etc.) your app uses to talk to Supabase without writing raw API calls

---

*Note: Supabase's product surface (especially branching and dashboard features) has been evolving quickly, so it's worth checking supabase.com/docs directly once you're deep into implementation, in case anything has shifted since this was written (July 2026).*
