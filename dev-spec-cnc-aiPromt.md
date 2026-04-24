# CNC AI Технолог — Технічне завдання для React-розробника

## 🎯 Суть проєкту

Веб-додаток (React + TypeScript) — AI-асистент для технологів та операторів ЧПУ верстатів.
Допомагає оптимізувати токарну обробку: розраховує маршрути, режими різання, знаходить де скоротити час.

**КЛЮЧОВИЙ ПРИНЦИП: AI НЕ приймає рішень. AI ПРОПОНУЄ — людина ВИРІШУЄ.**
Кожен результат AI проходить перевірку технолога → тестування на верстаті → затвердження.
Система накопичує базу знань з кожного проєкту — AI стає точнішим з часом.

---

## 🏗 Архітектура

```
Frontend:  React 18+ / TypeScript / Tailwind CSS / Zustand (стейт)
Backend:   Node.js + Express (або Next.js API routes)
AI:        Claude API (Anthropic) — claude-sonnet-4-20250514
DB:        PostgreSQL (основна) + Redis (кеш/сесії)
Storage:   S3-сумісний (креслення, паспорти, PDF)
Auth:      JWT + refresh tokens
Deploy:    Docker → будь-який хостинг (VPS / Vercel / Railway)
```

### Структура проєкту

```
src/
├── app/                    # Next.js App Router або React Router
│   ├── layout.tsx
│   ├── page.tsx           # Dashboard
│   ├── machines/
│   │   ├── page.tsx       # Список верстатів
│   │   └── [id]/page.tsx  # Деталі верстата
│   ├── parts/
│   │   ├── page.tsx       # Список деталей
│   │   └── [id]/page.tsx
│   ├── projects/
│   │   ├── page.tsx       # Список проєктів + фільтри
│   │   ├── new/page.tsx   # Wizard створення
│   │   └── [id]/page.tsx  # Деталі проєкту + AI + коментарі
│   ├── knowledge/page.tsx # База знань
│   ├── tutorial/page.tsx  # Навчання
│   └── api/
│       ├── ai/analyze/route.ts
│       ├── ai/recognize/route.ts
│       ├── machines/route.ts
│       ├── parts/route.ts
│       └── projects/route.ts
├── components/
│   ├── layout/
│   │   ├── Header.tsx
│   │   ├── Sidebar.tsx    # Навігація
│   │   └── Toast.tsx
│   ├── machines/
│   │   ├── MachineCard.tsx
│   │   ├── MachineForm.tsx
│   │   └── MachineUpload.tsx  # AI розпізнавання паспорту
│   ├── parts/
│   │   ├── PartCard.tsx
│   │   ├── PartForm.tsx
│   │   └── PartUpload.tsx     # AI розпізнавання креслення
│   ├── projects/
│   │   ├── ProjectCard.tsx
│   │   ├── ProjectWizard.tsx  # Покроковий майстер
│   │   ├── ProjectDetail.tsx
│   │   ├── AIAnalysis.tsx     # Блок AI-аналізу
│   │   ├── CommentThread.tsx  # Коментарі технолога
│   │   ├── StatusActions.tsx  # Кнопки зміни статусу
│   │   └── ProjectLog.tsx     # Журнал дій
│   ├── tutorial/
│   │   ├── TutorialLayout.tsx
│   │   ├── LessonCard.tsx
│   │   └── TryItButton.tsx    # "Спробуйте зараз"
│   ├── ui/                    # Загальні компоненти
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Select.tsx
│   │   ├── FileUpload.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── FilterChips.tsx
│   │   └── ProgressBar.tsx
│   └── knowledge/
│       ├── KnowledgeStats.tsx
│       └── LearningPipeline.tsx
├── store/
│   ├── machineStore.ts
│   ├── partStore.ts
│   ├── projectStore.ts
│   └── uiStore.ts
├── lib/
│   ├── ai.ts              # Claude API wrapper
│   ├── db.ts              # Prisma client
│   ├── storage.ts         # S3 upload
│   └── pdf.ts             # Генерація PDF техкарт
├── types/
│   └── index.ts           # Всі TypeScript типи
└── prompts/
    ├── analyze.ts          # Промт для AI-аналізу
    ├── recognize-machine.ts
    └── recognize-part.ts
```

