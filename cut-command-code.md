# Cut Command Code Bundle

This is the actual source code for the Cut Command tracker app: manual food logging, restaurant photo estimates, goal tracking, and intermittent fasting.

## Quick Setup

```bash
pnpm install
pnpm run dev
```

The app runs as a Next/Vinext project. The main app is in `app/page.tsx`; the visual styling is in `app/globals.css`.
## package.json

```json
{
  "name": "cut-command",
  "version": "0.1.0",
  "private": true,
  "engines": {
    "node": ">=22.13.0"
  },
  "scripts": {
    "dev": "WRANGLER_LOG_PATH=.wrangler/wrangler.log vinext dev",
    "build": "WRANGLER_LOG_PATH=.wrangler/wrangler.log vinext build",
    "start": "WRANGLER_LOG_PATH=.wrangler/wrangler.log vinext start",
    "test": "vinext build && node --test tests/rendered-html.test.mjs",
    "lint": "eslint . --ignore-pattern dist --ignore-pattern .next",
    "db:generate": "drizzle-kit generate"
  },
  "dependencies": {
    "drizzle-orm": "0.45.2",
    "next": "16.2.6",
    "react": "19.2.6",
    "react-dom": "19.2.6"
  },
  "devDependencies": {
    "@cloudflare/vite-plugin": "1.37.1",
    "@tailwindcss/postcss": "4.2.1",
    "@types/node": "22.19.19",
    "@types/react": "19.2.14",
    "@types/react-dom": "19.2.3",
    "@vitejs/plugin-react": "6.0.2",
    "@vitejs/plugin-rsc": "0.5.26",
    "drizzle-kit": "0.31.10",
    "eslint": "9.39.4",
    "eslint-config-next": "16.2.6",
    "react-server-dom-webpack": "19.2.6",
    "tailwindcss": "4.2.1",
    "typescript": "5.9.3",
    "vinext": "0.0.50",
    "vite": "8.0.13",
    "wrangler": "4.92.0"
  },
  "type": "module"
}

```

## app/page.tsx

```tsx
"use client";

/* eslint-disable @next/next/no-img-element */
import { useEffect, useMemo, useState } from "react";

type ProteinBasis = "bodyWeight" | "leanMass" | "bodyFat";
type FoodSource = "manual" | "quick" | "photo";
type MealPreset =
  | "burritoBowl"
  | "burgerFries"
  | "pasta"
  | "sushi"
  | "salad"
  | "steakPotato"
  | "pizza"
  | "ramen"
  | "custom";
type PortionSize = "small" | "standard" | "large" | "huge";
type CookingFat = "none" | "light" | "unknown" | "heavy";
type SauceLevel = "none" | "light" | "standard" | "heavy";

type Goals = {
  calorieTarget: number;
  startWeight: number;
  currentWeight: number;
  targetWeight: number;
  goalDate: string;
  proteinPerPound: number;
  proteinBasis: ProteinBasis;
  bodyFatPercent: number;
};

type FoodItem = {
  id: string;
  name: string;
  calories: number;
  protein: number;
  carbs: number;
  fat: number;
  source?: FoodSource;
  confidence?: number;
  estimateRange?: {
    low: number;
    high: number;
  };
  restaurant?: string;
  photoDataUrl?: string;
  accuracyNotes?: string[];
};

type FoodDraft = Omit<FoodItem, "id">;

type PhotoDraft = {
  photoDataUrl: string;
  restaurant: string;
  mealPreset: MealPreset;
  portionSize: PortionSize;
  cookingFat: CookingFat;
  sauceLevel: SauceLevel;
  referenceInFrame: boolean;
  notes: string;
};

type FastingSettings = {
  enabled: boolean;
  fastHours: number;
  windowStart: string;
  windowEnd: string;
  lastMealAt: string;
};

const STORAGE_KEY = "cut-command-v1";
const MS_PER_DAY = 86_400_000;

const clamp = (value: number, min: number, max: number) =>
  Math.min(Math.max(value, min), max);

const round = (value: number, digits = 0) => {
  const multiplier = 10 ** digits;
  return Math.round(value * multiplier) / multiplier;
};

const todayIso = () => new Date().toISOString().slice(0, 10);

const isoDaysFromNow = (days: number) => {
  const date = new Date();
  date.setDate(date.getDate() + days);
  return date.toISOString().slice(0, 10);
};

const defaultGoals: Goals = {
  calorieTarget: 2500,
  startWeight: 215,
  currentWeight: 215,
  targetWeight: 205,
  goalDate: isoDaysFromNow(30),
  proteinPerPound: 0.8,
  proteinBasis: "bodyWeight",
  bodyFatPercent: 22,
};

const emptyDraft: FoodDraft = {
  name: "",
  calories: 0,
  protein: 0,
  carbs: 0,
  fat: 0,
  source: "manual",
};

const defaultPhotoDraft: PhotoDraft = {
  photoDataUrl: "",
  restaurant: "",
  mealPreset: "burritoBowl",
  portionSize: "standard",
  cookingFat: "unknown",
  sauceLevel: "standard",
  referenceInFrame: false,
  notes: "",
};

const defaultFasting: FastingSettings = {
  enabled: true,
  fastHours: 16,
  windowStart: "12:00",
  windowEnd: "20:00",
  lastMealAt: "",
};

const quickAdds: FoodDraft[] = [
  { name: "Chicken rice bowl", calories: 620, protein: 52, carbs: 68, fat: 14 },
  { name: "Greek yogurt + berries", calories: 260, protein: 28, carbs: 30, fat: 3 },
  { name: "Protein shake", calories: 190, protein: 32, carbs: 6, fat: 3 },
  { name: "Dining hall plate", calories: 760, protein: 48, carbs: 78, fat: 28 },
];

const basisLabels: Record<ProteinBasis, string> = {
  bodyWeight: "body weight",
  leanMass: "lean mass",
  bodyFat: "body fat",
};

const mealPresets: Record<
  MealPreset,
  { label: string; calories: number; protein: number; carbs: number; fat: number }
> = {
  burritoBowl: { label: "Restaurant bowl", calories: 760, protein: 46, carbs: 82, fat: 27 },
  burgerFries: { label: "Burger + fries", calories: 980, protein: 42, carbs: 96, fat: 46 },
  pasta: { label: "Pasta entree", calories: 890, protein: 34, carbs: 112, fat: 32 },
  sushi: { label: "Sushi order", calories: 720, protein: 38, carbs: 92, fat: 18 },
  salad: { label: "Loaded salad", calories: 640, protein: 40, carbs: 34, fat: 38 },
  steakPotato: { label: "Steak + potato", calories: 820, protein: 62, carbs: 58, fat: 36 },
  pizza: { label: "Pizza meal", calories: 860, protein: 38, carbs: 92, fat: 34 },
  ramen: { label: "Ramen bowl", calories: 930, protein: 42, carbs: 108, fat: 36 },
  custom: { label: "Custom restaurant plate", calories: 700, protein: 35, carbs: 65, fat: 30 },
};

const portionMultipliers: Record<PortionSize, { label: string; multiplier: number }> = {
  small: { label: "Light / left some", multiplier: 0.78 },
  standard: { label: "Normal restaurant serving", multiplier: 1 },
  large: { label: "Large / packed plate", multiplier: 1.22 },
  huge: { label: "Huge / split-worthy", multiplier: 1.45 },
};

const cookingFatAdds: Record<CookingFat, { label: string; calories: number; fat: number }> = {
  none: { label: "Grilled / no added oil", calories: 0, fat: 0 },
  light: { label: "Light oil or butter", calories: 60, fat: 7 },
  unknown: { label: "Unknown restaurant oil", calories: 120, fat: 13 },
  heavy: { label: "Fried / heavy oil", calories: 240, fat: 27 },
};

const sauceAdds: Record<SauceLevel, { label: string; calories: number; carbs: number; fat: number }> = {
  none: { label: "No sauce/dressing", calories: 0, carbs: 0, fat: 0 },
  light: { label: "Light sauce", calories: 50, carbs: 5, fat: 3 },
  standard: { label: "Normal sauce/dressing", calories: 120, carbs: 10, fat: 8 },
  heavy: { label: "Heavy sauce/dressing", calories: 220, carbs: 18, fat: 15 },
};

function proteinTarget(goals: Goals) {
  const basisWeight =
    goals.proteinBasis === "bodyWeight"
      ? goals.currentWeight
      : goals.proteinBasis === "leanMass"
        ? goals.currentWeight * (1 - goals.bodyFatPercent / 100)
        : goals.currentWeight * (goals.bodyFatPercent / 100);

  return Math.max(0, basisWeight * goals.proteinPerPound);
}

function macroCalories(food: Pick<FoodItem, "protein" | "carbs" | "fat">) {
  return food.protein * 4 + food.carbs * 4 + food.fat * 9;
}

function progressPercent(value: number, target: number) {
  if (target <= 0) return 0;
  return clamp((value / target) * 100, 0, 100);
}

function estimatePhotoMeal(draft: PhotoDraft) {
  const preset = mealPresets[draft.mealPreset];
  const portion = portionMultipliers[draft.portionSize];
  const fatAdd = cookingFatAdds[draft.cookingFat];
  const sauceAdd = sauceAdds[draft.sauceLevel];

  const calories = preset.calories * portion.multiplier + fatAdd.calories + sauceAdd.calories;
  const protein = preset.protein * portion.multiplier;
  const carbs = preset.carbs * portion.multiplier + sauceAdd.carbs;
  const fat = preset.fat * portion.multiplier + fatAdd.fat + sauceAdd.fat;

  let confidence = draft.photoDataUrl ? 42 : 15;
  const notes = [
    "Restaurant oil/sauce is counted so the estimate does not default low.",
  ];

  if (draft.mealPreset !== "custom") confidence += 15;
  if (draft.referenceInFrame) {
    confidence += 15;
    notes.push("Reference object or plate size visible improves portion accuracy.");
  } else {
    notes.push("Add a fork, hand, can, or full plate in frame for better portion scale.");
  }

  if (draft.cookingFat !== "unknown") {
    confidence += 14;
  } else {
    notes.push("Unknown cooking fat assumes +120 calories.");
  }

  if (draft.restaurant.trim()) confidence += 7;
  if (draft.notes.trim()) confidence += 5;

  const finalConfidence = round(clamp(confidence, 20, 93));
  const spread = finalConfidence >= 82 ? 0.12 : finalConfidence >= 64 ? 0.18 : 0.28;

  return {
    name: `${draft.restaurant.trim() ? `${draft.restaurant.trim()} - ` : ""}${preset.label}`,
    calories: round(calories),
    protein: round(protein),
    carbs: round(carbs),
    fat: round(fat),
    confidence: finalConfidence,
    estimateRange: {
      low: round(calories * (1 - spread)),
      high: round(calories * (1 + spread)),
    },
    accuracyNotes: notes,
  };
}

function formatDuration(milliseconds: number) {
  const safeMs = Math.max(0, milliseconds);
  const hours = Math.floor(safeMs / 3_600_000);
  const minutes = Math.floor((safeMs % 3_600_000) / 60_000);
  return `${hours}h ${minutes.toString().padStart(2, "0")}m`;
}

function toDateTimeLocal(milliseconds: number) {
  const date = new Date(milliseconds);
  const pad = (value: number) => value.toString().padStart(2, "0");
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())}T${pad(
    date.getHours(),
  )}:${pad(date.getMinutes())}`;
}

