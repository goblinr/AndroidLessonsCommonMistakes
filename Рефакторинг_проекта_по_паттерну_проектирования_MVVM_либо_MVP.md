# Рефакторинг проекта по паттерну проектирования MVVM либо MVP
Разберем частые ошибки при выполнении данного задания студентами

#### Из публичного интерфейса репозитория доступна `MutableLiveData`
Пример кода студента:
```java
public interface IssueRepository {
     MutableLiveData<ArrayList<Contact>> loadContactList(Context context);
     MutableLiveData<Contact> loadContact(Context context, String id);
}
```
Что здесь не верно:
1. Для любого кто будет использовать этот интерфейс есть возможность изменять данные в `LiveData`
2. Метод `loadContactList` предоставляет конкретную реализацию `ArrayList` что ограничивает в возможностях предоставить какую-либо другую реализацию интерфейса `List` в реализации `IssueRepository`

Исправленный вариант:
```java
public interface IssueRepository {
     LiveData<List<Contact>> loadContactList(Context context);
     LiveData<Contact> loadContact(Context context, String id);
}
```

#### Получение инстанса `ViewModel` при каждом запросе данных
```java
public void queryContactDetails(String id) {
    /*CODE*/
    ContactDetailsViewModel model = new ViewModelProvider(this).get(ContactDetailsViewModel.class);
    LiveData<Contact> data = model.getData(id);
    /*CODE*/
}
```
Инстанс `ViewModel` достаточно получать 1 раз в `onCreate` фрагмента/активности

#### Неверный выбор стратегии для команд `Moxy`
Студенты забывают изучить документацию - это приводит к неверному выбору стратегий: [стратегии](https://github.com/Arello-Mobile/Moxy/wiki/View-commands-state-strategy)
```java
public interface ContactDetailsView extends MvpView {
    @StateStrategyType(OneExecutionStateStrategy.class)
    void showContactDetail(Contact contact);
}
```
В данном случае, стратегия выбрана совершенно неверно. Исправленный вариант:
```java
public interface ContactDetailsView extends MvpView {
    @StateStrategyType(AddToEndSingleStateStrategy.class)
    void showContactDetail(Contact contact);
}
```

#### Использование одного презентера/ViewModel для активности на оба фрагмента
Такой подход нарушает принцип единственной ответственности, т.к. активность теперь становится ответственной за:
1. Показ и обновление списка контактов
2. Показ и обновление деталей контакта

А так же в случае с Moxy - нарушается принцип разделения интерфейсов, реальный при мер из работы студента, который нарушает данный принцип:
```java
public interface MainView extends MvpView {
    @StateStrategyType(SingleStateStrategy.class)
    void showList();
    @StateStrategyType(SingleStateStrategy.class)
    void updateList(ArrayList<Contact> contacts);
    @StateStrategyType(OneExecutionStateStrategy.class)
    void showDetails(int id);
    @StateStrategyType(SingleStateStrategy.class)
    void updateDetails(Contact contact);
}
```

#### Получение ViewModel без `ViewModelProvider`
```java
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getActivity().setTitle(R.string.toolbar_title_contactDetails);
    model = new ContactDetailsViewModel(requireActivity().getApplication());
}
```
Естественно, так делать нельзя.
