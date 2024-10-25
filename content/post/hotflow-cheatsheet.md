+++
author = "huyhunhngc"
title = "Hot Flow Cheat Sheet"
date = "2023-10-24"
description = "Hot flow cheat sheet: SharedFlow and StateFlow"
tags = [
    "Kotlin", "Kotlin Flow", "Cheat sheet"
]
toc = true
+++


## 1. SharedFlow

### 1.1. Key Principles

- A hot stream with multiple receivers receiving the same value.
- Useful for broadcasting values to many consumers or sharing states/events across app components.
- Never completes unless the entire scope is closed.
- `MutableSharedFlow` allows updating state with `emit` (suspend) or `tryEmit` (non-suspend).
- Supports replay configuration and buffer overflow handling.
- All methods are thread-safe and can be called safely from concurrent coroutines.

### 1.2. Configuration Parameters

```kotlin
fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
)
```

### 1.3. shareIn

- Transforms a Flow into a **SharedFlow**.
- Requires a **coroutine scope** as the first parameter to start collecting elements of the flow.
- The second parameter, **started**, defines when the SharedFlow starts listening for emitted values (uses `SharingStarted`).
- The third parameter, **replay** (default: 0), defines how many past values are replayed to new subscribers.

### 1.4. SharingStarted Options

- **Eagerly**: Starts listening immediately and continues until the scope is canceled.
- **Lazily**: Begins listening when the first subscriber appears and never stops until the scope is canceled.
- **WhileSubscribed()**: Begins listening when the first subscriber appears and stops after a delay when the last subscriber disappears. The delay can be configured with `stopTimeoutMillis`.

**Note on WhileSubscribed:**
> If your screen pauses (e.g., when opening a new Intent like the camera app), your `SharedFlow` might stop emitting when there are no subscribers left. When returning to the previous screen, subscribers will reappear, potentially causing unnecessary task restarts.

**Note on Eagerly and Lazily:**
> If using `ViewModelScope` or `LifecycleScope`, SharedFlow will stop sending elements when the screen is destroyed.

### 1.5. Converting Flow to SharedFlow

```kotlin
// From a ViewModel or a class with lifeCycleScope
myFlow.shareIn(
    scope = viewModelScope,
    started = SharingStarted.Lazily
)

// From a class without lifeCycleScope (e.g., repository or use case)
suspend fun myFunction() = coroutineScope {
    myFlow.shareIn(
        scope = this,
        started = SharingStarted.Lazily
    )
}
```

### 1.6. Use Case: Observing Database Changes from Multiple Places

If you're using Room for your database, you likely know it supports Flow. You can observe database changes and receive updates immediately. However, reading from disk can be heavy, so if you need to observe data across multiple screens, you can use SharedFlow to avoid fetching data for each screen individually.

```kotlin
@Dao
interface UserSettingsDao {
    @Query("SELECT * FROM user_settings")
    fun getAll(): Flow<List<UserSettings>>
}

class UserSettingsRepository @Inject constructor(
    private val dao: UserSettingsDao
) {
    // Only read from the database once, and all receivers will get the data
    suspend fun getAll(): SharedFlow<List<UserSettings>> = coroutineScope {
        dao.getAll().shareIn(
            scope = this,
            started = SharingStarted.Lazily,
            replay = 1
        )
    }
}
```

## 2. StateFlow

### 2.1. Key Principles

- Works like `SharedFlow` but with `replay` set to 1.
- **Always stores a single value**.
- The stored value can be accessed via the `value` property.
- Requires an initial value in the constructor.
- The modern replacement for LiveData.
- Will not emit a new value if the emitted value is the same as the previous one.

### 2.2. Setting and Reading a Value

```kotlin
val state = MutableStateFlow("A") // Initial value is "A"
state.value = "B" // Setting value to "B"
state.value = "B" // Won't emit a new element because the value hasn't changed
val myValue = state.value // Reading value from state, here it's "B"
```

### 2.3. stateIn

- Converts a flow into a `StateFlow`.
- Requires a **scope**.
- There are two types: suspend and non-suspend.

#### Suspend `stateIn`

Suspends until the first flow element is emitted and a new value is calculated.

```kotlin
suspend fun myFunction() = coroutineScope {
    myFlow.stateIn(this)
}
```

#### Non-suspend `stateIn`

Requires an **initial value**.

```kotlin
myFlow.stateIn(
    scope = viewModelScope,
    started = SharingStarted.Lazily,
    initialValue = "A"
)
```

### 2.4. Use Case: Emitting Data from ViewModel to View

Here’s how to convert a flow into a `StateFlow` to emit state from a ViewModel to a view that observes it:

```kotlin
class MyViewModel @Inject constructor(
    private val fetchDataUseCase: FetchDataUseCase
) : ViewModel() {
    val myState: StateFlow<MyState> =
        fetchDataUseCase.dataState
            .map {
                when (it) {
                    is FetchDataUseCase.FetchDataState.Loading -> MyState.Loading
                    is FetchDataUseCase.FetchDataState.Success -> MyState.Success(it.data)
                    is FetchDataUseCase.FetchDataState.Error -> MyState.Error(it.message)
                }
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = MyState.Loading
            )
    
    sealed interface MyState {
        data object Loading : MyState
        data class Success(val data: List<String>) : MyState
        data class Error(val message: String) : MyState
    }
}

@Composable
fun MyScreen(viewModel: MyViewModel = MyViewModel()) {
    val state = viewModel.myState.collectAsStateWithLifecycle()
    when (state) {
        is MyState.Loading -> // show loading view
        is MyState.Success -> // show success view
        is MyState.Error -> // show error view
    }
}
```