---

## 📦 Моделі даних (Prisma Schema)

```prisma
model Machine {
  id          String   @id @default(cuid())
  name        String                          // "PINACHO STH 400-105x3000"
  type        String                          // "Токарний ЧПУ"
  cncSystem   String?                         // "Fagor 8055"
  maxRpm      Int                             // 4000
  power       Float                           // 26 кВт
  maxDiameter Int                             // 400 мм
  rmc         Int                             // 3000 мм (відстань між центрами)
  hasLunet    Boolean  @default(false)
  turretSlots Int?                            // К-ть позицій револьвера
  coolantType String?                         // "емульсія", "масло", "суха"
  notes       String?
  passportUrl String?                         // URL файлу паспорту в S3
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  projects    Project[]
}

model Part {
  id          String   @id @default(cuid())
  name        String                          // "Шток"
  drawingNo   String?                         // "4690-2.01.2024 B.01"
  material    String                          // "C45E (CK45)"
  hardness    String?                         // "HRC 45-50"
  weight      Float?                          // 37 кг
  dimensions  Json                            // Структуровані розміри (див. нижче)
  rawDimText  String?                         // Текстовий опис розмірів
  drawingUrl  String?                         // URL креслення в S3
  surfaceReq  String?                         // "Ra 6.3 загальна"
  notes       String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  projects    Project[]
}

model Tool {
  id          String   @id @default(cuid())
  holder      String                          // "TCLNR 3232 P12"
  insert      String                          // "CNMG 120408 KCU10"
  maker       String?                         // "KENNAMETAL"
  type        ToolType                        // EXTERNAL, INTERNAL, GROOVING, THREADING, DRILLING
  notes       String?
  createdAt   DateTime @default(now())
}

enum ToolType {
  EXTERNAL
  INTERNAL
  GROOVING
  THREADING
  DRILLING
  OTHER
}

model Project {
  id           String        @id @default(cuid())
  name         String                          // "Оптимізація штока ø90"
  status       ProjectStatus @default(DRAFT)
  machineId    String
  machine      Machine       @relation(fields: [machineId], references: [id])
  partId       String
  part         Part          @relation(fields: [partId], references: [id])
  tools        Json                            // масив інструментів для цього проєкту
  currentTime  Int                             // 100 хв — поточний час обробки
  targetTime   Int?                            // бажаний час (опціонально)
  batchSize    Int           @default(50)
  setupCount   Int?                            // к-ть установів
  notes        String?
  
  // AI результати
  aiResult     String?                         // Текст AI-аналізу (маршрут + режими)
  aiOptTime    Int?                            // AI-розрахований оптимальний час
  aiSaving     Int?                            // Економія в хвилинах
  aiVersion    Int           @default(0)       // Лічильник перерахунків
  
  // Результати тестування
  realTime     Int?                            // Реальний час після тестування
  testNotes    String?                         // Примітки після тесту
  
  comments     Comment[]
  logs         ProjectLog[]
  
  createdAt    DateTime      @default(now())
  updatedAt    DateTime      @updatedAt
}

enum ProjectStatus {
  DRAFT           // Чернетка
  IN_PROGRESS     // В роботі (AI дав результат)
  AI_REVIEW       // AI аналізує
  TESTING         // Тестування на верстаті
  TESTED_OK       // Протестовано успішно
  TESTED_FAIL     // Тест невдалий
  REWORK          // Доопрацювання
  APPROVED        // Затверджено в базу знань
}

model Comment {
  id        String   @id @default(cuid())
  projectId String
  project   Project  @relation(fields: [projectId], references: [id])
  author    String   @default("Технолог")
  text      String
  createdAt DateTime @default(now())
}

model ProjectLog {
  id        String   @id @default(cuid())
  projectId String
  project   Project  @relation(fields: [projectId], references: [id])
  action    String                              // "Статус → В роботі"
  createdAt DateTime @default(now())
}
```

### Структура Part.dimensions (JSON)

