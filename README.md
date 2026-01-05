# DinoTech

Good dinosaurs code, great dinosaurs vibecode.

# Архитектура PWA‑приложения для управления товарами, техкартами и себестоимостью

## 1. Общее описание

**Цель**: создание прогрессивного веб‑приложения (PWA) для управления ассортиментом товаров кофейни, технологическими картами, закупочными ценами и расчётом себестоимости и цен продажи с поддержкой офлайн‑режима и синхронизации.

Система ориентирована на **управленческий и операционный учёт**: корректный пересчёт себестоимости при изменении цен сырья и закупаемых товаров, хранение истории расчётов и прозрачное ценообразование.

**Ключевые требования**:

* Полноценная работа без интернет‑соединения (чтение, создание, редактирование данных)
* Установка на домашний экран мобильных устройств (Android / iOS)
* Синхронизация данных при восстановлении соединения
* Адаптивный интерфейс (mobile‑first)
* Работа с иерархией товаров и категорий
* Поддержка разных типов товаров: производимые и перепродаваемые

---

## 2. Доменная модель (ключевые понятия)

### 2.1. Товары (Product)

**Product** — центральная сущность системы. Любой объект, который продаётся клиенту.

Товары делятся на два типа:

1. **Производимые товары (`manufactured`)**
   Изготавливаются из сырья по технологической карте (кофе, напитки, выпечка).

2. **Перепродаваемые товары (`resale`)**
   Закупаются и продаются без переработки (батончики, готовые сандвичи).

```ts
Product {
  id
  name
  sku?
  type: 'manufactured' | 'resale'
  categoryId
  isActive
}
```

---

### 2.2. Категории товаров (Category)

Товары могут быть сгруппированы в иерархические категории:

* Напитки
  * Кофе
    * Классический
    * Авторский
  * Чай
* Снеки
  * Выпечка
  * Батончики

```ts
Category {
  id
  name
  parentId?
}
```

Категории используются для:

* навигации и UI;
* аналитики;
* применения ценовых правил.

---

### 2.3. Сырьё и ингредиенты (RawMaterial)

**RawMaterial** — закупаемое сырьё, используемое для производства товаров.

Важно: сырьё **не равно товару**. Батончик — это товар, но не сырьё.

```ts
RawMaterial {
  id
  name
  baseUnit    // g | ml | pcs
}
```

---

### 2.4. Цены (PriceRecord) — обобщённая сущность

Цены на сырьё, закупочные цены на перепродаваемые товары и цены продажи хранятся в **единой, версионированной во времени** структуре.

```ts
PriceRecord {
  id
  entityType: 'RAW_MATERIAL' | 'RESALE_PRODUCT' | 'SALE_PRODUCT'
  entityId     // Ссылается на RawMaterial.id или Product.id
  price
  currency
  unit        // g, ml, pcs — для RAW_MATERIAL и RESALE_PRODUCT
  validFrom   // Дата начала действия цены
  metadata    // Доп. данные (например, pricingModel для SALE_PRODUCT)
}
```

**Как это работает**:
* **Сырьё (`RAW_MATERIAL`)**: `entityId` ссылается на `RawMaterial`, `unit` совпадает с его `baseUnit`.
* **Закупка товара (`RESALE_PRODUCT`)**: `entityId` ссылается на `Product` типа `resale`, `unit` почти всегда `pcs`.
* **Цена продажи (`SALE_PRODUCT`)**: `entityId` ссылается на `Product`, `metadata` содержит модель ценообразования (например, `{ pricingModel: 'markup', markupPercent: 100 }`).

**Преимущества**:
* Унифицированная логика поиска актуальной цены на любую дату.
* Упрощение кода и схемы базы данных.
* Лёгкое добавление новых типов цен в будущем.

---

### 2.5. Технологические карты (TechCard)

**TechCard** описывает способ производства товара типа `manufactured`. Существует в строгой связи 1:1 с таким товаром.

