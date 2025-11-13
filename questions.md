# Собеседование по Java

## СОДЕРЖАНИЕ
1. [Вопрос №1](#вопрос-1)
2. [Вопрос №2](#вопрос-2)
3. [Вопрос №3](#вопрос-3)
4. [Вопрос №4](#вопрос-4)
5. [Вопрос №5](#вопрос-5)

## Вопрос №1.
Нужно реализовать кэширование объектов, которые тяжело
создавать. Какой паттерн подходит лучше всего?
1) **Chain of Responsibility**, чтобы делегировать запросы разным
обработчикам
2) **Observer**, чтобы оповещать подписчиков об изменениях
3) **Proxy**, чтобы отложить создание объекта и кэшировать результат
4) **Factory Method**, чтобы создавать разные типы кэшей.

> **Ответ: Proxy**

Когда объекты тяжело создавать, **Proxy (Заместитель)** откладывает создание реального объекта до момента, когда он действительно понадобится.

### Как работает:

```java
// Тяжелый объект
class HeavyObject {
    public HeavyObject() {
        // Долгая инициализация: загрузка из БД, файлов и т.д.
        System.out.println("Создание тяжелого объекта...");
    }
    
    public void doWork() {
        System.out.println("Работа выполнена");
    }
}

// Proxy - кэширует результат
class ProxyObject {
    private HeavyObject realObject;
    
    public void doWork() {
        if (realObject == null) {
            realObject = new HeavyObject(); // Создаем только когда нужно
        }
        realObject.doWork();
    }
}
```

### Использование:

```java
ProxyObject proxy = new ProxyObject(); // Быстро, объект не создан
// ... много кода ...
proxy.doWork(); // Только здесь создается HeavyObject
```

### Почему не другие:

- **Chain of Responsibility** - для передачи запроса по цепочке обработчиков
- **Observer** - для уведомлений о событиях
- **Factory Method** - для создания объектов, но не кэширует их

**Proxy** идеален для lazy initialization (ленивой инициализации) тяжелых объектов.

---

## Вопрос №2.
При сохранении сущности выбрасывается `ConstraintViolationException`.
Какое объяснение корректное?
1) Нарушено ограничение БД (уникальность, not null, foreign key)
2) Исключение всегда связано с проблемами индексов
3) Нарушены правила маппинга в ORM
4) Hibernate генерирует неправильный SQL-запрос

> **Ответ: Нарушено ограничение БД (уникальность, not null, foreign key)**

`ConstraintViolationException` выбрасывается, когда нарушаются **constraint'ы базы данных** при попытке сохранить сущность.

### Типичные причины:

```java
@Entity
class User {
    @Column(unique = true, nullable = false)
    private String email;
    
    @ManyToOne
    @JoinColumn(name = "role_id", nullable = false)
    private Role role;
}
```

**Что вызовет `ConstraintViolationException`:**

1. **Unique constraint** - попытка сохранить дубликат email
2. **Not null constraint** - попытка сохранить `null` в обязательном поле
3. **Foreign key constraint** - ссылка на несуществующую запись в другой таблице
4. **Check constraint** - нарушение кастомного правила (например, `age > 0`)

### Пример:

```java
User user1 = new User("test@mail.com");
entityManager.persist(user1);

User user2 = new User("test@mail.com"); // Тот же email
entityManager.persist(user2); // ConstraintViolationException: duplicate key
```

### Почему не другие варианты:

- **Проблемы индексов** - это performance issue, не exception
- **Правила маппинга в ORM** - вызовут `MappingException` или `PersistenceException`
- **Неправильный SQL** - вызовет `SQLException` или `QueryException`

`ConstraintViolationException` = нарушение правил целостности данных на уровне БД.

---

## Вопрос №3
В проекте используется `CompletableFuture`. Нужно запустить два независимых
асинхронных вычисления и дождаться их завершения, после чего объединить
результат. Какой метод лучше использовать?
1) `CompletableFuture.allOf(f1, f2).thenApply(...)`
2) `CompletableFuture.runAfterBoth(f1, f2, runnable)`
3) `CompletableFuture.applyToEither(f1, f2, fn)`
4) `CompletableFuture.anyOf(f1, f2)`

> **Ответ: CompletableFuture.allOf(f1, f2).thenApply(...)**

`allOf()` ждет завершения **всех** futures и позволяет объединить результаты.

### Пример:

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Result1");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "Result2");

CompletableFuture<String> combined = CompletableFuture.allOf(f1, f2)
    .thenApply(v -> {
        // Все futures завершены, можно получить результаты
        String r1 = f1.join();
        String r2 = f2.join();
        return r1 + " + " + r2;
    });

String result = combined.join(); // "Result1 + Result2"
```

### Почему не другие:

- **`runAfterBoth(f1, f2, runnable)`** - запускает `Runnable` (без возврата значения), не позволяет объединить результаты
  ```java
  f1.runAfterBoth(f2, () -> System.out.println("Done")); // Нет доступа к результатам
  ```

- **`applyToEither(f1, f2, fn)`** - берет результат **первого** завершившегося (не ждет оба)
  ```java
  f1.applyToEither(f2, result -> result); // Только один результат
  ```

- **`anyOf(f1, f2)`** - тоже возвращает **первый** завершившийся

**`allOf()`** - единственный метод, который ждет все futures и дает доступ ко всем результатам.

---

## Вопрос №4.
Почему `flush()` не гарантирует немедленное выполнение SQL в БД?
1) flush работает только в асинхронных транзакциях
2) flush синхронизирует состояние persistence context с SQL-операциями,
но выполнение может отложиться
3) flush всегда откладывает все операции до commit
4) flush доступен только для batch операций

> **Ответ: flush синхронизирует состояние persistence context с SQL-операциями, но выполнение может отложиться**

`flush()` отправляет изменения из persistence context в БД (генерирует SQL), но **не гарантирует немедленное выполнение** этих SQL-запросов.

### Почему:

```java
entityManager.persist(user);
entityManager.flush(); // Генерирует INSERT, но...
```

**Что происходит:**

1. `flush()` создает SQL-запросы (INSERT/UPDATE/DELETE)
2. Запросы отправляются в БД
3. **НО**: они могут быть в буфере JDBC-драйвера, сетевом буфере, или ждать в очереди БД
4. Реальное выполнение зависит от настроек драйвера, сети, нагрузки БД

### Гарантия выполнения - только commit:

```java
entityManager.persist(user);
entityManager.flush();    // SQL отправлен, но может быть в буфере
transaction.commit();     // Гарантированно выполнен и зафиксирован
```

### Почему не другие варианты:

- **"работает только в асинхронных транзакциях"** - нет, работает в обычных
- **"всегда откладывает до commit"** - нет, отправляет SQL, но не коммитит
- **"доступен только для batch"** - нет, работает всегда

`flush()` = синхронизация с БД, но не commit. Выполнение может быть отложено на уровне драйвера/сети/БД.

---

## Вопрос №5.
Разработчик включает GC-логи:
`[GC pause (G1 Evacuation Pause) (young) 45ms]`
Что это означает?
1) Minor GC в G1GC, копирование объектов из Eden в Survivor/Old
2) Concurrent Phase GC без паузы
3) Полная остановка JVM для Major GC
4) Очистка Old Gen сегмента

> **Ответ: Minor GC в G1GC, копирование объектов из Eden в Survivor/Old**

`[GC pause (G1 Evacuation Pause) (young) 45ms]` - это **Minor GC** в сборщике мусора **G1GC**.

### Что происходит:

**G1 Evacuation Pause** = процесс эвакуации (копирования) живых объектов из молодого поколения (young generation).

```
Eden region → Survivor region (или Old, если объект старый)
     ↓
Копирование живых объектов
Мертвые объекты просто удаляются
```

### Детали лога:

- **`(young)`** - обрабатывается только молодое поколение (Eden + Survivor)
- **`45ms`** - пауза приложения на 45 миллисекунд
- **Evacuation** - G1GC копирует объекты между регионами, а не чистит "на месте"

### Почему не другие:

- **"Concurrent Phase GC без паузы"** - нет, `pause` означает остановку приложения (STW - Stop The World)
- **"Полная остановка для Major GC"** - нет, `(young)` указывает на Minor GC
- **"Очистка Old Gen"** - нет, это молодое поколение

**G1 Evacuation Pause** - стандартный Minor GC в G1, копирует живые объекты и освобождает регионы Eden.

---


> **Ответ: Настроить метрики GC (Pause Time, Allocation Rate) и проанализировать параметры JVM GC (например, G1GC tuning)**

При росте задержек во время GC-пауз нужно **сначала понять проблему через метрики**, а потом **настроить GC**.

## Правильный подход:

1. **Анализ метрик GC:**
   - **Pause Time** - длительность пауз
   - **Allocation Rate** - скорость создания объектов
   - **Heap usage** - использование памяти
   - **GC frequency** - частота сборок

2. **Настройка JVM:**
   ```bash
   # G1GC tuning
   -XX:MaxGCPauseMillis=200        # Целевое время паузы
   -XX:G1HeapRegionSize=16m        # Размер региона
   -XX:InitiatingHeapOccupancyPercent=45  # Порог для Mixed GC
   ```

## Почему не другие варианты:

- **"Увеличить heap до максимума"** - временное решение, не устраняет причину. Может даже ухудшить ситуацию (больше heap = дольше GC паузы)

- **"Включить больше pod-ов"** - не решает проблему GC внутри каждого pod'а. Задержки останутся

- **"Игнорировать GC"** - плохая идея. Prometheus собирает **все** метрики JVM, включая GC. Игнорирование не решит проблему

## Что делать после анализа:

- Если allocation rate высокий → оптимизировать код (меньше объектов)
- Если pause time большой → настроить GC параметры или сменить сборщик (G1GC → ZGC/Shenandoah)
- Если heap переполняется → увеличить память или найти утечки

**Метрики + tuning = правильный путь к решению проблемы.**

```shellscript
   # G1GC tuning
   -XX:MaxGCPauseMillis=200        # Целевое время паузы
   -XX:G1HeapRegionSize=16m        # Размер региона
   -XX:InitiatingHeapOccupancyPercent=45  # Порог для Mixed GC
```

**Ответ: В БД отсутствует уникальный индекс на колонку email**

Метод `findByEmail()` возвращает `List<User>`, потому что **Spring Data JPA не знает, что email уникален**. Без индекса в БД Spring предполагает, что может быть несколько записей.

## Проблема:

```java
List<User> findByEmail(String email); // Возвращает список
```

Spring Data генерирует SQL:
```sql
SELECT * FROM users WHERE email = ?
-- Может вернуть несколько строк
```

## Решение - добавить уникальный индекс:

```java
@Entity
class User {
    @Column(unique = true) // Уникальность на уровне БД
    private String email;
}
```

Теперь можно использовать:
```java
Optional<User> findByEmail(String email); // Вернет одного или пустой Optional
User findByEmail(String email);           // Вернет одного или null
```

## Почему не другие варианты:

- **"Spring Data не накладывает ограничений"** - неверно, Spring Data учитывает схему БД и аннотации
- **"Используется Criteria API"** - нет, Spring Data генерирует SQL из имени метода
- **"Метод не поддерживает поиск по уникальным полям"** - неверно, поддерживает, если поле уникально

**Без `@Column(unique = true)` Spring Data не может гарантировать уникальность и возвращает List.**

```java
List<User> findByEmail(String email); // Возвращает список
```

```sql
SELECT * FROM users WHERE email = ?
-- Может вернуть несколько строк
```

```java
@Entity
class User {
    @Column(unique = true) // Уникальность на уровне БД
    private String email;
}
```

```java
Optional<User> findByEmail(String email); // Вернет одного или пустой Optional
User findByEmail(String email);           // Вернет одного или null
```

**Ответ: Information Expert — ответственность возложена на того, у кого есть данные**

Принцип **Information Expert** из GRASP: операцию должен выполнять класс, у которого есть необходимая информация.

## Применение к Invoice:

```java
class Invoice {
    private List<InvoiceItem> items;
    
    // Invoice сам вычисляет сумму, т.к. у него есть items
    public BigDecimal calculateTotal() {
        return items.stream()
            .map(InvoiceItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

**Invoice** имеет данные (позиции) → **Invoice** вычисляет сумму.

## Неправильно (нарушение Information Expert):

```java
class InvoiceService {
    // Сервис не должен знать внутреннюю структуру Invoice
    public BigDecimal calculateTotal(Invoice invoice) {
        return invoice.getItems().stream()...
    }
}
```

## Почему не другие принципы:

- **Low Coupling** - о снижении зависимостей между модулями
- **Controller** - о том, кто принимает запросы от UI/API
- **Creator** - о том, кто создает объекты (кто должен вызывать `new`)

**Information Expert** = ответственность там, где данные. Invoice знает свои позиции → Invoice вычисляет сумму.

```java
class Invoice {
    private List<InvoiceItem> items;
    
    // Invoice сам вычисляет сумму, т.к. у него есть items
    public BigDecimal calculateTotal() {
        return items.stream()
            .map(InvoiceItem::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

```java
class InvoiceService {
    // Сервис не должен знать внутреннюю структуру Invoice
    public BigDecimal calculateTotal(Invoice invoice) {
        return invoice.getItems().stream()...
    }
}
```

**Ответ: Планировщик PostgreSQL выбирает Hash Join или Nested Loop в зависимости от статистики**

Производительность зависит от того, какой **план выполнения** выберет PostgreSQL, основываясь на статистике таблиц.

## Проблема запроса:

```sql
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

**LEFT JOIN + GROUP BY** может выполняться по-разному:

### 1. Hash Join (быстро):
- Строит хеш-таблицу из `orders`
- Проходит по `users` один раз
- O(n + m) сложность

### 2. Nested Loop (медленно):
- Для каждого `user` сканирует `orders`
- O(n * m) сложность
- Используется, если статистика неточная или таблицы маленькие

## Что влияет на выбор:

- **Статистика таблиц** (ANALYZE)
- **Размер таблиц**
- **Наличие индексов** на `orders.user_id`
- **Настройки PostgreSQL** (`work_mem`, `random_page_cost`)

## Проверить план:

```sql
EXPLAIN ANALYZE
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

## Почему не другие варианты:

- **"JOIN всегда одинаковый"** - нет, есть разные алгоритмы
- **"LEFT JOIN блокирует индексы"** - нет, индексы работают
- **"COUNT медленный"** - COUNT(column) быстрый с индексами

**Планировщик выбирает алгоритм JOIN на основе статистики → производительность может сильно отличаться.**

```sql
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

```sql
EXPLAIN ANALYZE
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;
```

**Ответ: Job прервётся, step3 не будет выполнен**

По умолчанию в Spring Batch, если step завершается с ошибкой, **job останавливается** и последующие steps не выполняются.

## Что происходит:

```java
@Bean
public Job job(JobRepository jobRepository) {
    return new JobBuilder("myJob", jobRepository)
        .start(step1())
        .next(step2())  // Если упадёт - job остановится
        .next(step3())  // Не выполнится
        .build();
}
```

**Поведение:**
- step1 выполнен успешно ✓
- step2 упал с ошибкой ✗
- **Job status: FAILED**
- step3 **не запускается**

## Как изменить поведение:

### 1. Игнорировать ошибку step2:
```java
.start(step1())
.next(step2())
.on("FAILED").to(step3())  // Даже при ошибке идём на step3
.from(step2()).on("*").to(step3())
.end()
```

### 2. Автоматический перезапуск step2:
```java
@Bean
public Step step2() {
    return new StepBuilder("step2", jobRepository)
        .tasklet(...)
        .faultTolerant()
        .retry(Exception.class)
        .retryLimit(3)  // Перезапустит до 3 раз
        .build();
}
```

## Почему не другие варианты:

- **"Batch автоматически перезапустит"** - нет, нужна явная настройка
- **"Step3 выполнится независимо"** - нет, steps выполняются последовательно
- **"Ошибка проигнорирована"** - нет, по умолчанию job падает

**Дефолтное поведение: ошибка в step → job прерывается → следующие steps не выполняются.**

```java
@Bean
public Job job(JobRepository jobRepository) {
    return new JobBuilder("myJob", jobRepository)
        .start(step1())
        .next(step2())  // Если упадёт - job остановится
        .next(step3())  // Не выполнится
        .build();
}
```

```java
.start(step1())
.next(step2())
.on("FAILED").to(step3())  // Даже при ошибке идём на step3
.from(step2()).on("*").to(step3())
.end()
```

```java
@Bean
public Step step2() {
    return new StepBuilder("step2", jobRepository)
        .tasklet(...)
        .faultTolerant()
        .retry(Exception.class)
        .retryLimit(3)  // Перезапустит до 3 раз
        .build();
}
```

**Ответ: Включить idempotent producer и transactional consumer для exactly-once semantics**

Для обработки каждого сообщения **ровно один раз** нужна **exactly-once semantics (EOS)** в Kafka.

## Настройка:

### Producer (idempotent):
```java
Properties props = new Properties();
props.put("enable.idempotence", "true");  // Идемпотентность
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
```

### Consumer (transactional):
```java
Properties props = new Properties();
props.put("isolation.level", "read_committed");  // Читать только committed
props.put("enable.auto.commit", "false");        // Ручной commit

// В коде
consumer.subscribe(Arrays.asList("topic"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // Обработка
    }
    consumer.commitSync();  // Commit после обработки
}
```

### Транзакции (если нужно read-process-write):
```java
producer.initTransactions();
try {
    producer.beginTransaction();
    // Читаем из Kafka
    // Обрабатываем
    // Пишем в Kafka
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

## Почему не другие варианты:

- **"Автокоммит offset каждые 5 секунд"** - может привести к дублям (consumer упал между обработкой и коммитом)
- **"Сохранять offset в БД"** - усложняет архитектуру, не гарантирует exactly-once без транзакций
- **"acks=0"** - at-most-once (может потерять сообщения)

**Idempotent producer + transactional consumer + manual commit = exactly-once semantics.**

```java
Properties props = new Properties();
props.put("enable.idempotence", "true");  // Идемпотентность
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
```

```java
Properties props = new Properties();
props.put("isolation.level", "read_committed");  // Читать только committed
props.put("enable.auto.commit", "false");        // Ручной commit

// В коде
consumer.subscribe(Arrays.asList("topic"));
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // Обработка
    }
    consumer.commitSync();  // Commit после обработки
}
```

```java
producer.initTransactions();
try {
    producer.beginTransaction();
    // Читаем из Kafka
    // Обрабатываем
    // Пишем в Kafka
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**Ответ: Entity находится в состоянии detached, изменения не отслеживаются**

Когда сохраняешь новую сущность `User`, старая версия остается в памяти в состоянии **detached** (отсоединена от persistence context).

## Что происходит:

```java
User user = new User("John");
entityManager.persist(user);     // user в состоянии MANAGED
entityManager.flush();

// После flush/commit user переходит в DETACHED
user.setName("Jane");            // Изменение не отслеживается
entityManager.flush();           // Ничего не произойдет - user detached
```

## Состояния Entity:

1. **Transient** - новый объект, Hibernate не знает о нём
2. **Managed** - объект в persistence context, изменения отслеживаются
3. **Detached** - был в persistence context, но сессия закрылась
4. **Removed** - помечен на удаление

## Решение - вернуть в managed:

```java
User user = new User("John");
entityManager.persist(user);
entityManager.flush();

// Вариант 1: merge
user = entityManager.merge(user);  // Возвращает managed копию
user.setName("Jane");              // Теперь отслеживается

// Вариант 2: find
User managedUser = entityManager.find(User.class, user.getId());
managedUser.setName("Jane");       // Отслеживается
```

## Почему не другие варианты:

- **"Hibernate не поддерживает update без merge"** - поддерживает, но только для managed entities
- **"Репозиторий работает только с immutable"** - нет, работает с любыми entities
- **"Объект должен быть @Embeddable"** - нет, @Entity достаточно

**После persist/flush entity становится detached → изменения не отслеживаются → нужен merge для возврата в managed.**

```java
User user = new User("John");
entityManager.persist(user);     // user в состоянии MANAGED
entityManager.flush();

// После flush/commit user переходит в DETACHED
user.setName("Jane");            // Изменение не отслеживается
entityManager.flush();           // Ничего не произойдет - user detached
```

```java
User user = new User("John");
entityManager.persist(user);
entityManager.flush();

// Вариант 1: merge
user = entityManager.merge(user);  // Возвращает managed копию
user.setName("Jane");              // Теперь отслеживается

// Вариант 2: find
User managedUser = entityManager.find(User.class, user.getId());
managedUser.setName("Jane");       // Отслеживается
```

**Ответ: Удалятся и пользователь, и связанные заказы**

`CascadeType.ALL` включает **все операции каскадирования**, в том числе **REMOVE** (удаление).

## Что происходит:

```java
@Entity
class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}

// При удалении User
entityManager.remove(user);
// Hibernate автоматически удалит все связанные Order
```

## CascadeType.ALL включает:

- **PERSIST** - сохранение связанных объектов
- **MERGE** - обновление связанных объектов
- **REMOVE** - **удаление связанных объектов**
- **REFRESH** - обновление из БД
- **DETACH** - отсоединение от контекста

## SQL, который выполнится:

```sql
DELETE FROM orders WHERE user_id = ?;
DELETE FROM users WHERE id = ?;
```

## Если не нужно удалять заказы:

```java
@OneToMany(mappedBy = "user", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<Order> orders;
```

Или использовать `orphanRemoval = false` (по умолчанию):

```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = false)
```

## Почему не другие варианты:

- **"Удалятся только связи"** - нет, удалятся сами объекты Order
- **"Удалится только пользователь"** - нет, каскад распространяется на Order
- **"Hibernate заблокирует транзакцию"** - нет, удаление пройдет успешно

**CascadeType.ALL + remove(user) = удаление User и всех его Order.**

```java
@Entity
class User {
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders;
}

// При удалении User
entityManager.remove(user);
// Hibernate автоматически удалит все связанные Order
```

```sql
DELETE FROM orders WHERE user_id = ?;
DELETE FROM users WHERE id = ?;
```

```java
@OneToMany(mappedBy = "user", cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private List<Order> orders;
```

```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = false)
```

**Ответ: `false`, так как `copyOf` всегда создаёт новый список**

`List.copyOf()` **всегда создаёт новую неизменяемую копию**, даже если исходный список не менялся.

## Что происходит:

```java
var x = List.of(1, 2, 3);      // Создаётся immutable список
var y = List.copyOf(x);        // Создаётся НОВЫЙ immutable список

System.out.println(x == y);    // false - разные объекты в памяти
```

## Проверка:

```java
System.out.println(x.equals(y));  // true - содержимое одинаковое
System.out.println(x == y);       // false - разные ссылки
```

## Почему создаётся новая копия:

`List.copyOf()` гарантирует **защитную копию** (defensive copy), чтобы изменения в исходном списке не влияли на копию.

```java
List<Integer> mutable = new ArrayList<>(List.of(1, 2, 3));
List<Integer> copy = List.copyOf(mutable);

mutable.add(4);
System.out.println(copy);  // [1, 2, 3] - не изменился
```

## Оптимизация (но не в этом случае):

В некоторых реализациях `copyOf()` может вернуть ту же ссылку, если список уже immutable и того же типа, но это **не гарантировано** спецификацией и не стоит на это полагаться.

**Вывод: `false` - `copyOf()` создаёт новый объект.**

```java
var x = List.of(1, 2, 3);      // Создаётся immutable список
var y = List.copyOf(x);        // Создаётся НОВЫЙ immutable список

System.out.println(x == y);    // false - разные объекты в памяти
```

```java
System.out.println(x.equals(y));  // true - содержимое одинаковое
System.out.println(x == y);       // false - разные ссылки
```

```java
List<Integer> mutable = new ArrayList<>(List.of(1, 2, 3));
List<Integer> copy = List.copyOf(mutable);

mutable.add(4);
System.out.println(copy);  // [1, 2, 3] - не изменился
```

**Ответ: Только 10, остальные 5 будут без партиций и простаивать**

В Kafka **количество активных консьюмеров в группе ограничено количеством партиций**.

## Как работает:

```
Топик: 10 партиций
Consumer Group: 15 экземпляров

Распределение:
- Консьюмер 1  → Партиция 0
- Консьюмер 2  → Партиция 1
- ...
- Консьюмер 10 → Партиция 9
- Консьюмер 11 → IDLE (простаивает)
- Консьюмер 12 → IDLE
- ...
- Консьюмер 15 → IDLE
```

## Правило Kafka:

**Одна партиция = максимум один консьюмер в группе**

- Если консьюмеров < партиций → один консьюмер обрабатывает несколько партиций
- Если консьюмеров = партиций → идеальное распределение 1:1
- Если консьюмеров > партиций → лишние консьюмеры простаивают

## Пример оптимального распределения:

```
10 партиций, 5 консьюмеров:
- Консьюмер 1 → Партиции 0, 1
- Консьюмер 2 → Партиции 2, 3
- Консьюмер 3 → Партиции 4, 5
- Консьюмер 4 → Партиции 6, 7
- Консьюмер 5 → Партиции 8, 9
```

## Зачем лишние консьюмеры:

Они служат **резервом** - если один из активных консьюмеров упадет, Kafka автоматически перебалансирует партиции на простаивающие экземпляры.

**Вывод: реально работают только 10 консьюмеров, 5 в режиме standby.**

```plaintext
Топик: 10 партиций
Consumer Group: 15 экземпляров

Распределение:
- Консьюмер 1  → Партиция 0
- Консьюмер 2  → Партиция 1
- ...
- Консьюмер 10 → Партиция 9
- Консьюмер 11 → IDLE (простаивает)
- Консьюмер 12 → IDLE
- ...
- Консьюмер 15 → IDLE
```

```plaintext
10 партиций, 5 консьюмеров:
- Консьюмер 1 → Партиции 0, 1
- Консьюмер 2 → Партиции 2, 3
- Консьюмер 3 → Партиции 4, 5
- Консьюмер 4 → Партиции 6, 7
- Консьюмер 5 → Партиции 8, 9
```

**Ответ: Обработчик события вызывается только после успешного commit транзакции**

`@TransactionalEventListener` позволяет **отложить выполнение слушателя** до определенной фазы транзакции.

## Как работает:

```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
        // Слушатель НЕ вызовется здесь
    } // Транзакция commit → слушатель вызывается ПОСЛЕ commit
}

@Component
class OrderEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Вызывается ТОЛЬКО если транзакция успешно закоммичена
        sendEmail(event.getOrder());
    }
}
```

## Фазы TransactionPhase:

- **AFTER_COMMIT** (по умолчанию) - после успешного commit
- **AFTER_ROLLBACK** - после rollback
- **AFTER_COMPLETION** - после commit или rollback
- **BEFORE_COMMIT** - перед commit (но внутри транзакции)

## Преимущества:

```java
// Обычный @EventListener
@EventListener
public void handle(OrderCreatedEvent event) {
    sendEmail(); // Вызовется ДО commit
    // Если транзакция откатится, email уже отправлен (плохо!)
}

// @TransactionalEventListener
@TransactionalEventListener
public void handle(OrderCreatedEvent event) {
    sendEmail(); // Вызовется ПОСЛЕ commit
    // Email отправится только если заказ реально сохранен (хорошо!)
}
```

## Почему не другие варианты:

- **"Слушатель может быть вызван до rollback"** - нет, по умолчанию вызывается после commit
- **"Не требует регистрации"** - требует, это Spring-аннотация
- **"Работает быстрее, не зависит от прокси"** - нет, зависит от прокси и транзакций

**`@TransactionalEventListener` = гарантия, что обработчик выполнится только после успешного commit транзакции.**

```java
@Service
class OrderService {
    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(order));
        // Слушатель НЕ вызовется здесь
    } // Транзакция commit → слушатель вызывается ПОСЛЕ commit
}

@Component
class OrderEventListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Вызывается ТОЛЬКО если транзакция успешно закоммичена
        sendEmail(event.getOrder());
    }
}
```

```java
// Обычный @EventListener
@EventListener
public void handle(OrderCreatedEvent event) {
    sendEmail(); // Вызовется ДО commit
    // Если транзакция откатится, email уже отправлен (плохо!)
}

// @TransactionalEventListener
@TransactionalEventListener
public void handle(OrderCreatedEvent event) {
    sendEmail(); // Вызовется ПОСЛЕ commit
    // Email отправится только если заказ реально сохранен (хорошо!)
}
```

**Ответ: Для реализации оптимистичной блокировки**

`@Version` используется для **Optimistic Locking** - предотвращения конфликтов при одновременном изменении одной записи разными транзакциями.

## Как работает:

```java
@Entity
class Product {
    @Id
    private Long id;
    
    @Version
    private Long version;  // Автоматически увеличивается при каждом update
    
    private String name;
    private BigDecimal price;
}
```

## Механизм:

```java
// Транзакция 1
Product p1 = em.find(Product.class, 1L);  // version = 0
p1.setPrice(100);

// Транзакция 2 (параллельно)
Product p2 = em.find(Product.class, 1L);  // version = 0
p2.setPrice(200);

// Транзакция 1 commit
em.merge(p1);  // UPDATE ... SET price=100, version=1 WHERE id=1 AND version=0
               // Успех! version теперь = 1

// Транзакция 2 commit
em.merge(p2);  // UPDATE ... SET price=200, version=1 WHERE id=1 AND version=0
               // FAIL! version уже = 1, а не 0
               // OptimisticLockException
```

## SQL, который генерируется:

```sql
UPDATE product 
SET price = ?, version = version + 1 
WHERE id = ? AND version = ?
```

Если `version` не совпадает → update не выполнится → exception.

## Почему не другие варианты:

- **"Для индексации"** - нет, индексы создаются через `@Index`
- **"Для хранения номера коммита"** - нет, это не связано с Git/VCS
- **"Для отслеживания времени создания"** - нет, для этого `@CreatedDate` или `@Temporal`

**`@Version` = оптимистичная блокировка для предотвращения lost updates.**

```java
@Entity
class Product {
    @Id
    private Long id;
    
    @Version
    private Long version;  // Автоматически увеличивается при каждом update
    
    private String name;
    private BigDecimal price;
}
```

```java
// Транзакция 1
Product p1 = em.find(Product.class, 1L);  // version = 0
p1.setPrice(100);

// Транзакция 2 (параллельно)
Product p2 = em.find(Product.class, 1L);  // version = 0
p2.setPrice(200);

// Транзакция 1 commit
em.merge(p1);  // UPDATE ... SET price=100, version=1 WHERE id=1 AND version=0
               // Успех! version теперь = 1

// Транзакция 2 commit
em.merge(p2);  // UPDATE ... SET price=200, version=1 WHERE id=1 AND version=0
               // FAIL! version уже = 1, а не 0
               // OptimisticLockException
```

```sql
UPDATE product 
SET price = ?, version = version + 1 
WHERE id = ? AND version = ?
```

**Ответ: Spring создаёт прокси только для public/protected методов**

Spring AOP работает через **прокси-объекты** (JDK dynamic proxy или CGLIB), которые могут перехватывать только **публичные методы**.

## Почему private не работает:

```java
@Component
class MyService {
    
    @Transactional  // НЕ РАБОТАЕТ
    private void saveData() {
        // ...
    }
    
    @Transactional  // РАБОТАЕТ
    public void saveDataPublic() {
        // ...
    }
}
```

## Как работает прокси:

```java
// Spring создаёт прокси
class MyServiceProxy extends MyService {
    
    @Override
    public void saveDataPublic() {  // Может переопределить public
        // Начать транзакцию
        super.saveDataPublic();
        // Закоммитить транзакцию
    }
    
    // НЕ МОЖЕТ переопределить private метод!
    // private void saveData() { ... }  // Компилятор не позволит
}
```

## Ограничения прокси:

1. **Private методы** - не перехватываются (нельзя переопределить)
2. **Final методы** - не перехватываются (нельзя переопределить)
3. **Self-invocation** - вызов метода внутри того же класса обходит прокси

```java
@Component
class MyService {
    
    public void methodA() {
        this.methodB();  // Прямой вызов, прокси НЕ работает!
    }
    
    @Transactional
    public void methodB() {
        // Транзакция НЕ откроется при вызове из methodA
    }
}
```

## Почему не другие варианты:

- **"Аспекты требуют @AspectTarget"** - нет такой аннотации
- **"Аспекты только для статических методов"** - наоборот, для instance методов
- **"Прокси вызывают методы напрямую"** - прокси перехватывают вызовы извне

**Spring AOP = прокси = только public/protected методы. Private недоступен для переопределения.**

```java
@Component
class MyService {
    
    @Transactional  // НЕ РАБОТАЕТ
    private void saveData() {
        // ...
    }
    
    @Transactional  // РАБОТАЕТ
    public void saveDataPublic() {
        // ...
    }
}
```

```java
// Spring создаёт прокси
class MyServiceProxy extends MyService {
    
    @Override
    public void saveDataPublic() {  // Может переопределить public
        // Начать транзакцию
        super.saveDataPublic();
        // Закоммитить транзакцию
    }
    
    // НЕ МОЖЕТ переопределить private метод!
    // private void saveData() { ... }  // Компилятор не позволит
}
```

```java
@Component
class MyService {
    
    public void methodA() {
        this.methodB();  // Прямой вызов, прокси НЕ работает!
    }
    
    @Transactional
    public void methodB() {
        // Транзакция НЕ откроется при вызове из methodA
    }
}
```

**Ответ: Micrometer с метрикой `http.server.requests{status="5xx"}` и алертингом в Prometheus/Grafana**

Spring Boot Actuator автоматически собирает метрики через **Micrometer**, включая количество HTTP-запросов с разбивкой по статус-кодам.

## Настройка:

### 1. Зависимости (обычно уже есть):
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 2. application.yml:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,metrics
  metrics:
    tags:
      application: ${spring.application.name}
```

### 3. Метрики доступны на `/actuator/prometheus`:
```
http_server_requests_seconds_count{status="500"} 15
http_server_requests_seconds_count{status="503"} 3
http_server_requests_seconds_count{uri="/api/users",status="500"} 10
```

### 4. Prometheus запрос:
```promql
# Количество 5xx за последние 5 минут
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))

