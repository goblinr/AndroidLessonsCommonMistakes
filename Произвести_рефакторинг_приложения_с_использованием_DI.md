# Произвести рефакторинг приложения с использованием DI
Разберем частые ошибки при выполнении данного задания студентами

#### Избыточный `inject`
Посмотрим на пример кода из работы студента:
```java
public class ContactApplication extends Application {
    private AppComponent appComponent;

    @Override
    public void onCreate() {
        super.onCreate();
        initDependencies();
    }

    private void initDependencies() {
        appComponent = DaggerAppComponent.builder()
                .repositoryModule(new RepositoryModule(getApplicationContext()))
                .build();
        appComponent.inject(this);
    }

    public AppComponent getAppComponent() {
        if(appComponent == null) {
            initDependencies();
        }
        return appComponent;
    }
}
```
Как видим, в классе `ContactApplication` - нет ни одной аннотации `@Inject`. В данном случае инъекция в этот класс избыточна.

#### Инъекция в поля или методы класса, который доступен для конструирования
Желательно, придерживаться стратегии инъекции в конструктор класса, если нам доступна возможность управления созданием этого класса.

#### Зависимость, которая требуется в нескольких модулях - каждый раз создается заново
```java
@Module
public class DetailsViewModelModule {
    @Provides
    public IssueRepository getRepository() {
        return new ContactsResolver();
    }
}

@Module
public class ContactsViewModelModule {
    @Provides
    public IssueRepository getRepository() {
        return new ContactsResolver();
    }
}
```
Решение - вынести зависимость в общий для этих модулей `@Scope`, например `@Singleton`