```typescript
interface PartDimensions {
  totalLength: number;           // 879
  totalLengthTol: string;        // "±0.3"
  steps: Array<{
    diameter: number;            // 90, 80, 75, 70, 65, 61, 50
    tolerance: string;           // "f7", "h9", ""
    length: number;              // довжина ділянки
    surface: string;             // "Ra 6.3"
    hardened: boolean;           // true для ø90
    hardenDepth?: number;        // 2 мм
  }>;
  threads: Array<{
    designation: string;         // "M65x1.5"
    length: number;
    position: string;            // "ліва сторона"
  }>;
  grooves: Array<{
    type: string;                // "зовнішня"
    width: number;
    depth: number;
    position: string;
  }>;
  chamfers: string[];            // ["1x45°", "2x45°"]
  tapers: Array<{
    angle: number;               // 20
    position: string;
  }>;
}
```

---

## 🤖 AI-промти (Claude API)

### 1. Промт для AI-аналізу проєкту (ГОЛОВНИЙ)

```typescript
// src/prompts/analyze.ts

export function buildAnalysisPrompt(project: Project, comments: Comment[]): string {
  const commentsText = comments.length > 0
    ? comments.map(c => `[${c.author}]: ${c.text}`).join('\n')
    : 'немає';

  // Якщо є попередні успішні проєкти на цьому верстаті — додати як контекст
  // Це і є "навчання" — AI бачить реальні дані минулих проєктів

  return `Ти — досвідчений інженер-технолог з 20+ роками досвіду в токарній обробці на ЧПУ.
Твоя задача — розробити оптимізовану технологічну карту.

ВАЖЛИВО: Ти НЕ приймаєш рішень. Ти ПРОПОНУЄШ — технолог перевірить і вирішить.
Будь конкретним. Числа, формули, обґрунтування. Не загальні фрази.

═══════════════ ВХІДНІ ДАНІ ═══════════════

ДЕТАЛЬ:
- Назва: ${project.part.name}
- Креслення: ${project.part.drawingNo}
- Матеріал: ${project.part.material}
- Твердість: ${project.part.hardness || 'не вказано'}
- Маса: ${project.part.weight} кг
- Розміри: ${project.part.rawDimText}

ВЕРСТАТ:
- Модель: ${project.machine.name}
- Тип: ${project.machine.type}
- Система ЧПУ: ${project.machine.cncSystem}
- Макс. оберти: ${project.machine.maxRpm} об/хв
- Потужність: ${project.machine.power} кВт
- Макс. діаметр: ${project.machine.maxDiameter} мм
- РМЦ: ${project.machine.rmc} мм
- Люнет: ${project.machine.hasLunet ? 'є' : 'немає'}
- ЗОР: ${project.machine.coolantType || 'не вказано'}

ІНСТРУМЕНТ:
${JSON.stringify(project.tools, null, 2)}

ПАРАМЕТРИ ОБРОБКИ:
- Поточний час: ${project.currentTime} хв
- Цільовий час: ${project.targetTime || 'оптимізувати максимально'}
- Партія: ${project.batchSize} шт
- К-ть установів: ${project.setupCount || 'не вказано'}