# Процент ошибок
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
/ 
sum(rate(http_server_requests_seconds_count[5m])) * 100
```

### 5. Grafana dashboard или Prometheus alert:
```yaml
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 10
  for: 5m
  annotations:
    summary: "High 5xx error rate"
```

## Почему не другие варианты:

- **"Периодический SQL-запрос к таблице логов"** - неэффективно, нагружает БД, нет real-time
- **"Только health-check"** - показывает только статус приложения (up/down), не детализирует ошибки
- **"Журнал ошибок без метрик"** - сложно агрегировать, нет алертинга

**Micrometer + Prometheus + Grafana = стандартный стек для мониторинга метрик в Spring Boot.**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,metrics
  metrics:
    tags:
      application: ${spring.application.name}
```

```plaintext
http_server_requests_seconds_count{status="500"} 15
http_server_requests_seconds_count{status="503"} 3
http_server_requests_seconds_count{uri="/api/users",status="500"} 10
```

```plaintext
# Количество 5xx за последние 5 минут
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))

# Процент ошибок
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) 
/ 
sum(rate(http_server_requests_seconds_count[5m])) * 100
```

```yaml
- alert: HighErrorRate
  expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 10
  for: 5m
  annotations:
    summary: "High 5xx error rate"
```

**Ответ: Сообщения будут разделены между консьюмерами группы по партициям**

Когда несколько консьюмеров находятся в **одной consumer group**, Kafka **распределяет партиции** между ними. Каждое сообщение обрабатывается **только одним консьюмером** из группы.

## Как работает:

```
Топик orders: 3 партиции
Consumer Group "order-processors": 3 консьюмера

Распределение:
┌─────────────┐
│ Партиция 0  │ → Консьюмер 1
│ msg1, msg4  │
└─────────────┘

┌─────────────┐
│ Партиция 1  │ → Консьюмер 2
│ msg2, msg5  │
└─────────────┘

┌─────────────┐
│ Партиция 2  │ → Консьюмер 3
│ msg3, msg6  │
└─────────────┘
```

## Ключевые моменты:

1. **Одна партиция = один консьюмер** (в рамках группы)
2. **Один консьюмер может читать несколько партиций**
3. **Каждое сообщение обрабатывается только один раз** в группе
4. **Балансировка автоматическая** (при добавлении/удалении консьюмеров)

## Пример конфигурации:

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processors");  // Одна группа
props.put("enable.auto.commit", "true");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));
```

## Почему не другие варианты:

- **"Каждый консьюмер получит все сообщения"** - это если консьюмеры в **разных группах**
- **"Первому уйдут все, остальные standby"** - нет, партиции распределяются равномерно
- **"Случайное распределение"** - нет, детерминированное по партициям

**Consumer Group = распределение партиций между консьюмерами для параллельной обработки.**

```plaintext
Топик orders: 3 партиции
Consumer Group "order-processors": 3 консьюмера

Распределение:
┌─────────────┐
│ Партиция 0  │ → Консьюмер 1
│ msg1, msg4  │
└─────────────┘

┌─────────────┐
│ Партиция 1  │ → Консьюмер 2
│ msg2, msg5  │
└─────────────┘

┌─────────────┐
│ Партиция 2  │ → Консьюмер 3
│ msg3, msg6  │
└─────────────┘
```

```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processors");  // Одна группа
props.put("enable.auto.commit", "true");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));
```

**Ответ: Нарушается Single Responsibility — один класс выполняет несколько обязанностей**

Класс `OrderManager` выполняет **слишком много разных задач**, что нарушает принцип **Single Responsibility Principle (SRP)** из SOLID.

## Проблема:

```java
class OrderManager {
    // Бизнес-логика заказа
    public void calculateTotal(Order order) { ... }
    public void applyDiscount(Order order) { ... }
    
    // Сохранение в БД
    public void saveOrder(Order order) { 
        // SQL, транзакции, маппинг
    }
    
    // REST-контроллер
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody Order order) { ... }
}
```

**Один класс имеет 3 причины для изменений:**
1. Изменение бизнес-правил (скидки, расчёты)
2. Изменение способа хранения (БД → NoSQL)
3. Изменение API (REST → GraphQL)

## Правильное разделение (SRP):

```java
// Бизнес-логика
@Service
class OrderService {
    public void calculateTotal(Order order) { ... }
    public void applyDiscount(Order order) { ... }
}

// Сохранение в БД
@Repository
class OrderRepository {
    public void save(Order order) { ... }
    public Order findById(Long id) { ... }
}

// REST API
@RestController
class OrderController {
    private final OrderService orderService;
    private final OrderRepository orderRepository;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        orderService.calculateTotal(order);
        orderRepository.save(order);
        return ResponseEntity.ok(order);
    }
}
```

## Почему не другие принципы:

- **Dependency Inversion** - нарушен, но это следствие нарушения SRP
- **Open/Closed** - класс сложно расширять, но основная проблема в SRP
- **Interface Segregation** - не применим, речь не о большом интерфейсе

**SRP = одна причина для изменения. `OrderManager` имеет 3 причины → нарушение SRP.**

```java
class OrderManager {
    // Бизнес-логика заказа
    public void calculateTotal(Order order) { ... }
    public void applyDiscount(Order order) { ... }
    
    // Сохранение в БД
    public void saveOrder(Order order) { 
        // SQL, транзакции, маппинг
    }
    
    // REST-контроллер
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody Order order) { ... }
}
```

```java
// Бизнес-логика
@Service
class OrderService {
    public void calculateTotal(Order order) { ... }
    public void applyDiscount(Order order) { ... }
}

// Сохранение в БД
@Repository
class OrderRepository {
    public void save(Order order) { ... }
    public Order findById(Long id) { ... }
}

// REST API
@RestController
class OrderController {
    private final OrderService orderService;
    private final OrderRepository orderRepository;
    
    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody Order order) {
        orderService.calculateTotal(order);
        orderRepository.save(order);
        return ResponseEntity.ok(order);
    }
}
```

**Ответ: ZGC минимизирует паузы (обычно <10мс), но требует больше памяти**

**ZGC (Z Garbage Collector)** — это low-latency сборщик мусора, появившийся в Java 11, оптимизированный для **минимальных пауз** приложения.

## Ключевые характеристики ZGC:

### ✅ Преимущества:
- **Паузы <10мс** (обычно 1-5мс), независимо от размера heap
- Работает с heap от 8MB до **16TB**
- Поддерживает Linux, Windows, macOS
- Concurrent - большая часть работы параллельно с приложением

### ❌ Недостатки:
- **Требует больше памяти** (overhead ~10-20% от heap)
- Больше нагрузка на CPU
- Throughput ниже, чем у G1GC/Parallel GC

## Использование:

```bash
# Java 11-14 (экспериментальный)
java -XX:+UnlockExperimentalVMOptions -XX:+UseZGC -Xmx16g MyApp

# Java 15+ (production ready)
java -XX:+UseZGC -Xmx16g MyApp
```

## Когда использовать ZGC:

- **Нужны предсказуемые низкие паузы** (latency-sensitive приложения)
- Real-time системы, торговые платформы
- Веб-сервисы с SLA на response time
- Большие heap (>100GB)

## Сравнение с другими GC:

| GC | Паузы | Throughput | Память |
|----|-------|------------|--------|
| **ZGC** | <10ms | Средний | Высокая |
| **G1GC** | 10-200ms | Высокий | Средняя |
| **Parallel** | 100ms+ | Очень высокий | Низкая |
| **Shenandoah** | <10ms | Средний | Высокая |

## Почему не другие варианты:

- **"Недоступен в Linux"** - работает на всех основных ОС
- **"Всегда медленнее G1GC"** - нет, паузы короче, но throughput ниже
- **"Только single-thread"** - наоборот, highly concurrent

**ZGC = минимальные паузы за счёт дополнительной памяти и CPU.**

```shellscript
# Java 11-14 (экспериментальный)
java -XX:+UnlockExperimentalVMOptions -XX:+UseZGC -Xmx16g MyApp

# Java 15+ (production ready)
java -XX:+UseZGC -Xmx16g MyApp
```

**Ответ: Обе транзакции будут заблокированы до ручного вмешательства**

Это классический случай **deadlock (взаимной блокировки)** в PostgreSQL.

## Что происходит:

```
Транзакция 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Получила lock на строку id=1
-- Ждёт lock на строку id=2...

Транзакция 2 (одновременно):
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Получила lock на строку id=2
-- Ждёт lock на строку id=1...

Результат: DEADLOCK
```

## Детально:

### Шаг 1:
- **T1**: блокирует строку `id=1` (exclusive lock)
- **T2**: блокирует строку `id=2` (exclusive lock)

### Шаг 2:
- **T1**: пытается изменить `id=2` → **ждёт T2**
- **T2**: пытается изменить `id=1` → **ждёт T1**

### Результат:
**PostgreSQL обнаруживает deadlock** (обычно через ~1 секунду) и:
```
ERROR: deadlock detected
DETAIL: Process A waits for ShareLock on transaction B; blocked by process B.
Process B waits for ShareLock on transaction A; blocked by process A.
```

Одна из транзакций будет **принудительно откачена** (rollback), другая продолжит выполнение.

## Как избежать:

### 1. Упорядочить обновления (по id):
```sql
-- Всегда обновлять в порядке возрастания id
UPDATE accounts SET balance = balance - 100 WHERE id IN (1, 2) ORDER BY id;
```

### 2. Использовать `SELECT FOR UPDATE`:
```sql
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Заблокировали обе строки сразу
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### 3. Использовать уровень изоляции Serializable (но медленнее):
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## Почему не другие варианты:

- **"Возможны блокировки, но Read Committed гарантирует целостность"** - не спасает от deadlock
- **"Автоматический rollback"** - да, но **после обнаружения** deadlock, не до
- **"Выполнятся параллельно"** - нет, будут ждать друг друга

**Deadlock → PostgreSQL откатит одну из транзакций через ~1 секунду.**

```plaintext
Транзакция 1:
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Получила lock на строку id=1
-- Ждёт lock на строку id=2...

Транзакция 2 (одновременно):
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Получила lock на строку id=2
-- Ждёт lock на строку id=1...

Результат: DEADLOCK
```

```plaintext
ERROR: deadlock detected
DETAIL: Process A waits for ShareLock on transaction B; blocked by process B.
Process B waits for ShareLock on transaction A; blocked by process A.
```

```sql
-- Всегда обновлять в порядке возрастания id
UPDATE accounts SET balance = balance - 100 WHERE id IN (1, 2) ORDER BY id;
```

```sql
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Заблокировали обе строки сразу
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Ответ: По умолчанию он возвращает только статус "UP" или "DOWN", подробности скрыты**

Endpoint `/actuator/health` по умолчанию показывает **минимальную информацию** о состоянии приложения из соображений безопасности.

## Поведение по умолчанию:

```json
{
  "status": "UP"
}
```

## Как включить детали:

### application.yml:
```yaml
management:
  endpoint:
    health:
      show-details: always  # или when-authorized
  endpoints:
    web:
      exposure:
        include: health
```

### Опции `show-details`:
- **`never`** (по умолчанию) - только статус
- **`when-authorized`** - детали только для авторизованных пользователей
- **`always`** - всегда показывать детали

## С деталями:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 336699858944,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

## Health Indicators (автоматические проверки):

Spring Boot автоматически проверяет:
- **DataSource** (БД)
- **DiskSpace** (место на диске)
- **Redis, MongoDB, Kafka** (если подключены)
- **Custom indicators** (можно добавить свои)

## Кастомный Health Indicator:

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean serviceAvailable = checkExternalService();
        if (serviceAvailable) {
            return Health.up()
                .withDetail("service", "available")
                .build();
        }
        return Health.down()
            .withDetail("service", "unavailable")
            .build();
    }
}
```

## Почему не другие варианты:

- **"Доступен только через JMX"** - нет, доступен через HTTP
- **"Всегда включает полную информацию"** - нет, по умолчанию минимум
- **"Требуется Spring Security"** - нет, работает без него (но можно защитить)

**По умолчанию `/health` показывает только UP/DOWN для безопасности. Детали нужно включать явно.**

```json
{
  "status": "UP"
}
```

```yaml
management:
  endpoint:
    health:
      show-details: always  # или when-authorized
  endpoints:
    web:
      exposure:
        include: health
```

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 336699858944,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean serviceAvailable = checkExternalService();
        if (serviceAvailable) {
            return Health.up()
                .withDetail("service", "available")
                .build();
        }
        return Health.down()
            .withDetail("service", "unavailable")
            .build();
    }
}
```

**Ответ: Через прокси вернётся один и тот же bean из контекста**

Spring создаёт **CGLIB прокси** для `@Bean` методов в `@Configuration` классе. При вызове метода напрямую прокси перехватывает вызов и возвращает bean из контекста вместо создания нового.

## Что происходит:

```java
@Configuration
public class AppConfig {
    
    @Bean
    public Service service() { 
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        // Вызываем service() напрямую
        return new Controller(service());  // Прокси вернёт существующий bean!
    }
}
```

### Реальное поведение:

```java
// Spring создаёт прокси AppConfig
AppConfig$$EnhancerBySpringCGLIB extends AppConfig {
    
    private Map<String, Object> beanCache;
    
    @Override
    public Service service() {
        // Проверяем кэш
        if (beanCache.contains("service")) {
            return beanCache.get("service");  // Возвращаем singleton
        }
        Service bean = super.service();  // Создаём только если нет
        beanCache.put("service", bean);
        return bean;
    }
}
```

## Проверка:

```java
@Configuration
public class AppConfig {
    @Bean
    public Service service() { 
        System.out.println("Creating Service");
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        Service s1 = service();  // "Creating Service" - первый вызов
        Service s2 = service();  // Ничего не выведет - вернётся тот же bean
        System.out.println(s1 == s2);  // true
        return new Controller(s1);
    }
}
```

**Вывод:**
```
Creating Service
true
```

## Важно: без @Configuration это не работает!

```java
// НЕТ @Configuration - обычный класс
public class AppConfig {
    @Bean
    public Service service() { 
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        Service s1 = service();  // Новый объект
        Service s2 = service();  // Ещё один новый объект
        System.out.println(s1 == s2);  // false!
        return new Controller(s1);
    }
}
```

## Почему не другие варианты:

- **"Будет создан новый объект"** - нет, прокси вернёт существующий
- **"Контейнер всегда возвращает singleton"** - да, но через прокси-механизм
- **"Метод вызовет сам себя рекурсивно"** - нет, прокси перехватывает вызов

**@Configuration + CGLIB прокси = вызов @Bean метода напрямую возвращает bean из контекста, а не создаёт новый.**

```java
@Configuration
public class AppConfig {
    
