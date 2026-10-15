# Practice 6 — Architectural Design Patterns

Project: WeatherPatterns, package uz.ttpu.weatherpatterns, Kotlin, Compose, Material 3, Min SDK 24.

## Question 1 — What a proper architecture gives you

Simplicity is lost because one class does five things. If I only want to fix the text of a validation error, I still have to read through the network code and the SharedPreferences code to find the right line. There is no place where one job ends and another starts.

Testability is lost because the logic lives inside an Activity. To check if the JSON is parsed right, I have to start the Activity, so I need an emulator. Parsing JSON has nothing to do with Android, but I still cannot test it with a normal JUnit test.

Maintenance costs a lot because everything is glued together. If the server changes its answer, I open the same file where the UI is drawn, and I can break the form by mistake. And when two of us work on this file, we get conflicts every time.

I would take the network call and the JSON parsing out first and put them in a repository. That part is the biggest one and it does not need Android. After that I can write real tests for it, and the Activity already becomes much shorter.

## Question 2 — Starting from nothing

1. First look at the app, before picking anything. What does each screen do, which screen is complicated, what changes often. A pattern is only an idea, and how you write it depends on what the app needs, so you cannot choose before you know that.
2. Take the data work out of the Activities: network, database, SharedPreferences all go into a repository. This is useful for every pattern, so this work is never wasted.
3. Learn one pattern well, for example MVVM, and write one small screen with it. The more you understand it, the better you can use it.
4. Use it on one module first, not on the whole app. Then look at what you got and fix what feels wrong.
5. After that go to the other modules one by one. Each module can have its own structure, so an easy screen can stay easy.

So the first step is understanding the app, and only then choosing a pattern.

"Use MVVM everywhere" is not a good answer. There is no one pattern that fits every project. A screen with two texts would get a ViewModel, a UiState and a factory for nothing, and the code becomes harder to read. And if he does not understand MVVM yet, he will just write something that looks like MVVM, so this advice does not help him at all.

## Question 3 — Project and the shared data layer

**data/repository/WeatherRepository.kt**

```kotlin
class WeatherRepository(private val api: WeatherApi) {

    suspend fun getWeather(city: String): Result<Weather> {
        return try {
            val weather = api.fetchWeather(city)
            Result.success(weather)
        } catch (e: IllegalArgumentException) {
            // only this exception is caught, so cancellation still goes up
            Result.failure(e)
        }
    }
}
```

**WeatherRepositoryTest.kt** (src/test)

```kotlin
    @Test
    fun known_city_returns_success() = runBlocking<Unit> {
        val result = repository.getWeather("Tashkent")

        assertTrue(result.isSuccess)
        assertEquals(24, result.getOrNull()?.temperatureC)
    }

    @Test
    fun unknown_city_returns_failure() = runBlocking<Unit> {
        val result = repository.getWeather("Atlantis")

        assertTrue(result.isFailure)
    }
```

Both tests pass from the green arrow in the gutter, without an emulator.

## Question 4 — Shared screen, module and launcher

I created the packages from the map and added the four Activities with New → Activity → Empty Activity. Android Studio put all four into the manifest next to MainActivity:

```xml
        <activity
            android:name=".mvc.view.MvcActivity"
            android:exported="false"
            android:label="MVC" />
        <activity
            android:name=".mvp.view.MvpActivity"
            android:exported="false"
            android:label="MVP" />
        <activity
            android:name=".mvvm.view.MvvmActivity"
            android:exported="false"
            android:label="MVVM" />
        <activity
            android:name=".mvi.view.MviActivity"
            android:exported="false"
            android:label="MVI" />
```

WeatherModule went to di and WeatherScreen to ui, both exactly as given. When I ran the app the launcher showed the title and the four buttons, every button opened its own screen, and Back always came back to the launcher.

## Question 5 — MVC roles and the missing Controller

Model: the Weather data class (1), WeatherRepository (2) and FakeWeatherApi (5). View: the composable WeatherScreen (3). Controller: the new WeatherController (6).

Piece 4 does two roles together. It takes the click and calls the repository, that is the Controller part, and then it puts the text into a TextView, that is the View part. Because of this the Activity gets bigger with every new button, since both the logic and the UI code go into the same class. And I cannot test that logic, because it only exists inside an Activity, so I would need an emulator for it.