```ts
TechCard {
  id
  productId   // Уникальная ссылка на Product (type: 'manufactured')
  outputQuantity
  outputUnit     // ml | g | pcs
  version        // Для отслеживания изменений рецептуры
  ingredients    // Связь имеет_many с TechCardIngredient
}
```

---

### 2.6. Состав техкарты (TechCardIngredient)

Связь между техкартой и используемым сырьём. Хранит нормы расхода.

```ts
TechCardIngredient {
  id
  techCardId
  rawMaterialId
  quantity      // Количество на одну единицу выходного продукта
  unit          // Должен быть конвертируем в baseUnit сырья
  wasteFactor?  // Коэффициент потерь/усушки, например 1.1 (10% потерь)
}
```

---

### 2.7. Расчёт себестоимости (CostCalculation)

Результат расчёта себестоимости товара на конкретный момент времени. Всегда фиксируется (не меняется при изменении цен в будущем).

```ts
CostCalculation {
  id
  productId
  calculatedAt   // Момент времени, на который сделан расчёт
  totalCost
  breakdown      // JSON: детализация по ингредиентам или ссылка на закупку
  source         // 'TECH_CARD' | 'PURCHASE_PRICE'
}
```

**Логика формирования**:
* Для `manufactured` товаров (`source: 'TECH_CARD'`): `breakdown` содержит массив расчётов по каждому ингредиенту техкарты, с указанием `rawMaterialId`, `quantity`, `usedPrice` (цена на дату `calculatedAt`).
* Для `resale` товаров (`source: 'PURCHASE_PRICE'`): `breakdown` содержит ссылку на `PriceRecord.id` (тип `RESALE_PRODUCT`), который использовался как себестоимость.

---

### 2.8. Ключевые сценарии использования

#### Сценарий 1: Ввод нового сезонного напитка
1.  **Создание товара**: В системе создаётся `Product` с типом `manufactured` (например, "Тыквенный латте").
2.  **Создание техкарты**: Создаётся `TechCard`, привязанная к товару. В неё добавляются `TechCardIngredient` (кофе, молоко, сироп тыквенный).
3.  **Расчёт себестоимости**: Система по запросу создаёт `CostCalculation`:
    *   Для каждого ингредиента ищется актуальная на текущую дату цена (`PriceRecord` с `entityType: 'RAW_MATERIAL'`).
    *   Рассчитывается стоимость с учётом `wasteFactor`.
    *   Результат фиксируется.
4.  **Установка цены продажи**: Менеджер создаёт `PriceRecord` с `entityType: 'SALE_PRODUCT'`, указывая модель наценки (`markup`) или фиксированную цену.

#### Сценарий 2: Подорожала ваниль
1.  **Ввод новой цены**: Бухгалтер создаёт новый `PriceRecord` для `RawMaterial` "Сироп ванильный" с `validFrom = сегодня`.
2.  **Анализ влияния**: Система может выполнить **проверочный расчёт** для всех `TechCard`, где используется этот сироп, и показать, как изменится их себестоимость.
3.  **Массовый пересчёт**: По решению пользователя для всех зависимых `manufactured` товаров создаются новые `CostCalculation` с актуальной датой.
4.  **Переоценка**: На основе новых себестоимостей можно скорректировать `PriceRecord` типа `SALE_PRODUCT`.

#### Сценарий 3: Работа в цеху без сети (офлайн)
1.  **Локальные действия**: Технолог открывает PWA на планшете, видит все данные из локальной БД (RxDB). Может создавать новые `Product` и `TechCard`. Изменения помечаются статусом `pending`.
2.  **Списание сырья**: При производстве партии фиксируется списание. Данные также сохраняются локально.
3.  **Восстановление связи**: При подключении к Wi-Fi Service Worker инициирует синхронизацию.
4.  **Синхронизация**: Все локальные изменения (`pending`) отправляются на сервер (`push`), затем загружаются изменения с сервера (`pull`). Пользователь получает уведомление об успехе или запрос на разрешение конфликтов.