    @Bean
    public Service service() { 
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        // Вызываем service() напрямую
        return new Controller(service());  // Прокси вернёт существующий bean!
    }
}
```

```java
// Spring создаёт прокси AppConfig
AppConfig$$EnhancerBySpringCGLIB extends AppConfig {
    
    private Map<String, Object> beanCache;
    
    @Override
    public Service service() {
        // Проверяем кэш
        if (beanCache.contains("service")) {
            return beanCache.get("service");  // Возвращаем singleton
        }
        Service bean = super.service();  // Создаём только если нет
        beanCache.put("service", bean);
        return bean;
    }
}
```

```java
@Configuration
public class AppConfig {
    @Bean
    public Service service() { 
        System.out.println("Creating Service");
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        Service s1 = service();  // "Creating Service" - первый вызов
        Service s2 = service();  // Ничего не выведет - вернётся тот же bean
        System.out.println(s1 == s2);  // true
        return new Controller(s1);
    }
}
```

```plaintext
Creating Service
true
```

```java
// НЕТ @Configuration - обычный класс
public class AppConfig {
    @Bean
    public Service service() { 
        return new ServiceImpl(); 
    }
    
    @Bean
    public Controller controller() {
        Service s1 = service();  // Новый объект
        Service s2 = service();  // Ещё один новый объект
        System.out.println(s1 == s2);  // false!
        return new Controller(s1);
    }
}
```

**Ответ: Допустим `null` как ключ и как значение, оба элемента сохранятся**

`HashMap` в Java **разрешает один `null` ключ и любое количество `null` значений**.

## Что происходит:

```java
Map<String, String> map = new HashMap<>();
map.put(null, "value1");  // ✓ Сохранится: null -> "value1"
map.put("key", null);     // ✓ Сохранится: "key" -> null

System.out.println(map.size());        // 2
System.out.println(map.get(null));     // "value1"
System.out.println(map.get("key"));    // null
System.out.println(map.containsKey(null));  // true
System.out.println(map.containsKey("key")); // true
```

## Детали:

### Один null-ключ:
```java
map.put(null, "value1");
map.put(null, "value2");  // Перезапишет предыдущее значение
System.out.println(map.get(null));  // "value2"
```

### Много null-значений:
```java
map.put("key1", null);
map.put("key2", null);
map.put("key3", null);
// Все сохранятся
```

## Внутренняя реализация:

```java
// Упрощённая логика HashMap
public V put(K key, V value) {
    int hash = (key == null) ? 0 : key.hashCode();  // null -> hash = 0
    // ... сохранение в bucket[0]
}

public V get(Object key) {
    int hash = (key == null) ? 0 : key.hashCode();
    // ... поиск в bucket[0]
}
```

`null` ключ всегда хранится в **bucket 0**.

## Почему не другие варианты:

- **"Нельзя хранить null как значение"** - можно
- **"Нельзя хранить null как ключ"** - можно (один)
- **"HashMap заменит null на пустую строку"** - нет, хранится как есть

## Отличие от других Map:

```java
// HashMap - разрешает null
HashMap<String, String> hashMap = new HashMap<>();
hashMap.put(null, null);  // ✓ OK

// Hashtable - НЕ разрешает null
Hashtable<String, String> hashtable = new Hashtable<>();
hashtable.put(null, "value");  // ✗ NullPointerException

// ConcurrentHashMap - НЕ разрешает null
ConcurrentHashMap<String, String> concurrent = new ConcurrentHashMap<>();
concurrent.put(null, "value");  // ✗ NullPointerException
concurrent.put("key", null);    // ✗ NullPointerException
```

**HashMap допускает один `null` ключ и множество `null` значений.**

```java
Map<String, String> map = new HashMap<>();
map.put(null, "value1");  // ✓ Сохранится: null -> "value1"
map.put("key", null);     // ✓ Сохранится: "key" -> null

System.out.println(map.size());        // 2
System.out.println(map.get(null));     // "value1"
System.out.println(map.get("key"));    // null
System.out.println(map.containsKey(null));  // true
System.out.println(map.containsKey("key")); // true
```

```java
map.put(null, "value1");
map.put(null, "value2");  // Перезапишет предыдущее значение
System.out.println(map.get(null));  // "value2"
```

```java
map.put("key1", null);
map.put("key2", null);
map.put("key3", null);
// Все сохранятся
```

```java
// Упрощённая логика HashMap
public V put(K key, V value) {
    int hash = (key == null) ? 0 : key.hashCode();  // null -> hash = 0
    // ... сохранение в bucket[0]
}

public V get(Object key) {
    int hash = (key == null) ? 0 : key.hashCode();
    // ... поиск в bucket[0]
}
```

```java
// HashMap - разрешает null
HashMap<String, String> hashMap = new HashMap<>();
hashMap.put(null, null);  // ✓ OK

// Hashtable - НЕ разрешает null
Hashtable<String, String> hashtable = new Hashtable<>();
hashtable.put(null, "value");  // ✗ NullPointerException

// ConcurrentHashMap - НЕ разрешает null
ConcurrentHashMap<String, String> concurrent = new ConcurrentHashMap<>();
concurrent.put(null, "value");  // ✗ NullPointerException
concurrent.put("key", null);    // ✗ NullPointerException
```

**Ответ: Исключение при сохранении приведёт к rollback и откату метода целиком**

Когда метод помечен `@Transactional`, **любое unchecked exception** (RuntimeException) вызывает **откат всей транзакции**, включая уже выполненные операции.

## Что происходит:

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);              // 1. Сохранение в БД (в рамках транзакции)
    sendNotification(user);       // 2. Если здесь exception...
}
```

### Сценарий с ошибкой:

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);              // UPDATE выполнен (но не закоммичен)
    
    sendNotification(user);       // RuntimeException!
    
    // Транзакция откатывается (ROLLBACK)
    // UPDATE отменяется
    // Уведомление НЕ отправлено
}
```

**Результат:** User **НЕ сохранён** в БД, уведомление **НЕ отправлено**.

## Как отправить уведомление независимо:

### Вариант 1: Отдельная транзакция
```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    // Транзакция commit
}

public void notifyUser(User user) {
    try {
        sendNotification(user);  // Вне транзакции
    } catch (Exception e) {
        log.error("Notification failed", e);
    }
}

// Использование
updateUser(user);
notifyUser(user);
```

### Вариант 2: @TransactionalEventListener
```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    eventPublisher.publishEvent(new UserUpdatedEvent(user));
}

@Component
class NotificationListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserUpdated(UserUpdatedEvent event) {
        sendNotification(event.getUser());  // Только после успешного commit
    }
}
```

### Вариант 3: Propagation.REQUIRES_NEW
```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    notificationService.sendAsync(user);  // Отдельная транзакция
}

@Service
class NotificationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendAsync(User user) {
        sendNotification(user);  // Независимая транзакция
    }
}
```

## Почему не другие варианты:

- **"Spring всегда выполняет save вне транзакции"** - нет, `@Transactional` управляет транзакцией
- **"sendNotification всегда асинхронный"** - нет, по умолчанию синхронный
- **"Notification вызывается до commit"** - да, но в рамках той же транзакции, откатится вместе с save

**@Transactional + exception = rollback всей транзакции, включая save(). Уведомление не отправится.**

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);              // 1. Сохранение в БД (в рамках транзакции)
    sendNotification(user);       // 2. Если здесь exception...
}
```

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);              // UPDATE выполнен (но не закоммичен)
    
    sendNotification(user);       // RuntimeException!
    
    // Транзакция откатывается (ROLLBACK)
    // UPDATE отменяется
    // Уведомление НЕ отправлено
}
```

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    // Транзакция commit
}

public void notifyUser(User user) {
    try {
        sendNotification(user);  // Вне транзакции
    } catch (Exception e) {
        log.error("Notification failed", e);
    }
}

// Использование
updateUser(user);
notifyUser(user);
```

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    eventPublisher.publishEvent(new UserUpdatedEvent(user));
}

@Component
class NotificationListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserUpdated(UserUpdatedEvent event) {
        sendNotification(event.getUser());  // Только после успешного commit
    }
}
```

```java
@Transactional
public void updateUser(User user) {
    repo.save(user);
    notificationService.sendAsync(user);  // Отдельная транзакция
}

@Service
class NotificationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendAsync(User user) {
        sendNotification(user);  // Независимая транзакция
    }
}
```

**Ответ: Функция NOW() делает условие нестабильным для планировщика**

`NOW()` возвращает **текущее время**, которое меняется при каждом выполнении запроса. PostgreSQL не может использовать индекс эффективно, потому что значение `NOW()` **не константа** — оно вычисляется в runtime.

## Проблема:

```sql
SELECT * FROM users 
WHERE created_at >= NOW() - INTERVAL '1 day';
--              ^^^^^^ Динамическое значение!
```

**Планировщик PostgreSQL:**
- Не знает заранее результат `NOW()`
- Не может предвычислить границу диапазона
- Вынужден выполнять **Seq Scan** или **Index Scan** с пересчётом условия для каждой строки

## Проверка плана:

```sql
EXPLAIN ANALYZE
SELECT * FROM users 
WHERE created_at >= NOW() - INTERVAL '1 day';
```

**Результат:**
```
Seq Scan on users  (cost=0.00..1234.56 rows=500 width=100)
  Filter: (created_at >= (now() - '1 day'::interval))
```

## Решение — использовать параметры:

### Вариант 1: Вычислить дату в приложении
```java
// Java
LocalDateTime oneDayAgo = LocalDateTime.now().minusDays(1);
jdbcTemplate.query(
    "SELECT * FROM users WHERE created_at >= ?", 
    oneDayAgo
);
```

```sql
-- Теперь это константа для планировщика
SELECT * FROM users 
WHERE created_at >= '2025-11-11 10:00:00';
```

**План:**
```
Index Scan using idx_users_created_at on users
  Index Cond: (created_at >= '2025-11-11 10:00:00')
```

### Вариант 2: Использовать prepared statement
```sql
PREPARE recent_users(timestamp) AS
SELECT * FROM users WHERE created_at >= $1;

EXECUTE recent_users(NOW() - INTERVAL '1 day');
```

### Вариант 3: Partial Index (если часто запрашиваются недавние пользователи)
```sql
-- Индекс только для недавних записей
CREATE INDEX idx_users_recent 
ON users(created_at) 
WHERE created_at >= NOW() - INTERVAL '7 days';
```

## Почему не другие варианты:

- **"Индекс не работает для диапазонных запросов"** — работает, если условие стабильное
- **"PostgreSQL не поддерживает индексы по TIMESTAMP"** — поддерживает отлично
- **"Индекс работает только с UNIQUE ключами"** — нет, с любыми колонками

## Дополнительно: volatility функций

```sql
-- NOW() имеет volatility = STABLE (меняется в рамках транзакции)
SELECT provolatile FROM pg_proc WHERE proname = 'now';
-- Result: 's' (stable)

-- Для оптимизации можно использовать IMMUTABLE функции
```

**Вывод: `NOW()` делает условие динамическим → планировщик не может эффективно использовать индекс. Решение — передавать вычисленное значение из приложения.**

```sql
SELECT * FROM users 
WHERE created_at >= NOW() - INTERVAL '1 day';
--              ^^^^^^ Динамическое значение!
```

```sql
EXPLAIN ANALYZE
SELECT * FROM users 
WHERE created_at >= NOW() - INTERVAL '1 day';
```

```plaintext
Seq Scan on users  (cost=0.00..1234.56 rows=500 width=100)
  Filter: (created_at >= (now() - '1 day'::interval))
```

```java
// Java
LocalDateTime oneDayAgo = LocalDateTime.now().minusDays(1);
jdbcTemplate.query(
    "SELECT * FROM users WHERE created_at >= ?", 
    oneDayAgo
);
```

```sql
-- Теперь это константа для планировщика
SELECT * FROM users 
WHERE created_at >= '2025-11-11 10:00:00';
```

```plaintext
Index Scan using idx_users_created_at on users
  Index Cond: (created_at >= '2025-11-11 10:00:00')
```

```sql
PREPARE recent_users(timestamp) AS
SELECT * FROM users WHERE created_at >= $1;

EXECUTE recent_users(NOW() - INTERVAL '1 day');
```

```sql
-- Индекс только для недавних записей
CREATE INDEX idx_users_recent 
ON users(created_at) 
WHERE created_at >= NOW() - INTERVAL '7 days';
```

```sql
-- NOW() имеет volatility = STABLE (меняется в рамках транзакции)
SELECT provolatile FROM pg_proc WHERE proname = 'now';
-- Result: 's' (stable)

-- Для оптимизации можно использовать IMMUTABLE функции
```

**Ответ: Создание и координация ForkJoin задач дороже выгоды при малом числе элементов**

`parallelStream()` использует **ForkJoin framework**, который разбивает работу на подзадачи и распределяет их по потокам. Для **малых коллекций** (100 элементов) **overhead** (накладные расходы) на создание задач, синхронизацию и координацию потоков **превышает выгоду от параллелизма**.

## Почему это происходит:

### Overhead в parallelStream():

1. **Splitting** — разбиение коллекции на части
2. **Task creation** — создание ForkJoin задач
3. **Thread coordination** — синхронизация между потоками
4. **Context switching** — переключение контекста процессора
5. **Merging results** — объединение результатов

```java
List<Integer> list = IntStream.range(0, 100).boxed().collect(Collectors.toList());

// Sequential stream
long start = System.nanoTime();
list.stream()
    .map(i -> i * 2)
    .collect(Collectors.toList());
long sequential = System.nanoTime() - start;

// Parallel stream
start = System.nanoTime();
list.parallelStream()
    .map(i -> i * 2)
    .collect(Collectors.toList());
long parallel = System.nanoTime() - start;

System.out.println("Sequential: " + sequential);
System.out.println("Parallel: " + parallel);
// Parallel обычно медленнее для 100 элементов!
```

## Когда parallelStream() эффективен:

### ✅ Используй parallel когда:
- **Большой объём данных** (тысячи/миллионы элементов)
- **CPU-intensive операции** (сложные вычисления)
- **Независимые операции** (нет shared state)

```java
// Хороший случай для parallel
List<Integer> bigList = IntStream.range(0, 1_000_000).boxed().toList();
bigList.parallelStream()
    .map(i -> expensiveComputation(i))  // Тяжёлая операция
    .collect(Collectors.toList());
```

### ❌ НЕ используй parallel когда:
- **Малое количество элементов** (<1000)
- **Быстрые операции** (простая арифметика)
- **I/O операции** (БД, сеть)
- **Shared mutable state** (требует синхронизации)

## Сравнение накладных расходов:

```java
// 100 элементов, простая операция
list.stream().map(i -> i * 2)           // ~0.1ms
list.parallelStream().map(i -> i * 2)   // ~2ms (overhead!)

// 1_000_000 элементов, сложная операция
bigList.stream().map(i -> complexCalc(i))          // ~1000ms
bigList.parallelStream().map(i -> complexCalc(i))  // ~250ms (выгода!)
```

## ForkJoin overhead:

```java
// Что происходит внутри parallelStream()
ForkJoinPool.commonPool()  // Получение пула (есть cost)
    .invoke(
        new RecursiveTask() {  // Создание задачи
            protected compute() {
                if (size < threshold) {
                    // Базовый случай
                } else {
                    fork();  // Разделение на подзадачи (cost!)
                    join();  // Ожидание и объединение (cost!)
                }
            }
        }
    );
```

## Почему не другие варианты:

- **"Коллекция обрабатывается последовательно"** — нет, parallel обрабатывает параллельно, но с overhead
- **"GC хуже справляется"** — GC может влиять, но не основная причина
- **"Количество элементов слишком мало, накладные расходы выше"** — **ДА, это правильный ответ**

**Вывод: для 100 элементов overhead на ForkJoin (splitting, coordination, merging) превышает выгоду от параллелизма. `parallelStream()` эффективен только на больших объёмах данных с тяжёлыми вычислениями.**

```java
List<Integer> list = IntStream.range(0, 100).boxed().collect(Collectors.toList());

// Sequential stream
long start = System.nanoTime();
list.stream()
    .map(i -> i * 2)
    .collect(Collectors.toList());
long sequential = System.nanoTime() - start;

// Parallel stream
start = System.nanoTime();
list.parallelStream()
    .map(i -> i * 2)
    .collect(Collectors.toList());
long parallel = System.nanoTime() - start;

System.out.println("Sequential: " + sequential);
System.out.println("Parallel: " + parallel);
// Parallel обычно медленнее для 100 элементов!
```

```java
// Хороший случай для parallel
List<Integer> bigList = IntStream.range(0, 1_000_000).boxed().toList();
bigList.parallelStream()
    .map(i -> expensiveComputation(i))  // Тяжёлая операция
    .collect(Collectors.toList());
```

```java
// 100 элементов, простая операция
list.stream().map(i -> i * 2)           // ~0.1ms
list.parallelStream().map(i -> i * 2)   // ~2ms (overhead!)

// 1_000_000 элементов, сложная операция
bigList.stream().map(i -> complexCalc(i))          // ~1000ms
bigList.parallelStream().map(i -> complexCalc(i))  // ~250ms (выгода!)
```

```java
// Что происходит внутри parallelStream()
ForkJoinPool.commonPool()  // Получение пула (есть cost)
    .invoke(
        new RecursiveTask() {  // Создание задачи
            protected compute() {
                if (size < threshold) {
                    // Базовый случай
                } else {
                    fork();  // Разделение на подзадачи (cost!)
                    join();  // Ожидание и объединение (cost!)
                }
            }
        }
    );
```

**Ответ: Выполнит последовательный просмотр всей таблицы (Seq Scan)**

Без индекса на колонке `status` PostgreSQL вынужден **просканировать все 1 млн строк** последовательно, чтобы найти записи со значением `'PAID'`.

## Что происходит:

```sql
SELECT * FROM orders WHERE status = 'PAID';
```

**План выполнения:**
```
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PAID';

Seq Scan on orders  (cost=0.00..25000.00 rows=50000 width=100)
  Filter: (status = 'PAID'::text)
  Rows Removed by Filter: 950000
Planning Time: 0.1 ms
Execution Time: 1234.5 ms  -- МЕДЛЕННО!
```

PostgreSQL:
1. Читает **каждую строку** из таблицы (1 млн строк)
2. Проверяет условие `status = 'PAID'` для каждой строки
3. Возвращает подходящие строки

## Решение — создать индекс:

```sql
CREATE INDEX idx_orders_status ON orders(status);
```

**После создания индекса:**
```
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PAID';

Bitmap Heap Scan on orders  (cost=500.00..5000.00 rows=50000 width=100)
  Recheck Cond: (status = 'PAID'::text)
  ->  Bitmap Index Scan on idx_orders_status
        Index Cond: (status = 'PAID'::text)
Planning Time: 0.2 ms
Execution Time: 45.3 ms  -- БЫСТРО!
```

## Оптимизация для малого числа значений:

Если `status` имеет мало уникальных значений (например: `PAID`, `PENDING`, `CANCELLED`), используй **partial index** или **expression index**:

### Partial Index (если часто запрашивается один статус):
```sql
-- Индекс только для PAID заказов
CREATE INDEX idx_orders_paid 
ON orders(status) 
WHERE status = 'PAID';
```

### B-tree индекс (стандартный):
```sql
CREATE INDEX idx_orders_status ON orders(status);
```

### Hash индекс (для точного равенства):
```sql
-- Быстрее для `=`, но не для диапазонов
CREATE INDEX idx_orders_status_hash ON orders USING HASH(status);
```

## Почему не другие варианты:

- **"Отклонит запрос"** — нет, выполнит, но медленно
- **"Использует хэш-индекс автоматически"** — нет, индексы нужно создавать явно
- **"Выполнит бинарный поиск по строковому столбцу"** — нет, без индекса нет структуры для бинарного поиска

## Проверка использования индекса:

```sql
-- Убедиться, что индекс используется
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'PAID';
```

**Вывод: без индекса PostgreSQL выполнит Seq Scan (полное сканирование таблицы) — 1 млн строк. Создание индекса ускорит запрос в десятки/сотни раз.**

```sql
SELECT * FROM orders WHERE status = 'PAID';
```

```plaintext
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PAID';

Seq Scan on orders  (cost=0.00..25000.00 rows=50000 width=100)
  Filter: (status = 'PAID'::text)
  Rows Removed by Filter: 950000
Planning Time: 0.1 ms
Execution Time: 1234.5 ms  -- МЕДЛЕННО!
```

```sql
CREATE INDEX idx_orders_status ON orders(status);
```

```plaintext
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'PAID';

Bitmap Heap Scan on orders  (cost=500.00..5000.00 rows=50000 width=100)
  Recheck Cond: (status = 'PAID'::text)
  ->  Bitmap Index Scan on idx_orders_status
        Index Cond: (status = 'PAID'::text)
Planning Time: 0.2 ms
Execution Time: 45.3 ms  -- БЫСТРО!
```

```sql
-- Индекс только для PAID заказов
CREATE INDEX idx_orders_paid 
ON orders(status) 
WHERE status = 'PAID';
```

```sql
CREATE INDEX idx_orders_status ON orders(status);
```

```sql
-- Быстрее для `=`, но не для диапазонов
CREATE INDEX idx_orders_status_hash ON orders USING HASH(status);
```

```sql
-- Убедиться, что индекс используется
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'PAID';
```

**Ответ: Использовать примитивные коллекции (например, IntStream или сторонние библиотеки как Trove, FastUtil)**

Автоупаковка `int → Integer` создаёт объекты в heap, что значительно увеличивает потребление памяти. **Примитивные коллекции** избегают этого overhead.

## Проблема autoboxing:

```java
// С автоупаковкой
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // int → Integer (1 млн объектов!)
}
// Память: ~40-50 MB
```

**Overhead Integer:**
- `int` — 4 байта
- `Integer` — 16 байт (заголовок объекта) + 4 байта (значение) = **20+ байт**
- Плюс overhead коллекции (ссылки, capacity)

## Решения:

### 1. Примитивные библиотеки (лучший вариант):

#### Eclipse Collections:
```java
import org.eclipse.collections.impl.list.mutable.primitive.IntArrayList;

IntArrayList list = new IntArrayList();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // Без boxing!
}
// Память: ~4 MB (в 10 раз меньше!)
```

#### Trove:
```java
import gnu.trove.list.array.TIntArrayList;

TIntArrayList list = new TIntArrayList();
list.add(42);  // Примитив
```

#### FastUtil:
```java
import it.unimi.dsi.fastutil.ints.IntArrayList;

IntArrayList list = new IntArrayList();
list.add(42);
```

### 2. Использовать массивы (если размер известен):
```java
int[] array = new int[1_000_000];
for (int i = 0; i < array.length; i++) {
    array[i] = i;  // Без boxing
}
// Память: ~4 MB
```

### 3. IntStream для обработки (не для хранения):
```java
// Для вычислений без создания коллекции
int sum = IntStream.range(0, 1_000_000)
    .filter(i -> i % 2 == 0)
    .sum();
```

## Сравнение потребления памяти:

```java
// ArrayList<Integer>
List<Integer> boxed = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    boxed.add(i);
}
// ~40-50 MB

// IntArrayList (Eclipse Collections)
IntArrayList primitive = new IntArrayList();
for (int i = 0; i < 1_000_000; i++) {
    primitive.add(i);
}
// ~4 MB (в 10 раз меньше!)

// int[]
int[] array = new int[1_000_000];
// ~4 MB
```

## Почему не другие варианты:

- **"OptionalInt вместо Integer"** — `OptionalInt` для возврата значений, не для коллекций
- **"Заменить Integer на Long"** — `Long` занимает ещё больше памяти (24+ байт)
- **"AtomicInteger для экономии"** — `AtomicInteger` ещё тяжелее (thread-safe overhead)

## Зависимости:

### Eclipse Collections (Maven):
```xml
<dependency>
    <groupId>org.eclipse.collections</groupId>
    <artifactId>eclipse-collections</artifactId>
    <version>11.1.0</version>
</dependency>
```

### Trove:
```xml
<dependency>
    <groupId>net.sf.trove4j</groupId>
    <artifactId>trove4j</artifactId>
    <version>3.0.3</version>
</dependency>
```

### FastUtil:
```xml
<dependency>
    <groupId>it.unimi.dsi</groupId>
    <artifactId>fastutil</artifactId>
    <version>8.5.12</version>