function minutesFromTime(value: string) {
  const [hours, minutes] = value.split(":").map(Number);
  return hours * 60 + minutes;
}

function isWithinWindow(currentMinutes: number, start: string, end: string) {
  const startMinutes = minutesFromTime(start);
  const endMinutes = minutesFromTime(end);

  if (startMinutes <= endMinutes) {
    return currentMinutes >= startMinutes && currentMinutes < endMinutes;
  }

  return currentMinutes >= startMinutes || currentMinutes < endMinutes;
}

function compressImage(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();

    reader.onload = () => {
      const image = new Image();

      image.onload = () => {
        const maxSide = 900;
        const scale = Math.min(1, maxSide / image.width, maxSide / image.height);
        const canvas = document.createElement("canvas");
        canvas.width = Math.max(1, Math.round(image.width * scale));
        canvas.height = Math.max(1, Math.round(image.height * scale));

        const context = canvas.getContext("2d");
        if (!context) {
          resolve(reader.result as string);
          return;
        }

        context.drawImage(image, 0, 0, canvas.width, canvas.height);
        resolve(canvas.toDataURL("image/jpeg", 0.78));
      };

      image.onerror = reject;
      image.src = reader.result as string;
    };

    reader.onerror = reject;
    reader.readAsDataURL(file);
  });
}

function MetricCard({
  label,
  value,
  detail,
  tone = "neutral",
}: {
  label: string;
  value: string;
  detail: string;
  tone?: "neutral" | "green" | "amber" | "red";
}) {
  return (
    <section className={`metric-card ${tone}`}>
      <span>{label}</span>
      <strong>{value}</strong>
      <small>{detail}</small>
    </section>
  );
}

function ProgressBar({
  value,
  target,
  label,
}: {
  value: number;
  target: number;
  label: string;
}) {
  const percent = progressPercent(value, target);

  return (
    <div className="progress-row">
      <div className="progress-label">
        <span>{label}</span>
        <strong>
          {round(value)}
          <small> / {round(target)}</small>
        </strong>
      </div>
      <div className="progress-track" aria-label={label}>
        <span style={{ width: `${percent}%` }} />
      </div>
    </div>
  );
}

function NumberInput({
  label,
  value,
  onChange,
  step = 1,
  min = 0,
}: {
  label: string;
  value: number;
  onChange: (value: number) => void;
  step?: number;
  min?: number;
}) {
  return (
    <label className="field">
      <span>{label}</span>
      <input
        min={min}
        step={step}
        type="number"
        value={Number.isFinite(value) ? value : 0}
        onChange={(event) => onChange(Number(event.target.value))}
      />
    </label>
  );
}

