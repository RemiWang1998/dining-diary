# Dining Diary V1 Coding-Agent Plan

## Objective

Build the first local-first foundation for the React Native app in `dining-diary-react-native`.

Use `design/mobile-app-engineering-design.md` as the source design document. This plan is the implementation handoff for coding agents and should be followed before adding OCR, barcode, remote AI, or sync features.

The V1 implementation must support:

- Today, History, and Settings tabs
- manual add/log flow
- macros-only nutrition model: calories, protein, carbs, fat
- SQLite-backed local repositories
- SecureStore-backed BYOK key storage
- recent food quick query
- 14-day editable window and archive compaction
- automated tests and CI

## Current Repo State

- The app is an Expo SDK 56 / React Native project in `dining-diary-react-native`.
- The current UI is still close to the generated Expo starter.
- Routing uses Expo Router with native tabs isolated in `src/components/app-tabs.tsx`.
- Existing product design lives in `design/mobile-app-engineering-design.md`.
- Existing save-flow diagram lives in `design/save-record-flowcharts.png`.

Use the Expo SDK 56 docs for:

- `expo-sqlite`
- `expo-secure-store`
- Expo Router native tabs / `unstable-native-tabs`

## Implementation Steps

1. Install persistence and secret-storage dependencies with Expo-compatible versions:

   ```sh
   npx expo install expo-sqlite expo-secure-store
   ```

2. Replace starter screens with product routes:

   - Today tab
   - History tab
   - Settings tab
   - Add/log route or modal opened from Today

3. Keep route files thin:

   - route files should compose screens
   - feature logic belongs under `src/features`
   - domain logic belongs under `src/domain`
   - persistence belongs under `src/data`
   - AI/provider abstractions belong under `src/services`

4. Add pure TypeScript domain modules:

   - nutrition macro types and calculations
   - diary entry types
   - meal draft types
   - local date and timestamp helpers
   - 14-day edit-window helper
   - recent-food normalization helper

5. Add SQLite data layer:

   - database initialization
   - migrations with `PRAGMA user_version`
   - WAL mode
   - repositories for diary entries, recent foods, and daily totals
   - archive compaction on app launch or database initialization

6. Add SecureStore data layer:

   - save BYOK provider key
   - read BYOK provider key
   - delete BYOK provider key
   - expose whether a key is configured without showing the full key

7. Implement manual logging:

   - user enters food name, serving text, calories, protein, carbs, and fat
   - input becomes a `MealDraft`
   - user reviews/edits
   - confirmed draft saves as a diary entry
   - save updates recent foods and daily totals

8. Implement recent food quick query:

   - empty add screen shows recent foods
   - typing filters by normalized name
   - tapping a recent food prefills the draft
   - repeated saves increment use count and update last-used timestamp
   - recent foods are reusable templates only, not diary history

9. Implement history and retention:

   - recent entries remain editable through their `editableThroughDate`
   - entries older than the 14-day local-calendar window compact into archive records
   - older-than-14-day records are read-only and contain ingredient summaries plus macro totals
   - archived records must not be editable through UI or repository methods

10. Add test infrastructure and CI:

   - add a Jest-based test setup compatible with Expo SDK 56
   - add an `npm test` script for unit/repository tests
   - add an `npm run typecheck` script using `tsc --noEmit`
   - keep `npm run lint` as the lint gate
   - add GitHub Actions CI that runs install, lint, typecheck, and tests
   - make retention, archive, and recent-food separation rules covered by automated tests

## Required Types and Interfaces

Use these concepts from the design doc:

```ts
type IsoDateTime = string;
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

Add repository interfaces equivalent to:

```ts
interface DiaryRepository {
  saveDraft(draft: MealDraft, mealType: MealType, eatenAt: IsoDateTime): Promise<RecentMealEntry>;
  getDay(date: LocalDate): Promise<DayDiary>;
  getHistory(range: DateRange): Promise<Array<RecentMealEntry | ArchivedMealEntry>>;
  updateEntry(entry: RecentMealEntry): Promise<void>;
  deleteEntry(id: string): Promise<void>;
  compactOlderThan(cutoffDate: LocalDate): Promise<void>;
}