</dependency>
```

**Вывод: примитивные коллекции (Eclipse Collections, Trove, FastUtil) избегают autoboxing и экономят ~90% памяти по сравнению с `List<Integer>`.**

```java
// С автоупаковкой
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // int → Integer (1 млн объектов!)
}
// Память: ~40-50 MB
```

```java
import org.eclipse.collections.impl.list.mutable.primitive.IntArrayList;

IntArrayList list = new IntArrayList();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // Без boxing!
}
// Память: ~4 MB (в 10 раз меньше!)
```

```java
import gnu.trove.list.array.TIntArrayList;

TIntArrayList list = new TIntArrayList();
list.add(42);  // Примитив
```

```java
import it.unimi.dsi.fastutil.ints.IntArrayList;

IntArrayList list = new IntArrayList();
list.add(42);
```

```java
int[] array = new int[1_000_000];
for (int i = 0; i < array.length; i++) {
    array[i] = i;  // Без boxing
}
// Память: ~4 MB
```

```java
// Для вычислений без создания коллекции
int sum = IntStream.range(0, 1_000_000)
    .filter(i -> i % 2 == 0)
    .sum();
```

```java
// ArrayList<Integer>
List<Integer> boxed = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    boxed.add(i);
}
// ~40-50 MB

// IntArrayList (Eclipse Collections)
IntArrayList primitive = new IntArrayList();
for (int i = 0; i < 1_000_000; i++) {
    primitive.add(i);
}
// ~4 MB (в 10 раз меньше!)

// int[]
int[] array = new int[1_000_000];
// ~4 MB
```

```xml
<dependency>
    <groupId>org.eclipse.collections</groupId>
    <artifactId>eclipse-collections</artifactId>
    <version>11.1.0</version>
</dependency>
```

```xml
<dependency>
    <groupId>net.sf.trove4j</groupId>
    <artifactId>trove4j</artifactId>
    <version>3.0.3</version>
</dependency>
```

```xml
<dependency>
    <groupId>it.unimi.dsi</groupId>
    <artifactId>fastutil</artifactId>
    <version>8.5.12</version>
</dependency>
```

**Ответ: Будет выброшено `IllegalStateException: stream has already been operated upon or closed`**

Stream в Java можно использовать **только один раз**. После выполнения терминальной операции (например, `forEach`) stream **закрывается** и больше не может быть использован.

## Что происходит:

```java
Stream<String> stream = Stream.of("a", "b", "c");

stream.forEach(System.out::println);  // 1-й вызов - OK
// Stream закрыт после forEach

stream.forEach(System.out::println);  // 2-й вызов - EXCEPTION!
// IllegalStateException: stream has already been operated upon or closed
```

## Почему так устроено:

Stream — это **одноразовый конвейер данных**, а не коллекция:

```java
// Stream != Коллекция
List<String> list = List.of("a", "b", "c");
list.forEach(System.out::println);  // OK
list.forEach(System.out::println);  // OK - можно много раз

Stream<String> stream = list.stream();
stream.forEach(System.out::println);  // OK
stream.forEach(System.out::println);  // EXCEPTION - только 1 раз!
```

## Терминальные операции (закрывают stream):

```java
stream.forEach(...)     // Закрывает
stream.collect(...)     // Закрывает
stream.count()          // Закрывает
stream.findFirst()      // Закрывает
stream.reduce(...)      // Закрывает
stream.anyMatch(...)    // Закрывает
```

## Решение — создать новый stream:

```java
List<String> list = List.of("a", "b", "c");

// Создаём stream каждый раз заново
list.stream().forEach(System.out::println);  // OK
list.stream().forEach(System.out::println);  // OK
list.stream().forEach(System.out::println);  // OK
```

## Или сохранить результат, а не stream:

```java
List<String> list = Stream.of("a", "b", "c")
    .filter(s -> s.length() > 1)
    .collect(Collectors.toList());  // Сохранили результат

// Теперь можно использовать много раз
list.forEach(System.out::println);
list.forEach(System.out::println);
```

## Пример с ошибкой:

```java
Stream<Integer> numbers = Stream.of(1, 2, 3, 4, 5);

long count = numbers.count();  // Терминальная операция
System.out.println(count);     // 5

// Попытка повторно использовать
numbers.forEach(System.out::println);  
// IllegalStateException: stream has already been operated upon or closed
```

## Почему не другие варианты:

- **"Поток автоматически пересоздаётся"** — нет, нужно создавать вручную
- **"Поток будет корректно обработан несколько раз"** — нет, только один раз
- **"Второй вызов проигнорируется"** — нет, выбросит exception

**Вывод: Stream одноразовый. После терминальной операции (forEach, collect и т.д.) повторное использование вызовет `IllegalStateException`.**

```java
Stream<String> stream = Stream.of("a", "b", "c");

stream.forEach(System.out::println);  // 1-й вызов - OK
// Stream закрыт после forEach

stream.forEach(System.out::println);  // 2-й вызов - EXCEPTION!
// IllegalStateException: stream has already been operated upon or closed
```

```java
// Stream != Коллекция
List<String> list = List.of("a", "b", "c");
list.forEach(System.out::println);  // OK
list.forEach(System.out::println);  // OK - можно много раз

Stream<String> stream = list.stream();
stream.forEach(System.out::println);  // OK
stream.forEach(System.out::println);  // EXCEPTION - только 1 раз!
```

```java
stream.forEach(...)     // Закрывает
stream.collect(...)     // Закрывает
stream.count()          // Закрывает
stream.findFirst()      // Закрывает
stream.reduce(...)      // Закрывает
stream.anyMatch(...)    // Закрывает
```

```java
List<String> list = List.of("a", "b", "c");

// Создаём stream каждый раз заново
list.stream().forEach(System.out::println);  // OK
list.stream().forEach(System.out::println);  // OK
list.stream().forEach(System.out::println);  // OK
```

```java
List<String> list = Stream.of("a", "b", "c")
    .filter(s -> s.length() > 1)
    .collect(Collectors.toList());  // Сохранили результат

// Теперь можно использовать много раз
list.forEach(System.out::println);
list.forEach(System.out::println);
```

```java
Stream<Integer> numbers = Stream.of(1, 2, 3, 4, 5);

long count = numbers.count();  // Терминальная операция
System.out.println(count);     // 5

// Попытка повторно использовать
numbers.forEach(System.out::println);  
// IllegalStateException: stream has already been operated upon or closed
```

**Ответ: LAZY-инициализация коллекции требует открытой сессии, иначе LazyInitializationException**

По умолчанию связи `@OneToMany` имеют **LAZY loading** — коллекция загружается только при первом обращении. Если сессия Hibernate закрыта (транзакция завершена), обращение к коллекции вызовет `LazyInitializationException`.

## Что происходит:

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")  // По умолчанию LAZY
    private List<Order> orders;
}

// В сервисе
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
    // Транзакция закрывается здесь
}

// В контроллере (вне транзакции)
User user = userService.getUser(1L);
int size = user.getOrders().size();  // LazyInitializationException!
// failed to lazily initialize a collection of role: User.orders, 
// could not initialize proxy - no Session
```

## Решения:

### 1. Явная загрузка (EAGER):
```java
@OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
private List<Order> orders;
```
⚠️ **Осторожно**: загружает всегда, даже когда не нужно (N+1 problem).

### 2. JOIN FETCH в запросе (рекомендуется):
```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

### 3. @Transactional на уровне контроллера/сервиса:
```java
@Transactional(readOnly = true)
public int getUserOrdersCount(Long userId) {
    User user = userRepository.findById(userId).orElseThrow();
    return user.getOrders().size();  // Сессия открыта - OK
}
```

### 4. Entity Graph:
```java
@EntityGraph(attributePaths = {"orders"})
@Query("SELECT u FROM User u WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

### 5. Hibernate.initialize():
```java
@Transactional
public User getUserWithOrders(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    Hibernate.initialize(user.getOrders());  // Принудительная загрузка
    return user;
}
```

## Почему size() требует загрузки:

```java
// size() - это метод List, не прокси
user.getOrders().size();  

// Hibernate должен загрузить коллекцию из БД:
// SELECT * FROM orders WHERE user_id = ?
```

## Почему не другие варианты:

- **"Репозиторий не подгружает автоматически"** — частично верно, но причина именно в LAZY + закрытой сессии
- **"Коллекция всегда пустая без транзакции"** — нет, вызовет exception, не вернёт пустую
- **"Метод size() запрещён в JPA"** — нет, разрешён, но требует инициализации

## Проверка lazy/eager:

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;  // LAZY по умолчанию
    
    // Проверить инициализацию
    public boolean isOrdersLoaded() {
        return Hibernate.isInitialized(orders);
    }
}
```

**Вывод: `@OneToMany` по умолчанию LAZY → коллекция загружается при обращении → требует открытой сессии → вне транзакции `size()` вызовет `LazyInitializationException`.**

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")  // По умолчанию LAZY
    private List<Order> orders;
}

// В сервисе
@Transactional
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
    // Транзакция закрывается здесь
}

// В контроллере (вне транзакции)
User user = userService.getUser(1L);
int size = user.getOrders().size();  // LazyInitializationException!
// failed to lazily initialize a collection of role: User.orders, 
// could not initialize proxy - no Session
```

```java
@OneToMany(mappedBy = "user", fetch = FetchType.EAGER)
private List<Order> orders;
```

```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

```java
@Transactional(readOnly = true)
public int getUserOrdersCount(Long userId) {
    User user = userRepository.findById(userId).orElseThrow();
    return user.getOrders().size();  // Сессия открыта - OK
}
```

```java
@EntityGraph(attributePaths = {"orders"})
@Query("SELECT u FROM User u WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

```java
@Transactional
public User getUserWithOrders(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    Hibernate.initialize(user.getOrders());  // Принудительная загрузка
    return user;
}
```

```java
// size() - это метод List, не прокси
user.getOrders().size();  

// Hibernate должен загрузить коллекцию из БД:
// SELECT * FROM orders WHERE user_id = ?
```

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    private List<Order> orders;  // LAZY по умолчанию
    
    // Проверить инициализацию
    public boolean isOrdersLoaded() {
        return Hibernate.isInitialized(orders);
    }
}
```

**Ответ: Приложение не стартует, если не включен fail-fast режим**

По умолчанию Spring Cloud Config Client имеет **`spring.cloud.config.fail-fast=false`**, что означает: если Config Server недоступен, приложение **стартует с локальными настройками** (из `application.yml`/`application.properties`).

Но если включить **fail-fast**, приложение **не стартует** при недоступности Config Server.

## Поведение по умолчанию (fail-fast=false):

```yaml
# bootstrap.yml или application.yml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: false  # По умолчанию
```

**Что происходит:**
1. Приложение пытается подключиться к Config Server
2. Если не получается — **логирует WARNING**
3. **Продолжает запуск** с локальными настройками

```
WARN: Could not locate PropertySource: I/O error on GET request 
for "http://config-server:8888/...": Connection refused
INFO: Starting application with local configuration
```

## С fail-fast=true:

```yaml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true  # Строгий режим
```

**Что происходит:**
1. Приложение пытается подключиться к Config Server
2. Если не получается — **выбрасывает exception**
3. **Приложение НЕ стартует**

```
ERROR: Could not locate PropertySource and the fail fast property is set
java.lang.IllegalStateException: Could not locate PropertySource 
and the fail fast property is set, failing
```

## Настройки retry (повторные попытки):

```yaml
spring:
  cloud:
    config:
      fail-fast: true
      retry:
        initial-interval: 1000  # 1 секунда
        max-attempts: 6         # 6 попыток
        max-interval: 2000      # Макс 2 секунды между попытками
        multiplier: 1.1         # Множитель задержки
```

С retry приложение будет пытаться подключиться несколько раз перед падением.

## Альтернатива — Config Client игнорирует ошибки:

```yaml
spring:
  cloud:
    config:
      enabled: false  # Полностью отключить Config Client
```

Или через property:
```bash
java -jar app.jar --spring.cloud.config.enabled=false
```

## Рекомендация для production:

```yaml
# bootstrap.yml
spring:
  application:
    name: my-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true   # Строго требовать Config Server
      retry:
        max-attempts: 10
        initial-interval: 2000
```

**+ Health check в Kubernetes/Docker:**
```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60  # Дать время на retry
```

## Почему не другие варианты:

- **"Config Client игнорирует при ошибке сети"** — да, но **только если fail-fast=false** (по умолчанию)
- **"Приложение загрузится с пустыми свойствами"** — нет, загрузится с локальными (из application.yml)
- **"Приложение стартует всегда"** — нет, зависит от fail-fast

**Вывод: по умолчанию (fail-fast=false) приложение стартует с локальными настройками. С fail-fast=true приложение упадёт, если Config Server недоступен.**

```yaml
# bootstrap.yml или application.yml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: false  # По умолчанию
```

```plaintext
WARN: Could not locate PropertySource: I/O error on GET request 
for "http://config-server:8888/...": Connection refused
INFO: Starting application with local configuration
```

```yaml
spring:
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true  # Строгий режим
```

```plaintext
ERROR: Could not locate PropertySource and the fail fast property is set
java.lang.IllegalStateException: Could not locate PropertySource 
and the fail fast property is set, failing
```

```yaml
spring:
  cloud:
    config:
      fail-fast: true
      retry:
        initial-interval: 1000  # 1 секунда
        max-attempts: 6         # 6 попыток
        max-interval: 2000      # Макс 2 секунды между попытками
        multiplier: 1.1         # Множитель задержки
```

```yaml
spring:
  cloud:
    config:
      enabled: false  # Полностью отключить Config Client
```

```shellscript
java -jar app.jar --spring.cloud.config.enabled=false
```

```yaml
# bootstrap.yml
spring:
  application:
    name: my-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true   # Строго требовать Config Server
      retry:
        max-attempts: 10
        initial-interval: 2000
```

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 60  # Дать время на retry
```

**Ответ: Настроить ErrorHandler и Dead Letter Topic для проблемных сообщений**

При ошибках парсинга (например, невалидный JSON) нужно **обработать exception** и **отправить проблемное сообщение в Dead Letter Topic (DLT)**, чтобы не блокировать обработку остальных сообщений.

## Решение:

### 1. ErrorHandler с DLT (рекомендуется):

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        
        // Настройка Dead Letter Publishing
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", -1) // DLT топик
        );
        
        // Error Handler с retry и DLT
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3L)  // 3 попытки с интервалом 1 сек
        );
        
        // Не ретраить для определенных ошибок (например, парсинг)
        errorHandler.addNotRetryableExceptions(
            JsonProcessingException.class,
            DeserializationException.class
        );
        
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

### 2. Consumer с обработкой ошибок:

```java
@Service
public class OrderConsumer {
    
    private static final Logger log = LoggerFactory.getLogger(OrderConsumer.class);
    private final ObjectMapper objectMapper;
    
    @KafkaListener(topics = "orders", groupId = "order-service")
    public void consume(String message, 
                       @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            Order order = objectMapper.readValue(message, Order.class);
            processOrder(order);
            
        } catch (JsonProcessingException e) {
            log.error("Failed to parse message at offset {}: {}", offset, message, e);
            // ErrorHandler автоматически отправит в DLT
            throw new RuntimeException("Parsing failed", e);
        }
    }
    
    // Consumer для Dead Letter Topic
    @KafkaListener(topics = "orders.DLT", groupId = "order-service-dlt")
    public void consumeDeadLetter(String message,
                                 @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                 @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {
        log.error("Message in DLT: {} | Error: {}", message, exceptionMessage);
        // Сохранить в БД для ручной обработки, отправить алерт и т.д.
        saveToDatabase(message, exceptionMessage);
    }
}
```

### 3. Альтернатива — try-catch без остановки:

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(String message) {
    try {
        Order order = objectMapper.readValue(message, Order.class);
        processOrder(order);
        
    } catch (JsonProcessingException e) {
        log.error("Failed to parse JSON, skipping message: {}", message, e);
        // Сообщение пропущено, offset закоммичен
        // Можно отправить в отдельный топик вручную
        kafkaTemplate.send("orders.error", message);
    }
}
```

### 4. Настройка application.yml:

```yaml
spring:
  kafka:
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false  # Ручной контроль commit
      properties:
        spring.json.trusted.packages: "*"
    listener:
      ack-mode: record  # Коммитить каждую запись отдельно
```

## Стратегии обработки ошибок:

### 1. **Skip** (пропустить):
```java
// Логировать и продолжить
catch (Exception e) {
    log.error("Error", e);
    // Offset автоматически закоммитится
}
```

### 2. **Retry + DLT**:
```java
// Несколько попыток, потом в DLT
DefaultErrorHandler errorHandler = new DefaultErrorHandler(
    recoverer,
    new FixedBackOff(1000L, 3L)
);
```

### 3. **Stop** (остановить consumer):
```java
// Для критичных ошибок
catch (CriticalException e) {
    throw e;  // Consumer остановится
}
```

### 4. **Manual commit**:
```java
@KafkaListener(topics = "orders")
public void consume(String message, Acknowledgment ack) {
    try {
        processMessage(message);
        ack.acknowledge();  // Коммитим только если успех
    } catch (Exception e) {
        log.error("Error", e);
        // Не коммитим, сообщение будет перечитано
    }
}
```

## Почему не другие варианты:

- **"ackMode=RECORD для ручного commit"** — полезно, но не решает проблему парсинга
- **"Прервать работу консьюмера"** — плохо, все сообщения остановятся
- **"Поймать Exception и игнорировать"** — можно, но лучше отправить в DLT для анализа

**Вывод: ErrorHandler + Dead Letter Topic — лучшая практика. Проблемные сообщения попадут в DLT для анализа, а обработка остальных продолжится.**

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(
            ConsumerFactory<String, String> consumerFactory,
            KafkaTemplate<String, String> kafkaTemplate) {
        
        ConcurrentKafkaListenerContainerFactory<String, String> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        
        // Настройка Dead Letter Publishing
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            kafkaTemplate,
            (record, ex) -> new TopicPartition(record.topic() + ".DLT", -1) // DLT топик
        );
        
        // Error Handler с retry и DLT
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            recoverer,
            new FixedBackOff(1000L, 3L)  // 3 попытки с интервалом 1 сек
        );
        
        // Не ретраить для определенных ошибок (например, парсинг)
        errorHandler.addNotRetryableExceptions(
            JsonProcessingException.class,
            DeserializationException.class
        );
        
        factory.setCommonErrorHandler(errorHandler);
        return factory;
    }
}
```

```java
@Service
public class OrderConsumer {
    
    private static final Logger log = LoggerFactory.getLogger(OrderConsumer.class);
    private final ObjectMapper objectMapper;
    
    @KafkaListener(topics = "orders", groupId = "order-service")
    public void consume(String message, 
                       @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                       @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            Order order = objectMapper.readValue(message, Order.class);
            processOrder(order);
            
        } catch (JsonProcessingException e) {
            log.error("Failed to parse message at offset {}: {}", offset, message, e);
            // ErrorHandler автоматически отправит в DLT
            throw new RuntimeException("Parsing failed", e);
        }
    }
    
    // Consumer для Dead Letter Topic
    @KafkaListener(topics = "orders.DLT", groupId = "order-service-dlt")
    public void consumeDeadLetter(String message,
                                 @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                 @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {
        log.error("Message in DLT: {} | Error: {}", message, exceptionMessage);
        // Сохранить в БД для ручной обработки, отправить алерт и т.д.
        saveToDatabase(message, exceptionMessage);
    }
}
```

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void consume(String message) {
    try {
        Order order = objectMapper.readValue(message, Order.class);
        processOrder(order);
        
    } catch (JsonProcessingException e) {
        log.error("Failed to parse JSON, skipping message: {}", message, e);
        // Сообщение пропущено, offset закоммичен
        // Можно отправить в отдельный топик вручную
        kafkaTemplate.send("orders.error", message);
    }
}
```

```yaml
spring:
  kafka:
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false  # Ручной контроль commit
      properties:
        spring.json.trusted.packages: "*"
    listener:
      ack-mode: record  # Коммитить каждую запись отдельно
```

```java
// Логировать и продолжить
catch (Exception e) {
    log.error("Error", e);
    // Offset автоматически закоммитится
}
```

```java
// Несколько попыток, потом в DLT
DefaultErrorHandler errorHandler = new DefaultErrorHandler(
    recoverer,
    new FixedBackOff(1000L, 3L)
);
```

```java
// Для критичных ошибок
catch (CriticalException e) {
    throw e;  // Consumer остановится
}
```

```java
@KafkaListener(topics = "orders")
public void consume(String message, Acknowledgment ack) {
    try {
        processMessage(message);
        ack.acknowledge();  // Коммитим только если успех
    } catch (Exception e) {
        log.error("Error", e);
        // Не коммитим, сообщение будет перечитано
    }
}
```

**Ответ: Операция `counter++` неатомарная: чтение + инкремент + запись происходят без блокировки**

`volatile` **НЕ делает операцию `counter++` атомарной**. Это составная операция из трёх шагов, и между ними могут вклиниться другие потоки.

## Что происходит:

```java
private volatile int counter = 0;

public void increment() {
    counter++;  // НЕ атомарная операция!
}
```

### Декомпозиция `counter++`:

```java
// counter++ распадается на 3 операции:
1. int temp = counter;      // Читаем текущее значение
2. temp = temp + 1;         // Инкрементируем
3. counter = temp;          // Записываем обратно
```

### Проблема race condition:

```
Thread 1                    Thread 2                    counter
-----------                 -----------                 -------
1. читает counter (0)                                   0
                            1. читает counter (0)       0
2. инкремент (0 + 1 = 1)                                0
                            2. инкремент (0 + 1 = 1)    0
3. записывает 1                                         1
                            3. записывает 1             1

Результат: counter = 1 (должно быть 2!)
```

## Что гарантирует `volatile`:

✅ **Видимость** — изменения видны всем потокам немедленно (без кэширования в CPU)
✅ **Happens-before** — запись происходит раньше чтения

❌ **НЕ гарантирует атомарность** составных операций

## Решения:

### 1. AtomicInteger (лучшее для счётчиков):

```java
private final AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();  // Атомарно!
}
```

### 2. synchronized:

```java
private int counter = 0;

public synchronized void increment() {
    counter++;  // Атомарно благодаря блокировке
}
```

### 3. ReentrantLock:

```java
private int counter = 0;
private final Lock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock();
    }
}
```

### 4. LongAdder (для высокой конкуренции):

```java
private final LongAdder counter = new LongAdder();

public void increment() {
    counter.increment();  // Очень быстро в многопоточной среде
}

public long getCount() {
    return counter.sum();
}
```

## Пример потери инкремента:

```java
class Counter {
    private volatile int counter = 0;
    
    public void increment() {
        counter++;
    }
}

// Тест
Counter counter = new Counter();
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> counter.increment());
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.MINUTES);

System.out.println(counter.counter);
// Ожидается: 1000
// Реально: ~970 (потеряны инкременты из-за race condition)
```

## Когда `volatile` достаточно:

```java
// Простое чтение/запись - OK
private volatile boolean flag = false;

public void setFlag(boolean value) {
    flag = value;  // Атомарная операция
}

public boolean getFlag() {
    return flag;   // Атомарная операция
}
```

## Почему не другие варианты:

- **"volatile делает инкремент атомарным"** — НЕТ, только read/write отдельных примитивов
- **"JIT оптимизация игнорирует volatile"** — нет, JIT учитывает volatile
- **"volatile гарантирует только кэширование"** — гарантирует видимость, но не атомарность

**Вывод: `counter++` состоит из 3 операций (read-modify-write), `volatile` НЕ делает это атомарным. Используй `AtomicInteger` или `synchronized`.**

```java
private volatile int counter = 0;

public void increment() {
    counter++;  // НЕ атомарная операция!
}
```

```java
// counter++ распадается на 3 операции:
1. int temp = counter;      // Читаем текущее значение
2. temp = temp + 1;         // Инкрементируем
3. counter = temp;          // Записываем обратно
```

```plaintext
Thread 1                    Thread 2                    counter
-----------                 -----------                 -------
1. читает counter (0)                                   0
                            1. читает counter (0)       0
2. инкремент (0 + 1 = 1)                                0
                            2. инкремент (0 + 1 = 1)    0
3. записывает 1                                         1
                            3. записывает 1             1

Результат: counter = 1 (должно быть 2!)
```

```java
private final AtomicInteger counter = new AtomicInteger(0);

