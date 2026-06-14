# Dining Diary Mobile App Engineering Design

Brief design notes for engineers and coding agents building the React Native mobile app.

## Product Direction

Dining Diary is an iOS-first nutrition record app built with React Native and Expo. Android remains a future target, so shared TypeScript domain logic should stay portable. The app is local-first, has no first-party backend, and uses BYOK for remote LLM calls.

The first version only tracks:

- calories
- protein
- carbs
- fat

Avoid adding micronutrients, sodium, sugar, fiber, or clinical nutrition concepts unless the product scope changes.

## Platform And UX Principles

- Use each OS native design language instead of a custom cross-platform visual system.
- For iOS-first, prefer native-feeling navigation, sheets, large titles, swipe actions, haptics, and system-style controls.
- Keep React Native route files thin. Product behavior should live in feature/domain/data/service modules.
- AI and OCR outputs are drafts. Users must review/edit before data is committed.
- Manual logging must remain first-class; AI is assistive, not required.

## Recommended Source Structure

Use `src/app` for Expo Router entry points only. Put product logic behind explicit boundaries.

```txt
src/
  app/
    _layout.tsx
    (tabs)/
      index.tsx
      history.tsx
      settings.tsx
    log/
      new.tsx
      [entryId].tsx
    modal/
      api-key.tsx
      food-search.tsx

  features/
    diary/
    food/
    ai/
    settings/

  domain/
    nutrition/
      types.ts
      calculations.ts
      serving.ts
    diary/
      types.ts
      meal-period.ts
    ai/
      meal-draft.ts
      schemas.ts

  data/
    local/
      db.ts
      migrations/
      diary-repository.ts
      food-repository.ts
      recent-food-repository.ts
    secure/
      api-key-store.ts

  services/
    ai/
      llm-provider.ts
      openai-provider.ts
      on-device-provider.ios.ts
      mock-provider.ts
    sync/
      sync-provider.ts
      cloudkit-provider.ios.ts

  ui/
    primitives/
    native/
    theme/

  lib/
    date.ts
    ids.ts
    result.ts
```

## Data Model Concepts

Keep these concepts separate:

- `MealDraft`: uncertain extracted data from text, image, OCR, barcode, local AI, or remote LLM.
- `DiaryEntry`: confirmed user record, editable while inside the recent window.
- `RecentFood`: reusable quick-add template, updated when a user confirms/logs food.
- `ArchivedMealEntry`: compact, read-only older record.

Suggested core types:

```ts
// ISO 8601 timestamp with timezone, for example "2026-06-14T18:30:00-07:00".
type IsoDateTime = string;

// Local diary date in the user's current calendar, formatted as "YYYY-MM-DD".
type LocalDate = string;

type NutritionMacros = {
  caloriesKcal: number;
  proteinGrams: number;
  carbsGrams: number;
  fatGrams: number;
};

type FoodItem = {
  id: string;
  name: string;
  servingText: string;
  macros: NutritionMacros;
};

type RecentMealEntry = {
  id: string;
  eatenAt: IsoDateTime;
  diaryDate: LocalDate;
  mealType: 'breakfast' | 'lunch' | 'dinner' | 'snack';
  items: FoodItem[];
  totals: NutritionMacros;
  editableThroughDate: LocalDate;
};

type ArchivedMealEntry = {
  id: string;
  date: LocalDate;
  mealType: 'breakfast' | 'lunch' | 'dinner' | 'snack';
  ingredientSummary: string[];
  totals: NutritionMacros;
  archivedAt: IsoDateTime;
};

type RecentFood = {
  id: string;
  normalizedName: string;
  displayName: string;
  servingText: string;
  macros: NutritionMacros;
  lastUsedAt: IsoDateTime;
  useCount: number;
  source: 'manual' | 'ai' | 'barcode' | 'copied';
  confidence?: 'userConfirmed' | 'estimated';
};
```

Use grams internally for macros and kcal for calories. Display can round values. Use timestamps for event ordering, but use local diary dates for retention and editability decisions.