export default function Home() {
  const [goals, setGoals] = useState<Goals>(defaultGoals);
  const [foods, setFoods] = useState<FoodItem[]>([]);
  const [draft, setDraft] = useState<FoodDraft>(emptyDraft);
  const [photoDraft, setPhotoDraft] = useState<PhotoDraft>(defaultPhotoDraft);
  const [fasting, setFasting] = useState<FastingSettings>(defaultFasting);
  const [selectedDate, setSelectedDate] = useState(todayIso);
  const [todayTime] = useState(() => new Date(`${todayIso()}T00:00:00`).getTime());
  const [clockNow, setClockNow] = useState(() => Date.now());
  const [hasLoaded, setHasLoaded] = useState(false);

  useEffect(() => {
    const frame = window.requestAnimationFrame(() => {
      const saved = window.localStorage.getItem(STORAGE_KEY);
      if (saved) {
        try {
          const parsed = JSON.parse(saved) as {
            goals?: Partial<Goals>;
            foods?: FoodItem[];
            fasting?: Partial<FastingSettings>;
            selectedDate?: string;
          };

          setGoals({ ...defaultGoals, ...parsed.goals });
          setFoods(Array.isArray(parsed.foods) ? parsed.foods : []);
          setFasting({ ...defaultFasting, ...parsed.fasting });
          setSelectedDate(parsed.selectedDate || todayIso());
        } catch {
          setGoals(defaultGoals);
          setFasting(defaultFasting);
        }
      }

      setHasLoaded(true);
    });

    return () => window.cancelAnimationFrame(frame);
  }, []);

  useEffect(() => {
    const interval = window.setInterval(() => setClockNow(Date.now()), 30_000);
    return () => window.clearInterval(interval);
  }, []);

  useEffect(() => {
    if (!hasLoaded) return;
    try {
      window.localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify({ goals, foods, fasting, selectedDate }),
      );
    } catch {
      window.localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify({
          goals,
          foods: foods.map((food) => {
            const copy = { ...food };
            delete copy.photoDataUrl;
            return copy;
          }),
          fasting,
          selectedDate,
        }),
      );
    }
  }, [fasting, foods, goals, hasLoaded, selectedDate]);

  const totals = useMemo(
    () =>
      foods.reduce(
        (sum, food) => ({
          calories: sum.calories + food.calories,
          protein: sum.protein + food.protein,
          carbs: sum.carbs + food.carbs,
          fat: sum.fat + food.fat,
        }),
        { calories: 0, protein: 0, carbs: 0, fat: 0 },
      ),
    [foods],
  );

  const proteinGoal = proteinTarget(goals);
  const caloriesLeft = goals.calorieTarget - totals.calories;
  const proteinLeft = proteinGoal - totals.protein;
  const totalToLose = Math.max(0, goals.startWeight - goals.targetWeight);
  const lostSoFar = clamp(goals.startWeight - goals.currentWeight, 0, totalToLose || 1);
  const weightProgress = totalToLose > 0 ? (lostSoFar / totalToLose) * 100 : 100;
  const daysLeft = Math.max(
    0,
    Math.ceil((new Date(`${goals.goalDate}T00:00:00`).getTime() - todayTime) / MS_PER_DAY),
  );
  const poundsLeft = Math.max(0, goals.currentWeight - goals.targetWeight);
  const weeklyPace = daysLeft > 0 ? (poundsLeft / daysLeft) * 7 : 0;
  const dailyDeficit = weeklyPace > 0 ? (weeklyPace * 3500) / 7 : 0;
  const caloriesFromMacros = macroCalories(totals);
  const photoEstimate = estimatePhotoMeal(photoDraft);
  const calorieTone = caloriesLeft >= 0 ? "green" : "red";
  const paceTone = weeklyPace > 2 ? "red" : weeklyPace > 1.25 ? "amber" : "green";
  const cutStatus =
    weeklyPace > 2
      ? "Aggressive pace"
      : weeklyPace > 1.25
        ? "Strong cut"
        : "Controlled pace";
  const lastMealTime = fasting.lastMealAt ? new Date(fasting.lastMealAt).getTime() : null;
  const fastTotal = fasting.fastHours * 3_600_000;
  const fastEndTime = lastMealTime ? lastMealTime + fastTotal : null;
  const fastElapsed = lastMealTime ? Math.max(0, clockNow - lastMealTime) : 0;
  const fastRemaining = fastEndTime ? Math.max(0, fastEndTime - clockNow) : fastTotal;
  const fastProgress = lastMealTime ? progressPercent(fastElapsed, fastTotal) : 0;
  const fastComplete = Boolean(fastEndTime && clockNow >= fastEndTime);
  const clockDate = new Date(clockNow);
  const currentMinutes = clockDate.getHours() * 60 + clockDate.getMinutes();
  const eatingWindowOpen = isWithinWindow(currentMinutes, fasting.windowStart, fasting.windowEnd);

  const addFood = (food: FoodDraft, metadata: Partial<FoodItem> = {}) => {
    if (!food.name.trim() && food.calories <= 0 && food.protein <= 0) return;

    setFoods((current) => [
      {
        ...food,
        ...metadata,
        id: `${Date.now()}-${Math.random().toString(36).slice(2)}`,
        name: metadata.name ?? (food.name.trim() || "Manual food"),
      },
      ...current,
    ]);
    setDraft(emptyDraft);
  };

  const updateFood = (id: string, patch: Partial<FoodItem>) => {
    setFoods((current) =>
      current.map((food) => (food.id === id ? { ...food, ...patch } : food)),
    );
  };

  const removeFood = (id: string) => {
    setFoods((current) => current.filter((food) => food.id !== id));
  };

  const updateGoal = <K extends keyof Goals>(key: K, value: Goals[K]) => {
    setGoals((current) => ({ ...current, [key]: value }));
  };

  const updateFasting = <K extends keyof FastingSettings>(
    key: K,
    value: FastingSettings[K],
  ) => {
    setFasting((current) => ({ ...current, [key]: value }));
  };

  const handlePhotoChange = async (file: File | undefined) => {
    if (!file) return;
    const photoDataUrl = await compressImage(file);
    setPhotoDraft((current) => ({ ...current, photoDataUrl }));
  };

  const addPhotoEstimate = () => {
    if (!photoDraft.photoDataUrl) return;

    addFood(
      {
        name: photoEstimate.name,
        calories: photoEstimate.calories,
        protein: photoEstimate.protein,
        carbs: photoEstimate.carbs,
        fat: photoEstimate.fat,
        source: "photo",
      },
      {
        source: "photo",
        confidence: photoEstimate.confidence,
        estimateRange: photoEstimate.estimateRange,
        restaurant: photoDraft.restaurant,
        photoDataUrl: photoDraft.photoDataUrl,
        accuracyNotes: photoEstimate.accuracyNotes,
      },
    );

    setPhotoDraft(defaultPhotoDraft);
  };

  return (
    <main className="app-shell">
      <section className="hero-panel">
        <div className="hero-copy">
          <p className="eyebrow">Cut Command</p>
          <h1>Today has a number. Hit it clean.</h1>
          <div className="hero-stats" aria-label="Daily cut snapshot">
            <MetricCard
              detail={`${round(totals.calories)} eaten of ${round(goals.calorieTarget)}`}
              label="Calories left"
              tone={calorieTone}
              value={`${round(caloriesLeft)}`}
            />
            <MetricCard
              detail={`${round(totals.protein)}g logged of ${round(proteinGoal)}g`}
              label="Protein left"
              tone={proteinLeft <= 0 ? "green" : "amber"}
              value={`${round(proteinLeft)}g`}
            />
            <MetricCard
              detail={`${round(poundsLeft, 1)} lb left over ${daysLeft} days`}
              label="Goal pace"
              tone={paceTone}
              value={`${round(weeklyPace, 1)} lb/wk`}
            />
          </div>
        </div>
        <div className="plate-visual" aria-hidden="true">
          <div className="plate-ring">
            <span className="food protein" />
            <span className="food carb" />
            <span className="food greens" />
            <span className="food sauce" />
          </div>
        </div>
      </section>

      <section className="dashboard-grid">
        <div className="main-column">
          <section className="panel daily-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Daily log</p>
                <h2>{selectedDate}</h2>
              </div>
              <label className="date-field">
                <span>Date</span>
                <input
                  type="date"
                  value={selectedDate}
                  onChange={(event) => setSelectedDate(event.target.value)}
                />
              </label>
            </div>

            <div className="macro-stack">
              <ProgressBar label="Calories" target={goals.calorieTarget} value={totals.calories} />
              <ProgressBar label="Protein" target={proteinGoal} value={totals.protein} />
              <ProgressBar label="Carbs" target={Math.max(180, goals.calorieTarget * 0.34 / 4)} value={totals.carbs} />
              <ProgressBar label="Fat" target={Math.max(55, goals.calorieTarget * 0.26 / 9)} value={totals.fat} />
            </div>
          </section>

          <section className="panel photo-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Restaurant photo log</p>
                <h2>Snap meal, tighten estimate</h2>
              </div>
              <span className="range-chip">
                {photoDraft.photoDataUrl
                  ? `${photoEstimate.estimateRange.low}-${photoEstimate.estimateRange.high} cal`
                  : "Photo needed"}
              </span>
            </div>

            <div className="photo-grid">
              <div className="camera-card">
                <label className="photo-drop">
                  <input
                    accept="image/*"
                    capture="environment"
                    className="file-input"
                    type="file"
                    onChange={(event) => handlePhotoChange(event.target.files?.[0])}
                  />
                  {photoDraft.photoDataUrl ? (
                    <img alt="Meal preview" className="photo-preview" src={photoDraft.photoDataUrl} />
                  ) : (
                    <span className="photo-empty">
                      <strong>Take food photo</strong>
                      <small>Use the back camera. Keep the full plate and a fork, hand, or cup in frame.</small>
                    </span>
                  )}
                </label>
                {photoDraft.photoDataUrl ? (
                  <button
                    className="ghost-button"
                    type="button"
                    onClick={() => setPhotoDraft((current) => ({ ...current, photoDataUrl: "" }))}
                  >
                    Retake
                  </button>
                ) : null}
              </div>

              <div className="photo-controls">
                <div className="photo-fields">
                  <label className="field">
                    <span>Restaurant</span>
                    <input
                      placeholder="Chipotle, local diner, sushi spot..."
                      type="text"
                      value={photoDraft.restaurant}
                      onChange={(event) =>
                        setPhotoDraft((current) => ({ ...current, restaurant: event.target.value }))
                      }
                    />
                  </label>
                  <label className="field">
                    <span>Meal type</span>
                    <select
                      value={photoDraft.mealPreset}
                      onChange={(event) =>
                        setPhotoDraft((current) => ({
                          ...current,
                          mealPreset: event.target.value as MealPreset,
                        }))
                      }
                    >
                      {Object.entries(mealPresets).map(([key, preset]) => (
                        <option key={key} value={key}>
                          {preset.label}
                        </option>
                      ))}
                    </select>
                  </label>
                  <label className="field">
                    <span>Portion</span>
                    <select
                      value={photoDraft.portionSize}
                      onChange={(event) =>
                        setPhotoDraft((current) => ({
                          ...current,
                          portionSize: event.target.value as PortionSize,
                        }))
                      }
                    >
                      {Object.entries(portionMultipliers).map(([key, portion]) => (
                        <option key={key} value={key}>
                          {portion.label}
                        </option>
                      ))}
                    </select>
                  </label>
                  <label className="field">
                    <span>Oil / butter</span>
                    <select
                      value={photoDraft.cookingFat}
                      onChange={(event) =>
                        setPhotoDraft((current) => ({
                          ...current,
                          cookingFat: event.target.value as CookingFat,
                        }))
                      }
                    >
                      {Object.entries(cookingFatAdds).map(([key, fatAdd]) => (
                        <option key={key} value={key}>
                          {fatAdd.label}
                        </option>
                      ))}
                    </select>
                  </label>
                  <label className="field">
                    <span>Sauce / dressing</span>
                    <select
                      value={photoDraft.sauceLevel}
                      onChange={(event) =>
                        setPhotoDraft((current) => ({
                          ...current,
                          sauceLevel: event.target.value as SauceLevel,
                        }))
                      }
                    >
                      {Object.entries(sauceAdds).map(([key, sauce]) => (
                        <option key={key} value={key}>
                          {sauce.label}
                        </option>
                      ))}
                    </select>
                  </label>
                  <label className="toggle-row">
                    <input
                      checked={photoDraft.referenceInFrame}
                      type="checkbox"
                      onChange={(event) =>
                        setPhotoDraft((current) => ({
                          ...current,
                          referenceInFrame: event.target.checked,
                        }))
                      }
                    />
                    <span>Fork, hand, can, or full plate visible for scale</span>
                  </label>
                </div>

                <label className="field">
                  <span>Accuracy notes</span>
                  <textarea
                    placeholder="Ate half the fries, sauce on side, shared appetizer..."
                    value={photoDraft.notes}
                    onChange={(event) =>
                      setPhotoDraft((current) => ({ ...current, notes: event.target.value }))
                    }
                  />
                </label>

                <div className="accuracy-card">
                  <div className="confidence-line">
                    <span>Confidence</span>
                    <strong>{photoEstimate.confidence}%</strong>
                  </div>
                  <div className="progress-track confidence-track">
                    <span style={{ width: `${photoEstimate.confidence}%` }} />
                  </div>
                  <div className="estimate-breakdown">
                    <strong>{photoEstimate.calories} cal</strong>
                    <span>{photoEstimate.protein}g P</span>
                    <span>{photoEstimate.carbs}g C</span>
                    <span>{photoEstimate.fat}g F</span>
                  </div>
                  <ul className="accuracy-list">
                    {photoEstimate.accuracyNotes.map((note) => (
                      <li key={note}>{note}</li>
                    ))}
                  </ul>
                </div>

                <button
                  className="primary-button wide-button"
                  disabled={!photoDraft.photoDataUrl}
                  type="button"
                  onClick={addPhotoEstimate}
                >
                  Add photo estimate
                </button>
              </div>
            </div>
          </section>

          <section className="panel food-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Manual entry</p>
                <h2>Add food</h2>
              </div>
              <button className="ghost-button" type="button" onClick={() => setFoods([])}>
                Clear day
              </button>
            </div>

            <div className="food-form">
              <label className="field food-name">
                <span>Food</span>
                <input
                  placeholder="Chicken bowl, protein bar, sauce..."
                  type="text"
                  value={draft.name}
                  onChange={(event) => setDraft((current) => ({ ...current, name: event.target.value }))}
                />
              </label>
              <NumberInput
                label="Calories"
                value={draft.calories}
                onChange={(value) => setDraft((current) => ({ ...current, calories: value }))}
              />
              <NumberInput
                label="Protein"
                value={draft.protein}
                onChange={(value) => setDraft((current) => ({ ...current, protein: value }))}
              />
              <NumberInput
                label="Carbs"
                value={draft.carbs}
                onChange={(value) => setDraft((current) => ({ ...current, carbs: value }))}
              />
              <NumberInput
                label="Fat"
                value={draft.fat}
                onChange={(value) => setDraft((current) => ({ ...current, fat: value }))}
              />
              <button className="primary-button" type="button" onClick={() => addFood(draft)}>
                Add
              </button>
            </div>

            <div className="quick-adds" aria-label="Quick add foods">
              {quickAdds.map((food) => (
                <button key={food.name} type="button" onClick={() => addFood(food)}>
                  + {food.name}
                </button>
              ))}
            </div>

            <div className="food-table">
              <div className="food-row table-head">
                <span>Food</span>
                <span>Cal</span>
                <span>P</span>
                <span>C</span>
                <span>F</span>
                <span />
              </div>
              {foods.length === 0 ? (
                <div className="empty-state">
                  <strong>No foods logged yet.</strong>
                  <span>Start with one meal and let the totals do the work.</span>
                </div>
              ) : (
                foods.map((food) => (
                  <div className="food-row" key={food.id}>
                    <div className="food-title-cell">
                      {food.photoDataUrl ? (
                        <img alt="" className="food-thumb" src={food.photoDataUrl} />
                      ) : null}
                      <div>
                        <input
                          aria-label="Food name"
                          type="text"
                          value={food.name}
                          onChange={(event) => updateFood(food.id, { name: event.target.value })}
                        />
                        {food.source === "photo" ? (
                          <small>
                            Photo estimate · {food.confidence}% confidence
                            {food.estimateRange
                              ? ` · ${food.estimateRange.low}-${food.estimateRange.high} cal`
                              : ""}
                          </small>
                        ) : null}
                      </div>
                    </div>
                    <input
                      aria-label="Calories"
                      min="0"
                      type="number"
                      value={food.calories}
                      onChange={(event) => updateFood(food.id, { calories: Number(event.target.value) })}
                    />
                    <input
                      aria-label="Protein"
                      min="0"
                      type="number"
                      value={food.protein}
                      onChange={(event) => updateFood(food.id, { protein: Number(event.target.value) })}
                    />
                    <input
                      aria-label="Carbs"
                      min="0"
                      type="number"
                      value={food.carbs}
                      onChange={(event) => updateFood(food.id, { carbs: Number(event.target.value) })}
                    />
                    <input
                      aria-label="Fat"
                      min="0"
                      type="number"
                      value={food.fat}
                      onChange={(event) => updateFood(food.id, { fat: Number(event.target.value) })}
                    />
                    <button
                      aria-label={`Remove ${food.name}`}
                      className="remove-button"
                      type="button"
                      onClick={() => removeFood(food.id)}
                    >
                      x
                    </button>
                  </div>
                ))
              )}
            </div>
          </section>
        </div>

        <aside className="side-column">
          <section className="panel fasting-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Intermittent fasting</p>
                <h2>{fasting.enabled ? "16-hour timer" : "Fasting off"}</h2>
              </div>
              <label className="switch">
                <input
                  checked={fasting.enabled}
                  type="checkbox"
                  onChange={(event) => updateFasting("enabled", event.target.checked)}
                />
                <span />
              </label>
            </div>

            <div className="fasting-status">
              <div className="fasting-time">
                <span>{fastComplete ? "Fast complete" : "Time remaining"}</span>
                <strong>
                  {fasting.enabled && lastMealTime ? formatDuration(fastRemaining) : "--"}
                </strong>
                <small>
                  Eating window {fasting.windowStart}-{fasting.windowEnd} is{" "}
                  {eatingWindowOpen ? "open" : "closed"}
                </small>
              </div>
              <div className="progress-track">
                <span style={{ width: `${fasting.enabled ? fastProgress : 0}%` }} />
              </div>
            </div>

            <div className="goal-fields">
              <NumberInput
                label="Fast hours"
                value={fasting.fastHours}
                onChange={(value) => updateFasting("fastHours", value)}
                step={1}
              />
              <label className="field">
                <span>Last meal</span>
                <input
                  type="datetime-local"
                  value={fasting.lastMealAt}
                  onChange={(event) => updateFasting("lastMealAt", event.target.value)}
                />
              </label>
              <label className="field">
                <span>Window start</span>
                <input
                  type="time"
                  value={fasting.windowStart}
                  onChange={(event) => updateFasting("windowStart", event.target.value)}
                />
              </label>
              <label className="field">
                <span>Window end</span>
                <input
                  type="time"
                  value={fasting.windowEnd}
                  onChange={(event) => updateFasting("windowEnd", event.target.value)}
                />
              </label>
            </div>

            <button
              className="primary-button wide-button"
              disabled={!fasting.enabled}
              type="button"
              onClick={() => updateFasting("lastMealAt", toDateTimeLocal(Date.now()))}
            >
              Log last meal now
            </button>
            <p className="fine-print">
              Default plan: 16-hour fast, eating from 12:00 to 20:00. The timer starts when
              you log your last meal.
            </p>
          </section>

          <section className="panel goal-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Goal setup</p>
                <h2>Cut target</h2>
              </div>
              <span className={`pace-pill ${paceTone}`}>{cutStatus}</span>
            </div>

            <div className="goal-fields">
              <NumberInput
                label="Daily calories"
                value={goals.calorieTarget}
                onChange={(value) => updateGoal("calorieTarget", value)}
              />
              <NumberInput
                label="Start weight"
                value={goals.startWeight}
                onChange={(value) => updateGoal("startWeight", value)}
                step={0.1}
              />
              <NumberInput
                label="Current weight"
                value={goals.currentWeight}
                onChange={(value) => updateGoal("currentWeight", value)}
                step={0.1}
              />
              <NumberInput
                label="Goal weight"
                value={goals.targetWeight}
                onChange={(value) => updateGoal("targetWeight", value)}
                step={0.1}
              />
              <label className="field">
                <span>Goal date</span>
                <input
                  type="date"
                  value={goals.goalDate}
                  onChange={(event) => updateGoal("goalDate", event.target.value)}
                />
              </label>
            </div>

            <div className="weight-track">
              <div className="ring-stat">
                <svg viewBox="0 0 120 120" role="img" aria-label="Weight loss progress">
                  <circle cx="60" cy="60" r="48" />
                  <circle
                    cx="60"
                    cy="60"
                    r="48"
                    style={{ strokeDashoffset: `${301.6 - (301.6 * clamp(weightProgress, 0, 100)) / 100}` }}
                  />
                </svg>
                <div>
                  <strong>{round(weightProgress)}%</strong>
                  <span>{round(lostSoFar, 1)} lb down</span>
                </div>
              </div>
              <div className="pace-details">
                <span>Required deficit</span>
                <strong>{round(dailyDeficit)} cal/day</strong>
                <small>{round(poundsLeft, 1)} lb to {round(goals.targetWeight, 1)}</small>
              </div>
            </div>
          </section>

          <section className="panel protein-panel">
            <div className="panel-heading">
              <div>
                <p className="eyebrow">Protein rule</p>
                <h2>{round(proteinGoal)}g target</h2>
              </div>
            </div>
            <div className="goal-fields">
              <NumberInput
                label="Grams per pound"
                value={goals.proteinPerPound}
                onChange={(value) => updateGoal("proteinPerPound", value)}
                step={0.1}
              />
              <label className="field">
                <span>Basis</span>
                <select
                  value={goals.proteinBasis}
                  onChange={(event) => updateGoal("proteinBasis", event.target.value as ProteinBasis)}
                >
                  <option value="bodyWeight">Body weight</option>
                  <option value="leanMass">Lean mass</option>
                  <option value="bodyFat">Body fat</option>
                </select>
              </label>
              <NumberInput
                label="Body fat %"
                value={goals.bodyFatPercent}
                onChange={(value) => updateGoal("bodyFatPercent", value)}
                step={0.5}
              />
            </div>
            <p className="fine-print">
              Formula: {goals.proteinPerPound}g per lb of {basisLabels[goals.proteinBasis]}.
            </p>
          </section>

          <section className="panel insight-panel">
            <p className="eyebrow">Readout</p>
            <h2>Today&apos;s check</h2>
            <div className="readout-list">
              <div>
                <span>Calories from macros</span>
                <strong>{round(caloriesFromMacros)}</strong>
              </div>
              <div>
                <span>Logged items</span>
                <strong>{foods.length}</strong>
              </div>
              <div>
                <span>Protein completion</span>
                <strong>{round(progressPercent(totals.protein, proteinGoal))}%</strong>
              </div>
            </div>
            <p className="fine-print">
              Estimates are for planning. If performance, mood, or sleep drops hard, bring the target up.
            </p>
          </section>
        </aside>
      </section>
    </main>
  );
}