public void increment() {
    counter.incrementAndGet();  // Атомарно!
}
```

```java
private int counter = 0;

public synchronized void increment() {
    counter++;  // Атомарно благодаря блокировке
}
```

```java
private int counter = 0;
private final Lock lock = new ReentrantLock();

public void increment() {
    lock.lock();
    try {
        counter++;
    } finally {
        lock.unlock();
    }
}
```

```java
private final LongAdder counter = new LongAdder();

public void increment() {
    counter.increment();  // Очень быстро в многопоточной среде
}

public long getCount() {
    return counter.sum();
}
```

```java
class Counter {
    private volatile int counter = 0;
    
    public void increment() {
        counter++;
    }
}

// Тест
Counter counter = new Counter();
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 1000; i++) {
    executor.submit(() -> counter.increment());
}

executor.shutdown();
executor.awaitTermination(1, TimeUnit.MINUTES);

System.out.println(counter.counter);
// Ожидается: 1000
// Реально: ~970 (потеряны инкременты из-за race condition)
```

```java
// Простое чтение/запись - OK
private volatile boolean flag = false;

public void setFlag(boolean value) {
    flag = value;  // Атомарная операция
}

public boolean getFlag() {
    return flag;   // Атомарная операция
}
```

**Ответ: LAZY-загрузка связанных сущностей может вызвать LazyInitializationException**

Когда метод контроллера возвращает Entity напрямую, Spring пытается **сериализовать её в JSON**. Если у `User` есть LAZY-связи (например, `@OneToMany`), при попытке сериализации Jackson попытается обратиться к этим коллекциям **вне транзакции** → `LazyInitializationException`.

## Проблема:

```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "user")  // LAZY по умолчанию
    private List<Order> orders;
}

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return repo.findById(id).orElseThrow();
    // Транзакция закрывается здесь
}
// Jackson пытается сериализовать User → обращается к orders → LazyInitializationException!
```

### Исключение:

```
com.fasterxml.jackson.databind.JsonMappingException: 
failed to lazily initialize a collection of role: User.orders, 
could not initialize proxy - no Session
```

## Решения:

### 1. DTO (рекомендуется):

```java
public record UserDTO(Long id, String name) {}

@GetMapping("/users/{id}")
public UserDTO getUser(@PathVariable Long id) {
    User user = repo.findById(id).orElseThrow();
    return new UserDTO(user.getId(), user.getName());
    // Сериализация только нужных полей, без LAZY-связей
}
```

### 2. @JsonIgnore на LAZY-свойствах:

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    @JsonIgnore  // Исключить из сериализации
    private List<Order> orders;
}
```

### 3. @Transactional на контроллере (плохая практика):

```java
@GetMapping("/users/{id}")
@Transactional(readOnly = true)  // Держим сессию открытой
public User getUser(@PathVariable Long id) {
    return repo.findById(id).orElseThrow();
    // Транзакция закроется после сериализации
}
```

⚠️ **Не рекомендуется**: транзакция на контроллере — anti-pattern.

### 4. Open Session in View (по умолчанию включено в Spring Boot):

```yaml
spring:
  jpa:
    open-in-view: true  # По умолчанию true
```

Держит Hibernate сессию открытой до завершения HTTP-ответа. Решает проблему, но может вызвать N+1 queries и другие performance issues.

### 5. Явная загрузка с JOIN FETCH:

```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return repo.findByIdWithOrders(id).orElseThrow();
    // orders загружены заранее
}
```

### 6. @ResponseBody + projection:

```java
public interface UserProjection {
    Long getId();
    String getName();
    // Не включаем orders
}

@GetMapping("/users/{id}")
public UserProjection getUser(@PathVariable Long id) {
    return repo.findById(id, UserProjection.class).orElseThrow();
}
```

## Почему не другие варианты:

- **"Репозиторий возвращает detached entity"** — да, но проблема в LAZY-загрузке при сериализации
- **"Метод должен возвращать DTO"** — правильно, это лучшее решение
- **"Всегда требуется @ResponseBody"** — нет, `@RestController` уже включает это

## Best practice:

```java
// Service
@Service
public class UserService {
    @Transactional(readOnly = true)
    public UserDTO getUser(Long id) {
        User user = repo.findById(id).orElseThrow();
        return UserDTO.fromEntity(user);  // Конвертация в DTO
    }
}

// Controller
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.getUser(id);  // Возвращаем DTO
    }
}

// DTO
public record UserDTO(Long id, String name) {
    public static UserDTO fromEntity(User user) {
        return new UserDTO(user.getId(), user.getName());
    }
}
```

**Вывод: при сериализации Entity с LAZY-связями Jackson попытается обратиться к ним вне транзакции → `LazyInitializationException`. Решение: используй DTO или явно загружай связи с JOIN FETCH.**

```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "user")  // LAZY по умолчанию
    private List<Order> orders;
}

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return repo.findById(id).orElseThrow();
    // Транзакция закрывается здесь
}
// Jackson пытается сериализовать User → обращается к orders → LazyInitializationException!
```

```plaintext
com.fasterxml.jackson.databind.JsonMappingException: 
failed to lazily initialize a collection of role: User.orders, 
could not initialize proxy - no Session
```

```java
public record UserDTO(Long id, String name) {}

@GetMapping("/users/{id}")
public UserDTO getUser(@PathVariable Long id) {
    User user = repo.findById(id).orElseThrow();
    return new UserDTO(user.getId(), user.getName());
    // Сериализация только нужных полей, без LAZY-связей
}
```

```java
@Entity
public class User {
    @OneToMany(mappedBy = "user")
    @JsonIgnore  // Исключить из сериализации
    private List<Order> orders;
}
```

```java
@GetMapping("/users/{id}")
@Transactional(readOnly = true)  // Держим сессию открытой
public User getUser(@PathVariable Long id) {
    return repo.findById(id).orElseThrow();
    // Транзакция закроется после сериализации
}
```

```yaml
spring:
  jpa:
    open-in-view: true  # По умолчанию true
```

```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);

@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return repo.findByIdWithOrders(id).orElseThrow();
    // orders загружены заранее
}
```

```java
public interface UserProjection {
    Long getId();
    String getName();
    // Не включаем orders
}

@GetMapping("/users/{id}")
public UserProjection getUser(@PathVariable Long id) {
    return repo.findById(id, UserProjection.class).orElseThrow();
}
```

```java
// Service
@Service
public class UserService {
    @Transactional(readOnly = true)
    public UserDTO getUser(Long id) {
        User user = repo.findById(id).orElseThrow();
        return UserDTO.fromEntity(user);  // Конвертация в DTO
    }
}

// Controller
@RestController
public class UserController {
    @GetMapping("/users/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.getUser(id);  // Возвращаем DTO
    }
}

// DTO
public record UserDTO(Long id, String name) {
    public static UserDTO fromEntity(User user) {
        return new UserDTO(user.getId(), user.getName());
    }
}
```

**Ответ: Оно загружает все зависимости сразу, что может привести к неоптимальным запросам**

`FetchType.EAGER` загружает **все связанные сущности немедленно**, даже если они не нужны. Это приводит к:

1. **N+1 problem** — множественные запросы к БД
2. **Избыточная загрузка данных** — загружается то, что не используется
3. **Картезианское произведение** — при нескольких EAGER-связях

## Проблема N+1:

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)  // ❌ EAGER
    private List<Order> orders;
}

// Запрос
List<User> users = userRepository.findAll();
```

**SQL запросы:**
```sql
-- 1 запрос для users
SELECT * FROM users;  

-- N запросов для orders (для каждого user)
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
SELECT * FROM orders WHERE user_id = 3;
...
SELECT * FROM orders WHERE user_id = 100;

-- Итого: 1 + 100 = 101 запрос!
```

## Проблема картезианского произведения:

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)
    private List<Order> orders;  // 10 заказов
    
    @OneToMany(fetch = FetchType.EAGER)
    private List<Address> addresses;  // 5 адресов
}

User user = userRepository.findById(1L);
```

**SQL (с JOIN):**
```sql
SELECT * FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN addresses a ON u.id = a.user_id
WHERE u.id = 1;

-- Результат: 10 * 5 = 50 строк (картезианское произведение)
-- Hibernate должен склеить это обратно в 1 User + 10 Orders + 5 Addresses
```

## Неоптимальная загрузка:

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)
    private List<Order> orders;  // Всегда загружается!
}

// Нужен только email, но orders тоже загрузятся
String email = userRepository.findById(1L).getEmail();
// Лишние данные загружены зря
```

## Решение — используй LAZY:

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.LAZY)  // ✅ По умолчанию
    private List<Order> orders;
}

// Загружаем явно, когда нужно
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

## Когда EAGER допустим:

```java
// Маленькая, всегда нужная связь
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)  // OK: всегда нужен user
    private User user;
}
```

**Но даже здесь лучше LAZY + JOIN FETCH при необходимости.**

## Сравнение:

| Аспект | EAGER | LAZY |
|--------|-------|------|
| Загрузка | Всегда | По требованию |
| N+1 problem | Часто | Контролируется |
| Производительность | Может быть плохой | Оптимизируется |
| Гибкость | Низкая | Высокая |

## Почему не другие варианты:

- **"Всегда приводит к N+1"** — не всегда, но очень часто
- **"Блокирует транзакцию"** — нет, но может замедлить из-за лишних запросов
- **"Требует Criteria API"** — нет, работает с любым API

## Best practice:

```java
// Всегда используй LAZY по умолчанию
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;

// Загружай явно, когда нужно
@EntityGraph(attributePaths = {"orders"})
User findById(Long id);

// Или JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

**Вывод: `FetchType.EAGER` — антипаттерн. Загружает всё сразу → N+1 queries → картезианское произведение → плохая производительность. Используй LAZY + явная загрузка через JOIN FETCH.**

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)  // ❌ EAGER
    private List<Order> orders;
}

// Запрос
List<User> users = userRepository.findAll();
```

```sql
-- 1 запрос для users
SELECT * FROM users;  

-- N запросов для orders (для каждого user)
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 2;
SELECT * FROM orders WHERE user_id = 3;
...
SELECT * FROM orders WHERE user_id = 100;

-- Итого: 1 + 100 = 101 запрос!
```

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)
    private List<Order> orders;  // 10 заказов
    
    @OneToMany(fetch = FetchType.EAGER)
    private List<Address> addresses;  // 5 адресов
}

User user = userRepository.findById(1L);
```

```sql
SELECT * FROM users u
LEFT JOIN orders o ON u.id = o.user_id
LEFT JOIN addresses a ON u.id = a.user_id
WHERE u.id = 1;

-- Результат: 10 * 5 = 50 строк (картезианское произведение)
-- Hibernate должен склеить это обратно в 1 User + 10 Orders + 5 Addresses
```

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER)
    private List<Order> orders;  // Всегда загружается!
}

// Нужен только email, но orders тоже загрузятся
String email = userRepository.findById(1L).getEmail();
// Лишние данные загружены зря
```

```java
@Entity
public class User {
    @OneToMany(fetch = FetchType.LAZY)  // ✅ По умолчанию
    private List<Order> orders;
}

// Загружаем явно, когда нужно
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders WHERE u.id = :id")
Optional<User> findByIdWithOrders(@Param("id") Long id);
```

```java
// Маленькая, всегда нужная связь
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)  // OK: всегда нужен user
    private User user;
}
```

```java
// Всегда используй LAZY по умолчанию
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;

// Загружай явно, когда нужно
@EntityGraph(attributePaths = {"orders"})
User findById(Long id);

// Или JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders WHERE u.id = :id")
User findByIdWithOrders(@Param("id") Long id);
```

**Ответ: Spring загрузит файл полностью в память перед отправкой**

`FileSystemResource` загружает весь файл в память, что при большом размере файла (например, несколько GB) вызовет **OutOfMemoryError** или серьёзную деградацию производительности.

## Проблема:

```java
@GetMapping("/download")
public ResponseEntity<Resource> download() {
    Resource file = new FileSystemResource("data.csv");  // Файл 5 GB
    return ResponseEntity.ok(file);
    // Spring загрузит весь файл в память → OutOfMemoryError!
}
```

## Решение — streaming (потоковая передача):

### 1. StreamingResponseBody (рекомендуется):

```java
@GetMapping("/download")
public ResponseEntity<StreamingResponseBody> download() {
    StreamingResponseBody stream = outputStream -> {
        try (InputStream inputStream = new FileInputStream("data.csv")) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    };
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(stream);
}
```

### 2. InputStreamResource:

```java
@GetMapping("/download")
public ResponseEntity<InputStreamResource> download() throws IOException {
    File file = new File("data.csv");
    InputStreamResource resource = new InputStreamResource(new FileInputStream(file));
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .contentLength(file.length())
        .body(resource);
}
```

### 3. Async с DeferredResult:

```java
@GetMapping("/download")
public DeferredResult<ResponseEntity<StreamingResponseBody>> download() {
    DeferredResult<ResponseEntity<StreamingResponseBody>> result = new DeferredResult<>();
    
    CompletableFuture.runAsync(() -> {
        StreamingResponseBody stream = outputStream -> {
            // Потоковая запись
            try (InputStream in = new FileInputStream("data.csv")) {
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = in.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                }
            }
        };
        
        result.setResult(ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(stream));
    });
    
    return result;
}
```

### 4. С поддержкой Range requests (частичная загрузка):

```java
@GetMapping("/download")
public ResponseEntity<Resource> download(
        @RequestHeader(value = "Range", required = false) String rangeHeader) throws IOException {
    
    File file = new File("data.csv");
    long fileSize = file.length();
    
    if (rangeHeader == null) {
        // Обычная загрузка
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .contentLength(fileSize)
            .body(new InputStreamResource(new FileInputStream(file)));
    }
    
    // Парсинг Range header (например, "bytes=0-1023")
    String[] ranges = rangeHeader.replace("bytes=", "").split("-");
    long start = Long.parseLong(ranges[0]);
    long end = ranges.length > 1 ? Long.parseLong(ranges[1]) : fileSize - 1;
    long contentLength = end - start + 1;
    
    InputStream inputStream = new FileInputStream(file);
    inputStream.skip(start);
    
    return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT)
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .header(HttpHeaders.CONTENT_RANGE, "bytes " + start + "-" + end + "/" + fileSize)
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .contentLength(contentLength)
        .body(new InputStreamResource(inputStream));
}
```

## Сравнение подходов:

| Подход | Память | Скорость | Сложность |
|--------|--------|----------|-----------|
| `FileSystemResource` | ❌ Весь файл | Медленно | Простой |
| `StreamingResponseBody` | ✅ Буфер 8KB | Быстро | Средний |
| `InputStreamResource` | ✅ Буфер | Быстро | Простой |
| Range requests | ✅ Буфер | Быстро | Сложный |

## Почему не другие варианты:

- **"ResponseEntity не поддерживает файлы"** — поддерживает, но нужен правильный тип (streaming)
- **"Ресурс будет сериализован в JSON"** — нет, если `contentType` правильный
- **"FileSystemResource не работает с большими файлами"** — работает, но загружает в память

## Best practice для больших файлов:

```java
@GetMapping("/download/{filename}")
public ResponseEntity<StreamingResponseBody> downloadLargeFile(
        @PathVariable String filename) {
    
    Path filePath = Paths.get("/data/" + filename);
    
    if (!Files.exists(filePath)) {
        return ResponseEntity.notFound().build();
    }
    
    StreamingResponseBody stream = outputStream -> {
        try (InputStream in = Files.newInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    };
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, 
            "attachment; filename=\"" + filename + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(stream);
}
```

**Вывод: `FileSystemResource` загружает весь файл в память → проблема с большими файлами. Используй `StreamingResponseBody` или `InputStreamResource` для потоковой передачи.**

```java
@GetMapping("/download")
public ResponseEntity<Resource> download() {
    Resource file = new FileSystemResource("data.csv");  // Файл 5 GB
    return ResponseEntity.ok(file);
    // Spring загрузит весь файл в память → OutOfMemoryError!
}
```

```java
@GetMapping("/download")
public ResponseEntity<StreamingResponseBody> download() {
    StreamingResponseBody stream = outputStream -> {
        try (InputStream inputStream = new FileInputStream("data.csv")) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = inputStream.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    };
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(stream);
}
```

```java
@GetMapping("/download")
public ResponseEntity<InputStreamResource> download() throws IOException {
    File file = new File("data.csv");
    InputStreamResource resource = new InputStreamResource(new FileInputStream(file));
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .contentLength(file.length())
        .body(resource);
}
```

```java
@GetMapping("/download")
public DeferredResult<ResponseEntity<StreamingResponseBody>> download() {
    DeferredResult<ResponseEntity<StreamingResponseBody>> result = new DeferredResult<>();
    
    CompletableFuture.runAsync(() -> {
        StreamingResponseBody stream = outputStream -> {
            // Потоковая запись
            try (InputStream in = new FileInputStream("data.csv")) {
                byte[] buffer = new byte[8192];
                int bytesRead;
                while ((bytesRead = in.read(buffer)) != -1) {
                    outputStream.write(buffer, 0, bytesRead);
                }
            }
        };
        
        result.setResult(ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(stream));
    });
    
    return result;
}
```

```java
@GetMapping("/download")
public ResponseEntity<Resource> download(
        @RequestHeader(value = "Range", required = false) String rangeHeader) throws IOException {
    
    File file = new File("data.csv");
    long fileSize = file.length();
    
    if (rangeHeader == null) {
        // Обычная загрузка
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .contentLength(fileSize)
            .body(new InputStreamResource(new FileInputStream(file)));
    }
    
    // Парсинг Range header (например, "bytes=0-1023")
    String[] ranges = rangeHeader.replace("bytes=", "").split("-");
    long start = Long.parseLong(ranges[0]);
    long end = ranges.length > 1 ? Long.parseLong(ranges[1]) : fileSize - 1;
    long contentLength = end - start + 1;
    
    InputStream inputStream = new FileInputStream(file);
    inputStream.skip(start);
    
    return ResponseEntity.status(HttpStatus.PARTIAL_CONTENT)
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=data.csv")
        .header(HttpHeaders.CONTENT_RANGE, "bytes " + start + "-" + end + "/" + fileSize)
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .contentLength(contentLength)
        .body(new InputStreamResource(inputStream));
}
```

```java
@GetMapping("/download/{filename}")
public ResponseEntity<StreamingResponseBody> downloadLargeFile(
        @PathVariable String filename) {
    
    Path filePath = Paths.get("/data/" + filename);
    
    if (!Files.exists(filePath)) {
        return ResponseEntity.notFound().build();
    }
    
    StreamingResponseBody stream = outputStream -> {
        try (InputStream in = Files.newInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    };
    
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, 
            "attachment; filename=\"" + filename + "\"")
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(stream);
}
```

**Ответ: Автовайринг всегда требует запуска Spring Context**

`@Autowired` работает только внутри **Spring Application Context**. В unit-тестах (без Spring Context) зависимости не будут внедрены автоматически, и `repo` останется `null` → `NullPointerException`.

## Проблема:

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;  // Внедряется Spring'ом
    
    public User getUser(Long id) {
        return repo.findById(id).orElseThrow();
    }
}

// Unit-тест БЕЗ Spring Context
class UserServiceTest {
    @Test
    void testGetUser() {
        UserService service = new UserService();  // Создали вручную
        User user = service.getUser(1L);  
        // NullPointerException! repo == null
    }
}
```

## Решения:

### 1. Mock зависимости (unit-тест):

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository repo;
    
    @InjectMocks  // Mockito внедрит mock'и
    private UserService service;
    
    @Test
    void testGetUser() {
        User expectedUser = new User(1L, "John");
        when(repo.findById(1L)).thenReturn(Optional.of(expectedUser));
        
        User actualUser = service.getUser(1L);
        
        assertEquals(expectedUser, actualUser);
        verify(repo).findById(1L);
    }
}
```

### 2. Constructor injection (лучшая практика):

```java
@Service
public class UserService {
    private final UserRepository repo;
    
    // Constructor injection - проще тестировать
    @Autowired  // Можно опустить, если один конструктор
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
    
    public User getUser(Long id) {
        return repo.findById(id).orElseThrow();
    }
}

// Unit-тест без Spring
class UserServiceTest {
    @Test
    void testGetUser() {
        UserRepository mockRepo = mock(UserRepository.class);
        UserService service = new UserService(mockRepo);  // Внедрили вручную
        
        when(mockRepo.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));
        
        User user = service.getUser(1L);
        assertEquals("John", user.getName());
    }
}
```

### 3. Integration тест со Spring Context:

```java
@SpringBootTest
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService service;  // Spring внедрит
    
    @MockBean  // Mock для репозитория
    private UserRepository repo;
    
    @Test
    void testGetUser() {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));
        
        User user = service.getUser(1L);
        assertEquals("John", user.getName());
    }
}
```

### 4. @WebMvcTest для контроллеров:

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService service;
    
    @Test
    void testGetUserEndpoint() throws Exception {
        when(service.getUser(1L)).thenReturn(new User(1L, "John"));
        
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

## Сравнение подходов:

| Тип теста | Spring Context | Скорость | Изоляция |
|-----------|----------------|----------|----------|
| **Unit** (Mockito) | ❌ Нет | ⚡ Быстро | ✅ Полная |
| **@SpringBootTest** | ✅ Да | 🐌 Медленно | ❌ Низкая |
| **@WebMvcTest** | ✅ Частично | 🏃 Средне | ✅ Средняя |
| **@DataJpaTest** | ✅ Только JPA | 🏃 Средне | ✅ Высокая |

## Best practice:

```java
// Используй constructor injection
@Service
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}

// Unit-тест без Spring
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository repo;
    
    private UserService service;
    
    @BeforeEach
    void setUp() {
        service = new UserService(repo);  // Явное создание
    }
    
    @Test
    void testGetUser() {
        // Тест без Spring Context
    }
}
```

## Почему не другие варианты:

- **"Полевая инъекция усложняет тестирование"** — ДА, это правда
- **"Вызов репозитория невозможен без транзакций"** — в unit-тестах используем mock, транзакции не нужны
- **"Сервис не зарегистрирован как Bean"** — в unit-тестах Spring Context не запускается

**Вывод: `@Autowired` требует Spring Context. В unit-тестах зависимости `null` → используй `@Mock` + `@InjectMocks` (Mockito) или constructor injection для ручного внедрения.**

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;  // Внедряется Spring'ом
    
    public User getUser(Long id) {
        return repo.findById(id).orElseThrow();
    }
}

// Unit-тест БЕЗ Spring Context
class UserServiceTest {
    @Test
    void testGetUser() {
        UserService service = new UserService();  // Создали вручную
        User user = service.getUser(1L);  
        // NullPointerException! repo == null
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository repo;
    
    @InjectMocks  // Mockito внедрит mock'и
    private UserService service;
    
    @Test
    void testGetUser() {
        User expectedUser = new User(1L, "John");
        when(repo.findById(1L)).thenReturn(Optional.of(expectedUser));
        
        User actualUser = service.getUser(1L);
        
        assertEquals(expectedUser, actualUser);
        verify(repo).findById(1L);
    }
}
```

```java
@Service
public class UserService {
    private final UserRepository repo;
    
    // Constructor injection - проще тестировать
    @Autowired  // Можно опустить, если один конструктор
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
    
    public User getUser(Long id) {
        return repo.findById(id).orElseThrow();
    }
}

// Unit-тест без Spring
class UserServiceTest {
    @Test
    void testGetUser() {
        UserRepository mockRepo = mock(UserRepository.class);
        UserService service = new UserService(mockRepo);  // Внедрили вручную
        
        when(mockRepo.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));
        
        User user = service.getUser(1L);
        assertEquals("John", user.getName());
    }
}
```

