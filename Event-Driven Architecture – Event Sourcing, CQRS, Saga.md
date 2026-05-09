
> Практический разбор event-driven подхода: как проектировать потоки событий, когда применять Event Sourcing и CQRS, и как реализовывать Saga для распределенных транзакций

> **Event-Driven Architecture (EDA)** переносит фокус с синхронных вызовов на поток доменных событий. Это дает гибкость и масштабируемость, но требует дисциплины в контракте событий, идемпотентности и наблюдаемости. Практически EDA часто строится вокруг трех паттернов: **Event Sourcing**, **CQRS** и **Saga**.

- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#Когда нужен и когда не нужен EDA|Когда нужен и когда не нужен EDA]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Event-Driven фундамент**|Event-Driven фундамент]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Event Design**|Event Design]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Event Schema Versioning**|Event Schema Versioning]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Event Sourcing**|Event Sourcing]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**CQRS**|CQRS]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Saga**|Saga]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Saga стили координации**|Saga: стили координации]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Матрица выбора**|Матрица выбора]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Частые ошибки**|Частые ошибки]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Dead Letter Queue (DLQ)**|Dead Letter Queue (DLQ)]]
- [[Event-Driven Architecture – Event Sourcing, CQRS, Saga#**Мини-чеклист внедрения**|Мини-чеклист внедрения]]

---
## **Когда нужен и когда не нужен EDA**

##### **EDA не нужен если:**

- простой CRUD;
- нет независимых bounded contexts;
- нет высокой нагрузки;
- нет async integration needs;
- данные критично consistent;
- orchestration проще сделать синхронно.

##### **EDA добавляет:**

- eventual consistency;
- retries;
- duplicate delivery;
- observability complexity;
- schema evolution;
- operational overhead.

> Асинхронность — это компромисс, а не бесплатная масштабируемость.

---
## **Event-Driven фундамент**

##### **Событие - факт**
События описывают то, что уже произошло в домене, и должны быть неизменяемыми.

##### **Асинхронность по умолчанию**
Производитель и потребители слабо связаны, обмениваются через broker/log.

##### **Eventual consistency**
Между write и read-моделью появляется лаг, который нужно учитывать в UX и API.

---
## **Event Design**

> **Событие = immutable business fact**

**Хорошие события:**
- `OrderCreated`
- `PaymentCaptured`
- `InventoryReserved`

**Плохие события:**
- `UpdateOrder`
- `DoPayment`
- `SyncUser`

##### **Событие должно отвечать:**
- что произошло;
- когда;
- с каким aggregate/entity;
- какая версия события.

##### **Что обычно содержит событие**
```json
{
	"eventId": "...",
	"eventType": "OrderCreated",
	"aggregateId": "...",
	"version": 3,
	"occurredAt": "...",
	"payload": {...}
}
```

##### **⚠️ Anti-pattern**
> Не публикуйте внутренние DB-схемы как public events.

---
## **Event Schema Versioning**

> Event contract живет дольше конкретной версии producer-а и consumer-а.

В EDA producer и consumer обновляются независимо.

**Это значит:**
- старые consumer-ы могут читать новые события;
- новые consumer-ы могут replay-ить старые события;
- в системе одновременно живут разные версии event schema.

⚠️ Поэтому **schema evolution** — одна из главных проблем EDA.

### **Главный принцип**

> **События immutable**

**Уже опубликованные события:**
- нельзя менять;
- нельзя «исправлять»;
- нельзя переписывать историю.

> Старые события должны оставаться читаемыми спустя годы.

### **Практические стратегии versioning**

#### **Подход 1 — version field**
```json
{
	"eventType": "OrderCreated",
	"version": 2
}
```
**Consumer:**
- читает version;
- выбирает parser/handler.

#### **Подход 2 — version в имени события**
```
OrderCreatedV1OrderCreatedV2
```
Просто, но плодит типы событий.

#### **Подход 3 — schema registry**
**Отдельный registry хранит:**
- schema;
- version;
- compatibility rules.
**Популярно с:**
- Kafka + Avro;
- Protobuf;
- Confluent Schema Registry.

---
## **Event Sourcing**

> State хранится как последовательность событий, а не как последнее значение.
##### **Когда применять**
- Нужен полный аудит и воспроизводимость изменений.
- Важно пересчитывать проекции по истории событий.
- Домен естественно выражается через события.
##### **Trade-offs**
- Сложнее миграции event schema и versioning.
- Нужны snapshot и replay-стратегии для производительности.

---
## **CQRS**

> Разделение write-модели (commands) и read-модели (queries).
##### **Когда применять**
- Read/write профили сильно различаются.
- Нужны отдельные read-оптимизированные проекции.
- Система растет и требует независимого масштабирования путей.
##### **Trade-offs**
- Увеличивается операционная сложность и число компонентов.
- Read-model часто eventual consistent относительно write-model.

---
## **Saga**

> Управление распределенной транзакцией через цепочку локальных шагов + компенсации.
##### **Когда применять**
- Операция затрагивает несколько сервисов/хранилищ.
- 2PC недоступен или непрактичен.
- Нужен контролируемый rollback через compensating actions.
##### **Trade-offs**
- Нужно проектировать идемпотентность и повторные доставки.
- Сложнее отладка долгоживущих и частично завершенных процессов.

---
## **Saga: стили координации**

### **Choreography**

> Сервисы подписываются на события и реагируют без центрального координатора.
###### **Плюсы**
- Слабая связность
- Меньше центральных bottleneck
###### **Риски**
- Сложнее трассировка end-to-end
- Риск «event spaghetti»
### **Orchestration**

> Оркестратор явно управляет шагами и компенсациями.
###### **Плюсы**
- Прозрачный flow и observability
- Проще верифицировать бизнес-процесс
###### **Риски**
- Центральная точка сложности
- Нужно масштабировать orchestration layer

---
## **Матрица выбора**

|Потребность|Рекомендация|Почему|
|---|---|---|
|Полный audit trail и replay|Event Sourcing|История изменений - first-class data.|
|Отдельные read/write SLA|CQRS|Независимая оптимизация и масштабирование read/write-path.|
|Distributed transaction без 2PC|Saga|Локальные транзакции + компенсирующие действия.|
|Простая CRUD-система без высокой сложности|Не форсировать EDA|Операционная сложность может быть выше бизнес-выгоды.|

---
## **Частые ошибки**

- Пытаться внедрить все паттерны сразу без явного SLA и bounded context.
- Считать событие «фактом», но публиковать в него технический шум без бизнес-смысла.
- Не делать идемпотентность consumer-ов при at-least-once доставке.
- Игнорировать schema versioning и обратную совместимость событий.
- Не отслеживать DLQ, lag, duplicate rate и время завершения saga.

---
## **Dead Letter Queue (DLQ)**

> **DLQ** - это карантин для сообщений, которые не удалось обработать после retry. Цель DLQ - не «спрятать» ошибку, а сохранить проблемные события для безопасного разбора и контролируемого повторного запуска.

##### **Когда отправлять в DLQ**
Когда message превышает лимит попыток, нарушает schema contract или consistently падает из-за data issues.

##### **Что сохранять**
`messageId`, `attempts`, `lastError`, `originalTopic`, `failedAt`, версию контракта и ссылку на payload.

##### **Что делать после**
Делать triage, фиксить root cause и запускать re-drive batch с контролем rate limit и idempotency.

##### **Практический checklist для DLQ**

- Выделяйте отдельный DLQ на каждый критичный поток, чтобы не смешивать домены и приоритеты.
- Храните причину ошибки, число попыток, исходный topic/queue и payload reference для расследования.
- Разделяйте transient и non-transient ошибки: не всё должно попадать в DLQ после одинакового числа retry.
- Настройте re-drive процесс: ручной или автоматический возврат сообщений после исправления причины сбоя.
- Добавьте алерты на скорость роста DLQ и SLA по времени разбора poison messages.

---
## **Мини-чеклист внедрения**

1. Зафиксируйте event contract и policy versioning.
2. Обеспечьте idempotency для producer/consumer.
3. Настройте DLQ, retry policy и poison-message handling.
4. Мониторьте lag, replay time и success-rate saga.
