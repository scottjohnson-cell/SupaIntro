# Module 2 (Continued): Extending the Schema — Units, Conversion, and Conditional BEO Logic

*A follow-on to `Supabase-Intro-Course-for-Catering-App.md`. Read Module 3 of that document first — this picks up directly from the base schema (`recipes`, `ingredients`, `recipe_ingredients`, `events`, `event_menu_items`).*

---

## 1. The Full Table Set

The original four tables (`recipes`, `ingredients`, `recipe_ingredients`, `events`) are the right backbone. To support unit conversion and service-window logic, add four more:

| Table | Purpose |
|---|---|
| `recipes` | The recipe itself |
| `ingredients` | Master ingredient list |
| `recipe_ingredients` | Join: which ingredients, how much, per recipe |
| `units` | Reference table of every unit you use (g, oz, cup, tbsp, each...) |
| `ingredient_densities` | The missing link that makes weight ↔ volume conversion possible |
| `events` | The BEO header info (client, date, venue, headcount) |
| `event_menu_items` | Join: which recipes are served at which event, at what scale |
| `event_timeline` | Service windows within an event (cocktail hour, dinner service, etc.) |

---

## 2. Weight ↔ Volume Conversion

**The core problem:** you can't convert weight to volume with one universal factor. A cup of flour and a cup of honey weigh very different amounts. The conversion always has to go: *volume unit → density of THIS specific ingredient → weight unit* (or the reverse). There's no shortcut around storing density per ingredient — it's a real physical property, not a fixed ratio.

### Schema

```sql
-- Reference table: every unit, its type, and its conversion factor to a base unit
create table units (
  id uuid primary key default gen_random_uuid(),
  name text not null,           -- 'gram', 'cup', 'tablespoon'
  abbreviation text not null,   -- 'g', 'cup', 'tbsp'
  unit_type text not null check (unit_type in ('weight', 'volume', 'count')),
  to_base_factor numeric not null  -- multiplier to convert into the base unit for its type
);

-- Base units: grams for weight, milliliters for volume
insert into units (name, abbreviation, unit_type, to_base_factor) values
  ('gram', 'g', 'weight', 1),
  ('kilogram', 'kg', 'weight', 1000),
  ('ounce', 'oz', 'weight', 28.3495),
  ('pound', 'lb', 'weight', 453.592),
  ('milliliter', 'ml', 'volume', 1),
  ('liter', 'l', 'volume', 1000),
  ('teaspoon', 'tsp', 'volume', 4.92892),
  ('tablespoon', 'tbsp', 'volume', 14.7868),
  ('cup', 'cup', 'volume', 236.588),
  ('each', 'ea', 'count', 1);

-- The key table: density per ingredient, which bridges weight and volume
create table ingredient_densities (
  ingredient_id uuid primary key references ingredients(id),
  grams_per_ml numeric not null  -- e.g. flour ≈ 0.53, honey ≈ 1.42, water = 1.0
);
```

### Using it in a query

Converting any quantity of a specific ingredient into a common unit (grams, here) becomes a formula rather than a lookup table of every possible pairing:

```sql
select
  ri.id,
  i.name,
  ri.quantity,
  u.abbreviation as entered_unit,
  case
    when u.unit_type = 'weight' then ri.quantity * u.to_base_factor
    when u.unit_type = 'volume' then ri.quantity * u.to_base_factor * d.grams_per_ml
  end as grams
from recipe_ingredients ri
join ingredients i on i.id = ri.ingredient_id
join units u on u.abbreviation = ri.unit
left join ingredient_densities d on d.ingredient_id = i.id;
```

### Practical notes

- **Populate density once per ingredient.** Reliable gram-per-cup or gram-per-ml figures for common ingredients are easy to find, or you can weigh a known volume yourself for house items. This is a one-time data-entry cost per ingredient, not per recipe.
- **Count-based ingredients skip conversion.** If something is always measured by count ("2 eggs," "1 lemon"), don't add a density row — the `count` unit type bypasses the weight/volume math entirely.
- **This is what makes the purchase guide accurate.** It's what lets you correctly sum "3 cups flour" from one recipe with "2 lb flour" from another into a single, correct total on a purchase order.

---

## 3. Conditional Output: Service Windows, Item Counts, and Business Rules

This part is less about adding new columns everywhere and more about **one new table plus smart queries.** Most of the "conditionality" you're describing lives in how you query and render data — not in special schema fields.

### Service time windows

Add a table for the timeline within an event, and link menu items to it:

```sql
create table event_timeline (
  id uuid primary key default gen_random_uuid(),
  event_id uuid references events(id) on delete cascade,
  label text not null,       -- 'Cocktail Hour', 'Dinner Service', 'Dessert'
  start_time timestamptz,
  end_time timestamptz,
  notes text
);

-- Link each menu item to a specific window, not just the event as a whole
alter table event_menu_items
  add column timeline_id uuid references event_timeline(id),
  add column course_type text check (course_type in ('passed_app', 'stationed', 'plated', 'family_style', 'dessert'));
```

Now a BEO is just a query grouped by `event_timeline` — each window shows only the items assigned to it, and no one has to manually re-sort the document when a client moves cocktail hour from 6:00–6:45 to 6:00–7:00.

### Counting and conditional logic

Separate two things that are easy to conflate:

**Counting/aggregating** doesn't need any new schema — it's just `count()` in a query:

```sql
select course_type, count(*) as item_count
from event_menu_items
where event_id = $1
group by course_type;
```

**Conditional output** (e.g., "if there are more than 4 passed apps, flag that we need an extra server") is business logic, not data. There are two reasonable places to put it:

1. **In the query, using `case`** — good for straightforward, single-value rules:

```sql
select
  count(*) filter (where course_type = 'passed_app') as passed_app_count,
  case
    when count(*) filter (where course_type = 'passed_app') > 4 then 'Add extra server'
    else 'Standard staffing'
  end as staffing_flag
from event_menu_items
where event_id = $1;
```

2. **In your application code**, once the data comes back — better as rules get more complex (multiple conditions, formatting decisions, PDF layout choices). Push most "how should this look on the printed BEO" logic here rather than into SQL; it's easier to read, test, and change without touching a migration.

### The rule of thumb

> **The database should store facts.** This item is served at this time, in this quantity, in this category.
> **The application should decide what those facts mean for output.** Does this trigger a staffing note? Does this section get a header? Does this line get bolded?

Trying to encode every business rule directly into the schema gets brittle fast. Keep the schema describing *what happened*, and let queries and application code answer *what that implies*.

---

## Quick Reference: New Tables Added in This Module

```sql
units                  -- unit definitions + conversion factors
ingredient_densities   -- grams-per-ml, keyed by ingredient
event_timeline         -- service windows within an event
```

Plus two new columns on `event_menu_items`: `timeline_id`, `course_type`.