```java
@SpringBootTest
class UserServiceIntegrationTest {
    
    @Autowired
    private UserService service;  // Spring внедрит
    
    @MockBean  // Mock для репозитория
    private UserRepository repo;
    
    @Test
    void testGetUser() {
        when(repo.findById(1L)).thenReturn(Optional.of(new User(1L, "John")));
        
        User user = service.getUser(1L);
        assertEquals("John", user.getName());
    }
}
```

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService service;
    
    @Test
    void testGetUserEndpoint() throws Exception {
        when(service.getUser(1L)).thenReturn(new User(1L, "John"));
        
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

```java
// Используй constructor injection
@Service
public class UserService {
    private final UserRepository repo;
    
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}

// Unit-тест без Spring
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository repo;
    
    private UserService service;
    
    @BeforeEach
    void setUp() {
        service = new UserService(repo);  // Явное создание
    }
    
    @Test
    void testGetUser() {
        // Тест без Spring Context
    }
}
```

**Ответ: Большое количество short-lived объектов засоряет Eden space и вызывает частые Minor GC**

Создание **1 миллиона строк** внутри цикла создаёт огромное количество **короткоживущих объектов** (short-lived), которые быстро заполняют **Eden space** (часть Young Generation). Это вызывает:

1. **Частые Minor GC** (сборка мусора в Young Generation)
2. **Паузы приложения** (Stop-The-World)
3. **Нагрузка на CPU**

## Что происходит:

```java
for (int i = 0; i < 1_000_000; i++) {
    String s = new String("abc");  // Каждая итерация создаёт новый объект
    // Объект сразу становится мусором после итерации
}
```

### Heap структура:

```
Young Generation (Eden + Survivor)
  ├── Eden Space       ← Новые объекты создаются здесь
  ├── Survivor 0 (S0)  ← Выжившие после Minor GC
  └── Survivor 1 (S1)

Old Generation         ← Долгоживущие объекты
```

### Что происходит в Eden:

```
Итерация 1-10000:   Eden заполняется строками
Minor GC #1:        Eden очищается (все строки мусор)
Итерация 10001-20000: Eden снова заполняется
Minor GC #2:        Eden очищается
...
Minor GC #100+:     Постоянные паузы!
```

## Проверка влияния GC:

```java
// Включить GC логи
java -Xlog:gc* -Xmx512m MyApp

// Вывод:
[GC (Allocation Failure) 131072K->256K(524288K), 0.0015432 secs]
[GC (Allocation Failure) 131328K->320K(524288K), 0.0012345 secs]
[GC (Allocation Failure) 131392K->384K(524288K), 0.0013456 secs]
// ... сотни Minor GC
```

## Решения:

### 1. Переиспользовать объект (если возможно):

```java
String s = "abc";  // Literal в StringPool, переиспользуется
for (int i = 0; i < 1_000_000; i++) {
    // Используем ту же строку
    process(s);
}
```

### 2. StringBuilder для конкатенации:

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1_000_000; i++) {
    sb.append("abc");
    sb.append(i);
    String result = sb.toString();
    sb.setLength(0);  // Очистить buffer
}
```

### 3. Увеличить Eden size:

```bash
# Увеличить Young Generation
java -Xmn512m -Xmx2g MyApp
```

### 4. Оптимизировать алгоритм:

```java
// Вместо создания 1M объектов
List<String> list = new ArrayList<>(1_000_000);
for (int i = 0; i < 1_000_000; i++) {
    list.add(new String("abc"));  // Все еще плохо
}

// Лучше:
List<String> list = Collections.nCopies(1_000_000, "abc");  // Одна строка, 1M ссылок
```

## Мониторинг GC:

```java
// Добавить в код
long startTime = System.currentTimeMillis();
long startGcCount = getGcCount();

for (int i = 0; i < 1_000_000; i++) {
    String s = new String("abc");
}

long endTime = System.currentTimeMillis();
long endGcCount = getGcCount();

System.out.println("Time: " + (endTime - startTime) + "ms");
System.out.println("GC count: " + (endGcCount - startGcCount));

private static long getGcCount() {
    return ManagementFactory.getGarbageCollectorMXBeans().stream()
        .mapToLong(GarbageCollectorMXBean::getCollectionCount)
        .sum();
}
```

**Результат:**
```
Time: 250ms
GC count: 45  // 45 Minor GC за 250ms!
```

## Почему не другие варианты:

- **"GC игнорирует объекты внутри цикла"** — нет, собирает их активно
- **"String автоматически кэшируются в StringPool"** — только **литералы** (`"abc"`), не `new String("abc")`
- **"String всегда остаются в Old Gen"** — нет, создаются в Eden

## String literal vs new String:

```java
// Literal - в StringPool (переиспользуется)
String s1 = "abc";
String s2 = "abc";
System.out.println(s1 == s2);  // true

// new String - в heap (новый объект каждый раз)
String s3 = new String("abc");
String s4 = new String("abc");
System.out.println(s3 == s4);  // false
```

**Вывод: 1 млн `new String("abc")` создаёт 1 млн объектов в Eden → частые Minor GC → паузы приложения. Используй литералы или переиспользуй объекты.**

```java
for (int i = 0; i < 1_000_000; i++) {
    String s = new String("abc");  // Каждая итерация создаёт новый объект
    // Объект сразу становится мусором после итерации
}
```

```plaintext
Young Generation (Eden + Survivor)
  ├── Eden Space       ← Новые объекты создаются здесь
  ├── Survivor 0 (S0)  ← Выжившие после Minor GC
  └── Survivor 1 (S1)

Old Generation         ← Долгоживущие объекты
```

```plaintext
Итерация 1-10000:   Eden заполняется строками
Minor GC #1:        Eden очищается (все строки мусор)
Итерация 10001-20000: Eden снова заполняется
Minor GC #2:        Eden очищается
...
Minor GC #100+:     Постоянные паузы!
```

```java
// Включить GC логи
java -Xlog:gc* -Xmx512m MyApp

// Вывод:
[GC (Allocation Failure) 131072K->256K(524288K), 0.0015432 secs]
[GC (Allocation Failure) 131328K->320K(524288K), 0.0012345 secs]
[GC (Allocation Failure) 131392K->384K(524288K), 0.0013456 secs]
// ... сотни Minor GC
```

```java
String s = "abc";  // Literal в StringPool, переиспользуется
for (int i = 0; i < 1_000_000; i++) {
    // Используем ту же строку
    process(s);
}
```

```java
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1_000_000; i++) {
    sb.append("abc");
    sb.append(i);
    String result = sb.toString();
    sb.setLength(0);  // Очистить buffer
}
```

```shellscript
# Увеличить Young Generation
java -Xmn512m -Xmx2g MyApp
```

```java
// Вместо создания 1M объектов
List<String> list = new ArrayList<>(1_000_000);
for (int i = 0; i < 1_000_000; i++) {
    list.add(new String("abc"));  // Все еще плохо
}

// Лучше:
List<String> list = Collections.nCopies(1_000_000, "abc");  // Одна строка, 1M ссылок
```

```java
// Добавить в код
long startTime = System.currentTimeMillis();
long startGcCount = getGcCount();

for (int i = 0; i < 1_000_000; i++) {
    String s = new String("abc");
}

long endTime = System.currentTimeMillis();
long endGcCount = getGcCount();

System.out.println("Time: " + (endTime - startTime) + "ms");
System.out.println("GC count: " + (endGcCount - startGcCount));

private static long getGcCount() {
    return ManagementFactory.getGarbageCollectorMXBeans().stream()
        .mapToLong(GarbageCollectorMXBean::getCollectionCount)
        .sum();
}
```

```plaintext
Time: 250ms
GC count: 45  // 45 Minor GC за 250ms!
```

```java
// Literal - в StringPool (переиспользуется)
String s1 = "abc";
String s2 = "abc";
System.out.println(s1 == s2);  // true

// new String - в heap (новый объект каждый раз)
String s3 = new String("abc");
String s4 = new String("abc");
System.out.println(s3 == s4);  // false
```

**Ответ: Переполнение Humongous Objects и невозможность их разместить**

G1GC использует **регионы фиксированного размера**. Объекты, которые занимают **≥50% размера региона**, называются **Humongous Objects** и требуют **непрерывных регионов**. Если таких объектов много или они не помещаются, G1GC вынужден запускать **Full GC**.

## Что такое Humongous Objects:

```java
// Размер региона G1 (по умолчанию ~1-32MB, зависит от heap)
-XX:G1HeapRegionSize=4m  // Например, 4MB

// Humongous Object = объект ≥ 2MB (50% от региона)
byte[] hugeArray = new byte[3_000_000];  // 3MB - Humongous!
```

### Проблема:

1. **Humongous Objects размещаются напрямую в Old Generation**
2. **Требуют непрерывные регионы** (несколько подряд)
3. **Не собираются при Minor GC**
4. **Фрагментация heap** → нет места для новых Humongous
5. **Триггер Full GC** для дефрагментации

## Лог Full GC из-за Humongous:

```
[Full GC (Allocation Failure)
  [Humongous: 512M->256M(512M)]
  2048M->1536M(4096M), 1.234 secs]
```

## Почему это происходит на продакшене:

### 1. Большие объекты в бизнес-логике:
```java
// JSON/XML парсинг больших файлов
String bigJson = new String(Files.readAllBytes(path));  // 10MB

// Большие коллекции
List<Order> orders = orderRepository.findAll();  // 5M записей

// Byte arrays для файлов
byte[] fileContent = downloadFile();  // 50MB
```

### 2. Фрагментация heap:
```
Heap:
[Region 1: occupied] [Region 2: occupied] [Region 3: free]
[Region 4: occupied] [Region 5: free]    [Region 6: free]

Попытка создать Humongous (нужно 3 региона подряд):
❌ Не помещается → Full GC для дефрагментации
```

## Решения:

### 1. Увеличить размер региона G1:
```bash
# По умолчанию: автоматический (обычно 1-4MB)
# Увеличить для больших объектов
java -XX:G1HeapRegionSize=8m -Xmx8g MyApp

# Формула: регион должен быть > 2 * размер типичного большого объекта
# Если объекты ~10MB → регион 32MB
java -XX:G1HeapRegionSize=32m -Xmx16g MyApp
```

**⚠️ Ограничение:** размер региона от 1MB до 32MB, должен быть степенью 2.

### 2. Избегать создания больших объектов:

#### Streaming вместо загрузки в память:
```java
// ❌ Плохо: загружаем весь файл
String content = new String(Files.readAllBytes(path));

// ✅ Хорошо: потоковая обработка
try (Stream<String> lines = Files.lines(path)) {
    lines.forEach(this::processLine);
}
```

#### Chunking для больших данных:
```java
// ❌ Плохо: все записи сразу
List<Order> orders = repository.findAll();

// ✅ Хорошо: пагинация
Pageable pageable = PageRequest.of(0, 1000);
Page<Order> page;
do {
    page = repository.findAll(pageable);
    processOrders(page.getContent());
    pageable = pageable.next();
} while (page.hasNext());
```

### 3. Мониторинг Humongous Objects:

```bash
# Включить детальные логи G1GC
java -Xlog:gc*,gc+humongous=debug -Xmx8g MyApp

# Вывод покажет:
# [gc,humongous] Humongous allocation: 4194304 bytes
# [gc,humongous] Allocation failed for humongous object
```

### 4. JVM параметры для G1GC:

```bash
java \
  -XX:+UseG1GC \
  -Xmx8g \
  -Xms8g \
  -XX:G1HeapRegionSize=16m \           # Увеличить регион
  -XX:MaxGCPauseMillis=200 \           # Целевое время паузы
  -XX:InitiatingHeapOccupancyPercent=45 \  # Порог для Concurrent Mark
  -XX:G1ReservePercent=10 \            # Резерв для эвакуации
  MyApp
```

### 5. Альтернатива — использовать ZGC/Shenandoah:

```bash
# ZGC - лучше справляется с большими объектами
java -XX:+UseZGC -Xmx16g MyApp

# Shenandoah
java -XX:+UseShenandoahGC -Xmx16g MyApp
```

## Диагностика проблемы:

```bash
# Heap dump при Full GC
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     MyApp

# Анализ в VisualVM/JProfiler:
# - Найти самые большие объекты
# - Проверить где они создаются
```

## Почему не другие варианты:

- **"Слишком мало синхронизаций"** — не влияет на Full GC
- **"Нехватка потоков в ForkJoinPool"** — не связано с Full GC
- **"GC всегда вызывает Full GC каждые 10 минут"** — нет, Full GC запускается по необходимости

**Вывод: Full GC в G1GC на продакшене чаще всего из-за Humongous Objects (≥50% размера региона), которые не помещаются из-за фрагментации heap. Решение: увеличить G1HeapRegionSize или избегать создания больших объектов.**

```java
// Размер региона G1 (по умолчанию ~1-32MB, зависит от heap)
-XX:G1HeapRegionSize=4m  // Например, 4MB

// Humongous Object = объект ≥ 2MB (50% от региона)
byte[] hugeArray = new byte[3_000_000];  // 3MB - Humongous!
```

```plaintext
[Full GC (Allocation Failure)
  [Humongous: 512M->256M(512M)]
  2048M->1536M(4096M), 1.234 secs]
```

```java
// JSON/XML парсинг больших файлов
String bigJson = new String(Files.readAllBytes(path));  // 10MB

// Большие коллекции
List<Order> orders = orderRepository.findAll();  // 5M записей

// Byte arrays для файлов
byte[] fileContent = downloadFile();  // 50MB
```

```plaintext
Heap:
[Region 1: occupied] [Region 2: occupied] [Region 3: free]
[Region 4: occupied] [Region 5: free]    [Region 6: free]

Попытка создать Humongous (нужно 3 региона подряд):
❌ Не помещается → Full GC для дефрагментации
```

```shellscript
# По умолчанию: автоматический (обычно 1-4MB)
# Увеличить для больших объектов
java -XX:G1HeapRegionSize=8m -Xmx8g MyApp

# Формула: регион должен быть > 2 * размер типичного большого объекта
# Если объекты ~10MB → регион 32MB
java -XX:G1HeapRegionSize=32m -Xmx16g MyApp
```

```java
// ❌ Плохо: загружаем весь файл
String content = new String(Files.readAllBytes(path));

// ✅ Хорошо: потоковая обработка
try (Stream<String> lines = Files.lines(path)) {
    lines.forEach(this::processLine);
}
```

```java
// ❌ Плохо: все записи сразу
List<Order> orders = repository.findAll();

// ✅ Хорошо: пагинация
Pageable pageable = PageRequest.of(0, 1000);
Page<Order> page;
do {
    page = repository.findAll(pageable);
    processOrders(page.getContent());
    pageable = pageable.next();
} while (page.hasNext());
```

```shellscript
# Включить детальные логи G1GC
java -Xlog:gc*,gc+humongous=debug -Xmx8g MyApp

# Вывод покажет:
# [gc,humongous] Humongous allocation: 4194304 bytes
# [gc,humongous] Allocation failed for humongous object
```

```shellscript
java \
  -XX:+UseG1GC \
  -Xmx8g \
  -Xms8g \
  -XX:G1HeapRegionSize=16m \           # Увеличить регион
  -XX:MaxGCPauseMillis=200 \           # Целевое время паузы
  -XX:InitiatingHeapOccupancyPercent=45 \  # Порог для Concurrent Mark
  -XX:G1ReservePercent=10 \            # Резерв для эвакуации
  MyApp
```

```shellscript
# ZGC - лучше справляется с большими объектами
java -XX:+UseZGC -Xmx16g MyApp

# Shenandoah
java -XX:+UseShenandoahGC -Xmx16g MyApp
```

```shellscript
# Heap dump при Full GC
java -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/tmp/heapdump.hprof \
     MyApp

# Анализ в VisualVM/JProfiler:
# - Найти самые большие объекты
# - Проверить где они создаются
```

**Ответ: Chain of Responsibility, чтобы выстраивать обработчики в цепочку**

Для системы логирования с **несколькими обработчиками**, которые должны **выполняться последовательно** (цепочка), идеально подходит паттерн **Chain of Responsibility**.

## Как работает:

```java
// Абстрактный обработчик
abstract class LogHandler {
    protected LogHandler next;
    
    public void setNext(LogHandler handler) {
        this.next = handler;
    }
    
    public void log(String message, LogLevel level) {
        handle(message, level);
        
        // Передаём следующему в цепочке
        if (next != null) {
            next.log(message, level);
        }
    }
    
    protected abstract void handle(String message, LogLevel level);
}

// Обработчик 1: запись в файл
class FileLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[FILE] Writing to file: " + message);
        // Логика записи в файл
    }
}

// Обработчик 2: отправка в Kafka
class KafkaLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[KAFKA] Sending to Kafka: " + message);
        // Логика отправки в Kafka
    }
}

// Обработчик 3: вывод в консоль
class ConsoleLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[CONSOLE] " + message);
    }
}
```

## Использование:

```java
// Построение цепочки
LogHandler fileHandler = new FileLogHandler();
LogHandler kafkaHandler = new KafkaLogHandler();
LogHandler consoleHandler = new ConsoleLogHandler();

fileHandler.setNext(kafkaHandler);
kafkaHandler.setNext(consoleHandler);

// Запуск цепочки
fileHandler.log("User logged in", LogLevel.INFO);

// Вывод:
// [FILE] Writing to file: User logged in
// [KAFKA] Sending to Kafka: User logged in
// [CONSOLE] User logged in
```

## С фильтрацией по уровню:

```java
abstract class LogHandler {
    protected LogHandler next;
    protected LogLevel minLevel;
    
    public LogHandler(LogLevel minLevel) {
        this.minLevel = minLevel;
    }
    
    public void log(String message, LogLevel level) {
        // Обрабатываем, если уровень подходит
        if (level.ordinal() >= minLevel.ordinal()) {
            handle(message, level);
        }
        
        // Передаём дальше
        if (next != null) {
            next.log(message, level);
        }
    }
    
    protected abstract void handle(String message, LogLevel level);
}

// Использование
LogHandler fileHandler = new FileLogHandler(LogLevel.DEBUG);    // Всё пишет
LogHandler kafkaHandler = new KafkaLogHandler(LogLevel.WARN);   // Только WARN+
LogHandler consoleHandler = new ConsoleLogHandler(LogLevel.ERROR); // Только ERROR

fileHandler.setNext(kafkaHandler);
kafkaHandler.setNext(consoleHandler);

fileHandler.log("Debug info", LogLevel.DEBUG);  
// [FILE] Writing to file: Debug info

fileHandler.log("Critical error", LogLevel.ERROR);
// [FILE] Writing to file: Critical error
// [KAFKA] Sending to Kafka: Critical error
// [CONSOLE] Critical error
```

## Альтернатива с Builder:

```java
class Logger {
    private final LogHandler chain;
    
    private Logger(LogHandler chain) {
        this.chain = chain;
    }
    
    public void log(String message, LogLevel level) {
        chain.log(message, level);
    }
    
    static class Builder {
        private LogHandler first;
        private LogHandler last;
        
        public Builder addHandler(LogHandler handler) {
            if (first == null) {
                first = handler;
                last = handler;
            } else {
                last.setNext(handler);
                last = handler;
            }
            return this;
        }
        
        public Logger build() {
            return new Logger(first);
        }
    }
}

// Использование
Logger logger = new Logger.Builder()
    .addHandler(new FileLogHandler(LogLevel.DEBUG))
    .addHandler(new KafkaLogHandler(LogLevel.WARN))
    .addHandler(new ConsoleLogHandler(LogLevel.ERROR))
    .build();

logger.log("Error occurred", LogLevel.ERROR);
```

## Почему не другие паттерны:

### Template Method:
```java
// Задаёт шаги алгоритма, но не цепочку обработчиков
abstract class LogTemplate {
    public final void log(String message) {
        format(message);
        write(message);
        notify(message);
    }
    protected abstract void format(String message);
    protected abstract void write(String message);
}
// НЕ подходит: нельзя динамически добавлять/убирать обработчики
```

### Strategy:
```java
// Выбирает ОДИН алгоритм, а не цепочку
interface LogStrategy {
    void log(String message);
}
// НЕ подходит: нужно выполнить ВСЕ обработчики, не один
```

### Singleton:
```java
// Один экземпляр логгера
class Logger {
    private static Logger instance;
}
// НЕ подходит: не решает проблему цепочки обработчиков
```

## Real-world пример (SLF4J/Logback):

```xml
<!-- logback.xml -->
<configuration>
  <!-- Обработчик 1: файл -->
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>app.log</file>
  </appender>
  
  <!-- Обработчик 2: консоль -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  </appender>
  
  <!-- Обработчик 3: Kafka -->
  <appender name="KAFKA" class="com.github.danielwegener.logback.kafka.KafkaAppender">
  </appender>
  
  <root level="info">
    <appender-ref ref="FILE" />
    <appender-ref ref="CONSOLE" />
    <appender-ref ref="KAFKA" />
  </root>
</configuration>
```

**Вывод: Chain of Responsibility — идеальный паттерн для системы логирования с множественными обработчиками. Позволяет выстраивать цепочку, динамически добавлять/удалять обработчики и фильтровать по уровню.**

```java
// Абстрактный обработчик
abstract class LogHandler {
    protected LogHandler next;
    
    public void setNext(LogHandler handler) {
        this.next = handler;
    }
    
    public void log(String message, LogLevel level) {
        handle(message, level);
        
        // Передаём следующему в цепочке
        if (next != null) {
            next.log(message, level);
        }
    }
    
    protected abstract void handle(String message, LogLevel level);
}

// Обработчик 1: запись в файл
class FileLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[FILE] Writing to file: " + message);
        // Логика записи в файл
    }
}

// Обработчик 2: отправка в Kafka
class KafkaLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[KAFKA] Sending to Kafka: " + message);
        // Логика отправки в Kafka
    }
}

// Обработчик 3: вывод в консоль
class ConsoleLogHandler extends LogHandler {
    @Override
    protected void handle(String message, LogLevel level) {
        System.out.println("[CONSOLE] " + message);
    }
}
```

```java
// Построение цепочки
LogHandler fileHandler = new FileLogHandler();
LogHandler kafkaHandler = new KafkaLogHandler();
LogHandler consoleHandler = new ConsoleLogHandler();

fileHandler.setNext(kafkaHandler);
kafkaHandler.setNext(consoleHandler);

// Запуск цепочки
fileHandler.log("User logged in", LogLevel.INFO);

// Вывод:
// [FILE] Writing to file: User logged in
// [KAFKA] Sending to Kafka: User logged in
// [CONSOLE] User logged in
```

```java
abstract class LogHandler {
    protected LogHandler next;
    protected LogLevel minLevel;
    
    public LogHandler(LogLevel minLevel) {
        this.minLevel = minLevel;
    }
    
    public void log(String message, LogLevel level) {
        // Обрабатываем, если уровень подходит
        if (level.ordinal() >= minLevel.ordinal()) {
            handle(message, level);
        }
        
        // Передаём дальше
        if (next != null) {
            next.log(message, level);
        }
    }
    
    protected abstract void handle(String message, LogLevel level);
}

// Использование
LogHandler fileHandler = new FileLogHandler(LogLevel.DEBUG);    // Всё пишет
LogHandler kafkaHandler = new KafkaLogHandler(LogLevel.WARN);   // Только WARN+
LogHandler consoleHandler = new ConsoleLogHandler(LogLevel.ERROR); // Только ERROR

fileHandler.setNext(kafkaHandler);
kafkaHandler.setNext(consoleHandler);

fileHandler.log("Debug info", LogLevel.DEBUG);  
// [FILE] Writing to file: Debug info

fileHandler.log("Critical error", LogLevel.ERROR);
// [FILE] Writing to file: Critical error
// [KAFKA] Sending to Kafka: Critical error
// [CONSOLE] Critical error
```

```java
class Logger {
    private final LogHandler chain;
    
    private Logger(LogHandler chain) {
        this.chain = chain;
    }
    
    public void log(String message, LogLevel level) {
        chain.log(message, level);
    }
    
    static class Builder {
        private LogHandler first;
        private LogHandler last;
        
        public Builder addHandler(LogHandler handler) {
            if (first == null) {
                first = handler;
                last = handler;
            } else {
                last.setNext(handler);
                last = handler;
            }
            return this;
        }
        
        public Logger build() {
            return new Logger(first);
        }
    }
}

// Использование
Logger logger = new Logger.Builder()
    .addHandler(new FileLogHandler(LogLevel.DEBUG))
    .addHandler(new KafkaLogHandler(LogLevel.WARN))
    .addHandler(new ConsoleLogHandler(LogLevel.ERROR))
    .build();