The Model has no knowledge of the View.

## Question 6 — MVC with a separate Controller

**mvc/controller/WeatherController.kt**

```kotlin
    fun onCitySelected(city: String) {
        scope.launch {
            model.startLoading()
            view.render()

            model.load(city)   // suspends until the fake network answers
            view.render()
        }
    }
```

**mvc/view/MvcActivity.kt**

```kotlin
    // No parameters here, because in MVC the View can read the Model by itself.
    // The Controller only says that the Model changed. If it sent the Weather in,
    // it would also decide what the View shows, and that is MVP, not MVC.
    override fun render() {
        status = when {
            model.isLoading -> "Loading…"
            model.error != null -> "Error: ${model.error}"
            model.weather != null -> {
                val w = model.weather!!
                "${w.city}: ${w.temperatureC}°C, ${w.condition}"
            }
            else -> "Pick a city"
        }
    }
```

When I ran it the status line showed Loading… first and then Tashkent: 24°C, Sunny. Samarkand gave Samarkand: 21°C, Cloudy, and Atlantis gave Error: No forecast for Atlantis.

## Question 7 — MVP rules: find the violations

(A) The Activity calls repository.getWeather() by itself. So the View goes straight to the Model, and that is the first rule broken.

(B) The Activity looks at the result and decides what text to show. Here the View is thinking, but it should be dumb and only report clicks. That is the third rule.

(C) WeatherPresenter(this) gives the Presenter a real Activity. No rule is broken, but in a unit test I cannot make a WeatherActivity, because an Activity only exists on a device. So I can test this Presenter only on an emulator, and that is the thing MVP wanted to fix.

(D) The Presenter writes into activity.statusLabel, which is a real TextView. Again no rule is broken, but now the Presenter needs Android classes. In a local test there is no TextView, so the test would just crash.

(C) and (D) both keep the second rule: one Presenter belongs to one View.

Rewritten constructor parameter:

```kotlin
class WeatherPresenter(private val view: WeatherContract.View) {
```

Now the Presenter only sees an interface, so in a test I can give it a fake View.

## Question 8 — MVP with a testable Presenter

**mvp/presenter/WeatherPresenter.kt**

```kotlin
    fun onCitySelected(city: String) {
        view.showLoading()
        scope.launch {
            repository.getWeather(city)
                .onSuccess { view.showWeather(it) }
                .onFailure { view.showError(it.message ?: "Unknown error") }
        }
    }
```

**WeatherPresenterTest.kt** (src/test), the two missing tests

```kotlin
    @Test
    fun unknown_city_shows_loading_then_error() {
        presenter.onCitySelected("Atlantis")
        assertEquals(
            listOf("loading", "error:No forecast for Atlantis"),
            view.events,
        )
    }

    @Test
    fun two_requests_in_a_row_show_two_loading_events() {
        presenter.onCitySelected("Tashkent")
        presenter.onCitySelected("Samarkand")

        assertEquals(
            listOf("loading", "weather:Tashkent", "loading", "weather:Samarkand"),
            view.events,
        )
        assertEquals(2, view.events.count { it == "loading" })
    }
```

All three Presenter tests pass on my computer, with no emulator.

## Question 9 — The Presenter that outlives its View

(1) and (2) I opened the MVP screen, tapped Bukhara and rotated the emulator while it was still loading. Logcat with tag:MVP showed this:

```
D MVP: onCreate   activity=60451332
D MVP: onDestroy  activity=60451332
D MVP: onCreate   activity=177820158
D MVP: showWeather called on activity=60451332
```

The old Activity 60451332 was destroyed, the new one 177820158 was created, and about two seconds later showWeather arrived at the old, already destroyed instance 60451332. The new Activity never heard about the request.

(3) After adding destroy() and calling it from onDestroy I repeated the same steps:

```
D MVP: onCreate   activity=131150711
D MVP: onDestroy  activity=131150711
D MVP: onCreate   activity=206101726
```

The difference is that there is no showWeather line at all now. Cancelling the scope kills the coroutine together with the Activity, so no callback can reach a dead View. The leak is gone, but the result of the request is gone with it.

**WeatherPresenter**

```kotlin
    fun destroy() {
        // the coroutine is cancelled, so no callback reaches a destroyed View
        scope.cancel()
    }
```

**MvpActivity**