interface RecentFoodRepository {
  upsertFromFoodItem(item: FoodItem, source: RecentFood['source']): Promise<void>;
  search(query: string, limit: number): Promise<RecentFood[]>;
  listRecent(limit: number): Promise<RecentFood[]>;
  clear(): Promise<void>;
}

interface ApiKeyStore {
  getProviderKey(provider: string): Promise<string | null>;
  setProviderKey(provider: string, key: string): Promise<void>;
  deleteProviderKey(provider: string): Promise<void>;
}
```

## Data Persistence Plan

Use SQLite as the app's canonical local data store. Do not use Core Data for V1.

Minimum SQLite tables:

- recent meal entries
- food items attached to recent entries
- archived meal entries
- recent foods
- daily totals cache

Persistence rules:

- Store timestamps as ISO 8601 strings with timezone.
- Store diary dates as local `YYYY-MM-DD` strings.
- Use local diary dates for editability and retention.
- Use parameterized SQL queries only.
- Treat recent entries and food items as source of truth during the editable window.
- Treat archived summaries as source of truth after compaction.
- Treat daily totals as derived/cache data that can be recalculated.
- Keep recent foods separate from diary history; recent foods are quick-add templates and must not be used as historical records.

Compaction rules:

- Run on app launch or database initialization.
- Find entries older than the 14-day local-calendar editable window.
- Create archived summaries with ingredient names and macro totals.
- Remove detailed food rows and source metadata for compacted entries.
- Keep older-than-14-day archived records read-only.

SecureStore rules:

- Store BYOK credentials only in SecureStore.
- Never store API keys in SQLite, AsyncStorage, logs, route params, crash output, or plain files.
- Settings may show whether a key exists, but must not display the full key after save.

## UI Flow

Today tab:

- show today's local date
- show calories, protein, carbs, and fat totals
- show meal sections for breakfast, lunch, dinner, snack
- show recent editable entries
- expose Add/log action

Add/log flow:

- allow manual entry of food name, serving text, calories, protein, carbs, and fat
- show recent food suggestions before and during search
- tapping a recent food prefills the draft
- require review/edit before save
- save confirmed draft to diary, recent foods, and daily totals

History tab:

- show recent editable records
- show archived records as read-only summaries
- do not allow editing archived entries
- do not show recent-food templates as historical diary entries

Settings tab:

- allow saving and removing BYOK credentials
- allow clearing recent foods
- do not implement remote LLM calls yet

## Test and Verification Plan

Add test and CI tooling as part of V1, then run locally:

```sh
npm run typecheck
npm run lint
npm test
```

CI must run the same gates on pull requests and pushes to the main development branch:

```sh
npm ci
npm run lint
npm run typecheck
npm test
```

Verify domain behavior:

- macro totals sum correctly
- local diary date helper returns `YYYY-MM-DD`
- edit window uses local calendar days, not rolling 336-hour timestamps
- recent-food normalization merges repeated foods

Verify repository behavior:

- saving a manual draft creates a diary entry and food item
- saving updates daily totals
- saving upserts recent food
- repeated recent food increments use count and updates last-used timestamp
- recent food search sorts by match quality, recency, then use count
- compaction archives entries older than 14 local calendar days
- compaction removes detailed food rows for archived entries
- archived entries cannot be edited or deleted through editable-entry methods
- recent foods are not returned from diary history queries

Verify UI behavior:

- Today loads empty state
- manual add creates a visible meal entry
- daily totals update after save
- recent food tap prefills the add form
- History displays archived entries as read-only
- Settings saves/removes BYOK key without exposing the full value after save

## Explicit Non-Goals

Do not implement these in the first pass:

- OCR
- barcode scanning
- OpenFoodFacts integration
- remote LLM calls
- on-device AI
- image classification
- camera/photo capture
- CloudKit sync
- HealthKit
- widgets
- Android-specific polish
- micronutrients, sodium, sugar, fiber, or other nutrition fields beyond calories/protein/carbs/fat