logger.log("Error occurred", LogLevel.ERROR);
```

```java
// Задаёт шаги алгоритма, но не цепочку обработчиков
abstract class LogTemplate {
    public final void log(String message) {
        format(message);
        write(message);
        notify(message);
    }
    protected abstract void format(String message);
    protected abstract void write(String message);
}
// НЕ подходит: нельзя динамически добавлять/убирать обработчики
```

```java
// Выбирает ОДИН алгоритм, а не цепочку
interface LogStrategy {
    void log(String message);
}
// НЕ подходит: нужно выполнить ВСЕ обработчики, не один
```

```java
// Один экземпляр логгера
class Logger {
    private static Logger instance;
}
// НЕ подходит: не решает проблему цепочки обработчиков
```

```xml
<!-- logback.xml -->
<configuration>
  <!-- Обработчик 1: файл -->
  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
    <file>app.log</file>
  </appender>
  
  <!-- Обработчик 2: консоль -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
  </appender>
  
  <!-- Обработчик 3: Kafka -->
  <appender name="KAFKA" class="com.github.danielwegener.logback.kafka.KafkaAppender">
  </appender>
  
  <root level="info">
    <appender-ref ref="FILE" />
    <appender-ref ref="CONSOLE" />
    <appender-ref ref="KAFKA" />
  </root>
</configuration>
```

**Ответ: Конкатенация `"item" + i` создаёт новые String на каждой итерации**

Операция `"item" + i` создаёт **новый объект String** на каждой итерации (1 миллион раз), хотя многие из них идентичны. Java **не переиспользует** результаты конкатенации автоматически — каждый результат создаётся в heap.

## Что происходит:

```java
List<String> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);  
    // Создаётся новый String: "item0", "item1", "item2", ...
}
```

### Декомпозиция конкатенации:

```java
// "item" + i компилируется примерно так:
String temp = new StringBuilder()
    .append("item")
    .append(i)
    .toString();  // Создаёт новый String
list.add(temp);
```

**Каждая итерация:**
1. Создаёт `StringBuilder`
2. Добавляет `"item"`
3. Добавляет `i` (boxing `int` → `Integer` → `String`)
4. Вызывает `toString()` → новый `String` в heap
5. Добавляет в список

**Результат:** 1 млн объектов `String` + временные объекты `StringBuilder`.

## Почему в StringTable много объектов:

При профилировании видны объекты в **StringTable** (внутренняя структура JVM для интернированных строк), потому что:

1. Литерал `"item"` в StringPool (переиспользуется)
2. Результаты конкатенации (`"item0"`, `"item1"`, ...) **НЕ интернированы** автоматически
3. Но профайлер показывает их как часть String statistics

## Проверка:

```java
List<String> list = new ArrayList<>();

// Без intern()
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);
}
// Heap: ~50-100 MB (1M разных String объектов)

// С intern() (НЕ рекомендуется для большого числа уникальных строк)
for (int i = 0; i < 1_000_000; i++) {
    list.add(("item" + i).intern());  // Добавляет в StringPool
}
// StringPool переполнится, производительность упадёт
```

## Решения для оптимизации:

### 1. Если строки повторяются — переиспользовать:

```java
// Если значения повторяются
Map<String, String> cache = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    int key = i % 1000;  // Например, 1000 уникальных значений
    String item = cache.computeIfAbsent("item" + key, k -> k);
    list.add(item);
}
```

### 2. Использовать более эффективную структуру:

```java
// Если формат предсказуем
record Item(int id) {
    @Override
    public String toString() {
        return "item" + id;
    }
}

List<Item> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(new Item(i));  // Только int, String генерится по требованию
}
```

### 3. Если нужны String, но не все сразу:

```java
// Ленивая генерация
class ItemList extends AbstractList<String> {
    private final int size;
    
    public ItemList(int size) {
        this.size = size;
    }
    
    @Override
    public String get(int index) {
        return "item" + index;  // Генерируется на лету
    }
    
    @Override
    public int size() {
        return size;
    }
}

List<String> list = new ItemList(1_000_000);
// Strings создаются только при обращении к элементу
```

### 4. Оптимизация конкатенации (если нужны все String):

```java
// Предварительное выделение памяти
List<String> list = new ArrayList<>(1_000_000);
StringBuilder sb = new StringBuilder(10);  // Примерная длина "item12345"

for (int i = 0; i < 1_000_000; i++) {
    sb.setLength(0);
    sb.append("item").append(i);
    list.add(sb.toString());  // Всё равно создаёт String, но быстрее
}
```

## Почему не другие варианты:

- **"JIT не оптимизирует конкатенацию строк с числами"** — оптимизирует через `StringBuilder`, но результат всё равно новый `String`
- **"Используется intern(), что ведёт к росту пула строк"** — нет, `intern()` не вызывается автоматически при конкатенации
- **"Компилятор всегда кеширует строки в StringPool"** — только **литералы**, не результаты конкатенации

## Мониторинг:

```bash
# Посмотреть StringTable
jcmd <pid> VM.stringtable

# Heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# Анализ в VisualVM:
# Strings: 1,000,000 instances, ~50MB
```

**Вывод: конкатенация `"item" + i` создаёт 1 миллион уникальных String объектов в heap. JIT оптимизирует через `StringBuilder`, но каждый `toString()` всё равно создаёт новый String. Это не StringPool, а обычные объекты в heap.**

```java
List<String> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);  
    // Создаётся новый String: "item0", "item1", "item2", ...
}
```

```java
// "item" + i компилируется примерно так:
String temp = new StringBuilder()
    .append("item")
    .append(i)
    .toString();  // Создаёт новый String
list.add(temp);
```

```java
List<String> list = new ArrayList<>();

// Без intern()
for (int i = 0; i < 1_000_000; i++) {
    list.add("item" + i);
}
// Heap: ~50-100 MB (1M разных String объектов)

// С intern() (НЕ рекомендуется для большого числа уникальных строк)
for (int i = 0; i < 1_000_000; i++) {
    list.add(("item" + i).intern());  // Добавляет в StringPool
}
// StringPool переполнится, производительность упадёт
```

```java
// Если значения повторяются
Map<String, String> cache = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) {
    int key = i % 1000;  // Например, 1000 уникальных значений
    String item = cache.computeIfAbsent("item" + key, k -> k);
    list.add(item);
}
```

```java
// Если формат предсказуем
record Item(int id) {
    @Override
    public String toString() {
        return "item" + id;
    }
}

List<Item> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(new Item(i));  // Только int, String генерится по требованию
}
```

```java
// Ленивая генерация
class ItemList extends AbstractList<String> {
    private final int size;
    
    public ItemList(int size) {
        this.size = size;
    }
    
    @Override
    public String get(int index) {
        return "item" + index;  // Генерируется на лету
    }
    
    @Override
    public int size() {
        return size;
    }
}

List<String> list = new ItemList(1_000_000);
// Strings создаются только при обращении к элементу
```

```java
// Предварительное выделение памяти
List<String> list = new ArrayList<>(1_000_000);
StringBuilder sb = new StringBuilder(10);  // Примерная длина "item12345"

for (int i = 0; i < 1_000_000; i++) {
    sb.setLength(0);
    sb.append("item").append(i);
    list.add(sb.toString());  // Всё равно создаёт String, но быстрее
}
```

```shellscript
# Посмотреть StringTable
jcmd <pid> VM.stringtable

# Heap dump
jmap -dump:format=b,file=heap.hprof <pid>

# Анализ в VisualVM:
# Strings: 1,000,000 instances, ~50MB
```

**Ответ: Strategy, чтобы подставлять нужную реализацию динамически**

Паттерн **Strategy** позволяет **выбирать реализацию в runtime** в зависимости от типа платёжной системы, избегая цепочки `if-else`.

## Реализация:

### 1. Интерфейс стратегии:

```java
public interface PaymentProcessor {
    void processPayment(Payment payment);
    boolean supports(PaymentType type);
}
```

### 2. Реализации для каждой платёжной системы:

```java
@Component
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing credit card payment: " + payment.getAmount());
        // Логика обработки кредитной карты
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.CREDIT_CARD;
    }
}

@Component
public class PayPalProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing PayPal payment: " + payment.getAmount());
        // Логика обработки PayPal
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.PAYPAL;
    }
}

@Component
public class CryptoProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing crypto payment: " + payment.getAmount());
        // Логика обработки криптовалют
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.CRYPTO;
    }
}
```

### 3. Сервис для выбора стратегии:

```java
@Service
public class PaymentService {
    private final List<PaymentProcessor> processors;
    
    @Autowired
    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = processors.stream()
            .filter(p -> p.supports(payment.getType()))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No processor found for type: " + payment.getType()));
        
        processor.processPayment(payment);
    }
}
```

### 4. Использование:

```java
@RestController
@RequestMapping("/payments")
public class PaymentController {
    private final PaymentService paymentService;
    
    @PostMapping
    public ResponseEntity<String> processPayment(@RequestBody Payment payment) {
        paymentService.processPayment(payment);
        return ResponseEntity.ok("Payment processed");
    }
}
```

## Альтернатива — Map стратегий:

```java
@Service
public class PaymentService {
    private final Map<PaymentType, PaymentProcessor> processors;
    
    @Autowired
    public PaymentService(List<PaymentProcessor> processorList) {
        this.processors = processorList.stream()
            .collect(Collectors.toMap(
                p -> p.getSupportedType(),
                p -> p
            ));
    }
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = processors.get(payment.getType());
        if (processor == null) {
            throw new IllegalArgumentException("Unsupported payment type");
        }
        processor.processPayment(payment);
    }
}

// Реализации
public interface PaymentProcessor {
    void processPayment(Payment payment);
    PaymentType getSupportedType();
}

@Component
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public PaymentType getSupportedType() {
        return PaymentType.CREDIT_CARD;
    }
    
    @Override
    public void processPayment(Payment payment) {
        // ...
    }
}
```

## С использованием фабрики + Strategy:

```java
@Component
public class PaymentProcessorFactory {
    private final Map<PaymentType, PaymentProcessor> processors = new EnumMap<>(PaymentType.class);
    
    @Autowired
    public PaymentProcessorFactory(List<PaymentProcessor> processorList) {
        processorList.forEach(processor -> 
            processors.put(processor.getSupportedType(), processor)
        );
    }
    
    public PaymentProcessor getProcessor(PaymentType type) {
        PaymentProcessor processor = processors.get(type);
        if (processor == null) {
            throw new IllegalArgumentException("No processor for: " + type);
        }
        return processor;
    }
}

@Service
public class PaymentService {
    private final PaymentProcessorFactory factory;
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = factory.getProcessor(payment.getType());
        processor.processPayment(payment);
    }
}
```

## Без паттерна (плохо):

```java
// ❌ Плохо: if-else по типам
public void processPayment(Payment payment) {
    if (payment.getType() == PaymentType.CREDIT_CARD) {
        // Логика кредитной карты
    } else if (payment.getType() == PaymentType.PAYPAL) {
        // Логика PayPal
    } else if (payment.getType() == PaymentType.CRYPTO) {
        // Логика криптовалют
    } else {
        throw new IllegalArgumentException("Unknown type");
    }
}
// При добавлении новой системы нужно менять этот метод (нарушение OCP)
```

## Почему не другие паттерны:

### Singleton:
```java
// НЕ подходит: нужно несколько реализаций, не один экземпляр
```

### Adapter:
```java
// Для адаптации СУЩЕСТВУЮЩИХ интерфейсов, не для выбора алгоритма
class OldPaymentSystemAdapter implements PaymentProcessor {
    private OldPaymentSystem oldSystem;
    // Адаптирует старый интерфейс к новому
}
// Можно использовать ВМЕСТЕ со Strategy, если нужно
```

### Observer:
```java
// Для подписки на события, не для выбора реализации
```

## Преимущества Strategy:

✅ **Open/Closed Principle** — легко добавлять новые платёжные системы  
✅ **Single Responsibility** — каждая реализация в отдельном классе  
✅ **Нет if-else** — выбор через Map или List  
✅ **Тестируемость** — каждую стратегию можно тестировать отдельно  

## Добавление новой платёжной системы:

```java
// Просто создаём новый компонент
@Component
public class ApplePayProcessor implements PaymentProcessor {
    @Override
    public PaymentType getSupportedType() {
        return PaymentType.APPLE_PAY;
    }
    
    @Override
    public void processPayment(Payment payment) {
        // Логика Apple Pay
    }
}
// Spring автоматически подхватит, ничего менять не нужно!
```

**Вывод: Strategy — идеальный паттерн для избежания if-else по типам платёжек. Позволяет динамически выбирать реализацию, легко добавлять новые системы и следовать SOLID принципам.**

```java
public interface PaymentProcessor {
    void processPayment(Payment payment);
    boolean supports(PaymentType type);
}
```

```java
@Component
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing credit card payment: " + payment.getAmount());
        // Логика обработки кредитной карты
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.CREDIT_CARD;
    }
}

@Component
public class PayPalProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing PayPal payment: " + payment.getAmount());
        // Логика обработки PayPal
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.PAYPAL;
    }
}

@Component
public class CryptoProcessor implements PaymentProcessor {
    @Override
    public void processPayment(Payment payment) {
        System.out.println("Processing crypto payment: " + payment.getAmount());
        // Логика обработки криптовалют
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.CRYPTO;
    }
}
```

```java
@Service
public class PaymentService {
    private final List<PaymentProcessor> processors;
    
    @Autowired
    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = processors.stream()
            .filter(p -> p.supports(payment.getType()))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException(
                "No processor found for type: " + payment.getType()));
        
        processor.processPayment(payment);
    }
}
```

```java
@RestController
@RequestMapping("/payments")
public class PaymentController {
    private final PaymentService paymentService;
    
    @PostMapping
    public ResponseEntity<String> processPayment(@RequestBody Payment payment) {
        paymentService.processPayment(payment);
        return ResponseEntity.ok("Payment processed");
    }
}
```

```java
@Service
public class PaymentService {
    private final Map<PaymentType, PaymentProcessor> processors;
    
    @Autowired
    public PaymentService(List<PaymentProcessor> processorList) {
        this.processors = processorList.stream()
            .collect(Collectors.toMap(
                p -> p.getSupportedType(),
                p -> p
            ));
    }
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = processors.get(payment.getType());
        if (processor == null) {
            throw new IllegalArgumentException("Unsupported payment type");
        }
        processor.processPayment(payment);
    }
}

// Реализации
public interface PaymentProcessor {
    void processPayment(Payment payment);
    PaymentType getSupportedType();
}

@Component
public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public PaymentType getSupportedType() {
        return PaymentType.CREDIT_CARD;
    }
    
    @Override
    public void processPayment(Payment payment) {
        // ...
    }
}
```

```java
@Component
public class PaymentProcessorFactory {
    private final Map<PaymentType, PaymentProcessor> processors = new EnumMap<>(PaymentType.class);
    
    @Autowired
    public PaymentProcessorFactory(List<PaymentProcessor> processorList) {
        processorList.forEach(processor -> 
            processors.put(processor.getSupportedType(), processor)
        );
    }
    
    public PaymentProcessor getProcessor(PaymentType type) {
        PaymentProcessor processor = processors.get(type);
        if (processor == null) {
            throw new IllegalArgumentException("No processor for: " + type);
        }
        return processor;
    }
}

@Service
public class PaymentService {
    private final PaymentProcessorFactory factory;
    
    public void processPayment(Payment payment) {
        PaymentProcessor processor = factory.getProcessor(payment.getType());
        processor.processPayment(payment);
    }
}
```

```java
// ❌ Плохо: if-else по типам
public void processPayment(Payment payment) {
    if (payment.getType() == PaymentType.CREDIT_CARD) {
        // Логика кредитной карты
    } else if (payment.getType() == PaymentType.PAYPAL) {
        // Логика PayPal
    } else if (payment.getType() == PaymentType.CRYPTO) {
        // Логика криптовалют
    } else {
        throw new IllegalArgumentException("Unknown type");
    }
}
// При добавлении новой системы нужно менять этот метод (нарушение OCP)
```

```java
// НЕ подходит: нужно несколько реализаций, не один экземпляр
```

```java
// Для адаптации СУЩЕСТВУЮЩИХ интерфейсов, не для выбора алгоритма
class OldPaymentSystemAdapter implements PaymentProcessor {
    private OldPaymentSystem oldSystem;
    // Адаптирует старый интерфейс к новому
}
// Можно использовать ВМЕСТЕ со Strategy, если нужно
```

```java
// Для подписки на события, не для выбора реализации
```

```java
// Просто создаём новый компонент
@Component
public class ApplePayProcessor implements PaymentProcessor {
    @Override
    public PaymentType getSupportedType() {
        return PaymentType.APPLE_PAY;
    }
    
    @Override
    public void processPayment(Payment payment) {
        // Логика Apple Pay
    }
}
// Spring автоматически подхватит, ничего менять не нужно!
```

**Ответ: Будет выброшено UnsupportedOperationException**

`List.of()` создаёт **неизменяемый (immutable) список**. Любая попытка модификации (add, set, remove) вызовет `UnsupportedOperationException`.

## Что происходит:

```java
var list = List.of("a", "b", "c");  // Immutable list
list.set(1, "x");  // UnsupportedOperationException!
```

**Exception:**
```
Exception in thread "main" java.lang.UnsupportedOperationException
    at java.base/java.util.ImmutableCollections.uoe(ImmutableCollections.java:142)
    at java.base/java.util.ImmutableCollections$AbstractImmutableList.set(ImmutableCollections.java:260)
```

## Immutable коллекции в Java:

### List.of() — неизменяемый:
```java
List<String> immutable = List.of("a", "b", "c");
immutable.set(0, "x");    // ❌ UnsupportedOperationException
immutable.add("d");       // ❌ UnsupportedOperationException
immutable.remove(0);      // ❌ UnsupportedOperationException
immutable.clear();        // ❌ UnsupportedOperationException
```

### ArrayList — изменяемый:
```java
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));
mutable.set(1, "x");      // ✅ OK, теперь ["a", "x", "c"]
mutable.add("d");         // ✅ OK, теперь ["a", "x", "c", "d"]
```

## Другие immutable методы:

```java
// Set
Set<String> set = Set.of("a", "b", "c");
set.add("d");  // ❌ UnsupportedOperationException

// Map
Map<String, Integer> map = Map.of("a", 1, "b", 2);
map.put("c", 3);  // ❌ UnsupportedOperationException

// Arrays.asList() — частично изменяемый (fixed-size)
List<String> fixedSize = Arrays.asList("a", "b", "c");
fixedSize.set(1, "x");  // ✅ OK, можно изменять элементы
fixedSize.add("d");     // ❌ UnsupportedOperationException (нельзя менять размер)
```

## Как сделать изменяемую копию:

```java
// Способ 1: через конструктор ArrayList
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));
mutable.set(1, "x");  // ✅ OK

// Способ 2: через Stream
List<String> mutable2 = List.of("a", "b", "c")
    .stream()
    .collect(Collectors.toCollection(ArrayList::new));

// Способ 3: через List.copyOf() + ArrayList
List<String> mutable3 = new ArrayList<>(List.copyOf(List.of("a", "b", "c")));
```

## Преимущества immutable коллекций:

✅ **Thread-safe** — безопасны в многопоточности  
✅ **Память** — более компактны  
✅ **Производительность** — быстрее для чтения  
✅ **Безопасность** — нельзя случайно изменить  

## Сравнение:

| Метод | Изменяемый | Null элементы | Производительность |
|-------|------------|---------------|-------------------|
| `List.of()` | ❌ | ❌ | Быстрая |
| `Arrays.asList()` | Частично | ✅ | Средняя |
| `new ArrayList<>()` | ✅ | ✅ | Средняя |
| `Collections.unmodifiableList()` | ❌ (обёртка) | ✅ | Средняя |

## Другие операции с immutable списком:

```java
var list = List.of("a", "b", "c");

// Чтение — OK
String first = list.get(0);        // ✅ "a"
int size = list.size();            // ✅ 3
boolean contains = list.contains("b");  // ✅ true

// Модификация — Exception
list.set(0, "x");     // ❌
list.add("d");        // ❌
list.remove("a");     // ❌
list.clear();         // ❌
list.replaceAll(String::toUpperCase);  // ❌
list.sort(null);      // ❌
```

## Проверка на immutable:

```java
List<String> list = List.of("a", "b", "c");

// Нет встроенного метода, но можно попробовать изменить
try {
    list.add("test");
    System.out.println("Mutable");
} catch (UnsupportedOperationException e) {
    System.out.println("Immutable");  // Выведет это
}
```

**Вывод: `List.of()` создаёт immutable список. Попытка вызвать `set()`, `add()` или другие модифицирующие методы приведёт к `UnsupportedOperationException`. Для изменяемого списка используй `new ArrayList<>(List.of(...))`.**

```java
var list = List.of("a", "b", "c");  // Immutable list
list.set(1, "x");  // UnsupportedOperationException!
```

```plaintext
Exception in thread "main" java.lang.UnsupportedOperationException
    at java.base/java.util.ImmutableCollections.uoe(ImmutableCollections.java:142)
    at java.base/java.util.ImmutableCollections$AbstractImmutableList.set(ImmutableCollections.java:260)
```

```java
List<String> immutable = List.of("a", "b", "c");
immutable.set(0, "x");    // ❌ UnsupportedOperationException
immutable.add("d");       // ❌ UnsupportedOperationException
immutable.remove(0);      // ❌ UnsupportedOperationException
immutable.clear();        // ❌ UnsupportedOperationException
```

```java
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));
mutable.set(1, "x");      // ✅ OK, теперь ["a", "x", "c"]
mutable.add("d");         // ✅ OK, теперь ["a", "x", "c", "d"]
```

```java
// Set
Set<String> set = Set.of("a", "b", "c");
set.add("d");  // ❌ UnsupportedOperationException

// Map
Map<String, Integer> map = Map.of("a", 1, "b", 2);
map.put("c", 3);  // ❌ UnsupportedOperationException

// Arrays.asList() — частично изменяемый (fixed-size)
List<String> fixedSize = Arrays.asList("a", "b", "c");
fixedSize.set(1, "x");  // ✅ OK, можно изменять элементы
fixedSize.add("d");     // ❌ UnsupportedOperationException (нельзя менять размер)
```

```java
// Способ 1: через конструктор ArrayList
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));
mutable.set(1, "x");  // ✅ OK

// Способ 2: через Stream
List<String> mutable2 = List.of("a", "b", "c")
    .stream()
    .collect(Collectors.toCollection(ArrayList::new));

// Способ 3: через List.copyOf() + ArrayList
List<String> mutable3 = new ArrayList<>(List.copyOf(List.of("a", "b", "c")));
```

```java
var list = List.of("a", "b", "c");

// Чтение — OK
String first = list.get(0);        // ✅ "a"
int size = list.size();            // ✅ 3
boolean contains = list.contains("b");  // ✅ true

// Модификация — Exception
list.set(0, "x");     // ❌
list.add("d");        // ❌
list.remove("a");     // ❌
list.clear();         // ❌
list.replaceAll(String::toUpperCase);  // ❌
list.sort(null);      // ❌
```

```java
List<String> list = List.of("a", "b", "c");

// Нет встроенного метода, но можно попробовать изменить
try {
    list.add("test");
    System.out.println("Mutable");
} catch (UnsupportedOperationException e) {
    System.out.println("Immutable");  // Выведет это
}
```

**Ответ: `count()` выполняется до выхода из блока try-with-resources**

Метод `count()` — это **терминальная операция Stream**, которая **запускает обработку** всего потока. Она выполняется **внутри try-with-resources блока**, поэтому **может выбросить `IOException`**, которое компилятор требует обработать.

## Проблема:

```java
try (var reader = new BufferedReader(new FileReader("data.txt"))) {
    return reader.lines().count();  
    // count() читает файл → может выбросить IOException
    // Но IOException не обработан!
}
```

**Ошибка компиляции:**
```
error: unreported exception IOException; must be caught or declared to be thrown
    return reader.lines().count();
                         ^
```

## Почему это происходит:

### Ленивость Stream:
```java
Stream<String> stream = reader.lines();  // Не читает файл, только создаёт Stream
// Файл НЕ прочитан здесь

