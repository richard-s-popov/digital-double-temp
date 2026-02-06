# Life Story Interview — Алгоритм и флоу

## Обзор

Life Story Interview — это адаптивное интервью о жизненной истории участника. Система задаёт вопросы из заранее подготовленного скрипта (~100+ вопросов), но адаптирует их в реальном времени на основе ответов пользователя. LLM решает, когда задать уточняющий вопрос, когда перейти к следующей теме, и какие вопросы можно пропустить.

## Архитектура

```
Фронтенд (/interview)  ←→  API (/api/chat)  ←→  ChatService  ←→  LLM (Anthropic / Together AI)
                                                       ↕
                                                  SessionStorage (PostgreSQL)
```

### Основные файлы

| Компонент | Файл |
|-----------|------|
| Сервис | `backend/src/chat/chat.service.ts` |
| Контроллер | `backend/src/chat/chat.controller.ts` |
| LLM-сервис | `backend/src/chat/services/llm.service.ts` |
| Хранение сессий | `backend/src/chat/services/session-storage.service.ts` |
| Экспертные рефлексии | `backend/src/chat/services/expert-reflections.service.ts` |
| Скрипт интервью | `backend/src/chat/data/interview-script.json` |
| Типы | `backend/src/chat/types/interview.types.ts` |
| DTO | `backend/src/chat/dto/chat.dto.ts` |
| Фронтенд | `frontend/app/interview/page.tsx` |

### API-эндпоинты

| Метод | Маршрут | Описание |
|-------|---------|----------|
| POST | `/api/chat/start` | Начать новое интервью |
| POST | `/api/chat` | Отправить ответ |
| GET | `/api/chat/my-session` | Активная сессия текущего юзера |
| GET | `/api/chat/session/:id/resume` | Возобновить сессию |
| GET | `/api/chat/session/:id` | Состояние сессии |
| GET | `/api/chat/session/:id/transcript` | Полный транскрипт |
| GET | `/api/chat/session/:id/reflections` | Экспертные рефлексии |

---

## Этап 1: Старт интервью

**Эндпоинт:** `POST /api/chat/start { participant_name }`

```
┌─────────────────────────────────────────────────┐
│  1. Архивировать старые сессии юзера             │
│  2. Создать новую сессию (UUID, status=INTRO)    │
│  3. Загрузить скрипт (interview-script.json)     │
│     └── 100+ вопросов про жизнь                  │
│  4. Сохранить snapshot скрипта в БД              │
│  5. Сформировать intro-текст (подставить имя)    │
│  6. Записать intro в transcript                  │
│                                                   │
│  7. ── LLM вызов #1: personalizeQuestion ──      │
│     │  Вход: первый вопрос + пустые заметки      │
│     │  Выход: тот же вопрос (заметок ещё нет)    │
│                                                   │
│  8. Записать первый вопрос в transcript          │
│  9. status → WAITING_ANSWER                      │
│                                                   │
│  Ответ: { intro, message (вопрос), progress 1/N }│
└─────────────────────────────────────────────────┘
```

### Детали

- Старые сессии юзера архивируются, чтобы у каждого юзера была только одна активная сессия.
- Snapshot скрипта фиксирует версию вопросов на момент старта — даже если `interview-script.json` обновится, сессия продолжит работать со своей версией.
- Первый вопрос проходит через `personalizeQuestion`, но т.к. заметок ещё нет, возвращается без изменений.

---

## Этап 2: Обработка ответа пользователя (главный цикл)

**Эндпоинт:** `POST /api/chat { session_id, answer }`

