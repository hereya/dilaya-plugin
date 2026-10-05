---
name: dilaya-write-skill
description: "Use when writing or updating the skill (stored instructions) of a Dilaya app."
---

Tool names below are the Dilaya connector's tools; your client may show them with a prefix (e.g. `mcp__dilaya__query`).

# How to Write a Good Skill

A skill is a set of instructions stored in Dilaya that tells agents how to use an app. Good skills make the app immediately usable by any agent in any conversation.

## Structure

### 1. Overview (required)
One paragraph explaining what the app does, who it's for, and what problems it solves.

### 2. Tables (required)
List every table with:
- Table name and purpose
- Each column: name, type, meaning, constraints
- Relationships: foreign keys, join patterns

### 3. Common operations (required)
Provide ready-to-use SQL patterns for the most frequent operations:
- **Read patterns**: SELECT queries for listing, filtering, searching, aggregating
- **Write patterns**: INSERT templates with all required columns
- **Update patterns**: UPDATE templates for common state changes
- **Delete patterns**: DELETE with proper cascade awareness

Use `:param_name` syntax for parameterized values.

A calculation the user relies on (a price, a quantity, an invoice) belongs in an **action** of the app's backend, not in a SQL recipe — **even when the app has no web page**: a backend can serve actions only. Write « pour X, appelle l'action `nom` » here and keep SQL for exploration (the `actions` section of the `dilaya-frontend` skill).

### 4. Business rules (recommended)
Document constraints that aren't enforced by the database:
- Validation rules (e.g., "servings must be between 1 and 100")
- State machines (e.g., "an order goes from pending → confirmed → shipped → delivered")
- Computed values (e.g., "total_cost = SUM of ingredient costs × quantity")
- Access patterns (e.g., "recipes are per-user, filter by user_id")

### 5. File storage conventions (if applicable)
Document the folder structure and naming conventions:
- Where files are stored (e.g., `recipes/photos/{recipe_id}/`)
- Naming conventions (e.g., `{recipe_name}-{timestamp}.jpg`)
- File types accepted

### 6. Stored scripts & templates (if applicable)
Document any reusable scripts or templates stored in S3 that speed up recurrent tasks:
- What each file does and when to use it
- Path in S3 (e.g., `recipes/scripts/weekly-report.sql`)
- Any placeholders or parameters that need substitution

## Example

```
# Recipe Manager

Manages cooking recipes with ingredients and nutritional information. Designed for home cooks who want to organize, search, and scale their recipes.

## Tables

### recipes
| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PK | Auto-generated ID (INTEGER PRIMARY KEY AUTOINCREMENT) |
| name | TEXT | Recipe name (required, unique per user) |
| servings | INTEGER | Number of portions this recipe makes |
| prep_time | INTEGER | Preparation time in minutes |
| cook_time | INTEGER | Cooking time in minutes |
| category | TEXT | Category: appetizer, main, dessert, snack, drink |
| created_at | TEXT | Auto-set on creation (DEFAULT CURRENT_TIMESTAMP) |

### ingredients
| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PK | Auto-generated ID (INTEGER PRIMARY KEY AUTOINCREMENT) |
| recipe_id | INTEGER FK → recipes(id) | Parent recipe (CASCADE delete) |
| name | TEXT | Ingredient name |
| quantity | REAL | Amount needed |
| unit | TEXT | Unit: g, kg, ml, l, piece, tbsp, tsp, cup |

## Common Operations

### List recipes by category
query: SELECT id, name, servings, prep_time + cook_time AS total_time FROM recipes WHERE category = :cat ORDER BY name
params: { cat: "desserts" }

### Get recipe with ingredients
query: SELECT r.*, i.name AS ingredient, i.quantity, i.unit FROM recipes r LEFT JOIN ingredients i ON i.recipe_id = r.id WHERE r.id = :id ORDER BY i.name
params: { id: 1 }

### Add a recipe
execute: INSERT INTO recipes (name, servings, prep_time, cook_time, category) VALUES (:name, :servings, :prep, :cook, :cat)
params: { name: "Tarte Tatin", servings: 8, prep: 30, cook: 45, cat: "dessert" }

### Scale a recipe
query: SELECT name, quantity * :factor AS scaled_qty, unit FROM ingredients WHERE recipe_id = :id
params: { id: 1, factor: 2.0 }

## Business Rules
- Category must be one of: appetizer, main, dessert, snack, drink
- Servings must be >= 1
- Times are in minutes, must be >= 0

## File Storage
Photos stored at: recipes/photos/{recipe_id}/{filename}
Accepted types: image/jpeg, image/png, image/webp
```

## Saving the skill
```
save-skill({
  schema: "recipes",
  name: "main",
  description: "Manage cooking recipes with ingredients and portions",
  content: "... the full skill text above ..."
})
```

You can save multiple skills per schema for different use cases (e.g., "cost-analysis", "meal-planning").