long count = stream.count();  // Здесь происходит чтение файла
// IOException может быть выброшен здесь
```

### Выполнение внутри try-with-resources:
```java
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    return reader.lines().count();  
    // ← count() выполняется ДО выхода из try
    // ← reader еще НЕ закрыт
    // ← IOException может произойти здесь
}
// ← reader закрывается здесь
```

## Решения:

### 1. Добавить `throws IOException`:

```java
public long countLines() throws IOException {
    try (var reader = new BufferedReader(new FileReader("data.txt"))) {
        return reader.lines().count();
    }
}
```

### 2. Обработать IOException:

```java
public long countLines() {
    try (var reader = new BufferedReader(new FileReader("data.txt"))) {
        return reader.lines().count();
    } catch (IOException e) {
        throw new RuntimeException("Failed to read file", e);
    }
}
```

### 3. Использовать Files.lines() (рекомендуется):

```java
public long countLines() throws IOException {
    try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
        return lines.count();
    }
}
```

### 4. Или без try-with-resources (Files.lines закроет автоматически):

```java
public long countLines() throws IOException {
    return Files.lines(Path.of("data.txt")).count();
    // Files.lines() создаёт Stream, который закроется после count()
}
```

⚠️ **Внимание:** последний вариант может привести к утечке ресурсов при ошибке до вызова `count()`. Лучше использовать явный try-with-resources.

## Почему не другие варианты:

### ❌ "Поток ленивый, чтение после выхода из try":
```java
// Неверно! count() выполняется ДО выхода
try (var reader = new BufferedReader(...)) {
    return reader.lines().count();  // Здесь, внутри try
}
```

### ❌ "Stream всегда завершает ресурсы до конца чтения":
```java
// Неверно! Stream не закрывает reader автоматически
// Только try-with-resources закрывает
```

### ❌ "lines() всегда буферизует весь файл":
```java
// Неверно! lines() читает файл построчно (ленивый Stream)
// Но count() должен прочитать все строки
```

## Детально про reader.lines():

```java
Stream<String> lines = reader.lines();
// Возвращает Stream, который:
// 1. НЕ прочитал файл еще
// 2. Прочитает его при вызове терминальной операции (count, forEach, collect)
// 3. НЕ закрывает reader автоматически (нужен try-with-resources)
```

## Правильный паттерн:

```java
// Хорошо
public long countLines(String filename) throws IOException {
    try (Stream<String> lines = Files.lines(Path.of(filename))) {
        return lines.count();
    }
}

// Еще лучше (с обработкой ошибок)
public long countLines(String filename) {
    try (Stream<String> lines = Files.lines(Path.of(filename))) {
        return lines.count();
    } catch (IOException e) {
        log.error("Failed to count lines in {}", filename, e);
        return 0;
    }
}
```

## Альтернатива без Stream:

```java
public long countLines(String filename) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
        long count = 0;
        while (reader.readLine() != null) {
            count++;
        }
        return count;
    }
}
```

**Вывод: `count()` — терминальная операция, которая выполняется внутри try-with-resources блока и может выбросить `IOException`. Компилятор требует обработать это checked exception через `throws` или `try-catch`.**

```java
try (var reader = new BufferedReader(new FileReader("data.txt"))) {
    return reader.lines().count();  
    // count() читает файл → может выбросить IOException
    // Но IOException не обработан!
}
```

```plaintext
error: unreported exception IOException; must be caught or declared to be thrown
    return reader.lines().count();
                         ^
```

```java
Stream<String> stream = reader.lines();  // Не читает файл, только создаёт Stream
// Файл НЕ прочитан здесь

long count = stream.count();  // Здесь происходит чтение файла
// IOException может быть выброшен здесь
```

```java
try (BufferedReader reader = new BufferedReader(new FileReader("data.txt"))) {
    return reader.lines().count();  
    // ← count() выполняется ДО выхода из try
    // ← reader еще НЕ закрыт
    // ← IOException может произойти здесь
}
// ← reader закрывается здесь
```

```java
public long countLines() throws IOException {
    try (var reader = new BufferedReader(new FileReader("data.txt"))) {
        return reader.lines().count();
    }
}
```

```java
public long countLines() {
    try (var reader = new BufferedReader(new FileReader("data.txt"))) {
        return reader.lines().count();
    } catch (IOException e) {
        throw new RuntimeException("Failed to read file", e);
    }
}
```

```java
public long countLines() throws IOException {
    try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
        return lines.count();
    }
}
```

```java
public long countLines() throws IOException {
    return Files.lines(Path.of("data.txt")).count();
    // Files.lines() создаёт Stream, который закроется после count()
}
```

```java
// Неверно! count() выполняется ДО выхода
try (var reader = new BufferedReader(...)) {
    return reader.lines().count();  // Здесь, внутри try
}
```

```java
// Неверно! Stream не закрывает reader автоматически
// Только try-with-resources закрывает
```

```java
// Неверно! lines() читает файл построчно (ленивый Stream)
// Но count() должен прочитать все строки
```

```java
Stream<String> lines = reader.lines();
// Возвращает Stream, который:
// 1. НЕ прочитал файл еще
// 2. Прочитает его при вызове терминальной операции (count, forEach, collect)
// 3. НЕ закрывает reader автоматически (нужен try-with-resources)
```

```java
// Хорошо
public long countLines(String filename) throws IOException {
    try (Stream<String> lines = Files.lines(Path.of(filename))) {
        return lines.count();
    }
}

// Еще лучше (с обработкой ошибок)
public long countLines(String filename) {
    try (Stream<String> lines = Files.lines(Path.of(filename))) {
        return lines.count();
    } catch (IOException e) {
        log.error("Failed to count lines in {}", filename, e);
        return 0;
    }
}
```

```java
public long countLines(String filename) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(filename))) {
        long count = 0;
        while (reader.readLine() != null) {
            count++;
        }
        return count;
    }
}
```

**Ответ: Использовать пагинацию через Pageable или Stream API**

Загрузка 10,000 пользователей сразу в `List<User>` создаёт огромный объект в памяти. Лучше использовать **пагинацию** (обработка порциями) или **Stream API** (потоковая обработка).

## Проблема:

```java
List<User> users = repo.findAll();  // 10,000 объектов в памяти сразу
// Heap: ~50-200 MB (зависит от размера User)
// Все данные загружены, даже если не используются
```

## Решения:

### 1. Пагинация через Pageable (рекомендуется):

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
    
    public void processAllUsers() {
        int pageSize = 100;
        int pageNumber = 0;
        Page<User> page;
        
        do {
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            page = repo.findAll(pageable);
            
            // Обработка текущей страницы
            page.getContent().forEach(this::processUser);
            
            pageNumber++;
        } while (page.hasNext());
    }
    
    private void processUser(User user) {
        // Обработка одного пользователя
    }
}
```

**Преимущества:**
- ✅ Загружается только 100 пользователей за раз
- ✅ Память: ~1-2 MB вместо 50-200 MB
- ✅ Предсказуемое потребление памяти

### 2. Stream API с @QueryHints:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    @QueryHints(value = @QueryHint(name = HINT_FETCH_SIZE, value = "100"))
    @Query("SELECT u FROM User u")
    Stream<User> streamAll();
}

@Service
public class UserService {
    @Transactional(readOnly = true)
    public void processAllUsers() {
        try (Stream<User> users = repo.streamAll()) {
            users.forEach(this::processUser);
            // Обрабатывается по одному, не загружает все в память
        }
    }
}
```

**⚠️ Важно:**
- Требует `@Transactional` (Stream должен работать в рамках транзакции)
- Нужно закрывать Stream (try-with-resources)

### 3. Scrollable Results (Hibernate):

```java
@Repository
public class UserDao {
    @PersistenceContext
    private EntityManager entityManager;
    
    public void processAllUsers() {
        Session session = entityManager.unwrap(Session.class);
        
        ScrollableResults<User> results = session
            .createQuery("FROM User", User.class)
            .setFetchSize(100)
            .scroll(ScrollMode.FORWARD_ONLY);
        
        while (results.next()) {
            User user = results.get();
            processUser(user);
            
            // Очистить persistence context для освобождения памяти
            if (results.getRowNumber() % 100 == 0) {
                session.flush();
                session.clear();
            }
        }
        results.close();
    }
}
```

### 4. Batch processing с пагинацией (самый эффективный):

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
    
    @Transactional(readOnly = true)
    public void processAllUsers() {
        int batchSize = 100;
        int pageNumber = 0;
        
        Slice<User> slice;
        do {
            Pageable pageable = PageRequest.of(pageNumber++, batchSize);
            slice = repo.findAll(pageable);
            
            // Обработка батча
            processUserBatch(slice.getContent());
            
        } while (slice.hasNext());
    }
    
    private void processUserBatch(List<User> users) {
        users.forEach(this::processUser);
    }
}
```

### 5. Проекция (если нужны не все поля):

```java
// DTO или Projection
public interface UserSummary {
    Long getId();
    String getName();
}

public interface UserRepository extends JpaRepository<User, Long> {
    Page<UserSummary> findAllBy(Pageable pageable);
}

// Использование
@Service
public class UserService {
    public void processAllUsers() {
        int pageNumber = 0;
        Page<UserSummary> page;
        
        do {
            page = repo.findAllBy(PageRequest.of(pageNumber++, 1000));
            page.forEach(user -> {
                // Работаем только с id и name, экономим память
            });
        } while (page.hasNext());
    }
}
```

## Сравнение подходов:

| Подход | Память | Скорость | Сложность | Транзакция |
|--------|--------|----------|-----------|------------|
| **`findAll()`** | ❌ Высокая | Средняя | Простая | Нет |
| **Pageable** | ✅ Низкая | Средняя | Простая | Нет |
| **Stream API** | ✅ Низкая | Быстрая | Средняя | ✅ Да |
| **ScrollableResults** | ✅ Низкая | Быстрая | Сложная | ✅ Да |
| **Projection** | ✅ Очень низкая | Быстрая | Средняя | Нет |

## Почему не другие варианты:

### ❌ "Хранить в Set, а не в List":
```java
Set<User> users = new HashSet<>(repo.findAll());
// Проблема та же: 10,000 объектов в памяти
```

### ❌ "Заменить JPA на чистый JDBC":
```java
// Может помочь, но усложняет код
// Лучше использовать Stream или Pageable с JPA
```

### ❌ "Добавить кэш второго уровня":
```java
// Не решает проблему: всё равно загружаем 10,000 объектов
```

## Best practice для больших таблиц:

```java
@Service
public class UserService {
    private static final int BATCH_SIZE = 500;
    
    @Autowired
    private UserRepository repo;
    
    @Autowired
    private EntityManager entityManager;
    
    @Transactional(readOnly = true)
    public void processAllUsers() {
        long totalUsers = repo.count();
        int totalPages = (int) Math.ceil((double) totalUsers / BATCH_SIZE);
        
        for (int page = 0; page < totalPages; page++) {
            Pageable pageable = PageRequest.of(page, BATCH_SIZE);
            List<User> users = repo.findAll(pageable).getContent();
            
            processUserBatch(users);
            
            // Очистить persistence context
            entityManager.clear();
        }
    }
    
    private void processUserBatch(List<User> users) {
        users.forEach(user -> {
            // Обработка пользователя
        });
    }
}
```

**Вывод: `findAll()` загружает все 10,000 записей в память → OutOfMemoryError или деградация производительности. Используй **Pageable** для пагинации или **Stream API** для потоковой обработки. Память снизится с ~100MB до ~1-5MB.**

```java
List<User> users = repo.findAll();  // 10,000 объектов в памяти сразу
// Heap: ~50-200 MB (зависит от размера User)
// Все данные загружены, даже если не используются
```

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
    
    public void processAllUsers() {
        int pageSize = 100;
        int pageNumber = 0;
        Page<User> page;
        
        do {
            Pageable pageable = PageRequest.of(pageNumber, pageSize);
            page = repo.findAll(pageable);
            
            // Обработка текущей страницы
            page.getContent().forEach(this::processUser);
            
            pageNumber++;
        } while (page.hasNext());
    }
    
    private void processUser(User user) {
        // Обработка одного пользователя
    }
}
```

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    @QueryHints(value = @QueryHint(name = HINT_FETCH_SIZE, value = "100"))
    @Query("SELECT u FROM User u")
    Stream<User> streamAll();
}

@Service
public class UserService {
    @Transactional(readOnly = true)
    public void processAllUsers() {
        try (Stream<User> users = repo.streamAll()) {
            users.forEach(this::processUser);
            // Обрабатывается по одному, не загружает все в память
        }
    }
}
```

```java
@Repository
public class UserDao {
    @PersistenceContext
    private EntityManager entityManager;
    
    public void processAllUsers() {
        Session session = entityManager.unwrap(Session.class);
        
        ScrollableResults<User> results = session
            .createQuery("FROM User", User.class)
            .setFetchSize(100)
            .scroll(ScrollMode.FORWARD_ONLY);
        
        while (results.next()) {
            User user = results.get();
            processUser(user);
            
            // Очистить persistence context для освобождения памяти
            if (results.getRowNumber() % 100 == 0) {
                session.flush();
                session.clear();
            }
        }
        results.close();
    }
}
```

```java
@Service
public class UserService {
    @Autowired
    private UserRepository repo;
    
    @Transactional(readOnly = true)
    public void processAllUsers() {
        int batchSize = 100;
        int pageNumber = 0;
        
        Slice<User> slice;
        do {
            Pageable pageable = PageRequest.of(pageNumber++, batchSize);
            slice = repo.findAll(pageable);
            
            // Обработка батча
            processUserBatch(slice.getContent());
            
        } while (slice.hasNext());
    }
    
    private void processUserBatch(List<User> users) {
        users.forEach(this::processUser);
    }
}
```

```java
// DTO или Projection
public interface UserSummary {
    Long getId();
    String getName();
}

public interface UserRepository extends JpaRepository<User, Long> {
    Page<UserSummary> findAllBy(Pageable pageable);
}

// Использование
@Service
public class UserService {
    public void processAllUsers() {
        int pageNumber = 0;
        Page<UserSummary> page;
        
        do {
            page = repo.findAllBy(PageRequest.of(pageNumber++, 1000));
            page.forEach(user -> {
                // Работаем только с id и name, экономим память
            });
        } while (page.hasNext());
    }
}
```

```java
Set<User> users = new HashSet<>(repo.findAll());
// Проблема та же: 10,000 объектов в памяти
```

```java
// Может помочь, но усложняет код
// Лучше использовать Stream или Pageable с JPA
```

```java
// Не решает проблему: всё равно загружаем 10,000 объектов
```

```java
@Service
public class UserService {
    private static final int BATCH_SIZE = 500;
    
    @Autowired
    private UserRepository repo;
    
    @Autowired
    private EntityManager entityManager;
    
    @Transactional(readOnly = true)
    public void processAllUsers() {
        long totalUsers = repo.count();
        int totalPages = (int) Math.ceil((double) totalUsers / BATCH_SIZE);
        
        for (int page = 0; page < totalPages; page++) {
            Pageable pageable = PageRequest.of(page, BATCH_SIZE);
            List<User> users = repo.findAll(pageable).getContent();
            
            processUserBatch(users);
            
            // Очистить persistence context
            entityManager.clear();
        }
    }
    
    private void processUserBatch(List<User> users) {
        users.forEach(user -> {
            // Обработка пользователя
        });
    }
}
```

**Ответ: Open/Closed — расширение через новые реализации, без изменения старого кода**

Принцип **Open/Closed (OCP)** из SOLID: класс должен быть **открыт для расширения**, но **закрыт для модификации**. Новые алгоритмы расчёта скидок добавляются через новые классы, без изменения существующего кода.

## Как применяется к алгоритмам скидок:

### ❌ Плохо (нарушение OCP):

```java
public class DiscountCalculator {
    public double calculate(Order order, String discountType) {
        if (discountType.equals("PERCENTAGE")) {
            return order.getTotal() * 0.1;
        } else if (discountType.equals("FIXED")) {
            return 50.0;
        } else if (discountType.equals("BLACK_FRIDAY")) {
            return order.getTotal() * 0.3;
        }
        // При добавлении нового типа нужно менять ЭТОТ код!
        return 0;
    }
}
```

**Проблема:** При добавлении нового алгоритма (например, "VIP_DISCOUNT") нужно **модифицировать** класс `DiscountCalculator` → нарушение OCP.

### ✅ Хорошо (следование OCP):

```java
// Интерфейс (абстракция)
public interface DiscountStrategy {
    double calculate(Order order);
}

// Существующие реализации
public class PercentageDiscount implements DiscountStrategy {
    private final double percentage;
    
    public PercentageDiscount(double percentage) {
        this.percentage = percentage;
    }
    
    @Override
    public double calculate(Order order) {
        return order.getTotal() * percentage;
    }
}

public class FixedDiscount implements DiscountStrategy {
    private final double amount;
    
    public FixedDiscount(double amount) {
        this.amount = amount;
    }
    
    @Override
    public double calculate(Order order) {
        return amount;
    }
}

// Новый алгоритм — НЕ МЕНЯЕМ старый код!
public class BlackFridayDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        return order.getTotal() * 0.3;
    }
}

// Еще один новый алгоритм
public class VipDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        if (order.getCustomer().isVip()) {
            return order.getTotal() * 0.2;
        }
        return 0;
    }
}
```

**Использование:**

```java
@Service
public class OrderService {
    public double applyDiscount(Order order, DiscountStrategy strategy) {
        double discount = strategy.calculate(order);
        return order.getTotal() - discount;
    }
}

// Применение
Order order = new Order(1000);

// Старые алгоритмы
orderService.applyDiscount(order, new PercentageDiscount(0.1));
orderService.applyDiscount(order, new FixedDiscount(50));

// Новые алгоритмы (добавлены БЕЗ изменения старого кода)
orderService.applyDiscount(order, new BlackFridayDiscount());
orderService.applyDiscount(order, new VipDiscount());
```

## Open/Closed в действии:

**Открыт для расширения:**
```java
// Добавляем новый алгоритм — создаём новый класс
public class SeasonalDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        LocalDate now = LocalDate.now();
        if (now.getMonth() == Month.DECEMBER) {
            return order.getTotal() * 0.25;
        }
        return 0;
    }
}
```

**Закрыт для модификации:**
```java
// Классы PercentageDiscount, FixedDiscount, BlackFridayDiscount
// НЕ ИЗМЕНЯЮТСЯ при добавлении SeasonalDiscount
```

## Почему не другие принципы:

### Dependency Inversion:
```java
// Зависимость от абстракции, а не реализации
public class OrderService {
    private final DiscountStrategy strategy;  // Зависимость от интерфейса
    // ...
}
// Это DIP, но не решает проблему добавления новых алгоритмов
```

### Liskov Substitution:
```java
// Подклассы должны быть заменяемы без изменения поведения
class A {}
class B extends A {}
// Не о добавлении новых алгоритмов
```

### Interface Segregation:
```java
// Разделение больших интерфейсов на мелкие
interface PaymentProcessor {
    void processPayment();
    void refund();
    void generateReport();  // Не все реализации нуждаются в этом
}

// Лучше разделить
interface PaymentProcessor {
    void processPayment();
}

interface RefundablePayment {
    void refund();
}
// Не о расширяемости через новые реализации
```

## OCP с другими паттернами:

### Strategy + Factory:
```java
@Component
public class DiscountFactory {
    public DiscountStrategy getStrategy(String type) {
        return switch (type) {
            case "PERCENTAGE" -> new PercentageDiscount(0.1);
            case "FIXED" -> new FixedDiscount(50);
            case "BLACK_FRIDAY" -> new BlackFridayDiscount();
            case "VIP" -> new VipDiscount();
            default -> throw new IllegalArgumentException("Unknown discount type");
        };
    }
}
```

### Spring + автоматическая регистрация:
```java
@Component
public class PercentageDiscount implements DiscountStrategy {
    // ...
}

@Component
public class BlackFridayDiscount implements DiscountStrategy {
    // ...
}

@Service
public class DiscountService {
    private final Map<String, DiscountStrategy> strategies;
    
    @Autowired
    public DiscountService(List<DiscountStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                s -> s.getClass().getSimpleName(),
                s -> s
            ));
    }
    
    public double applyDiscount(Order order, String strategyName) {
        DiscountStrategy strategy = strategies.get(strategyName);
        return strategy.calculate(order);
    }
}
```

## Преимущества OCP:

✅ **Расширяемость** — новые алгоритмы без изменения кода  
✅ **Тестируемость** — каждый алгоритм тестируется отдельно  
✅ **Поддерживаемость** — старый код не ломается  
✅ **Single Responsibility** — каждый класс отвечает за один алгоритм  

**Вывод: Open/Closed — ключевой принцип для расширяемости. Новые алгоритмы скидок добавляются через новые классы (открыт для расширения), без модификации существующих классов (закрыт для изменения). Используй абстракции (интерфейсы) + Strategy pattern.**

```java
public class DiscountCalculator {
    public double calculate(Order order, String discountType) {
        if (discountType.equals("PERCENTAGE")) {
            return order.getTotal() * 0.1;
        } else if (discountType.equals("FIXED")) {
            return 50.0;
        } else if (discountType.equals("BLACK_FRIDAY")) {
            return order.getTotal() * 0.3;
        }
        // При добавлении нового типа нужно менять ЭТОТ код!
        return 0;
    }
}
```

```java
// Интерфейс (абстракция)
public interface DiscountStrategy {
    double calculate(Order order);
}

// Существующие реализации
public class PercentageDiscount implements DiscountStrategy {
    private final double percentage;
    
    public PercentageDiscount(double percentage) {
        this.percentage = percentage;
    }
    
    @Override
    public double calculate(Order order) {
        return order.getTotal() * percentage;
    }
}

public class FixedDiscount implements DiscountStrategy {
    private final double amount;
    
    public FixedDiscount(double amount) {
        this.amount = amount;
    }
    
    @Override
    public double calculate(Order order) {
        return amount;
    }
}

// Новый алгоритм — НЕ МЕНЯЕМ старый код!
public class BlackFridayDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        return order.getTotal() * 0.3;
    }
}

// Еще один новый алгоритм
public class VipDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        if (order.getCustomer().isVip()) {
            return order.getTotal() * 0.2;
        }
        return 0;
    }
}
```

```java
@Service
public class OrderService {
    public double applyDiscount(Order order, DiscountStrategy strategy) {
        double discount = strategy.calculate(order);
        return order.getTotal() - discount;
    }
}

// Применение
Order order = new Order(1000);

// Старые алгоритмы
orderService.applyDiscount(order, new PercentageDiscount(0.1));
orderService.applyDiscount(order, new FixedDiscount(50));

// Новые алгоритмы (добавлены БЕЗ изменения старого кода)
orderService.applyDiscount(order, new BlackFridayDiscount());
orderService.applyDiscount(order, new VipDiscount());
```

```java
// Добавляем новый алгоритм — создаём новый класс
public class SeasonalDiscount implements DiscountStrategy {
    @Override
    public double calculate(Order order) {
        LocalDate now = LocalDate.now();
        if (now.getMonth() == Month.DECEMBER) {
            return order.getTotal() * 0.25;
        }
        return 0;
    }
}
```

```java
// Классы PercentageDiscount, FixedDiscount, BlackFridayDiscount
// НЕ ИЗМЕНЯЮТСЯ при добавлении SeasonalDiscount
```

```java
// Зависимость от абстракции, а не реализации
public class OrderService {
    private final DiscountStrategy strategy;  // Зависимость от интерфейса
    // ...
}
// Это DIP, но не решает проблему добавления новых алгоритмов
```

```java
// Подклассы должны быть заменяемы без изменения поведения
class A {}
class B extends A {}
// Не о добавлении новых алгоритмов
```

```java
// Разделение больших интерфейсов на мелкие
interface PaymentProcessor {
    void processPayment();
    void refund();
    void generateReport();  // Не все реализации нуждаются в этом
}

// Лучше разделить
interface PaymentProcessor {
    void processPayment();
}

interface RefundablePayment {
    void refund();
}
// Не о расширяемости через новые реализации
```

```java
@Component
public class DiscountFactory {
    public DiscountStrategy getStrategy(String type) {
        return switch (type) {
            case "PERCENTAGE" -> new PercentageDiscount(0.1);
            case "FIXED" -> new FixedDiscount(50);
            case "BLACK_FRIDAY" -> new BlackFridayDiscount();
            case "VIP" -> new VipDiscount();
            default -> throw new IllegalArgumentException("Unknown discount type");
        };
    }
}
```

```java
@Component
public class PercentageDiscount implements DiscountStrategy {
    // ...
}

@Component
public class BlackFridayDiscount implements DiscountStrategy {
    // ...
}

@Service
public class DiscountService {
    private final Map<String, DiscountStrategy> strategies;
    
    @Autowired
    public DiscountService(List<DiscountStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(
                s -> s.getClass().getSimpleName(),
                s -> s
            ));
    }
    
    public double applyDiscount(Order order, String strategyName) {
        DiscountStrategy strategy = strategies.get(strategyName);
        return strategy.calculate(order);
    }
}
```