```

## app/globals.css

```css
@import "tailwindcss";

:root {
  --background: #07090d;
  --surface: #10141b;
  --surface-2: #151b24;
  --surface-3: #1c2430;
  --foreground: #f6f7f2;
  --muted: #9ba6b5;
  --line: rgba(255, 255, 255, 0.1);
  --green: #75f2a8;
  --green-2: #1fd071;
  --amber: #f5c86b;
  --red: #ff735f;
  --cyan: #79d8ff;
  --ink: #07090d;
  --shadow: 0 24px 80px rgba(0, 0, 0, 0.38);
}

* {
  box-sizing: border-box;
}

html {
  background: var(--background);
}

body {
  min-height: 100vh;
  margin: 0;
  background:
    radial-gradient(circle at 16% 0%, rgba(117, 242, 168, 0.12), transparent 28rem),
    radial-gradient(circle at 86% 12%, rgba(121, 216, 255, 0.1), transparent 24rem),
    linear-gradient(180deg, #0a0d13 0%, var(--background) 44%, #050609 100%);
  color: var(--foreground);
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
}

button,
input,
select,
textarea {
  font: inherit;
}

button {
  cursor: pointer;
}

input,
select,
textarea {
  width: 100%;
  min-width: 0;
  border: 1px solid var(--line);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.05);
  color: var(--foreground);
  outline: none;
  transition:
    border-color 160ms ease,
    background 160ms ease,
    box-shadow 160ms ease;
}