```kotlin
    override fun onDestroy() {
        presenter.destroy()
        Log.d("MVP", "onDestroy  activity=${hashCode()}")
        super.onDestroy()
    }
```

(4) After the rotation the status line shows "Pick a city" again. The new Activity starts with a new Presenter and a new status field, and nothing tells it that a request was running. To survive the rotation the state would have to live outside the Activity, in something that is not destroyed with it, like an Android ViewModel.

## Question 10 — MVVM: where does each class live?

data/api: WeatherApi, FakeWeatherApi. data/model: Weather. data/repository: WeatherRepository. di: WeatherModule. presentation/viewmodel: WeatherViewModel, WeatherUiState. presentation/view: WeatherScreen, MvvmActivity.

(a) The ViewModel lives longer than the Activity, it stays alive after a rotation. So if it held the Activity, that link would point to a dead object, which is a leak, and the ViewModel would try to draw on a screen that is not there. Instead of that it only gives out state, a StateFlow of WeatherUiState, and the View reads it and draws itself.

(b) Since the ViewModel does not touch the View at all, I can make it in a normal JUnit test with FakeWeatherApi, call onCitySelected("Atlantis") and check that uiState is WeatherUiState.Error with the message "No forecast for Atlantis". No Activity, no emulator.

## Question 11 — MVVM with a ViewModel and StateFlow

**mvvm/viewmodel/WeatherViewModel.kt**

```kotlin
    fun onCitySelected(city: String) {
        _uiState.value = WeatherUiState.Loading
        viewModelScope.launch {
            repository.getWeather(city)
                .onSuccess { _uiState.value = WeatherUiState.Success(it) }
                .onFailure {
                    _uiState.value = WeatherUiState.Error(it.message ?: "Unknown error")
                }
        }
    }
```

**mvvm/view/MvvmActivity.kt**

```kotlin
private fun WeatherUiState.toStatusText(): String = when (this) {
    WeatherUiState.Idle -> "Pick a city"
    WeatherUiState.Loading -> "Loading…"
    is WeatherUiState.Success ->
        "${weather.city}: ${weather.temperatureC}°C, ${weather.condition}"
    is WeatherUiState.Error -> "Error: $message"
}
```

I added lifecycle-viewmodel-compose and lifecycle-runtime-compose, and WeatherModule got the factory.

The experiment: I tapped Bukhara, the status showed Loading…, and I rotated the screen about a second later. After the rotation the screen still showed Loading… and then changed to Bukhara: 27°C, Windy. The request was not lost. The ViewModel is not destroyed with the Activity, so the coroutine kept running, and the new Activity simply collected the same StateFlow and got the result.

## Question 12 — MVI principles: spot the violations

(1) The state is four public vars, and each one is just overwritten. There is never one state value that gets replaced by a new one, so this is against immutability. Somebody can also say it is separation of concerns, because no one owns the state, but I think immutability fits better, because the problem is the changing in place.

(2) The function is called after a widget and it changes the state itself. An Intent should say what the user wants, not which spinner moved, so here the Intent and the Model work get mixed. This is separation of concerns.

(3) WeatherState uses var, so the state can still change after it is made. This is immutability again, and here it is very clear.

(4) The Activity writes into the ViewModel field. Data goes from the View into the Model directly, instead of coming back as an Intent, so unidirectional data flow is broken. You could also call it separation of concerns, but the real problem is the direction, so I choose unidirectional flow.

The three declarations an MVI version needs:

```kotlin
data class WeatherState(
    val isLoading: Boolean = false,
    val weather: Weather? = null,
    val error: String? = null,
)

sealed interface WeatherIntent {
    data class SelectCity(val city: String) : WeatherIntent
    data object DismissError : WeatherIntent
}

fun onIntent(intent: WeatherIntent)
```

## Question 13 — MVI: one state, immutable updates, intents

**mvi/viewmodel/WeatherMviViewModel.kt**

```kotlin
            WeatherIntent.DismissError -> {
                // copy() gives a new state, the old one is not touched
                _state.update { it.copy(error = null) }
            }
```

```kotlin
    private fun load(city: String) {
        _state.update { it.copy(isLoading = true, error = null) }
        viewModelScope.launch {
            repository.getWeather(city)
                .onSuccess { weather ->
                    _state.update {
                        it.copy(isLoading = false, weather = weather, error = null)
                    }
                }
                .onFailure { e ->
                    _state.update {
                        it.copy(isLoading = false, error = e.message ?: "Unknown error")
                    }
                }
        }
    }
```

