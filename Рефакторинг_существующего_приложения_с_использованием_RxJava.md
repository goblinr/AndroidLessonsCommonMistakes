# Рефакторинг существующего приложения с использованием RxJava
Разберем частые ошибки при выполнении данного задания студентами

#### Просачивание Rx на уровень View
Очень важно держать View как можно более простой. Когда View начинает внутри себя использовать `Rx`, организовывать подписки и управлятьэтими подписками - нарушается принцип **S**ingle Responsobility

#### Не следят за жизненным циклом в контексте подписки
```java
observable.map(...)
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(...);
```
Подписка в данном случае просто игнорируется, что как минимум, может привести к утечке памяти.

#### Постоянно копятся подписки в `CompositeDisposable`
```java
public void updateList(@Nullable String selector) {
    disposable.add(repository.getContacts(selector)
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .doOnSubscribe(d -> getViewState().showLoading())
            .doFinally(() -> getViewState().hideLoading())
            .subscribe(contacts -> getViewState().updateList(contacts)));
}
```

В данном случае метод `updateList` вызывается каждый раз при изменении пользовательского ввода, что приводит к добавлению каждый раз новой подписки в `CompositeDisposable`, при чем старые подписки - не отписываются. Это приводит к ухудшению производительности приложения.  
**Выход:** обернуть события пользовательского ввода в "горячий" `Observable` и воспользоваться оператором `switchMapSingle`, бонусом, можно добавить оператор `debounce`(Но будьте осторожны, обязательно почитайте документацию по этому оператору). Подписка на этот "горячий" `Observable`, в итоге, будет только одна.

#### Не обрабатывают терминальное состояние `onError`
Это всегда приводит к крэшу приложения в случае возникновения ошибки, что максимально негативно сказывается на пользовательском опыте.

#### Неверное понимание работы оператора `just`
```java
Observable.just(contactDetailsRepository.getDetailsContact(id));
```
В данном случае, данные из репозитория начнут загружаться в момент создания `Observable`, а не когда произойдет подписка на него. Естественно, такое поведение заблокирует главный поток - что очень плохо.  
Нужно использовать асинхронные операторы:
```java
Single.fromCallable(() -> contactDetailsRepository.getDetailsContact(id));
```