input:focus,
select:focus,
textarea:focus {
  border-color: rgba(117, 242, 168, 0.75);
  box-shadow: 0 0 0 3px rgba(117, 242, 168, 0.14);
}

textarea {
  min-height: 80px;
  resize: vertical;
  padding: 11px;
}

select option {
  background: #10141b;
  color: var(--foreground);
}

.app-shell {
  width: min(1440px, 100%);
  margin: 0 auto;
  padding: 24px;
}

.hero-panel {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 280px;
  gap: 24px;
  align-items: stretch;
  min-height: 310px;
  border: 1px solid var(--line);
  border-radius: 8px;
  background:
    linear-gradient(135deg, rgba(16, 20, 27, 0.92), rgba(12, 15, 20, 0.8)),
    linear-gradient(90deg, rgba(117, 242, 168, 0.14), transparent 44%),
    var(--surface);
  box-shadow: var(--shadow);
  overflow: hidden;
}

.hero-copy {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  padding: 32px;
}

.eyebrow {
  margin: 0 0 8px;
  color: var(--green);
  font-size: 0.72rem;
  font-weight: 800;
  text-transform: uppercase;
}

h1,
h2 {
  margin: 0;
  font-weight: 850;
  line-height: 1;
}

h1 {
  max-width: 820px;
  font-size: clamp(2.75rem, 7vw, 6.75rem);
}