---

## 3. Алгоритм расчёта себестоимости (детализация)

### 3.1. Для производимых товаров (`manufactured`)

**Входные данные**: `Product.id`, `calculationDate` (по умолчанию — текущая дата).

**Шаги**:
1.  Найти активную `TechCard`, связанную с товаром.
2.  Для каждого `TechCardIngredient` в составе техкарты:
    *   Найти `RawMaterial` по `rawMaterialId`.
    *   Найти актуальную **цену сырья**: `PriceRecord` где `entityType = 'RAW_MATERIAL'`, `entityId = rawMaterial.id`, и `validFrom` — максимальная дата, не превышающая `calculationDate`.
    *   Привести `quantity` к `baseUnit` сырья, если единицы измерения (`unit`) различаются.
    *   **Стоимость ингредиента** = `quantity * price * (wasteFactor || 1.0)`.
3.  **Итоговая себестоимость** (`totalCost`) = Сумма стоимостей всех ингредиентов.
4.  Создать запись `CostCalculation`:
    *   `productId` = ID товара.
    *   `source` = `'TECH_CARD'`.
    *   `breakdown` = JSON-массив с деталями по каждому ингредиенту (ID сырья, цена, итоговая стоимость).
    *   `calculatedAt` = `calculationDate`.

### 3.2. Для перепродаваемых товаров (`resale`)

**Шаги**:
1.  Найти актуальную **закупочную цену**: `PriceRecord` где `entityType = 'RESALE_PRODUCT'`, `entityId = product.id`, и `validFrom` — максимальная дата, не превышающая `calculationDate`.
2.  **Себестоимость** (`totalCost`) = `price` из записи.
3.  Создать запись `CostCalculation`:
    *   `source` = `'PURCHASE_PRICE'`.
    *   `breakdown` = `{ purchasePriceRecordId: priceRecord.id }`.

**Инвариант**: Рассчитанная себестоимость фиксируется и не меняется при изменении цен в будущем. Это гарантирует воспроизводимость финансовой отчётности.

---

## 4. Архитектура приложения

### 4.1. Общая схема

```
┌─────────────────┐    HTTP / Sync API    ┌─────────────────┐
│   PWA Клиент    │◄────────────────────►│   API Сервер    │
│  (Vue + Quasar) │                      │   (NestJS)      │
│                 │                      │                 │
│  • RxDB         │                      │  • PostgreSQL   │
│  • IndexedDB    │                      │  • Business     │
│  • Offline UI   │                      │    Logic        │
└─────────────────┘                      └─────────────────┘
```

---

## 5. Синхронизация и офлайн‑режим (детали реализации)

### 5.1. Модель данных на клиенте (RxDB)

Каждая локальная запись в RxDB дополняется служебными полями для синхронизации:

```ts
LocalRecord {
  // Стандартные поля сущности (id, name, price...)
  _id: string;           // Локальный уникальный ID (например, UUID)
  serverId?: string;     // ID, присвоенный на сервере после синхронизации
  syncStatus: 'synced' | 'pending' | 'error';
  lastModified: number;  // Локальная метка времени изменения
  _deleted?: boolean;    // Флаг мягкого удаления для синхронизации
}
```

### 5.2. Алгоритм синхронизации

1.  **Инициализация**: При запуске PWA проверяется соединение. Загружается `serverId` для всех локально созданных записей.
2.  **Локальная работа**: Все изменения создают/обновляют записи со статусом `pending`.
3.  **Фоновая синхронизация** (при появлении сети):
    *   **Push**: Отправляются на сервер все записи со статусом `pending`. Для каждой успешной операции локальная запись обновляется: `syncStatus = 'synced'`, `serverId` заполняется.
    *   **Pull**: Клиент запрашивает изменения с сервера с момента последней успешной синхронизации. Полученные записи сохраняются или обновляются в локальной БД.
