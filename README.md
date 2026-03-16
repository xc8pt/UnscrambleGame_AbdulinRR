# Лабораторная работа №13-14. ViewModel и архитектура Android-приложений
Цель работы: Познакомиться с архитектурными компонентами Android Jetpack,
научиться использовать ViewModel для управления состоянием UI и сохранения данных
при изменении конфигурации устройства (поворот экрана, смена темы и т.д.).
Разработать игру с подсчётом очков, которая не теряет данные при повороте экрана.

## Технологии
### UI-фреймворк
Jetpack Compose - declarative UI toolkit для Android. Используются `@Composable` функции (`GameScreen`, `GameStatus`), модификаторы (`Modifier`), состояния (`collectAsState`, `mutableStateOf`) и layouts (`Column`, `Row`, `Box`).
### Material Design 3
MaterialTheme - стили, типографика (`titleLarge`, `displayMedium`), цвета (`colorScheme`), компоненты (`Card`, `OutlinedTextField`, `Button`, `AlertDialog`, `TextButton`).
### State Management
 - ViewModel (`gameViewModel = viewModel()`) для хранения состояния игры.
 - StateFlow (`uiState.collectAsState()`) для реактивного обновления UI.
 - `remember` и `mutableStateOf` для локальных состояний.
### Дополнительные элементы
 - `KeyboardOptions` и `KeyboardActions` для обработки клавиатуры.
 - `rememberScrollState()` для прокрутки.
 - `Arrangement` и `Alignment` для компоновки элементов.

## Функциональность приложения
 - Генерирует рандомное слово + перемешивает буквы
 - Проверяет введёное слово на правильность
 - Засчитывает правильное слово и добавляет 1 балл
 - Если же не правильно, пишет ошибка

## Контрольные вопросы:

### Что такое ViewModel и зачем он нужен?
 - ViewModel хранит данные UI (состояние игры), живёт дольше Activity и не теряет их при повороте экрана.
 - Проблема без него: при повороте Activity пересоздаётся, все локальные данные (score, слово) сбрасываются.
 - С ViewModel: uiState сохраняется, UI обновляется через collectAsState() без мигания.

### В чём разница между MutableStateFlow и StateFlow?
 - MutableStateFlow vs StateFlow:
MutableStateFlow - изменяемый (может emit() новые значения), StateFlow - только для чтения (asStateFlow()). StateFlow только наблюдают в UI (collectAsState()).
 - Два поля _uiState и uiState:
_uiState = MutableStateFlow() - приватное для изменений в ViewModel.
val uiState = _uiState.asStateFlow() - публичное/безопасное для UI. UI не может случайно изменить состояние.
 - Backing property pattern:
Это шаблон с приватной изменяемой "подложкой" (_uiState) и публичным readonly свойством (uiState). Защищает состояние от внешних изменений, стандарт для ViewModel.

### Почему в ViewModel нельзя хранить Context или View?
 - Нельзя хранить Context/View в ViewModel, ViewModel живёт дольше Activity (до её полного уничтожения), а Context/View - нет.
 - Если сохранить Activity в ViewModel:
Activity не GC'ится -> утечка памяти (занимает RAM).
При повороте/сворачивании: ViewModel попытается обновить уничтоженный Activity -> crash (NullPointerException или IllegalStateException).

### Чем StateFlow отличается от remember и rememberSaveable?
| Подход                | Хранит           | Выживает при повороте? | Выживает при убийстве процесса? |
| --------------------- | ---------------- |---------------------| ---------------------------- |
| remember              | В памяти Compose |  Нет                |  Нет                         |
| rememberSaveable      | В Bundle         |  Да                 |  Да              |
| StateFlow (ViewModel) | В ViewModel      |  Да                 |  Да              |

### Что такое data class в Kotlin и зачем он используется для UI State?
 - Data class - специальный класс в Kotlin для хранения данных (UI State), автоматически генерирует полезные методы.
 - Зачем для UI State: неизменяемые объекты состояния (например, GameUiState), легко сравниваются, копируются, отлаживаются (toString()).
 - Автогенерируемые методы:
   - `equals()` - сравнение по содержимому
   - `hashCode()` - хэш по полям
   - `toString()` - читаемый вывод
   - `copy()` - копия с изменениями
   - `componentN()` - деструктуризация (val (x, y) = obj)
 - copy(): создаёт новый объект с изменёнными полями

### Как работает collectAsState() в Compose?
 - collectAsState() преобразует StateFlow в State для Compose, автоматически собирая новые значения.
 - Как работает:
   - Подписывается на StateFlow при первой рекомпозиции
   - StateFlow.value -> начальное значение State
   - При emit() нового значения → State.value обновляется -> рекомпозиция только нужных @Composable
 -
```kotlin
val gameUiState by gameViewModel.uiState.collectAsState()
```
Счёт/слово меняется в ViewModel -> UI мгновенно обновляется без ручного setState().
 - Почему автоматически: Compose отслеживает все State.value в композиции. Любое изменение = рекомпозиция.

### Зачем разделять код на пакеты data и ui?
 - Разделение на data и ui пакеты отделяет логику данных (модели, репозитории, API) от UI (Composable, ViewModel).
 - Преимущества:
   - Чистота кода: UI не знает о БД/сети, data - о кнопках.
   - Переиспользование: data-модуль можно тестировать/использовать отдельно.
   - Масштабируемость: легко менять источники данных без правок UI.
 - Принципы архитектуры:
   - Clean Architecture (Layers: Data/Domain/UI)
   - MVVM (UI -> ViewModel -> Data)
   - Separation of Concerns (разделение ответственности)
   - Dependency Inversion (UI зависит от абстракций data).

### Что происходит с ViewModel, когда приложение полностью закрывается (process death)?
 - При process death (убийство процесса системой): ViewModel уничтожается полностью - все данные в ней (StateFlow, переменные) теряются.
 - Сохраняются ли данные: Нет, ViewModel не предназначен для этого. Только то, что вручную сохраняется в SavedStateHandle (сериализуемые данные через onSaveInstanceState()).
 - Альтернативы для долговременного хранения:
   - SavedStateHandle - в ViewModel для простых данных (int, String).
   - Room DB - локальная БД (счёт игры).
   - DataStore - preferences для ключ-значение.
   - SharedPreferences (устарело).
   - Файлы - для больших данных.