Это ядро системы — каждый ответ пользователя проходит через цепочку из нескольких LLM-вызовов.

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ШАГ 1: Сохранить ответ в transcript                    │
│                                                          │
│  ШАГ 2: Подготовить контекст                            │
│     • transcriptTail (последние ~5000 символов)          │
│     • currentQuestion (текущий вопрос из скрипта)        │
│     • timeLeft (сколько секунд осталось в блоке)         │
│                                                          │
│  ШАГ 3: ── LLM вызов #1: updateReflectionNotes ──       │
│     │  Вход: старые заметки + ответ юзера                │
│     │  Выход: обновлённые key-value заметки              │
│     │  Пример: {"place_of_birth": "Moscow",             │
│     │           "has_children": "2 kids",                │
│     │           "current_mood": "reflective"}            │
│     └── Сохранить в БД                                   │
│                                                          │
│  ШАГ 4: ── LLM вызов #2: planNextAction ──              │
│     │  Вход: transcriptTail, заметки, текущий вопрос,    │
│     │        timeLeft, кол-во follow-up'ов                │
│     │  LLM анализирует:                                  │
│     │    • Достигнута ли цель вопроса?                   │
│     │    • Есть ли эмоциональный контент?                │
│     │    • Хочет ли юзер двигаться дальше?               │
│     │  Выход: { action, next_utterance, emotional,       │
│     │           transition, assessment }                  │
│     └──────────────────────────────────────────          │
│                                                          │
│  ШАГ 5: Решение ─────────────────────────────────────── │
│     │                                                    │
│     ├── A) FOLLOW_UP (и timeLeft > 30 сек)               │
│     │   ├── Вернуть follow-up вопрос                     │
│     │   ├── followups_count++                             │
│     │   ├── question_index НЕ меняется                   │
│     │   └── status → WAITING_ANSWER                      │
│     │                                                    │
│     └── B) NEXT_QUESTION (или timeLeft < 30)             │
│         │                                                │
│         ├── Если emotional_content → сгенерировать       │
│         │   эмпатический переход (LLM вызов #3)          │
│         │                                                │
│         ├── Цикл поиска следующего вопроса:              │
│         │   ┌────────────────────────────────────────┐   │
│         │   │ для каждого вопроса[i] в скрипте:      │   │
│         │   │                                        │   │
│         │   │ ── LLM вызов: personalizeQuestion ──   │   │
│         │   │   Вход: вопрос + reflection notes      │   │
│         │   │   Выход:                               │   │
│         │   │     • action="ask" → адаптировать      │   │
│         │   │       и задать этот вопрос              │   │
│         │   │     • action="skip" → пропустить       │   │
│         │   │       (тема уже раскрыта), i++         │   │
│         │   │   (макс. 5 пропусков подряд)           │   │
│         │   └────────────────────────────────────────┘   │
│         │                                                │
│         ├── Если все вопросы кончились:                   │
│         │   ├── Добавить outro в transcript               │
│         │   ├── status → COMPLETED                       │
│         │   └── Фоново запустить генерацию               │
│         │       экспертных рефлексий                      │
│         │                                                │
│         └── Иначе:                                       │
│             ├── [переход] + персонализированный вопрос    │
│             ├── question_index = targetIndex              │
│             ├── followups_count = 0                       │
│             └── status → WAITING_ANSWER                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Шаг 3: Reflection Notes (структурированная память)

LLM обновляет key-value заметки о пользователе. Это основная "память" системы между вопросами.

**Промпт задача:**
- Добавить новые факты из ответа
- Обновить существующие, если противоречат
- Удалить только если явно опровергнуты
- Фокус: биографические факты, ценности, эмоции, ключевые события жизни

**Пример заметок после нескольких ответов:**
```json
{
  "place_of_birth": "Saint Petersburg, Russia",
  "current_city": "Berlin, Germany",
  "occupation": "software engineer",
  "has_children": "yes, 2 kids (boy 7, girl 3)",
  "marital_status": "married",
  "moved_to_germany": "2019, for work",
  "emotional_topic_parents_divorce": "mentioned with sadness"
}
```

### Шаг 4: Planner (планировщик действий)

LLM-планировщик получает контекст и принимает решение: задать уточняющий вопрос (FOLLOW_UP) или перейти к следующему (NEXT_QUESTION).

**Факторы решения:**
- Достигнута ли цель текущего вопроса? (interview objective)
- Оставшееся время в блоке
- Количество уже заданных follow-up'ов
- Эмоциональный контент (нужен ли эмпатический переход)
- Желание юзера двигаться дальше (автономность)

**Правила:**
- Если юзер явно хочет пропустить → NEXT_QUESTION
- Если юзер отказывается отвечать по личным причинам → NEXT_QUESTION (не давить)
- Если timeLeft < 30 сек → принудительно NEXT_QUESTION

### Шаг 5A: Follow-up

Когда LLM решает задать уточняющий вопрос:
- `question_index` не меняется (мы всё ещё на том же вопросе)
- `followups_count` увеличивается на 1
- Вопрос приходит с `kind: "follow_up"`

### Шаг 5B: Переход к следующему вопросу

1. **Эмпатический переход** — если ответ содержал эмоциональный контент, LLM генерирует фразу-мостик (1-2 предложения).

2. **Цикл персонализации** — для следующего кандидата из скрипта вызывается `personalizeQuestion`:
   - `"ask"` — вопрос адаптируется (убираются части, ответ на которые уже известен) и задаётся
   - `"skip"` — вопрос полностью избыточен, переходим к следующему
   - Максимум 5 пропусков подряд (защита от зацикливания)

3. **Сброс блока** — `question_index` обновляется, `followups_count` обнуляется, таймер блока перезапускается.

---

## Этап 3: Завершение интервью

Когда все вопросы исчерпаны (или все оставшиеся пропущены):

```
┌──────────────────────────────────────────────────┐
│  1. Добавить outro-текст в transcript             │
│  2. status → COMPLETED                            │
│                                                   │
│  Фоново (async, не блокирует ответ юзеру):       │
│                                                   │
│  3. Собрать полный transcript                     │
│  4. Отправить 4 экспертам (4 LLM-вызова):        │
│     • Психолог                                    │
│     • Экономист                                   │
│     • Политолог                                   │
│     • Демограф                                    │
│  5. Сохранить рефлексии в БД                      │
│  6. Запустить калибровку реплики (reliability)     │
└──────────────────────────────────────────────────┘
```

Экспертные рефлексии генерируются асинхронно — пользователь получает ответ `status: COMPLETED` сразу, а рефлексии подтягиваются по мере готовности.

---

## LLM-вызовы на каждый ответ пользователя

| # | Метод | Тип | Модель | Что делает |
|---|-------|-----|--------|------------|
| 1 | `updateReflectionNotes` | `reflect` | Fast (Haiku) | Обновляет структурированную "память" (key-value заметки) |
| 2 | `planNextAction` | `plan` | Main (Sonnet) | Решает: follow-up или следующий вопрос |
| 3 | `personalizeQuestion` | `personalize` | Main (Sonnet) | Адаптирует/пропускает следующий вопрос |
| 4* | `generateEmpathyTransition` | `empathy_transition` | Main (Sonnet) | Эмпатический переход (только если emotional_content=true) |

*Вызов #4 — опциональный, срабатывает только при эмоциональном контенте.*

**Итого: 2-4 LLM-вызова на каждый ответ пользователя.**

### Конфигурация моделей

| Тир | Anthropic (по умолчанию) | Together AI | Используется для |
|-----|--------------------------|-------------|------------------|
| Main | `claude-sonnet-4-5-20250929` | `Llama-3.3-70B-Instruct-Turbo` | plan, personalize, empathy_transition |
| Fast | `claude-3-5-haiku-20241022` | `Llama-3.1-8B-Instruct-Turbo` | reflect (reflection notes) |

Переменные окружения: `LLM_PROVIDER`, `ANTHROPIC_CHAT_MODEL`, `ANTHROPIC_FAST_MODEL`.

---

## Промпты LLM-вызовов

Все промпты находятся в `backend/src/chat/services/llm.service.ts`.

### 1. updateReflectionNotes (reflect) — быстрая модель

Обновляет структурированные key-value заметки о пользователе на основе его последнего ответа.

**System prompt:**
```
You are a helpful assistant that produces structured JSON output.
```

**User prompt (шаблон):**
```
Given these existing notes about the interviewee:
{oldNotesJson}

And this new {role} utterance:
"{newUtterance}"

Task:
Produce updated notes by adding new facts, revising existing ones if contradicted,
or keeping them unchanged.

Rules:
- Use snake_case keys (e.g., "place_of_birth", "current_occupation")
- Values should be concise strings
- Add "uncertain_" prefix to keys if information is ambiguous
- Remove keys only if explicitly contradicted
- Focus on: biographical facts, values/beliefs, emotional states, key life events
- Keep notes concise: aim for 5-15 key-value pairs max

IMPORTANT: Return ONLY a valid JSON object with the updated notes. Do not include
any explanation, markdown formatting, or code blocks. Just the raw JSON object
starting with { and ending with }.
```

**Вход:** старые заметки (JSON) + новый ответ пользователя
**Выход:** обновлённый JSON с заметками
**Режим:** JSON mode, fast model

---

### 2. planNextAction (plan) — основная модель

Решает, что делать дальше: задать уточняющий вопрос (FOLLOW_UP) или перейти к следующему (NEXT_QUESTION).

**System prompt:**
```
You are a helpful assistant that produces structured JSON output for interview planning.
```

**User prompt (шаблон):**
```
Here is a conversation between an interviewer and an interviewee.
{transcriptTail}

=*=*=

Meta info:
- Language: Auto-detect from conversation
- Description of the interviewer (Isabella): friendly and curious
- Notes on the interviewee:
{
  {notesFormatted}
}
- Time left in this question block (sec): {timeLeftSec}
- Follow-ups already asked: {followupsCount}

Context:
This is an interview between the interviewer and an interviewee. In this conversation,
the interviewer is trying to ask the following question: "{scriptedQuestion}"

=*=*=

{interviewGuidance — если передан}

Task Description:
Interview objective: By the end of this conversation, the interviewer has to learn
the following: {interviewObjective}

Safety note: In an extreme case where the interviewee *explicitly* refuses to answer
the question for privacy reasons, do not force the interviewee to answer by pivoting
to other relevant topics.

Autonomy note: If the interviewee expresses ANY desire to move on, skip, or indicates
they don't want to elaborate further (in any language), immediately choose NEXT_QUESTION.
Respect their boundaries completely.

Output the following:
1) Assess the interview progress by reasoning step by step -- what did the interviewee
   say so far, and in your view, what would count as the interview objective being
   achieved? Write a short (3~4 sentences) assessment on whether the interview objective
   is being achieved. While staying on the current topic, what kind of follow-up questions
   should the interviewer further ask the interviewee to better achieve your interview
   objective?

2) Author the interviewer's next utterance. To not go too far astray from the interview
   objective, author a follow-up question that would better achieve the interview
   objective. OR if the objective is sufficiently met, time is almost up, or user wants
   to move on, indicate NEXT_QUESTION.

3) If the interviewee shared something emotionally difficult (trauma, loss, illness,
   hardship), acknowledge with empathy before asking your question.

IMPORTANT - Vary your language:
- Do NOT copy-paste the same empathetic phrases across different follow-ups
- Use different words and sentence structures each time
- Examples of varied empathy: "That sounds really challenging...",
  "I appreciate you sharing that...", "It takes courage to talk about...",
  "Thank you for opening up about..."
- Keep empathy genuine but brief (1 sentence max), then ask your question

IMPORTANT: Return ONLY a valid JSON object with exactly these fields:
{
  "assessment": "3-4 sentences from step 1",
  "emotional_content": true or false,
  "action": "FOLLOW_UP" or "NEXT_QUESTION",
  "transition": "if NEXT_QUESTION and emotional_content is true: empathetic transition
                 phrase. Otherwise empty string.",
  "next_utterance": "if FOLLOW_UP: the follow-up question (with empathy prefix if
                     emotional). If NEXT_QUESTION: empty string.",
  "reason": "short explanation for your decision"
}
```

**Вход:** transcriptTail, reflection notes, текущий вопрос, timeLeft, followupsCount
**Выход:** JSON с решением `PlannerDecision`
**Режим:** JSON mode, main model

---

### 3. personalizeQuestion (personalize) — основная модель

Адаптирует следующий скриптовый вопрос на основе того, что уже известно о пользователе. Может решить пропустить вопрос, если тема полностью раскрыта.

**System prompt:**
```
You are a helpful assistant that personalizes interview questions based on what we
already know about the interviewee. You produce structured JSON output.
```

**User prompt (шаблон):**
```
You are conducting an interview.
{guideSection — если передан}

Here is what we already know about the interviewee:
{
  {notesFormatted}
}

We want to ask the following scripted question:
"{scriptedQuestion}"

Task:
Decide whether to ask this question or skip it, and if asking — adapt it to account
for what we already know.

Rules:
- Keep the core intent and goal of the original question
- If the question asks about something we already know (e.g., "Do you have children?"
  when we know they have 2 kids), skip that part and focus on details we don't know yet
- If the question asks multiple things, remove redundant parts but keep unknown parts
- Keep the tone friendly and conversational
- Keep the same language as the original question
- If the entire question is redundant (we already know everything it asks),
  set action to "skip"
- DO NOT add extra questions beyond the scope of the original question

IMPORTANT: Return ONLY a valid JSON object with exactly these fields:
{
  "action": "ask" or "skip",
  "question": "the adapted question text (empty string if action is skip)",
  "reason": "brief explanation of your decision"
}
```

**Вход:** скриптовый вопрос + reflection notes
**Выход:** JSON `{ action: "ask"|"skip", question, reason }`
**Режим:** JSON mode, main model
**Особенность:** если notes пустые — пропускает LLM-вызов и возвращает оригинальный вопрос

---

### 4. generateEmpathyTransition (empathy_transition) — основная модель

Генерирует эмпатическую фразу-мостик при переходе к следующему вопросу после эмоционально тяжёлого ответа.

**System prompt:**
```
You are a compassionate interviewer who acknowledges emotional content with genuine
empathy before transitioning to the next topic.
```

**User prompt (шаблон):**
```
Here is the recent conversation:
{transcriptTail}
{followUpContext — если был запланирован follow-up, но время вышло}

Task:
Generate a brief, empathetic transition phrase (1-2 sentences max) that:
1. Acknowledges what the interviewee just shared with genuine empathy
2. Does NOT ask any questions
3. Does NOT give advice or try to fix anything
4. Does NOT use clichés like "Everything happens for a reason"
5. Feels natural and warm, not formulaic
{languageHint — в том же языке, что и разговор}

Examples of good transitions:
- "Thank you for sharing something so personal with me."
- "I really appreciate you opening up about that."
- "That sounds like it was a significant experience for you."
- "I can hear how much that meant to you."

Return ONLY the transition phrase, nothing else.
```

**Вход:** transcriptTail, (опционально) оригинальный follow-up
**Выход:** строка — фраза-переход (1-2 предложения)
**Режим:** текст (не JSON), main model
**Вызывается только** когда `emotional_content=true` и нужен переход к следующему вопросу

---

## Хранение данных (SessionStorage → PostgreSQL)

| Что | Формат | Описание |
|-----|--------|----------|
| Session | объект | Состояние сессии: `current_question_index`, `status`, `followups_count`, `block_started_at` |
| Transcript | массив `TranscriptEntry[]` | Полная история диалога: `{ts, role, text, kind, question_index}` |
| Reflection Notes | `Record<string, string>` | Обновляемая структурированная "память" о пользователе |
| Script Snapshot | `InterviewScript` | Замороженный скрипт на момент старта сессии |
| LLM Logs | массив `LLMCallLog[]` | Все вызовы LLM с input/output для аудита |
| Expert Reflections | массив | Рефлексии 4 экспертов (после завершения) |

---

## Типы сообщений (TranscriptKind)

| Kind | Роль | Описание |
|------|------|----------|
| `intro` | assistant | Вступление интервьюера |
| `scripted_question` | assistant | Вопрос из скрипта (персонализированный) |
| `follow_up` | assistant | Уточняющий вопрос / эмпатический переход |
| `user_answer` | user | Ответ пользователя |
| `outro` | assistant | Завершение интервью |

---

## Статусы сессии (SessionStatus)

```
INTRO → WAITING_ANSWER ←→ ASKING → COMPLETED
                 ↑              │
                 └──────────────┘
                 (цикл вопрос-ответ)
```

| Статус | Описание |
|--------|----------|
| `INTRO` | Сессия только создана, intro отправлен |
| `ASKING` | Система формирует следующий вопрос |
| `WAITING_ANSWER` | Ожидание ответа пользователя |
| `COMPLETED` | Интервью завершено |

---

## Фронтенд (interview/page.tsx)

### Возможности
- Чат-интерфейс с историей сообщений
- Текстовый ввод с авто-фокусом
- Голосовой ввод с real-time транскрипцией (Deepgram WebSocket)
- Прогресс-бар (текущий вопрос / всего)
- Счётчик прошедшего времени
- Text-to-speech для сообщений интервьюера
- Сохранение сессии в localStorage для возобновления
- Визуальные бейджи для типов сообщений (Overview, Question N, Follow-up)

### Логика возобновления
1. При загрузке страницы проверяет `localStorage` на наличие `sessionId`
2. Если есть — запрашивает `GET /api/chat/my-session`
3. Если сессия найдена — восстанавливает transcript и состояние
4. Если нет — начинает новую через `POST /api/chat/start`