4.  **Разрешение конфликтов**: При обнаружении конфликта (одна и та же запись изменена и на сервере, и офлайн) применяется стратегия **"last-write-wins"** на основе метки времени `lastModified`. Пользователь получает уведомление о конфликте для ключевых сущностей (опционально).
5.  **Подтверждение**: UI обновляется, отображая актуальные данные. Статус-индикатор показывает, что синхронизация завершена.

### 5.3. API эндпоинты для синхронизации

```
POST   /api/sync/push     - Приём пачки изменений { updates: [...], deleted: [...] }
POST   /api/sync/pull     - Отдача изменений с меткой времени { since: timestamp }
GET    /api/sync/status   - Статус и время последней синхронизации
```

---

## 6. Технологический стек

### 6.1. Backend (API Сервер)
- **Язык**: TypeScript 5.x
- **Фреймворк**: NestJS 10.x
- **База данных**: PostgreSQL 15+ с расширением JSONB
- **ORM**: Prisma 5.x
- **Аутентификация**: JWT через Passport.js
- **Валидация**: class-validator, class-transformer
- **Документация API**: @nestjs/swagger (OpenAPI 3)
- **Контейнеризация**: Docker, Docker Compose

### 6.2. Frontend (PWA Клиент)
- **Фреймворк**: Vue 3 (Composition API) с TypeScript
- **UI-Фреймворк**: Quasar Framework 2.x
- **Управление состоянием**: Pinia 2.x
- **Клиентская база данных**: RxDB 15.x (на базе IndexedDB)
- **HTTP-клиент**: Axios
- **Роутинг**: Vue Router 4
- **Service Worker**: Workbox 7.x (через Quasar PWA plugin)
- **Утилиты**: date-fns (работа с датами), lodash-es (утилиты)

### 6.3. Инфраструктура и инструменты
- **Сборка**: Vite 5.x (через Quasar)
- **Контроль версий**: Git
- **CI/CD**: GitHub Actions / GitLab CI
- **Хостинг (frontend)**: Netlify / Vercel / статический хостинг
- **Хостинг (backend)**: VPS (Ubuntu) с nginx + PM2 или облачный PaaS
- **Мониторинг**: Sentry (опционально)

---

## 7. Ключевые архитектурные принципы

* **Product — центральная сущность**
* **TechCard вторична** и существует только для производимых товаров (связь 1:1)
* **Сырьё и товары — разные домены**
* **Все цены версионируются во времени** через единую сущность `PriceRecord`
* **Себестоимость и цена продажи не пересчитываются «задним числом»** — новые расчёты создаются с новой датой

---

## 8. Масштабирование (заложено в модель)

Модель поддерживает:

* Несколько точек продаж (через добавление поля `locationId` к ключевым сущностям).
* Сложные модели ценообразования (через `metadata` в `PriceRecord`).
* Расширение ассортимента (иерархия категорий).
* Аналитику маржи по категориям, товарам, периодам.
* Внедрение комплектов и комбо‑товаров (как новый тип `Product` с виртуальной `TechCard`).

---

## 9. Конфигурационные файлы

### 9.1. Docker Compose (разработка)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: dinotech_dev
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  backend:
    build: ./backend
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: "postgresql://devuser:devpass@postgres:5432/dinotech_dev"
      JWT_SECRET: "development-secret-change-in-production"
    depends_on:
      - postgres
    volumes:
      - ./backend:/app
      - /app/node_modules

volumes:
  postgres_data:
