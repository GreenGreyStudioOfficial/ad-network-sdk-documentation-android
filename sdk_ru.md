[Manual](#manual) | [API](#api)

# Принципы работы SDK 
## Оглавление
- [Инициализация библиотеки](#initialization)
- [Загрузка рекламных объявлений](#manual_load)
- [Показ рекламного объявления](#manual_show)
- [Особенности работы системы кэширования](#cache)
- [Как подключить библиотеку](#connect_lib)
- [Как работать с библиотекой - пример](#lib_work)


Данный SDK используется для работы с рекламой трех видов:

- видеорекламой
- рекламой, отображаемой внутри **web-view**.
- баннерной рекламой

Решение о показе какого-либо типа рекламы принимается в зависимости от решений сервера.

В основе работы **SDK** лежит статический класс **AdvSDK**, который:

- инициирует библиотеку;
- загружает рекламные объявления;
- показывает рекламные объявления;
- показывает баннер;

Для того, чтобы совершать эту работу, в классе имеются методы, которые, как правило, выполняются в фоновом режиме. Чтобы реагировать на их выполнение, существуют интерфейсы слушателей. Пользователю SDK предстоит самостоятельно реализовать эти интерфейсы, в зависимости от своих нужд.
Колбеки слушателей возвращают результат на основной поток
Пример реализации интерфейса слушателя смотрите [здесь](#lib_work).


## Инициализация библиотеки <a name="initialization"></a>
На данном этапе **SDK** подготавливается к загрузке и показу рекламных объявлений.

Пример:

```Kotlin
class InitActivity : AppCompatActivity() {  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
  
        AdvSDK.initialize(this, MY_GAME_ID, true, object : IAdInitializationListener {  
            override fun onInitializationComplete() {  
                Log.d("AdvSDK", "initialization complete")  
            }  
            override fun onInitializationError(error: InitializationErrorType, message: String) {  
                Log.d("AdvSDK", "initialization error ${error.name} $message")  
            }        
        })    
    }
}
```

## Загрузка рекламных объявлений <a name="manual_load"></a>

На данном этапе библиотека просит сервис подобрать и скачать подходящую рекламу для данного пользователя. После того, как успешный ответ получен, рекламное объявление загружается в кэш.

#### Типы рекламных объявлений

| Тип | Описание |
|---|---|
| `INTERSTITIAL` | Видео без вознаграждения|
| `REWARDED` | Видео с вознаграждением|
| `BANNER` | Рекламный баннер|

Пример:

```Kotlin
class LoadActivity : AppCompatActivity() {  
    private val advSDK = AdvSDK.INSTANCE
    lateinit var advId : String  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
        afterSdkInit {  
            advSDK.load(AdvertiseType.REWARDED, object : IAdLoadListener {  
                override fun onLoadComplete(id: String) {  
                    advId = id  
                    Log.d("AdvSDK", "load complete, id = $id")  
                }  
                override fun onLoadError(error: LoadErrorType, message: String, id: String) {  
                    Log.d("AdvSDK", "load error, id = $id, message $message")  
                }            
            })        
        }  
    }  
  
    private fun afterSdkInit(onError:((String) -> Unit)? = null, onInit :() -> Unit){  
        advSDK.initialize(this.application, MY_GAME_ID, true, object : IAdInitializationListener {  
            override fun onInitializationComplete() {  
                Log.d("AdvSDK", "initialization complete")  
                onInit()  
            }  
            override fun onInitializationError(error: InitializationErrorType, message: String) {  
                Log.d("AdvSDK", "initialization error ${error.name} $message")  
                onError?.invoke("initialization error $message")  
            }        
        })    
    }
}
```

## Нюансы работы системы кэширования <a name="cache"></a>

Кэширование креативов и их очистка происходит автоматически без возможности пользователя SDK влиять на данный процесс.

Например, очистка кэша происходит по завершению показа ролика, или по истечению срока его актуальности. Срок актуальности объявления регулирует рекламодатель. Отсюда возможна ситуация, когда загруженный в кэш ролик при попытке показа выдает ошибку.

## Показ рекламного объявления <a name="manual_show"></a>

На данном этапе **SDK** показывает пользователю рекламное объявление из кэша.

Пример:

```Kotlin
class ShowActivity : AppCompatActivity() {  
    private lateinit var advId: String  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
        initAndLoadAdv { id ->  
            advId = id  
        }  
  
        findViewById<View>(R.id.btnShow).setOnClickListener {  
            AdvSDK.show(advId, object : IAdShowListener {  
                override fun onShowChangeState(id: String, state: ShowCompletionState) {  
                    Log.d("AdvSDK", "show change state, id = $id ${state.name}")  
                }  
                override fun onShowError(id: String, error: ShowErrorType, message: String) {  
                    Log.d("AdvSDK", "show error, id = $id message = $message")  
                }            
            })        
        }  
    }  
  
    private fun initAndLoadAdv(onError: ((String) -> Unit)? = null, onLoad: (String) -> Unit) {  
        AdvSDK.initialize(this, MY_GAME_ID, true, object : IAdInitializationListener {  
            override fun onInitializationComplete() {  
                Log.d("AdvSDK", "initialization complete")  
                AdvSDK.load(AdvertiseType.REWARDED, object : IAdLoadListener {  
                    override fun onLoadComplete(id: String) {  
                        Log.d("AdvSDK", "load complete, id = $id")  
                        onLoad(id)  
                    }  
                    override fun onLoadError(error: LoadErrorType, message: String, id: String) {  
                        Log.d("AdvSDK", "load error, id = $id, message $message")  
                        onError?.invoke("load error, id = $id, message $message")  
                    }
              })
            }

            override fun onInitializationError(error: InitializationErrorType, message: String) {  
                Log.d("AdvSDK", "initialization error ${error.name} $message")  
                onError?.invoke("initialization error $message")  
            }        
        })    
    }
}
```

```Kotlin
class ShowBannerActivity : AppCompatActivity() {  
    private lateinit var advId: String  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
        initAndLoadAdv { id ->  
            advId = id  
        }  
  
        findViewById<View>(R.id.btnShow).setOnClickListener {  
            AdvSDK.showBanner(advId, object : IAdShowBannerListener {  
                override fun onBannerShow(id: String?) {
                    Log.d("AdvSDK", "onBannerShow, id = $id")
                }

                override fun onBannerShowError(error: ShowErrorType, errorMessage: String, id: String?) {
                    Log.d("AdvSDK", "onBannerShowError, id = $id errorMessage = ${error.name}")
                }         
            })        
        }

        findViewById<View>(R.id.btnHide).setOnClickListener {  
            AdvSDK.hideBanner(advId, object : IAdHideBannerListener {  
                override fun onBannerHide(id: String?) {
                    Log.d("AdvSDK","onBannerHide, id = $id")
                }

                override fun onBannerHideError(error: ShowErrorType, errorMessage: String, id: String?) {
                    Log.d("AdvSDK","onBannerHideError, id = $id errorMessage = ${error.name}")
                }          
            })        
        }  
    }  
  
    private fun initAndLoadAdv(onError: ((String) -> Unit)? = null, onLoad: (String) -> Unit) {  
        advSDK.initialize(this, MY_GAME_ID, true, object : IAdInitializationListener {  
            override fun onInitializationComplete() {  
                Log.d("AdvSDK", "initialization complete")  
                advSDK.load(AdvertiseType.BANNER, object : IAdLoadListener {  
                    override fun onLoadComplete(id: String) {  
                        Log.d("AdvSDK", "load complete, id = $id")  
                        onLoad(id)  
                    }  
                    override fun onLoadError(error: LoadErrorType, message: String, id: String) {  
                        Log.d("AdvSDK", "load error, id = $id, message $message")  
                        onError?.invoke("load error, id = $id, message $message")  
                    }
              })
            }

            override fun onInitializationError(error: InitializationErrorType, message: String) {  
                Log.d("AdvSDK", "initialization error ${error.name} $message")  
                onError?.invoke("initialization error $message")  
            }        
        })    
    }
}
```

# Как подключить библиотеку <a name="connect_lib"></a>

Библиотека распространяется как артефакт для **maven** `com.greengray:advsdk`

актуальная версия библиотеки `com.greengray:advsdk:1.5.1`

Для работы с библиотекой необходим **GAME_ID** - идентификатор приложения в системе показа рекламы.
Напишите на [a.bobkov@mobidriven.com](a.bobkov@mobidriven.com), чтобы получить идентификатор.

Для этого необходимо добавить maven репозиторий в build.gradle уровня проекта

http://nexus.ggsinternal.space/repository/maven-public-hosted/

![Screenshot_2.png](/images/Screenshot_2.png)

```Kotlin
repositories {  
    jcenter()  
    google()  
    maven { url 'https://nexus.ggsinternal.space/repository/maven-public-hosted/' }  
}
```

для **Gradle 7** и выше (новая структура проекта) необходимо добавить репозиторий в **settings.gradle**

![Screenshot_3.png](/images/Screenshot_3.png)

```Kotlin
dependencyResolutionManagement {  
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)  
    repositories {  
        maven {  
            url "https://nexus.ggsinternal.space/repository/maven-public-hosted/"  
        }  
        google()  
        mavenCentral()  
    }}
```

2. добавить библиотеку `com.greengray:advsdk:1.5.1` в блок зависимостей **build.gradle** уровня модуля

```implementation 'com.greengray:advsdk:1.5.1'```
    
![Screenshot_4.png](/images/Screenshot_4.png)

3. Синхронизируйте **gradle** проект

5. Для запуска примера необходимо прописать полученный идентификатор **GAME_ID** в методе инициализации **AdvSDK**, более подробно можно посмотреть  [в примере](#lib_work)

```Kotlin
AdvSDK.initialize(  
    context: Activity,  
    gameId: String,  
    isTestMode: Boolean,  
    listener: IAdInitializationListener  
)
```


# Как работать с библиотекой - пример <a name="lib_work"></a> 

Пример кода с объяснениями, который иллюстрирует максимально быстрый в разработке способ показать рекламу.

Создается **Activity**, которая реализует все 3 интерфейса слушателей: инициации, загрузки и показа.

Соответствующие функции класса **AdvSDK** вызываются при нажатии кнопок на экране.

Для загрузки используется функция **Load**, которая загружает рекламное объявление из сети и сохраняет в кеш.

Для отображения рекламы используется функция **show**, которая показывает рекламное объявление из из кэша, если загрузка была проведена ранее.

Для отображения используется функция **showBanner** , которая показывает рекламное объявление из из кэша, если загрузка была проведена ранее. Для скрытия баннера используется метод **hideBanner**.

Данный код можно протестировать в  **MainActivity**, поставляющейся в пакете вместе с **SDK**.

```Kotlin
class MainActivity : AppCompatActivity(), IAdInitializationListener, IAdLoadListener, IAdShowListener,  IAdShowBannerListener, IAdHideBannerListener {

    var advId :  String? = null  
    private lateinit var recyclerView: RecyclerView  
  
    private val logsAdapter by lazy {  
        LogsAdapter()  
    }  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
        recyclerView = findViewById<RecyclerView>(R.id.rvLogs).apply {  
            adapter = logsAdapter  
        }  
  
        findViewById<View>(R.id.btnInit).setOnClickListener {  
            AdvSDK.initialize(this.application, MY_GAME_ID,  true, this)  
        }  
  
        findViewById<View>(R.id.btnLoadRewarded).setOnClickListener {  
            AdvSDK.load(AdvertiseType.REWARDED,this)  
        }

        findViewById<View>(R.id.btnLoadInterstitial).setOnClickListener {  
            AdvSDK.load(AdvertiseType.INTERSTITIAL, object : IAdLoadListener {  
                override fun onLoadComplete(id: String) {  
                    advId = id  
                    addLog("INTERSTITIAL onLoadComplete, id = $id")  
                }  
                override fun onLoadError(error: LoadErrorType, errorMessage: String, id: String) {  
                    addLog("INTERSTITIAL onLoadError, id = $id, ${error.name}, errorMessage $errorMessage")  
                }
            })
        }

        findViewById<View>(R.id.btnShow).setOnClickListener {  
            advId?.let {  
                AdvSDK.show(it, this)  
            }  
        }

        findViewById<View>(R.id.btnLoadBanner).setOnClickListener {
            AdvSDK.load(AdvertiseType.BANNER, listener = object : IAdLoadListener {
                override fun onLoadComplete(id: String?) {
                    addLog("BANNER onLoadComplete, id = $id")
                }

                override fun onLoadError(error: LoadErrorType, errorMessage: String, id: String?) {
                    addLog("BANNER onLoadError, id = $id, ${error.name} , errorMessage $errorMessage")
                }
            })
        }

        findViewById<View>(R.id.btnShowBanner).setOnClickListener {
            advId?.let {
                AdvSDK.showBanner(it, this) 
            } 
        }
    }  
    
    
    private fun addLog(log: String) {  
        logsAdapter.addLog(log)  
        recyclerView.smoothScrollToPosition(logsAdapter.itemCount - 1)  
    }

    override fun onInitializationComplete() {  
        addLog("initialization complete")  
    }

    override fun onInitializationError(error: InitializationErrorType, message: String) {  
        addLog("initialization error = ${error.name}, $message")  
    }

    override fun onShowChangeState(id: String, state: ShowCompletionState) {  
        addLog("show change state, id = $id state = ${state.name}")  
    }

    override fun onShowError(id: String, error: ShowErrorType, message: String) {  
        addLog("show error, id = $id message = $message")  
    }
 
    override fun onLoadComplete(id: String) {  
        advId = id  
        addLog("load complete, id = $id")  
    }

    override fun onLoadError(error: LoadErrorType, message: String, id: String) {  
        addLog("load error, id = $id, message $message")  
    }

    override fun onBannerShow(id: String?) {
        addLog("onBannerShow, id = $id")
    }

    override fun onBannerShowError(error: ShowErrorType, errorMessage: String, id: String?) {
        addLog("onBannerShowError, id = $id errorMessage = ${error.name}")
    }

    override fun onBannerHide(id: String?) {
        addLog("onBannerHide, id = $id")
    }

    override fun onBannerHideError(error: ShowErrorType, errorMessage: String, id: String?) {
        addLog("onBannerHideError, id = $id errorMessage = ${error.name}")
    }
}
```
