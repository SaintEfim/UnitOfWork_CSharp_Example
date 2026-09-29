# Unit of Work

Unit of Work (UoW) — паттерн, который объединяет изменения, сделанные в рамках одной бизнес-операции, и координирует их сохранение в хранилище на стороне бэкенда.

Чтобы легче понять, зачем нам нужен этот паттерн, можно начать с проблемы.

## Проблема

```csharp
orderRepository.Delete(orderId);
sampleRepository.Delete(sampleId);

await orderRepository.SaveChangesAsync(cancellationToken);
```

На первый взгляд проблемы как таковой нет, ведь метод `SaveChangesAsync` может быть обёрткой над базовой реализацией сохранения результата класса `DbContext`. Да и ошибки у нас никакой не будет: все данные сохранятся в базу.

В самом примере проблема скорее архитектурная. Например, предположим, что мы пишем сервис, придерживаясь паттерна DDD. Наши репозитории в таком случае должны отражать, какие операции мы можем проводить при работе с хранилищем, и всё это должно быть на языке бизнеса. Почему один из репозиториев должен определять границу транзакции, охватывающей изменения нескольких агрегатов?

Вернёмся к примеру выше:

```csharp
orderRepository.Delete(orderId);
sampleRepository.Delete(sampleId);

await orderRepository.SaveChangesAsync(cancellationToken);
```

Почему мы здесь решили, что надо вызвать `SaveChangesAsync` именно у `orderRepository`? Почему не у `sampleRepository`?

Как раз для решения этой дилеммы и существует такой паттерн, как UoW. Всю ответственность за транзакции и сохранение изменений мы полностью возлагаем на него. Тогда наш пример немного изменится:

```csharp
orderRepository.Delete(orderId);
sampleRepository.Delete(sampleId);

await context.SaveChangesAsync(cancellationToken);
```

При этом оба репозитория должны работать с одним экземпляром `DbContext` в рамках текущей операции.

Здесь мы обратились напрямую к `DbContext`, который сам по себе является реализацией этого паттерна. При необходимости поверх контекста можно написать обёртку, которую обычно называют `UnitOfWork`. Она позволяет явно обозначить границу бизнес-операции и не связывать прикладной слой непосредственно с EF Core.

```csharp
orderRepository.Delete(orderId);
sampleRepository.Delete(sampleId);

await unitOfWork.SaveChangesAsync(cancellationToken);
```

Базовая реализация UoW будет выглядеть так:

```csharp
internal sealed class UnitOfWork(ApplicationDbContext context) : IUnitOfWork
{
    public Task<int> SaveChangesAsync(
        CancellationToken cancellationToken = default)
    {
        return context.SaveChangesAsync(cancellationToken);
    }

    // Остальная реализация опущена
}
```

## А что, если репозитории сами сохраняют изменения?

А что, если у меня есть базовая реализация репозитория, где при каждом изменении вызывается `context.SaveChangesAsync()`?

Тогда наш пример будет выглядеть так:

```csharp
await orderRepository.DeleteAsync(orderId, token);
await sampleRepository.DeleteAsync(sampleId, token);
```

Ура, кода стало меньше, да и посредник пропал за ненадобностью, скажете вы. Но теперь, если второй вызов `sampleRepository.DeleteAsync` завершится с ошибкой, результат предыдущей операции не откатится. Два наших вызова стали атомарными по отдельности: каждый из них завершается в своей транзакции. Для решения этой проблемы нам всё равно придётся добавить явную транзакцию с возможностью отката:

```csharp
await unitOfWork.BeginTransactionAsync(token);

try
{
    await orderRepository.DeleteAsync(orderId, token);
    await sampleRepository.DeleteAsync(sampleId, token);

    await unitOfWork.CommitTransactionAsync(token);
}
catch
{
    await unitOfWork.RollbackTransactionAsync(token);
    throw;
}
```

Кода стало намного больше, да и сложность увеличилась. Теперь на разработчике лежит ответственность за атомарность всей операции. Такой подход с ручным управлением транзакциями в целом приемлем, но всё зависит от архитектуры. В большинстве задач хватит и `SaveChangesAsync`.

Вот такие пироги.