```

### 9.2. Prisma Schema (основные сущности)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Product {
  id          String   @id @default(uuid())
  name        String
  sku         String?  @unique
  type        ProductType // MANUFACTURED, RESALE
  isActive    Boolean  @default(true)

  categoryId  String?
  category    Category? @relation(fields: [categoryId], references: [id])

  // Связи
  techCard    TechCard?                // Только для type=MANUFACTURED
  costCalculations CostCalculation[]
  priceRecords     PriceRecord[]       // Цены продажи (SALE_PRODUCT)

  @@index([categoryId])
  @@index([type])
}

model Category {
  id       String   @id @default(uuid())
  name     String
  parentId String?
  parent   Category? @relation("CategoryToCategory", fields: [parentId], references: [id])
  children Category[] @relation("CategoryToCategory")
  products Product[]
}

model RawMaterial {
  id        String   @id @default(uuid())
  name      String
  baseUnit  UnitType // G, ML, PCS

  // Связи
  ingredients TechCardIngredient[]
  priceRecords PriceRecord[]        // Цены закупки (RAW_MATERIAL)
}

model TechCard {
  id             String   @id @default(uuid())
  version        Int      @default(1)
  outputQuantity Float
  outputUnit     UnitType

  // Связь 1:1 с производимым товаром
  productId      String   @unique
  product        Product  @relation(fields: [productId], references: [id])

  ingredients    TechCardIngredient[]

  @@index([productId])
}

model TechCardIngredient {
  id            String      @id @default(uuid())
  quantity      Float
  unit          UnitType
  wasteFactor   Float       @default(1.0)

  techCardId    String
  techCard      TechCard    @relation(fields: [techCardId], references: [id], onDelete: Cascade)

  rawMaterialId String
  rawMaterial   RawMaterial @relation(fields: [rawMaterialId], references: [id])

  @@index([techCardId])
  @@index([rawMaterialId])
}

model PriceRecord {
  id         String      @id @default(uuid())
  entityType EntityType  // RAW_MATERIAL, RESALE_PRODUCT, SALE_PRODUCT
  entityId   String      // ID сырья или товара
  price      Float
  currency   String      @default("RUB")
  unit       UnitType?
  validFrom  DateTime
  metadata   Json?       // Для SALE_PRODUCT: { pricingModel, markupPercent, ... }

  // Связи (необязательно, для валидации)
  product      Product?      @relation(fields: [entityId], references: [id], onDelete: Cascade)
  rawMaterial  RawMaterial?  @relation(fields: [entityId], references: [id], onDelete: Cascade)

  @@index([entityType, entityId, validFrom])
}

model CostCalculation {
  id           String    @id @default(uuid())
  productId    String
  product      Product   @relation(fields: [productId], references: [id])
  calculatedAt DateTime  @default(now())
  totalCost    Float
  source       CalculationSource // TECH_CARD, PURCHASE_PRICE
  breakdown    Json      // Детализация расчёта

  @@index([productId])
  @@index([calculatedAt])
}

// Типы
enum ProductType {
  MANUFACTURED
  RESALE
}

enum UnitType {
  G
  ML
  PCS
}

enum EntityType {
  RAW_MATERIAL
  RESALE_PRODUCT
  SALE_PRODUCT
}

enum CalculationSource {
  TECH_CARD
  PURCHASE_PRICE
}
```

---

## 10. Развёртывание

### 10.1. Локальная разработка

```bash
# Запуск базы данных
docker-compose up -d postgres

# Инициализация backend
cd backend
npm install
npx prisma migrate dev
npm run start:dev

# Инициализация frontend
cd frontend
npm install
quasar dev -m pwa
```

### 10.2. Продакшен сборка

```bash
# Backend
cd backend
npm run build
docker build -t dinotech-backend .

# Frontend
cd frontend
quasar build -m pwa
# Результат в dist/pwa
```

---

## 11. Мониторинг и логирование

### 11.1. Backend
- Логирование через Winston или NestJS встроенный логгер
- Мониторинг здоровья эндпоинтов `/health`
- Метрики производительности (количество расчётов, время ответа API)

### 11.2. Frontend
- Логирование офлайн-действий в IndexedDB
- Отправка ошибок в Sentry (при наличии сети)
- Мониторинг состояния синхронизации и размера локальной БД

---