h2 {
  font-size: 1.35rem;
}

.hero-stats {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
  margin-top: 28px;
}

.metric-card,
.panel {
  border: 1px solid var(--line);
  border-radius: 8px;
  background: rgba(16, 20, 27, 0.78);
}

.metric-card {
  min-height: 126px;
  padding: 18px;
}

.metric-card span,
.field span,
.date-field span,
.progress-label span,
.readout-list span,
.fine-print,
.metric-card small,
.pace-details span,
.pace-details small,
.empty-state span {
  color: var(--muted);
}

.metric-card strong {
  display: block;
  margin-top: 10px;
  font-size: clamp(1.6rem, 3vw, 2.5rem);
  line-height: 1;
}

.metric-card small {
  display: block;
  margin-top: 10px;
  font-size: 0.86rem;
}

.metric-card.green strong {
  color: var(--green);
}

.metric-card.amber strong {
  color: var(--amber);
}

.metric-card.red strong {
  color: var(--red);
}

.plate-visual {
  position: relative;
  display: grid;
  min-height: 100%;
  place-items: center;
  border-left: 1px solid var(--line);
  background:
    linear-gradient(180deg, rgba(255, 255, 255, 0.06), transparent),
    rgba(255, 255, 255, 0.03);
}

.plate-ring {
  position: relative;
  width: min(220px, 72vw);
  aspect-ratio: 1;
  border: 16px solid rgba(255, 255, 255, 0.14);
  border-radius: 50%;
  background:
    radial-gradient(circle at center, rgba(255, 255, 255, 0.13), transparent 56%),
    #161b22;
  box-shadow:
    inset 0 0 0 1px rgba(255, 255, 255, 0.18),
    0 30px 70px rgba(0, 0, 0, 0.45);
}

.food {
  position: absolute;
  display: block;
  border-radius: 999px;
}

