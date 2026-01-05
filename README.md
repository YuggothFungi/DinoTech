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

### 2.4. Цены закупки сырья (RawMaterialPrice)

Цены на сырьё хранятся **во времени**.

```ts
RawMaterialPrice {
  rawMaterialId
  price
  unit        // совпадает с baseUnit
  validFrom
}
```

Это обеспечивает:

* корректный пересчёт себестоимости;
* воспроизводимость исторических расчётов.

---

### 2.5. Технологические карты (TechCard)

**TechCard** описывает способ производства товара типа `manufactured`.

```ts
TechCard {
  id
  productId
  outputQuantity
  outputUnit     // ml | g | pcs
  version
}
```

---

### 2.6. Состав техкарты (TechCardIngredient)

Связь между техкартой и используемым сырьём.

```ts
TechCardIngredient {
  techCardId
  rawMaterialId
  quantity
  unit
  wasteFactor?
}
```

---

### 2.7. Закупка перепродаваемых товаров (PurchaseItem)

Для товаров типа `resale` хранится цена закупки.

```ts
PurchaseItem {
  productId
  price
  unit        // pcs
  validFrom
}
```

---

### 2.8. Расчёт себестоимости (CostCalculation)

Результат расчёта себестоимости товара на конкретный момент времени.

```ts
CostCalculation {
  id
  productId
  calculatedAt
  totalCost
  breakdown      // детализация расчёта (JSON)
  source         // 'ingredients' | 'purchase'
}
```

Логика:

* `manufactured` → расчёт по техкарте и ценам сырья;
* `resale` → себестоимость = цена закупки.

---

### 2.9. Цена продажи (SalePrice)

Цена продажи товара с привязкой к себестоимости и модели ценообразования.

```ts
SalePrice {
  productId
  price
  currency
  pricingModel   // fixed | markup | margin
  baseCostId
  validFrom
}
```

---

## 3. Расчёт себестоимости (инварианты)

* Себестоимость рассчитывается **на дату расчёта**
* Используются цены, действующие на эту дату
* Потери (`wasteFactor`) применяются мультипликативно
* Результат фиксируется и не меняется при изменении цен в будущем

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

### 4.2. Синхронизация данных (офлайн/онлайн)
- **Локальное хранилище**: RxDB с плагинами репликации
- **Стратегия синхронизации**: Оптимистичные обновления + конфликт-разрешение
- **Очередь запросов**: Автоматическое помещение в очередь при офлайн-режиме
- **Фоновая синхронизация**: При появлении соединения через Service Worker

---

## 5. Технологический стек

### 5.1. Backend (API Сервер)
- **Язык**: TypeScript 5.x
- **Фреймворк**: NestJS 10.x
- **База данных**: PostgreSQL 15+ с расширением JSONB
- **ORM**: Prisma 5.x
- **Аутентификация**: JWT через Passport.js
- **Валидация**: class-validator, class-transformer
- **Документация API**: @nestjs/swagger (OpenAPI 3)
- **Контейнеризация**: Docker, Docker Compose

### 5.2. Frontend (PWA Клиент)
- **Фреймворк**: Vue 3 (Composition API) с TypeScript
- **UI-Фреймворк**: Quasar Framework 2.x
- **Управление состоянием**: Pinia 2.x
- **Клиентская база данных**: RxDB 15.x (на базе IndexedDB)
- **HTTP-клиент**: Axios
- **Роутинг**: Vue Router 4
- **Service Worker**: Workbox 7.x (через Quasar PWA plugin)
- **Утилиты**: date-fns (работа с датами), lodash-es (утилиты)

### 5.3. Инфраструктура и инструменты
- **Сборка**: Vite 5.x (через Quasar)
- **Контроль версий**: Git
- **CI/CD**: GitHub Actions / GitLab CI
- **Хостинг (frontend)**: Netlify / Vercel / статический хостинг
- **Хостинг (backend)**: VPS (Ubuntu) с nginx + PM2 или облачный PaaS
- **Мониторинг**: Sentry (опционально)

---

## 6. Синхронизация и офлайн‑режим

* Все изменения фиксируются локально в RxDB
* Изменения помечаются статусом `pending`
* При появлении сети выполняется `push / pull`
* Конфликты разрешаются по стратегии *last‑write‑wins* с уведомлением пользователя

---

## 7. Ключевые архитектурные принципы

* Product — центральная сущность
* TechCard вторична и существует только для производимых товаров
* Сырьё и товары — разные домены
* Все цены версионируются во времени
* Себестоимость и цена продажи не пересчитываются «задним числом»

---

## 8. Масштабирование (заложено в модель)

Модель поддерживает:

* несколько точек продаж;
* разные цены закупки;
* расширение ассортимента;
* аналитику маржи по категориям;
* внедрение комплектов и комбо‑товаров.

## 9. Конфигурационные файлы

### 9.1. Docker Compose (разработка)
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: techcards_dev
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
      DATABASE_URL: "postgresql://devuser:devpass@postgres:5432/techcards_dev"
      JWT_SECRET: "development-secret-change-in-production"
    depends_on:
      - postgres
    volumes:
      - ./backend:/app
      - /app/node_modules

volumes:
  postgres_data:
```

### 9.2. Prisma Schema (пример)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model TechCard {
  id          String    @id @default(uuid())
  name        String
  description String?
  output      Float
  ingredients Json      // JSONB массив ингредиентов
  steps       Json?     // JSONB этапов производства
  isActive    Boolean   @default(true)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  version     Int       @default(1)

  calculations CostCalculation[]
  user         User              @relation(fields: [userId], references: [id])
  userId       String

  @@index([userId])
  @@index([createdAt])
}

model CostCalculation {
  id         String    @id @default(uuid())
  techCard   TechCard  @relation(fields: [techCardId], references: [id])
  techCardId String
  date       DateTime  @default(now())
  costs      Json      // JSONB структура затрат
  totalCost  Float
  isCurrent  Boolean   @default(false)

  @@unique([techCardId, isCurrent])
  @@index([date])
}
```

## 10. Процесс синхронизации

### 10.1. Алгоритм синхронизации
1. **Инициализация**: При запуске PWA проверяется соединение
2. **Локальная работа**: Все изменения пишутся в RxDB с меткой `syncStatus: 'pending'`
3. **Фоновая синхронизация**: При появлении сети Service Worker запускает синхронизацию
4. **Разрешение конфликтов**: При конфликтах используется стратегия "последний пишет" с уведомлением пользователя
5. **Подтверждение**: После успешной синхронизации статус изменяется на `synced`

### 10.2. API эндпоинты для синхронизации
```
POST   /api/sync/push     - Отправка локальных изменений
POST   /api/sync/pull     - Получение изменений с сервера
GET    /api/sync/status   - Статус синхронизации
POST   /api/sync/resolve  - Разрешение конфликтов
```

## 11. Развёртывание

### 11.1. Локальная разработка
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

### 11.2. Продакшен сборка
```bash
# Backend
cd backend
npm run build
docker build -t tech-cards-backend .

# Frontend
cd frontend
quasar build -m pwa
# Результат в dist/pwa
```

## 12. Мониторинг и логирование

### 12.1. Backend
- Логирование через Winston или NestJS встроенный логгер
- Мониторинг здоровья эндпоинтов `/health`
- Метрики производительности

### 12.2. Frontend
- Логирование офлайн-действий в IndexedDB
- Отправка ошибок в Sentry (при наличии сети)
- Мониторинг состояния синхронизации