**di/WeatherModule.kt**

```kotlin
    val weatherMviViewModelFactory: ViewModelProvider.Factory = viewModelFactory {
        initializer { WeatherMviViewModel(repository) }
    }
```

**mvi/view/MviActivity.kt**

```kotlin
// isLoading comes first, so the old city does not stay on the screen
// while the new request is still running.
private fun WeatherState.toStatusText(): String = when {
    isLoading -> "Loading…"
    error != null -> "Error: $error"
    weather != null -> "${weather.city}: ${weather.temperatureC}°C, ${weather.condition}"
    else -> "Pick a city"
}
```

**WeatherMviViewModelTest.kt**, the two missing tests

```kotlin
    @Test
    fun selecting_an_unknown_city_ends_with_error() {
        val viewModel = newViewModel()

        viewModel.onIntent(WeatherIntent.SelectCity("Atlantis"))

        val state = viewModel.state.value
        assertFalse(state.isLoading)
        assertEquals("No forecast for Atlantis", state.error)
        assertNull(state.weather)
    }

    @Test
    fun dismissing_the_error_clears_it_and_keeps_the_old_weather() {
        val viewModel = newViewModel()

        viewModel.onIntent(WeatherIntent.SelectCity("Tashkent"))
        viewModel.onIntent(WeatherIntent.SelectCity("Atlantis"))
        viewModel.onIntent(WeatherIntent.DismissError)

        val state = viewModel.state.value
        assertNull(state.error)
        assertEquals("Tashkent", state.weather?.city)
    }
```

All three MVI tests pass. On the emulator the MVI screen behaves like the MVVM one: Bukhara shows Loading… and then Bukhara: 27°C, Windy, and the result is still on the screen after a rotation.

## Question 14 — What does each pattern depend on?

Find in Files over each pattern package, first "import android." and then "import androidx.":

| Pattern | Class with the logic | import android. | import androidx. | View holds | Survives rotation |
|---|---|---|---|---|---|
| MVC | WeatherController | no | no | the Controller | nothing |
| MVP | WeatherPresenter | no | no | the Presenter | nothing |
| MVVM | WeatherViewModel | no | yes (androidx.lifecycle.ViewModel, viewModelScope) | the ViewModel | the ViewModel with its uiState and the request in flight |
| MVI | WeatherMviViewModel | no | yes (androidx.lifecycle.ViewModel, viewModelScope) | the ViewModel | the ViewModel with its WeatherState and the request in flight |

In every package only the Activities import android. and androidx., and I think that is normal, because the Activity is the View.

MVC: the Controller takes a CoroutineScope, so I can give it a fake WeatherView and Dispatchers.Unconfined and test it with JUnit only.

MVP: same thing, the Presenter tests use FakeView and Dispatchers.Unconfined and need no extra library.

MVVM: the ViewModel works with viewModelScope, and that one runs on Dispatchers.Main, which does not exist in a local test, so here I need kotlinx-coroutines-test for Dispatchers.setMain.

MVI: same as MVVM, the test needs kotlinx-coroutines-test with UnconfinedTestDispatcher in @Before.

## Question 15 — Choose a pattern

(1) For an About screen with two texts I would not use any pattern, and not even a data layer. There is no state and nothing to load, so one composable is enough. The lecture says there is no pattern that fits every project, and the code depends on what the app needs. The cost is that if later the screen has to load something, like the app version from the server, I have to build it again from zero.

(2) The chat screen I would do with MVI. The message list, the typing indicator and the connection status all change together, and with one state they can never show a wrong mix. The cost is a lot of writing: a state class, an intent class, and copy() for every small change.

(3) The old app with 40 Activities I would not rewrite one month before the release. First I take the database code out into a repository, and then I use MVVM only on the screens we really touch. The lecture says that inside one big project each module can have its own structure, so I do not need to change everything. The cost is that for some time the app has two styles at the same time, and that is confusing for a new developer.

Yes, (1) and (2) can be in the same app with different structures. The lecture says that even inside one complex project each module can have its own structure, so a static screen stays a simple composable while the chat screen uses MVI.
