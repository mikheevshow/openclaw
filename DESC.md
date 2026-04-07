# OpenClaw: Skills и ClawHub — Исследовательский отчёт
 
**Дата:** 7 апреля 2026  
**Репозиторий:** mikheevshow/openclaw  
 
---
 
## Содержание
 
1. [Управление навыками (Skills)](#1-управление-навыками-skills)
   - [Что такое навык](#11-что-такое-навык)
   - [Структура данных](#12-структура-данных)
   - [Источники навыков](#13-источники-навыков)
   - [Пайплайн загрузки](#14-пайплайн-загрузки)
   - [Регистрация в рантайме](#15-регистрация-в-рантайме)
2. [Масштабирование: тысячи навыков](#2-масштабирование-тысячи-навыков)
   - [Жёсткие лимиты](#21-жёсткие-лимиты)
   - [Двухуровневое усечение промпта](#22-двухуровневое-усечение-промпта)
   - [Per-agent фильтры](#23-per-agent-фильтры)
   - [Удалённый поиск ClawHub](#24-удалённый-поиск-clawhub)
   - [Ограничения архитектуры](#25-ограничения-архитектуры)
3. [Интеграция с ClawHub](#3-интеграция-с-clawhub)
   - [Что такое ClawHub](#31-что-такое-clawhub)
   - [API-поверхность](#32-api-поверхность)
   - [Аутентификация](#33-аутентификация)
   - [Установка навыка](#34-установка-навыка)
   - [Обновление навыков](#35-обновление-навыков)
   - [Установка плагинов](#36-установка-плагинов)
   - [Локальное состояние и кэш](#37-локальное-состояние-и-кэш)
   - [Разделение ответственности](#38-разделение-ответственности)
4. [Ключевые файлы](#4-ключевые-файлы)
5. [Переменные окружения](#5-переменные-окружения)
 
---
 
## 1. Управление навыками (Skills)
 
### 1.1 Что такое навык
 
Навык — это **директория** с файлом `SKILL.md`, содержащим YAML-frontmatter и текстовые инструкции для агента. Навыки учат агента пользоваться специализированными инструментами или выполнять определённые задачи.
 
Пример `SKILL.md`:
 
```markdown
---
name: image-lab
description: Generate or edit images via a provider-backed image workflow
metadata: {
  "openclaw": {
    "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"] },
    "primaryEnv": "GEMINI_API_KEY"
  }
}
---
# Инструкции для агента...
```
 
### 1.2 Структура данных
 
**Файл:** `src/agents/skills/types.ts`
 
```typescript
type SkillEntry = {
  skill: Skill;                        // базовые данные навыка
  frontmatter: ParsedSkillFrontmatter; // YAML-метаданные
  metadata?: OpenClawSkillMetadata;    // openclaw-специфичные метаданные
  invocation?: SkillInvocationPolicy;  // правила вызова (user/model)
  exposure?: SkillExposure;            // контроль видимости
};
 
type OpenClawSkillMetadata = {
  always?: boolean;               // всегда включать, минуя фильтры
  emoji?: string;                 // иконка для UI
  os?: ("darwin" | "linux" | "win32")[];  // ограничение по ОС
  requires?: {
    bins?: string[];              // обязательные бинари на PATH
    anyBins?: string[];           // хотя бы один из списка
    env?: string[];               // переменные окружения
    config?: string[];            // конфиг-пути
  };
  primaryEnv?: string;            // основной API-ключ
  install?: InstallSpec;          // спецификация установки (brew/npm/go/uv)
};
 
type SkillExposure = {
  includeInRuntimeRegistry: boolean;
  includeInAvailableSkillsPrompt: boolean;
  userInvocable: boolean;
};
```
 
### 1.3 Источники навыков
 
Навыки загружаются из нескольких источников в порядке **убывания приоритета** (workspace перекрывает bundled):
 
| Приоритет | Источник | Путь |
|-----------|----------|------|
| 1 (высший) | Workspace skills | `<workspace>/skills` |
| 2 | Project agent skills | `<workspace>/.agents/skills` |
| 3 | Personal agent skills | `~/.agents/skills` |
| 4 | Managed skills | `~/.openclaw/skills` |
| 5 | Bundled (поставляются с пакетом) | внутри пакета |
| 6 (низший) | Extra dirs из конфига | `skills.load.extraDirs` |
 
### 1.4 Пайплайн загрузки
 
**Файл:** `src/agents/skills/workspace.ts`
 
```
Сканирование директорий (алфавитный порядок)
        ↓
Эвристика: есть ли паттерн skills/*/SKILL.md?
        ↓
Чтение SKILL.md + парсинг YAML frontmatter
        ↓
Валидация (имя, описание, размер файла)
        ↓
Фильтрация (ОС, бинари, env, конфиги)
        ↓
Дедупликация: Map<name, Skill> с приоритетом по источнику
        ↓
Форматирование в XML для промпта модели
```
 
**Хранилище:** только in-memory `Map`, пересоздаётся при каждой сессии. Никакой персистентной БД или индекса.
 
### 1.5 Регистрация в рантайме
 
- **В промпт модели** — `buildWorkspaceSkillsPrompt()` → XML с `<available_skills>`
- **Как slash-команды** — `buildWorkspaceSkillCommandSpecs()` → регистрация `/skill-name`
- **Синхронизация в Docker sandbox** — `syncSkillsToWorkspace()`
- **Прямой вызов инструмента** — через `command-dispatch: tool` в frontmatter (минует модель)
 
---
 
## 2. Масштабирование: тысячи навыков
 
### 2.1 Жёсткие лимиты
 
**Файл:** `src/agents/skills/workspace.ts:81-105`  
**Конфиг:** `src/config/types.skills.ts`
 
```typescript
const DEFAULT_MAX_CANDIDATES_PER_ROOT      = 300;    // директорий на сканирование
const DEFAULT_MAX_SKILLS_LOADED_PER_SOURCE = 200;    // навыков на источник
const DEFAULT_MAX_SKILLS_IN_PROMPT         = 150;    // навыков в LLM-контекст
const DEFAULT_MAX_SKILLS_PROMPT_CHARS      = 30_000; // символов на промпт
const DEFAULT_MAX_SKILL_FILE_BYTES         = 256_000; // байт на SKILL.md
```
 
Все значения **конфигурируемы** через `skills.limits.*`.
 
### 2.2 Двухуровневое усечение промпта
 
Когда навыки не помещаются в бюджет символов:
 
```
1. Полный формат: name + description + location
          ↓ (если не влезает)
2. Компактный формат: name + location (описания отброшены)
          ↓
3. Бинарный поиск максимального подмножества под бюджет
          ↓
4. Предупреждение: "⚠️ Skills truncated: included N of M (compact format)"
```
 
### 2.3 Per-agent фильтры
 
**Файл:** `src/agents/skills/agent-filter.ts`
 
```typescript
resolveEffectiveAgentSkillFilter(cfg, agentId): string[] | undefined
```
 
Через `agents.list[].skills` разным агентам назначаются разные наборы навыков — позволяет не грузить весь каталог для каждого агента.
 
### 2.4 Удалённый поиск ClawHub
 
Для тысяч навыков в удалённом каталоге — серверный полнотекстовый/векторный поиск с ранжированием:
 
```typescript
searchClawHubSkills({ query: "image generation", limit: 10 })
// GET /api/v1/search?q=image+generation&limit=10
// → [{ score: 0.92, slug: "image-lab", displayName: "...", summary: "..." }]
```
 
### 2.5 Ограничения архитектуры
 
| Аспект | Текущее решение | Ограничение |
|--------|----------------|-------------|
| Локальное хранилище | Сканирование директорий O(n) | Нет индекса/кэша |
| Поиск локально | Точное совпадение по имени | Нет семантического поиска |
| Загрузка | Вся сессия — один раз при старте | Нет ленивой загрузки |
| Масштаб | Сотни навыков локально | Тысячи — только через ClawHub |
| Эмбеддинги | Отсутствуют локально | Только на стороне сервера ClawHub |
 
---
 
## 3. Интеграция с ClawHub
 
### 3.1 Что такое ClawHub
 
**ClawHub** (`https://clawhub.ai`) — публичный реестр и маркетплейс навыков и плагинов для OpenClaw.
 
**Возможности:**
- Версионированное хранилище с semver и changelog
- Векторный поиск по навыкам и пакетам
- Stars, комментарии, community-фидбек
- Система модерации и репортов
- Три канала публикации: `official`, `community`, `private`
- Три типа пакетов: `skill`, `code-plugin`, `bundle-plugin`
 
### 3.2 API-поверхность
 
**Файл:** `src/infra/clawhub.ts`  
**Base URL:** `https://clawhub.ai` (конфигурируется)  
**Таймаут:** 30 секунд
 
#### Навыки
 
| Метод | Эндпоинт | Назначение |
|-------|----------|------------|
| GET | `/api/v1/search?q=...&limit=N` | Поиск навыков (векторный) |
| GET | `/api/v1/skills` | Список всех навыков с пагинацией |
| GET | `/api/v1/skills/{slug}` | Детали навыка + версии |
| GET | `/api/v1/download?slug=...&version=...` | Скачать ZIP-архив навыка |
 
#### Пакеты (плагины)
 
| Метод | Эндпоинт | Назначение |
|-------|----------|------------|
| GET | `/api/v1/packages/{name}` | Детали пакета (метаданные, верификация) |
| GET | `/api/v1/packages/{name}/versions/{version}` | Версия с compatibility-инфо |
| GET | `/api/v1/packages/search?q=...&family=...` | Поиск пакетов по семейству |
| GET | `/api/v1/packages/{name}/download` | Скачать ZIP-архив пакета |
 
### 3.3 Аутентификация
 
**Файл:** `src/infra/clawhub.ts:210-272`
 
Токен разрешается в порядке приоритета:
 
```
1. OPENCLAW_CLAWHUB_TOKEN
2. CLAWHUB_TOKEN
3. CLAWHUB_AUTH_TOKEN
4. Конфиг-файл (macOS): ~/Library/Application Support/clawhub/config.json
5. Конфиг-файл (Linux): $XDG_CONFIG_HOME/clawhub/config.json
```
 
В конфиг-файле рекурсивно ищутся поля: `accessToken`, `authToken`, `apiToken`, `token` внутри веток `auth.*`, `session.*`, `credentials.*`, `user.*`.
 
Токен передаётся как `Authorization: Bearer {token}`.
 
### 3.4 Установка навыка
 
**Команда:** `openclaw skills install <slug> [--version <ver>] [--force]`  
**Файл:** `src/agents/skills-clawhub.ts:400-409`
 
```
1. Валидация slug (только буквы/цифры/дефис)
         ↓
2. GET /api/v1/skills/{slug} → определение версии
         ↓
3. GET /api/v1/download?slug=...&version=... → ZIP + SHA256
         ↓
4. Распаковка во временную директорию, проверка SKILL.md
         ↓
5. Копирование в {workspace}/skills/{slug}
         ↓
6. Запись {skillDir}/.clawhub/origin.json
         ↓
7. Обновление {workspace}/.clawhub/lock.json
```
 
**Формат `origin.json`:**
 
```json
{
  "version": 1,
  "registry": "https://clawhub.ai",
  "slug": "image-lab",
  "installedVersion": "1.2.3",
  "installedAt": 1712500000000
}
```
 
### 3.5 Обновление навыков
 
**Команда:** `openclaw skills update [<slug> | --all]`  
**Файл:** `src/agents/skills-clawhub.ts:411-463`
 
```
1. Читает {workspace}/.clawhub/lock.json → список отслеживаемых навыков
         ↓
2. Для каждого — читает origin.json (slug, registry, installedVersion)
         ↓
3. Переустанавливает с force: true (тот же пайплайн, что и install)
         ↓
4. Сравнивает старую/новую версию → возвращает { changed: boolean }
```
 
### 3.6 Установка плагинов
 
**Команда:** `openclaw plugins install clawhub:<name>[@version]`  
**Файл:** `src/plugins/clawhub.ts`
 
При установке проверяется совместимость:
 
| Проверка | Описание |
|----------|----------|
| `pluginApiRange` | Версия Plugin API совместима |
| `minGatewayVersion` | Версия gateway не ниже требуемой |
| Канал `private` | Блокируется без auth-токена |
| Канал `community` | Предупреждение пользователю |
| `family: "skill"` | Редирект на `openclaw skills install` |
 
**Метаданные, записываемые при установке:**
 
```typescript
clawhub: {
  source: "clawhub",
  clawhubUrl: "https://clawhub.ai",
  clawhubPackage: "my-plugin",
  clawhubFamily: "code-plugin",
  clawhubChannel: "official",
  version: "1.2.3",
  integrity: "sha256-...",
  resolvedAt: "2026-04-07T...",
  installedAt: "2026-04-07T..."
}
```
 
**Коды ошибок:**
 
| Код | Описание |
|-----|----------|
| `invalid_spec` | Неверный формат `clawhub:...` |
| `package_not_found` | 404 от ClawHub |
| `version_not_found` | Запрошенная версия не существует |
| `skill_package` | Попытка установить навык как плагин |
| `private_package` | Приватный пакет без токена |
| `incompatible_plugin_api` | Несовместимая версия Plugin API |
| `incompatible_gateway` | Gateway слишком старый |
 
### 3.7 Локальное состояние и кэш
 
| Файл | Назначение |
|------|------------|
| `{workspace}/.clawhub/lock.json` | Lockfile: все установленные навыки + версии |
| `{skillDir}/.clawhub/origin.json` | Источник навыка для обновлений |
 
**Нет долгосрочного кэша:** ZIP-архивы удаляются сразу после распаковки.  
**Нет кэша поиска:** каждый запрос уходит на сервер.  
**Обратная совместимость:** поддерживается устаревший путь `.clawdhub` (старое опечаточное написание).
 
### 3.8 Разделение ответственности
 
| Задача | Инструмент |
|--------|------------|
| Поиск навыков в реестре | `openclaw skills search <query>` |
| Установка навыка | `openclaw skills install <slug>` |
| Обновление навыков | `openclaw skills update [--all]` |
| Установка плагина | `openclaw plugins install clawhub:<name>` |
| **Публикация навыков** | Отдельный `clawhub` CLI (`npm i -g clawhub`) |
| **Аутентификация для публикации** | `clawhub login` |
 
> **Важно:** Openclaw умеет только **потреблять** навыки из ClawHub. Публикация полностью делегирована отдельному инструменту `clawhub`.
 
---
 
## 4. Ключевые файлы
 
| Файл | Назначение |
|------|------------|
| `src/agents/skills/workspace.ts` | Основная загрузка, фильтрация, сборка промпта |
| `src/agents/skills/types.ts` | Типы: SkillEntry, OpenClawSkillMetadata, SkillExposure |
| `src/agents/skills/local-loader.ts` | Сканирование директорий и парсинг SKILL.md |
| `src/agents/skills/agent-filter.ts` | Per-agent allowlist-фильтры |
| `src/agents/skills/command-specs.ts` | Регистрация навыков как slash-команд |
| `src/agents/skills/skill-contract.ts` | Структура данных + XML-форматирование для промпта |
| `src/agents/skills-clawhub.ts` | Установка/обновление навыков из ClawHub |
| `src/infra/clawhub.ts` | HTTP-клиент ClawHub API |
| `src/plugins/clawhub.ts` | Установка плагинов из ClawHub |
| `src/config/types.skills.ts` | Схема конфига: лимиты, load, entries |
| `src/cli/skills-cli.ts` | CLI-команды: search, install, update |
| `src/gateway/server-methods/skills.ts` | Gateway-методы: skills.search/install/update |
| `docs/tools/skills.md` | Пользовательская документация по навыкам |
| `docs/tools/clawhub.md` | Пользовательская документация по ClawHub |
 
---
 
## 5. Переменные окружения
 
| Переменная | Назначение | Приоритет |
|------------|------------|-----------|
| `OPENCLAW_CLAWHUB_URL` | URL реестра | 1 (высший) |
| `CLAWHUB_URL` | URL реестра | 2 |
| `OPENCLAW_CLAWHUB_TOKEN` | Auth-токен | 1 (высший) |
| `CLAWHUB_TOKEN` | Auth-токен | 2 |
| `CLAWHUB_AUTH_TOKEN` | Auth-токен | 3 |
| `OPENCLAW_CLAWHUB_CONFIG_PATH` | Путь к конфиг-файлу | 1 (высший) |
| `CLAWHUB_CONFIG_PATH` | Путь к конфиг-файлу | 2 |
| `CLAWDHUB_CONFIG_PATH` | Путь к конфиг-файлу (legacy) | 3 |
| `XDG_CONFIG_HOME` | Базовая директория конфига Linux | — |
 
---
 
*Отчёт сформирован на основе анализа исходного кода репозитория mikheevshow/openclaw.*
