# Переделать экран списка контактов на RecyclerView
Разберем частые ошибки при выполнении данного задания студентами

#### Не обнуляется ссылка на adapter в `onDestroyView` фрагмента
Необходимо обнулять жесткую ссылку на `adapter` при уничтожении View фрагмента. Это поможет избежать утечки памяти, выделенной под `adapter` и все `ViewHolder`'ы в нем.

#### Небезопасное обращение к `adapter`
Прежде чем обращаться к адаптеру - следует проверить что еще не произошло `onDestroyView()` и адаптер не `null`

#### Дублированиее ссылки на список данных в наследниках `ListAdapter`
```java
public void setContacts(ArrayList<Contact> contacts) {
    this.contacts = contacts;
    submitList(this.contacts);
}
```
Так делать, естественно, не стоит. Это избыточно и ведет к рассинхрону данных, т.к. `ListAdapter` сам управляет списком данных в своей скрытой реализации

#### Проверка не всех полей участвующих в отображении списка на изменение
```java
public static final DiffUtil.ItemCallback<Contact> DIFF_CALLBACK = new DiffUtil.ItemCallback<Contact>() {
    @Override
    public boolean areItemsTheSame(@NonNull Contact oldItem, @NonNull Contact newItem) {
        return oldItem.getId() == newItem.getId();
    }

    @Override
    public boolean areContentsTheSame(@NonNull Contact oldItem, @NonNull Contact newItem) {
        return oldItem.getNumber().equals(newItem.getNumber());
    }
};
```
В данном случае, в методе `areContentsTheSame` проверяется только номер контакта, когда в списке еще отображается имя контакта. Т.е. при изменении имени контакта - `DiffUtils` не сгенерирует событие об изменении элемента списка и данные, которые будет наблюдать пользователь - будут устаревшими

**Обратный кейс** - проверка всех полей, которых только возможно. Естественно, это уже излишне для стандартных ситуаций. Бывают исключения, когда требуется оповестить пользователя об изменении данных в каком-то элементе списка, что бы он прошел в детали этого элемента - в этом случае, такой кейс оправдан.

#### В `ItemDecorator` не указывают единицы измерения в которых ожидаются входные данные
Пример из работы студента:
```java
public class ContactDecorator extends RecyclerView.ItemDecoration {
    private final int offset;
    
    public ContactDecorator(int offset) { ... }
}
```
Для пользователя данного `ItemDecorator` будет не понятно, в каких единицах измерения передавать `offset`. В `dp` или пикселях?

#### Не приводят размеры в `dp` преданные в `ItemDecorator` в пиксели
Пример:
```java
recyclerView.addItemDecoration(new ContactDecorator(15));
```
Т.к. код декоратора(в данном случае) рассчитывает на офсет в пикселях - нужно приводить значение из `dp` в пиксели, иначе на экранах с разной плотностью точек на дюйм - будет разный результат.  
Ну и еще - здесь используется нечетный размер, что может приводить к неточным визуальным результатам на некоторых устройствах

#### Прямое обращение к адаптеру в `ItemDecorator`
Пример из кода студента:
```java
int position = parent.getChildAdapterPosition(view);
if (position == parent.getAdapter().getItemCount() - 1) {
    /*CODE*/
}
```
Т.к. состояние адаптера в данный момент может отличаться от того, что требуется отрисовать на экране - прямое обращение к адаптеру в `ItemDecorator` может дать неверный результат. Необходимо использовать `RecyclerView.State` для получения необходимого состояния для текущей отрисовки.
Исправленный вариант:
```java
int position = parent.getChildAdapterPosition(view);
if (position == state.getItemCount() - 1) {
    /*CODE*/
}
```

#### Использование внутреннего класса для `ViewHolder`
Пример:
```java
class ContactsViewHolder extends RecyclerView.ViewHolder implements View.OnClickListener {
    ...
    ContactsViewHolder(View itemView) {
        ...
        itemView.setOnClickListener(this);
    }
    
    ...
    
    @Override
    public void onClick(View view) {
        int position = getAdapterPosition();
        if (itemClickListener != null && position != RecyclerView.NO_POSITION) {
            Contact contact = getItem(position);
            itemClickListener.onItemClicked(contact.getId());
        }
    }
}
```
В данном случае, внутренний класс нужен только для `Contact contact = getItem(position);`. Чем плохо использование внутреннего класса? Ответ - жесткая привязка к конкретному инстансу адаптера, т.е. нет возможности как-то повлиять на то каким образом будет организована связь с адаптером. Если в будущем потребуется поддержать функционал `RecyclerView.swapAdapter` это станет проблемой.

Выход - использовать вложенный класс.

#### Получение данных из `ListAdapter` не через `getItem(position)`
Т.к. `ListAdapter` сам управляет списком данных в нутри своей реализации и делает diff на бэкграунд потоке - получать актуальные данны необходимо только через `getItem(position)`

#### При каждом изменени данных - создают новый адаптер
Ну это классика. Так делать не стоит, т.к. это приводит к плохому пользовательскому опыту и проклятиям в адрес разработчиков приложения.

#### Фильтрация в адаптере
В большинстве случаев, лучше отфильтровать контакты на уровне работы с данными, т.е. при запросе в поставщик контактов. Т.к. БД в поставщике контактов сделает это значительно быстрее чем фильтрация в адаптере на большом количестве записей. И пагинацию данных будет сделать значительно проще.

#### Не проверяют `getAdapterPosition` на `RecyclerView.NO_POSITION`
`getAdapterPosition` необходимо проверять на `RecyclerView.NO_POSITION` прежде чем использовать.