## Retention Rule

Detailed food data is editable only for the most recent 14 local calendar days.

Recent 14 days:

- full meal entries
- food names
- serving text and quantities
- calories, protein, carbs, fat
- source metadata where useful
- editable

Older than 14 days:

- read-only
- brief ingredient summary
- daily or meal totals for calories, protein, carbs, fat
- no raw AI prompts/outputs
- no detailed serving/source metadata
- no photos unless the product explicitly changes this rule

Run a compaction task on app launch or once per day:

```txt
1. Find entries older than 14 local calendar days.
2. Build archived summaries and totals.
3. Save archive records.
4. Delete detailed rows/assets/source metadata.
```

Use local calendar dates, not a rolling 336-hour cutoff.

## Recent Food Quick Query

Recent foods are a quick-add index, not diary history. They may outlive the 14-day detailed diary window.

On successful save:

```txt
1. Save confirmed diary entry.
2. Upsert recent food templates.
3. Update or recalculate daily totals.
```

Search behavior for v1 can be simple:

```txt
1. exact/prefix name match
2. lastUsedAt descending
3. useCount descending
```

Keep a cap such as 100-300 recent foods and provide a way to clear them.

## Save Record Flow

Reference: `design/save-record-flowcharts.png`.

All paths should converge into one review/edit draft screen before saving.

```txt
Image or text input
  -> classify input type
  -> extract a MealDraft using the best local/source method
  -> optionally improve the draft with remote BYOK LLM
  -> user reviews/edits calories, protein, carbs, fat, quantity
  -> save DiaryEntry
  -> update RecentFood
  -> update daily totals
```

Image classification should distinguish:

- nutrition facts label
- barcode
- meal/food photo
- other/unknown

Extraction paths:

- Nutrition facts label: local OCR, then deterministic macro parser, then optional LLM fallback.
- Barcode: OpenFoodFacts lookup, then review screen. Cache successful lookups locally.
- Text request: parse locally if possible, then optional remote BYOK LLM.
- Meal/food photo: local classifier first, remote BYOK LLM if enabled and useful.

Required failure states:

- barcode not found
- OCR confidence too low
- no API key configured
- remote LLM unavailable or timed out
- user cancels
- user chooses manual entry

## AI And BYOK

Remote LLM calls are allowed because users bring their own API key. Store user keys only in secure platform storage:

- iOS: Keychain
- Android later: Keystore-backed secure storage

Do not store keys in AsyncStorage, SQLite, logs, crash reports, or plain files.

Use a provider interface:

```ts
interface LlmProvider {
  validateKey(): Promise<boolean>;
  extractMeal(input: MealInput): Promise<MealDraft>;
  estimatePortions(input: PortionInput): Promise<MealDraft>;
}
```

Recommended settings:

- always ask before remote AI
- use remote AI automatically for supported inputs
- never use remote AI

Default should be privacy- and cost-conscious. Never send full diary history unless the user explicitly asks for a feature that requires it.

## Persistence And Sync

For portability, prefer a shared local database abstraction. SQLite is the most portable default. If the product becomes deeply iOS-only, Core Data + CloudKit can be considered behind repositories.

Do not let UI or feature code depend directly on storage tables, CloudKit records, or native persistence details. Use repository interfaces.

Daily totals should be treated as derived/cache data. Entries and items are the source of truth during the editable window; archived summaries become the source of truth after compaction.

## Initial Screens

Recommended v1 tabs:

- Today
- History
- Settings

Add/log food should be an action or sheet from Today rather than a permanent tab unless user testing shows otherwise.

## Engineering Guardrails

- Keep domain logic pure TypeScript.
- Keep AI outputs as drafts until user confirmation.
- Keep route files thin.
- Keep native modules behind service/data abstractions.
- Avoid building a large custom design system before product flows stabilize.
- Do not expand nutrition scope beyond calories/protein/carbs/fat without an explicit product decision.