КОМЕНТАРІ ТЕХНОЛОГА (ВРАХУЙ ОБОВ'ЯЗКОВО):
${commentsText}

ПРИМІТКИ:
${project.notes || 'немає'}

═══════════════ ЗАВДАННЯ ═══════════════

Розроби технологічну карту з такими розділами:

## 1. СХЕМА БАЗУВАННЯ
Для кожного установу: як закріплювати, де люнет, де центр, обґрунтування.

## 2. МАРШРУТ ОБРОБКИ
Для КОЖНОГО установу — послідовність переходів:
- Номер
- Опис операції
- Інструмент (тримач + пластина зі списку)
- Оброблювані поверхні

## 3. РЕЖИМИ РІЗАННЯ
Таблиця для КОЖНОГО переходу:
| № | Операція | D мм | n об/хв | S мм/об | V м/хв | t мм | L мм | Проходів | To хв |

Формули:
- n = 1000·V / (π·D)
- To = L / (n·S) × кількість проходів
- Ra ≈ S² / (8·r) — перевірка шорсткості

## 4. НОРМУВАННЯ ЧАСУ
- Основний час To (сума)
- Допоміжний час Тд (встановлення, зняття, вимірювання, зміна інструменту — розписати)
- Час обслуговування Тобс (% від оперативного)
- Час відпочинку Твідп (% від оперативного)
- Підготовчо-заключний Тпз
- Штучний час Тшт = To + Тд + Тобс + Твідп
- Штучно-калькуляційний Тшк = Тшт + Тпз/n

## 5. ОПТИМІЗАЦІЯ
Порівняй поточний час (${project.currentTime} хв) з розрахованим.
Для КОЖНОЇ рекомендації вкажи:
- Що змінити (конкретно)
- Було → Стало
- Економія в хвилинах
- Ризики

## 6. ПІДСУМОК
- Поточний час: X хв
- Оптимізований час: Y хв
- Економія: Z хв (W%)

## 7. ВИТРАТИ ІНСТРУМЕНТУ
Аналіз на партію ${project.batchSize} шт: чи вистачить вказаної кількості пластин.

Відповідай УКРАЇНСЬКОЮ. Будь максимально конкретним.`;
}
```

### 2. Промт для розпізнавання паспорту верстата

```typescript
// src/prompts/recognize-machine.ts

export const MACHINE_RECOGNIZE_PROMPT = `Ти отримав фото або PDF паспорту токарного верстата.
Витягни з нього параметри і поверни ТІЛЬКИ JSON (без markdown, без пояснень):

{
  "name": "повна назва моделі верстата",
  "type": "Токарний ЧПУ / Токарний універсальний / Токарно-фрезерний",
  "cncSystem": "назва системи ЧПУ або null",
  "maxRpm": число_обертів_за_хвилину,
  "power": потужність_кВт,
  "maxDiameter": макс_діаметр_обробки_мм,
  "rmc": відстань_між_центрами_мм,
  "turretSlots": кількість_позицій_або_null,
  "notes": "додаткові важливі параметри"
}

Якщо параметр не видно або не можеш прочитати — став null.
ТІЛЬКИ JSON. Нічого більше.`;
```

### 3. Промт для розпізнавання креслення деталі

```typescript
// src/prompts/recognize-part.ts

export const PART_RECOGNIZE_PROMPT = `Ти отримав креслення машинобудівної деталі.
Витягни з нього параметри і поверни ТІЛЬКИ JSON:

{
  "name": "назва деталі",
  "drawingNo": "номер креслення",
  "material": "матеріал",
  "hardness": "твердість або null",
  "weight": маса_кг_або_null,
  "totalLength": загальна_довжина_мм,
  "surfaceFinish": "загальна шорсткість",
  "steps": [
    {"diameter": 90, "tolerance": "f7", "length": 100, "surface": "Ra 6.3", "hardened": true}
  ],
  "threads": [
    {"designation": "M65x1.5", "length": 25}
  ],
  "grooves": кількість,
  "chamfers": ["1x45°", "2x45°"],
  "tapers": [{"angle": 20}],
  "notes": "технічні вимоги з креслення"
}

ТІЛЬКИ JSON. Не пропускай допуски — вони КРИТИЧНІ (h9, f7, g6 тощо).`;
```

### 4. Промт із контекстом бази знань (НАВЧАННЯ AI)

```typescript
// src/prompts/analyze-with-context.ts

export function buildContextualPrompt(
  project: Project,
  comments: Comment[],
  pastProjects: Project[]  // Затверджені проєкти на цьому верстаті/матеріалі
): string {
  
  let context = '';
  if (pastProjects.length > 0) {
    context = `\n═══════ ДОСВІД З МИНУЛИХ ПРОЄКТІВ (БАЗА ЗНАНЬ) ═══════\n`;
    for (const pp of pastProjects) {
      context += `
Проєкт: ${pp.name}
Деталь: ${pp.part.name}, ${pp.part.material}
Верстат: ${pp.machine.name}
AI рекомендував: ${pp.aiOptTime} хв
Реальний результат: ${pp.realTime} хв
Коментарі технолога: ${pp.testNotes || 'немає'}
Статус: ${pp.status}
---`;
    }
    context += `
ВАЖЛИВО: Враховуй реальні результати минулих проєктів!
Якщо AI рекомендував 73 хв а реально вийшло 82 хв — значить AI був
занадто оптимістичний. Коригуй свої рекомендації відповідно.\n`;
  }

  return buildAnalysisPrompt(project, comments) + context;
}
```

---

## 📋 РЕАЛЬНИЙ ПРИКЛАД ДАНИХ (для тестування)

### Верстат

```json
{
  "name": "PINACHO STH 400-105x3000",
  "type": "Токарний ЧПУ",
  "cncSystem": "Fagor 8055",
  "maxRpm": 4000,
  "power": 26,
  "maxDiameter": 400,
  "rmc": 3000,
  "hasLunet": true,
  "turretSlots": 12,
  "coolantType": "емульсія"
}
```

### Деталь — Шток (Креслення 4690-2.01.2024 B.01)

```json
{
  "name": "Шток",
  "drawingNo": "4690-2.01.2024 B.01",
  "material": "C45E (CK45) EN 10083-2",
  "hardness": null,
  "weight": 37.0,
  "rawDimText": "Загальна довжина 879±0.3 мм. Діаметри: ø90(гартована,f7), ø80h9(-0.07), ø75(-0.1), ø70f7(-0.03/-0.06), ø65h9(-0.07), ø61, ø60.5, ø50. Різьби: M65x1.5, М64x2. Конуси 20° (4 ділянки). Фаски 1x45° та 2x45°. Канавки 4 шт різної конфігурації (Б,В,Г,Д). Шорсткість Ra 6.3 загальна, Ra 12.5 на торцях. Биття 0.06 мм.",
  "dimensions": {
    "totalLength": 879,
    "totalLengthTol": "±0.3",
    "steps": [
      {"diameter": 90, "tolerance": "f7", "length": 0, "surface": "Ra 6.3", "hardened": true, "hardenDepth": 2, "note": "заготовка, гартована зона"},
      {"diameter": 80, "tolerance": "h9 (-0.07)", "length": 250, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 75, "tolerance": "-0.1", "length": 50, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 70, "tolerance": "f7 (-0.03/-0.06)", "length": 150, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 65, "tolerance": "h9 (-0.07)", "length": 200, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 61, "tolerance": "", "length": 30, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 60.5, "tolerance": "", "length": 20, "surface": "Ra 6.3", "hardened": false},
      {"diameter": 50, "tolerance": "", "length": 100, "surface": "Ra 12.5", "hardened": false}
    ],
    "threads": [
      {"designation": "M65x1.5", "length": 25, "position": "ліва сторона"},
      {"designation": "M64x2", "length": 25, "position": "права сторона"}
    ],
    "grooves": [
      {"type": "Б - зовнішня", "width": 4, "depth": 2, "position": "ліва"},
      {"type": "В - зовнішня", "width": 3, "depth": 1.5, "position": "центр"},
      {"type": "Г - зовнішня, R1.8", "width": 5, "depth": 3, "position": "ліва"},
      {"type": "Д - зовнішня", "width": 5, "depth": 2.5, "position": "права"}
    ],
    "chamfers": ["1x45°", "2x45°"],
    "tapers": [
      {"angle": 20, "position": "ліва від ø90"},
      {"angle": 20, "position": "права від ø90"},
      {"angle": 20, "position": "перехід до ø50"},
      {"angle": 20, "position": "торець правий"}
    ]
  }
}
```

### Інструмент (витрати на партію 50 шт)

```json
[
  {"holder": "TCLNR 3232 P12", "insert": "CNMG 120408 KCU10", "maker": "KENNAMETAL", "type": "EXTERNAL", "qty": 8, "note": "чорнове точіння, зняття гартованого шару"},
  {"holder": "TOJNR 3232 P15", "insert": "DNMG 150604 PC", "maker": "TAEGUTEC", "type": "EXTERNAL", "qty": 1, "note": "чистове точіння"},
  {"holder": "SVJCR 2525 M16", "insert": "VCMN 150404 PC", "maker": "TAEGUTEC", "type": "EXTERNAL", "qty": 1, "note": "конуси, фасонні поверхні"},
  {"holder": "TTER 3232-4T25", "insert": "TDXU 4E-04", "maker": "TAEGUTEC", "type": "GROOVING", "qty": 1, "note": "канавки шир. 4мм"},
  {"holder": "TTER 3232-3T20", "insert": "TDT 3E-15-RU", "maker": "TAEGUTEC", "type": "GROOVING", "qty": 1, "note": "канавки шир. 3мм"},
  {"holder": "SER 3232 P16", "insert": "16ER 1.5 ISO", "maker": "TAEGUTEC", "type": "THREADING", "qty": 1, "note": "різьба M65x1.5"},
  {"holder": "SER 3232 P16", "insert": "16ER 2.0 ISO", "maker": "TAEGUTEC", "type": "THREADING", "qty": 1, "note": "різьба M64x2"}
]
```

### Проєкт

```json
{
  "name": "Оптимізація штока ø90 — PINACHO STH 400",
  "status": "DRAFT",
  "currentTime": 100,
  "targetTime": null,
  "batchSize": 50,
  "setupCount": 2,
  "notes": "Заготовка — шток гартований ø90f7, товщина гартування 2мм, довжина 883мм. Потрібно скоротити час зі 100 хв."
}
```

### Очікуваний AI-результат (приклад того, що має генерувати система)

```
## 1. СХЕМА БАЗУВАННЯ

УСТАНОВ А: Патрон 3-кулачковий за ø90 (L=60мм) + задній обертовий центр.
Люнет на ø80 після чорнової обробки. База: торець + ø90.

УСТАНОВ Б: Переворот. Патрон за ø80h9 + задній центр. Люнет на ø65h9.

## 2. МАРШРУТ (УСТАНОВ А — права сторона)

1. Підрізка торця — TCLNR 3232 P12 + CNMG 120408
2. Центрування
3. Чорнове точіння ø90→ø80 (гартований шар 2мм!) — TCLNR + CNMG
4. Чорнове точіння ø80→ø65→ø50 — TCLNR + CNMG
5. Чистове точіння ø80h9, ø70f7, ø65h9 — TOJNR + DNMG 150604
6. Конуси 20° — SVJCR + VCMN 150404
7. Канавки Б, В — TTER 3232-4T25 + TDXU
8. Різьба М64x2 — SER + 16ER 2.0

## 3. РЕЖИМИ РІЗАННЯ

| № | Операція        | D   | n    | S    | V   | t   | L   | Пр | To   |
|---|-----------------|-----|------|------|-----|-----|-----|----|------|
| 1 | Підрізка торця  | 90  | 700  | 0.20 | 198 | 2.0 | 45  | 1  | 0.32 |
| 3 | Чорн. гарт.шар | 90  | 350  | 0.15 | 99  | 2.0 | 700 | 1  | 13.3 |
| 4 | Чорн. сирцевий  | 88  | 800  | 0.35 | 221 | 3-4 | 400 | 3  | 4.3  |
| 5a| Чист. ø80h9     | 80  | 1000 | 0.12 | 251 | 0.3 | 250 | 1  | 2.08 |
| 5b| Чист. ø70f7     | 70  | 1100 | 0.10 | 242 | 0.25| 150 | 1  | 1.36 |

...і так далі...

## 5. ОПТИМІЗАЦІЯ

| Рекомендація                    | Було  | Стало | Економія | Ризик           |
|---------------------------------|-------|-------|----------|-----------------|
| Подача чорн. S=0.25→0.40        | 0.25  | 0.40  | -4 хв    | Перевірити шум  |
| Глибина чорн. t=3→4 мм          | 3     | 4     | -3 хв    | Навантаження    |
| Чист. подача S=0.08→0.15         | 0.08  | 0.15  | -2 хв    | Перевірити Ra   |
| Оптимізація траєкторій           | —     | —     | -5 хв    | Немає           |
| Скорочення вимірювань            | 12    | 6     | -3 хв    | Контроль якості |

## 6. ПІДСУМОК
Поточний: 100 хв → Оптимізований: ~73 хв → Економія: ~27 хв (27%)
```

---

## 🔄 Статуси та переходи

```
DRAFT ──→ IN_PROGRESS ──→ TESTING ──→ TESTED_OK ──→ APPROVED ★
  ↑          ↕                          ↓
  └── AI_REVIEW                    TESTED_FAIL ──→ REWORK ──→ (назад до AI_REVIEW)
```

Дозволені переходи:

| З             | В               | Хто          | Дія                      |
|---------------|-----------------|--------------|--------------------------|
| DRAFT         | AI_REVIEW       | Система      | При запуску AI-аналізу   |
| AI_REVIEW     | IN_PROGRESS     | Система      | Коли AI завершив         |
| IN_PROGRESS   | TESTING         | Технолог     | "На тестування"          |
| TESTING       | TESTED_OK       | Технолог     | "Тест ОК"               |
| TESTING       | TESTED_FAIL     | Технолог     | "Невдача"                |
| TESTED_OK     | APPROVED        | Технолог     | "Затвердити в базу"      |
| TESTED_FAIL   | REWORK          | Технолог     | "На доопрацювання"       |
| REWORK        | AI_REVIEW       | Система      | При перезапуску AI       |

**При переході в APPROVED:**
- Проєкт зберігається в базу знань
- Реальний час (realTime) та коментарі стають доступні для контексту AI в майбутніх проєктах

---

## 📱 UX / Екрани

### 1. Dashboard (Головна)
- 4 stat-карточки: верстати, деталі, інструменти, проєкти
- 3 quick-action кнопки
- Список останніх проєктів
- Банер "Пройдіть навчання" (якщо ще не пройдено)

### 2. Верстати / Деталі
- Список карточок
- Кнопка "+ Додати" → модалка/слайд з:
  - Зона drag-and-drop для файлу → POST на /api/ai/recognize з файлом
  - Форма ручного введення
  - Після AI-розпізнавання — ВСІ поля відображаються для перевірки людиною
  - Кнопка "Зберегти в базу"

### 3. Проєкти — список
- Горизонтальні фільтр-чіпси по статусах
- Карточки проєктів (назва, деталь, верстат, статус, дата, час)
- Кнопка "+ Новий проєкт"

### 4. Новий проєкт — Wizard
- Крок 1: Назва + вибір верстата + вибір деталі (з існуючих в базі)
- Крок 2: Поточний час, партія, примітки
- → Створити (статус DRAFT)

### 5. Проєкт — деталі (ГОЛОВНИЙ ЕКРАН)
- Шапка: назва, статус-бейдж, деталь, верстат, дата
- 3 info-карточки: деталь / верстат / час+партія
- **Блок AI-аналізу:**
  - Кнопка "Запустити AI" / "Перерахувати"
  - Лоадер під час аналізу
  - Результат у форматованому вигляді (markdown → HTML)
- **Блок коментарів:**
  - Список коментарів (дата, автор, текст)
  - Текстова область + кнопка "Надіслати"
  - Після відправки: toast "AI врахує при наступному аналізі"
- **Блок дій:**
  - Кнопки зміни статусу (залежать від поточного)
  - При TESTING → поле для введення реального часу
- **Журнал:** хронологія всіх дій

### 6. База знань
- Статистика: скільки верстатів, деталей, проєктів
- Список затверджених проєктів
- Візуалізація "Як AI навчається" (5 кроків)

### 7. Навчання
- Вступ: що це, для кого, можливості/ризики
- 8 уроків з прогрес-баром
- Кожен урок: кроки + попередження + кнопка "Спробуйте зараз"
- Кнопки навігації між уроками

---

## 🔐 API Endpoints

```
POST   /api/machines              — створити верстат
GET    /api/machines              — список верстатів
GET    /api/machines/:id          — деталі верстата
PUT    /api/machines/:id          — оновити
DELETE /api/machines/:id          — видалити

POST   /api/parts                 — створити деталь
GET    /api/parts                 — список
GET    /api/parts/:id
PUT    /api/parts/:id
DELETE /api/parts/:id

POST   /api/projects              — створити проєкт
GET    /api/projects              — список (query: ?status=TESTING)
GET    /api/projects/:id          — деталі з comments + logs
PUT    /api/projects/:id          — оновити (статус, realTime, etc.)
DELETE /api/projects/:id

POST   /api/projects/:id/comments — додати коментар
POST   /api/projects/:id/analyze  — запустити AI-аналіз

POST   /api/ai/recognize          — розпізнати файл (multipart/form-data)
                                    body: { type: "machine"|"part", file: File }
                                    → повертає JSON з розпізнаними даними

GET    /api/knowledge/stats       — статистика бази знань
GET    /api/knowledge/context     — контекст для AI (минулі проєкти)
```

---

## 🧠 Як працює "навчання" AI (без ML, без fine-tuning)

AI навчається через КОНТЕКСТ промту — це найпростіший і найефективніший спосіб:

```
1. Технолог створює проєкт, AI дає рекомендації
2. Технолог тестує → реальний час = 82 хв (AI казав 73 хв)
3. Технолог пише: "вібрації на ø50 при S>0.12, знизив до 0.08"
4. Проєкт затверджується зі статусом APPROVED
5. Зберігається в БД: aiOptTime=73, realTime=82, testNotes="вібрації..."

Наступний проєкт на цьому верстаті:
6. Система дістає всі APPROVED проєкти для цього верстата/матеріалу
7. Додає їх в промт як контекст (див. analyze-with-context.ts)
8. AI бачить: "минулого разу я рекомендував 73 хв, реально вийшло 82 хв, 
   технолог каже про вібрації на ø50" → AI коригує рекомендації

Ніякого ML, ніякого fine-tuning — просто розумне використання контексту.
```

---

## 📄 Генерація PDF

Для генерації техкарти у PDF використовувати:
- **@react-pdf/renderer** (React-компонент → PDF)
- або **jsPDF** + **jspdf-autotable** (більш гнучкий)

PDF має містити:
- Шапка з логотипом, датою, номером проєкту
- Інформація про деталь / верстат / інструмент
- Маршрут обробки (таблиця)
- Режими різання (таблиця)
- Нормування часу
- Підпис: "Розробив: AI + [ім'я технолога]" / "Затвердив: [ім'я]"

---

## 🚀 MVP — Порядок розробки

### Фаза 1 (1-2 тижні)
- [ ] Налаштування проєкту (Next.js + Prisma + PostgreSQL)
- [ ] Авторизація (JWT)
- [ ] CRUD верстатів, деталей
- [ ] Базовий UI (layout, навігація, форми)

### Фаза 2 (1-2 тижні)
- [ ] AI розпізнавання (upload → Claude API → JSON → форма)
- [ ] Створення проєктів (wizard)
- [ ] AI-аналіз проєкту (промт → Claude API → результат)

### Фаза 3 (1 тиждень)
- [ ] Коментарі технолога
- [ ] Статуси та переходи
- [ ] Перерахунок AI з урахуванням коментарів

### Фаза 4 (1 тиждень)
- [ ] База знань (контекст з минулих проєктів)
- [ ] Навчання (tutorial)
- [ ] Генерація PDF

### Фаза 5 (ongoing)
- [ ] Фільтри проєктів
- [ ] Пошук по базі
- [ ] Адаптив для мобільних
- [ ] Мультикористувацький режим
- [ ] Аналітика (скільки часу зекономлено загалом)

---

## ⚙ ENV змінні

```env
DATABASE_URL=postgresql://user:pass@localhost:5432/cnc_ai
ANTHROPIC_API_KEY=sk-ant-...
S3_BUCKET=cnc-ai-uploads
S3_REGION=eu-central-1
S3_ACCESS_KEY=...
S3_SECRET_KEY=...
JWT_SECRET=...
NEXT_PUBLIC_APP_URL=https://cnc-ai.yourcompany.com
```

---

## Контакти

При питаннях по доменній логіці (токарна обробка, режими різання, типи інструменту) — звертатись до технолога-замовника. AI-промти та структура даних описані вище. Демо-версія додатку (HTML) доступна для ознайомлення з UX-патернами.