.food.protein {
  width: 74px;
  height: 50px;
  top: 52px;
  left: 38px;
  background: linear-gradient(135deg, #f3c189, #d66e4c);
}

.food.carb {
  width: 76px;
  height: 76px;
  right: 34px;
  bottom: 34px;
  background: linear-gradient(135deg, #fff2b0, #d8aa4a);
}

.food.greens {
  width: 82px;
  height: 58px;
  top: 42px;
  right: 34px;
  background: linear-gradient(135deg, #75f2a8, #168b53);
}

.food.sauce {
  width: 44px;
  height: 44px;
  left: 64px;
  bottom: 38px;
  background: linear-gradient(135deg, #ff735f, #8c302a);
}

.dashboard-grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) 380px;
  gap: 18px;
  margin-top: 18px;
}

.main-column,
.side-column {
  display: grid;
  gap: 18px;
  align-content: start;
}

.panel {
  padding: 22px;
  box-shadow: 0 18px 48px rgba(0, 0, 0, 0.22);
}

.panel-heading {
  display: flex;
  gap: 16px;
  align-items: flex-start;
  justify-content: space-between;
  margin-bottom: 22px;
}

.date-field,
.field {
  display: grid;
  gap: 8px;
}

.field span,
.date-field span {
  font-size: 0.75rem;
  font-weight: 750;
  text-transform: uppercase;
}

.date-field {
  min-width: 170px;
}

.date-field input,
.field input,
.field select,
.field textarea,
.food-row input {
  padding: 0 11px;
}

.date-field input,
.field input,
.field select,
.food-row input {
  height: 42px;
}

.field textarea {
  padding: 11px;
}

.macro-stack {
  display: grid;
  gap: 16px;
}

.progress-row {
  display: grid;
  gap: 8px;
}

.progress-label {
  display: flex;
  justify-content: space-between;
  gap: 16px;
  font-size: 0.94rem;
}

.progress-label strong small {
  color: var(--muted);
  font-weight: 600;
}

.progress-track {
  height: 12px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.07);
  overflow: hidden;
}

.progress-track span {
  display: block;
  height: 100%;
  border-radius: inherit;
  background: linear-gradient(90deg, var(--green), var(--cyan));
}

.food-form {
  display: grid;
  grid-template-columns: minmax(220px, 1.6fr) repeat(4, minmax(84px, 0.5fr)) 88px;
  gap: 10px;
  align-items: end;
}

.food-name input {
  min-width: 0;
}

.primary-button,
.ghost-button,
.quick-adds button,
.remove-button,
.photo-drop {
  min-height: 42px;
  border: 1px solid transparent;
  border-radius: 8px;
  font-weight: 800;
}

.primary-button {
  background: var(--green);
  color: var(--ink);
}

.primary-button:disabled {
  cursor: not-allowed;
  opacity: 0.45;
  transform: none;
}

.ghost-button,
.quick-adds button {
  border-color: var(--line);
  background: rgba(255, 255, 255, 0.05);
  color: var(--foreground);
}

.primary-button:hover,
.quick-adds button:hover,
.ghost-button:hover {
  transform: translateY(-1px);
}

.quick-adds {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin: 18px 0;
}

.quick-adds button {
  padding: 0 13px;
  color: var(--muted);
}

.photo-grid {
  display: grid;
  grid-template-columns: minmax(260px, 0.75fr) minmax(0, 1.25fr);
  gap: 18px;
}

.camera-card {
  display: grid;
  gap: 10px;
  align-content: start;
}

.photo-drop {
  position: relative;
  display: grid;
  min-height: 360px;
  place-items: center;
  overflow: hidden;
  border-color: rgba(117, 242, 168, 0.22);
  background:
    linear-gradient(135deg, rgba(117, 242, 168, 0.1), transparent),
    rgba(255, 255, 255, 0.04);
}

.file-input {
  position: absolute;
  inset: 0;
  opacity: 0;
  cursor: pointer;
}

.photo-preview {
  width: 100%;
  height: 100%;
  min-height: 360px;
  object-fit: cover;
}

.photo-empty {
  display: grid;
  gap: 8px;
  max-width: 260px;
  padding: 24px;
  text-align: center;
}

.photo-empty strong {
  font-size: 1.35rem;
}

.photo-empty small {
  color: var(--muted);
  line-height: 1.45;
}

.photo-controls,
.photo-fields {
  display: grid;
  gap: 12px;
}

.photo-fields {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.toggle-row {
  display: flex;
  grid-column: 1 / -1;
  gap: 10px;
  align-items: center;
  min-height: 42px;
  color: var(--muted);
  font-size: 0.9rem;
}

.toggle-row input {
  width: 18px;
  height: 18px;
  accent-color: var(--green);
}

.accuracy-card {
  display: grid;
  gap: 10px;
  border: 1px solid rgba(117, 242, 168, 0.16);
  border-radius: 8px;
  padding: 14px;
  background: rgba(117, 242, 168, 0.06);
}

.confidence-line,
.estimate-breakdown {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.confidence-line span,
.accuracy-list,
.food-title-cell small {
  color: var(--muted);
}

.confidence-line strong {
  color: var(--green);
  font-size: 1.4rem;
}

.confidence-track span {
  background: linear-gradient(90deg, var(--amber), var(--green));
}

.estimate-breakdown {
  flex-wrap: wrap;
}

.estimate-breakdown strong {
  color: var(--foreground);
  font-size: 1.2rem;
}

.estimate-breakdown span {
  color: var(--cyan);
  font-weight: 800;
}

.accuracy-list {
  margin: 0;
  padding-left: 18px;
  line-height: 1.45;
  font-size: 0.86rem;
}

.range-chip {
  flex: 0 0 auto;
  border: 1px solid rgba(121, 216, 255, 0.22);
  border-radius: 999px;
  padding: 8px 10px;
  color: var(--cyan);
  background: rgba(121, 216, 255, 0.08);
  font-size: 0.74rem;
  font-weight: 850;
}

.wide-button {
  width: 100%;
}

.food-table {
  display: grid;
  gap: 8px;
}

.food-row {
  display: grid;
  grid-template-columns: minmax(180px, 1.5fr) repeat(4, minmax(64px, 0.42fr)) 44px;
  gap: 8px;
  align-items: center;
}

.food-title-cell {
  display: flex;
  gap: 10px;
  align-items: center;
  min-width: 0;
}

.food-title-cell > div {
  display: grid;
  gap: 5px;
  min-width: 0;
  width: 100%;
}

.food-title-cell small {
  overflow: hidden;
  font-size: 0.72rem;
  line-height: 1.25;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.food-thumb {
  width: 44px;
  height: 44px;
  flex: 0 0 44px;
  border: 1px solid var(--line);
  border-radius: 8px;
  object-fit: cover;
}

.table-head {
  padding: 0 8px;
  color: var(--muted);
  font-size: 0.72rem;
  font-weight: 800;
  text-transform: uppercase;
}

.remove-button {
  background: rgba(255, 115, 95, 0.1);
  color: var(--red);
}

.empty-state {
  display: grid;
  gap: 6px;
  min-height: 118px;
  place-items: center;
  border: 1px dashed rgba(255, 255, 255, 0.14);
  border-radius: 8px;
  text-align: center;
}

.goal-fields {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 12px;
}

.pace-pill {
  flex: 0 0 auto;
  border: 1px solid var(--line);
  border-radius: 999px;
  padding: 8px 10px;
  font-size: 0.74rem;
  font-weight: 850;
}

.pace-pill.green {
  background: rgba(117, 242, 168, 0.1);
  color: var(--green);
}

.pace-pill.amber {
  background: rgba(245, 200, 107, 0.1);
  color: var(--amber);
}

.pace-pill.red {
  background: rgba(255, 115, 95, 0.1);
  color: var(--red);
}

.weight-track {
  display: grid;
  grid-template-columns: 150px minmax(0, 1fr);
  gap: 18px;
  align-items: center;
  margin-top: 22px;
  padding-top: 20px;
  border-top: 1px solid var(--line);
}

.ring-stat {
  position: relative;
  display: grid;
  place-items: center;
}

.ring-stat svg {
  width: 136px;
  height: 136px;
  transform: rotate(-90deg);
}

.ring-stat circle {
  fill: none;
  stroke-width: 10;
}

.ring-stat circle:first-child {
  stroke: rgba(255, 255, 255, 0.08);
}

.ring-stat circle:last-child {
  stroke: var(--green);
  stroke-dasharray: 301.6;
  stroke-linecap: round;
}

.ring-stat div {
  position: absolute;
  display: grid;
  place-items: center;
  text-align: center;
}

.ring-stat strong {
  font-size: 1.8rem;
  line-height: 1;
}

.ring-stat span {
  margin-top: 4px;
  color: var(--muted);
  font-size: 0.78rem;
}

.pace-details {
  display: grid;
  gap: 7px;
}

.pace-details strong {
  color: var(--green);
  font-size: 1.55rem;
}

.fine-print {
  margin: 14px 0 0;
  font-size: 0.86rem;
  line-height: 1.5;
}

.readout-list {
  display: grid;
  gap: 10px;
  margin-top: 18px;
}

.readout-list div {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  border-bottom: 1px solid var(--line);
  padding-bottom: 10px;
}

.readout-list div:last-child {
  border-bottom: 0;
  padding-bottom: 0;
}

.fasting-status {
  display: grid;
  gap: 12px;
  margin-bottom: 16px;
  border: 1px solid rgba(121, 216, 255, 0.18);
  border-radius: 8px;
  padding: 16px;
  background: rgba(121, 216, 255, 0.06);
}

.fasting-time {
  display: grid;
  gap: 5px;
}

.fasting-time span,
.fasting-time small {
  color: var(--muted);
}

.fasting-time strong {
  color: var(--cyan);
  font-size: 2rem;
  line-height: 1;
}

.switch {
  position: relative;
  width: 54px;
  height: 30px;
}

.switch input {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  margin: 0;
  opacity: 0;
}

.switch span {
  position: absolute;
  inset: 0;
  border: 1px solid var(--line);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
}

.switch span::after {
  position: absolute;
  top: 4px;
  left: 4px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: var(--muted);
  content: "";
  transition:
    transform 160ms ease,
    background 160ms ease;
}

.switch input:checked + span {
  background: rgba(117, 242, 168, 0.18);
}

.switch input:checked + span::after {
  transform: translateX(24px);
  background: var(--green);
}

@media (max-width: 1120px) {
  .hero-panel,
  .dashboard-grid,
  .photo-grid {
    grid-template-columns: 1fr;
  }

  .plate-visual {
    min-height: 230px;
    border-top: 1px solid var(--line);
    border-left: 0;
  }

  .food-form {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }

  .food-name {
    grid-column: 1 / -1;
  }

  .primary-button {
    grid-column: span 2;
  }

  .photo-drop,
  .photo-preview {
    min-height: 300px;
  }
}

@media (max-width: 760px) {
  .app-shell {
    padding: 12px;
  }

  .hero-copy,
  .panel {
    padding: 18px;
  }

  h1 {
    font-size: clamp(2.25rem, 12vw, 4rem);
  }

  .hero-stats,
  .goal-fields,
  .photo-fields,
  .weight-track {
    grid-template-columns: 1fr;
  }

  .panel-heading {
    flex-direction: column;
  }

  .date-field {
    width: 100%;
  }

  .food-form {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }

  .food-row {
    grid-template-columns: minmax(136px, 1fr) repeat(4, minmax(48px, 0.36fr)) 38px;
    gap: 6px;
  }

  .food-row input {
    height: 38px;
    padding: 0 8px;
    font-size: 0.9rem;
  }

  .table-head {
    font-size: 0.64rem;
  }

  .photo-drop,
  .photo-preview {
    min-height: 250px;
  }

  .fasting-time strong {
    font-size: 1.65rem;
  }
}

@media (max-width: 560px) {
  .food-table {
    overflow-x: auto;
    padding-bottom: 4px;
  }

  .food-row {
    min-width: 540px;
  }

  .quick-adds button {
    flex: 1 1 100%;
  }
}

```

## app/layout.tsx

```tsx
import type { Metadata } from "next";
import { headers } from "next/headers";
import "./globals.css";

const siteTitle = "Cut Command";
const siteDescription =
  "A dark calorie, protein, and weight-goal tracker for a focused cut.";

export async function generateMetadata(): Promise<Metadata> {
  const headerStore = await headers();
  const host = headerStore.get("x-forwarded-host") ?? headerStore.get("host") ?? "localhost:3000";
  const protocol =
    headerStore.get("x-forwarded-proto") ?? (host.startsWith("localhost") ? "http" : "https");
  const origin = `${protocol}://${host}`;

  return {
    title: siteTitle,
    description: siteDescription,
    icons: {
      icon: "/favicon.svg",
      shortcut: "/favicon.svg",
    },
    openGraph: {
      title: siteTitle,
      description: siteDescription,
      type: "website",
      images: [
        {
          url: `${origin}/og.png`,
          width: 1200,
          height: 630,
          alt: "Cut Command dark nutrition tracker dashboard",
        },
      ],
    },
    twitter: {
      card: "summary_large_image",
      title: siteTitle,
      description: siteDescription,
      images: [`${origin}/og.png`],
    },
  };
}

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}

```

## tests/rendered-html.test.mjs

```js
import assert from "node:assert/strict";
import { readFile } from "node:fs/promises";
import test from "node:test";

test("cut tracker keeps the requested defaults and controls", async () => {
  const page = await readFile(new URL("../app/page.tsx", import.meta.url), "utf8");
  const layout = await readFile(new URL("../app/layout.tsx", import.meta.url), "utf8");
  const styles = await readFile(new URL("../app/globals.css", import.meta.url), "utf8");

  assert.match(layout, /const siteTitle = "Cut Command"/);
  assert.match(layout, /generateMetadata/);
  assert.match(layout, /\/og\.png/);
  assert.match(page, /calorieTarget:\s*2500/);
  assert.match(page, /startWeight:\s*215/);
  assert.match(page, /targetWeight:\s*205/);
  assert.match(page, /proteinPerPound:\s*0\.8/);
  assert.match(page, /Manual entry/);
  assert.match(page, /Restaurant photo log/);
  assert.match(page, /capture="environment"/);
  assert.match(page, /Add photo estimate/);
  assert.match(page, /Unknown cooking fat assumes \+120 calories/);
  assert.match(page, /Intermittent fasting/);
  assert.match(page, /fastHours:\s*16/);
  assert.match(page, /windowStart:\s*"12:00"/);
  assert.match(page, /windowEnd:\s*"20:00"/);
  assert.match(page, /Log last meal now/);
  assert.match(page, /Goal setup/);
  assert.match(page, /Protein rule/);
  assert.match(styles, /--background:\s*#07090d/);
  assert.doesNotMatch(page, /SkeletonPreview|codex-preview|react-loading-skeleton/);
});

```

## .openai/hosting.json

```json
{
  "project_id": "appgprj_6a59913aeef481918173db7bbbff5fe0",
  "d1": null,
  "r2": null
}

```

