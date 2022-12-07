[Manual](#manual) | [API](#api)

# SDK Workflow and Integration
## Contents
- [Library initialization](#initialization)
- [Advertisements upload](#manual_load)
- [Ad display](#manual_show)
- [Cache system features](#cache)
- [How to connect the library](#connect_lib)
- [How to work with library (example)](#lib_work)


This SDK is used for work with advertisements of two types:

- Video ads
- Ads shown inside **web-view**

Depending on the server decisions only one of the ad types will be shown.

There is a statistic class AdvSDK at the core of **SDK**, that:

- initialize the library;
- load an advertisement;
- show an advertisement

There are a few methods in this class that run in the background to manage the aforementioned processes. There are listener interfaces to respond to their execution. SDK users have to implement these interfaces on their own, depending on their needs.
Listener callbacks return the result to the main flow.
Listener interface [example](#lib_work).


## Library initialization <a name="initialization"></a>
**SDK** is prepared for loading and showing advertisements.

Example:

```Kotlin
class InitActivity : AppCompatActivity() {  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
  
        AdvSDK.INSTANCE.initialize(this.application, MY_GAME_ID, true, object : IAdInitializationListener {  
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

## Advertisements upload <a name="manual_load"></a>

SDK is prepared for loading and showing advertisements. After a valid response is successfully received, the advertisement is loaded into the cache.


#### Ad types

| Тип | Описание |
|---|---|
| `INTERSTITIAL` | Video without compensation within app|
| `REWARDED` | Video with compensation within app|

Example:

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

## Nuances of the caching system <a name="cache"></a>

Caching and refining of creatives is done automatically, without capability for user to manage this process.

E.g, cache refreshing occurs when video ends, or after its expiration period. Expiration period is set by the advertiser. Therefore, cached video may result in an error after an attempt to view them.

## Ad display <a name="manual_show"></a>

**SDK** shows the advertisement from the cache.

Example:

```Kotlin
class ShowActivity : AppCompatActivity() {  
    private val advSDK = AdvSDK.INSTANCE  
    private lateinit var advId: String  
  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        setContentView(R.layout.activity_main)  
  
        initAndLoadAdv { id ->  
            advId = id  
        }  
  
        findViewById<View>(R.id.btnShow).setOnClickListener {  
            advSDK.show(advId, object : IAdShowListener {  
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
        advSDK.initialize(this.application, MY_GAME_ID, true, object : IAdInitializationListener {  
            override fun onInitializationComplete() {  
                Log.d("AdvSDK", "initialization complete")  
                advSDK.load(AdvertiseType.REWARDED, object : IAdLoadListener {  
                    override fun onLoadComplete(id: String) {  
                        Log.d("AdvSDK", "load complete, id = $id")  
                        onLoad(id)  
                    }  
                    override fun onLoadError(error: LoadErrorType, message: String, id: String) {  
                        Log.d("AdvSDK", "load error, id = $id, message $message")  
                        onError?.invoke("load error, id = $id, message $message")  
                    }                })            }  
            override fun onInitializationError(error: InitializationErrorType, message: String) {  
                Log.d("AdvSDK", "initialization error ${error.name} $message")  
                onError?.invoke("initialization error $message")  
            }        
        })    
    }
}
```

# How to connect the library <a name="connect_lib"></a>

Library is distributed as an artifact for **maven** `com.greengray:advsdk`

current library version `com.greengray:advsdk:1.3.1`

You need GAME_ID - an application identifier in the ad serving system to work with the library.
Contact [partners@mobidriven.com](partners@mobidriven.com) to receive the identifier.

You have to add maven repository in build.gradle project level

http://nexus.ggsinternal.space/repository/maven-public-hosted/

![Screenshot_2.png](/images/Screenshot_2.png)

```Kotlin
repositories {  
    jcenter()  
    google()  
    maven { url 'https://nexus.ggsinternal.space/repository/maven-public-hosted/' }  
}
```

for **Gradle 7** and following (new project structure) - a repository is to be added to **settings.gradle**

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

2. add library `com.greengray:advsdk:1.3.1` to dependancy block **build.gradle** of module level

```implementation 'com.greengray:advsdk:1.3.1'```
    
![Screenshot_4.png](/images/Screenshot_4.png)

3. Synchronize **gradle** project

4. To run the example, you need to register recieved identifier **GAME_ID** in the initialization method **AdvSDK** [example](#lib_work)

```Kotlin
AdvSDK.INSTANCE.initialize(  
    context: Application,  
    gameId: String,  
    isTestMode: Boolean,  
    listener: IAdInitializationListener  
)
```


# How to work with library (example) <a name="lib_work"></a> 

An example of the code illustrates the most reliable and fastest way for a developer to get an advertisement shown

An **Activity** is created that implements all 3 listener interfaces: initiate, load, and show.

The corresponding functions of the **AdvSDK** class are called when the buttons on the screen are pressed.

For loading, the **Load** function is used, which loads the advertisement from the network or from the cache, if the download was carried out earlier.

This code can be tested in **MainActivity** supplied with the **SDK**.

```Kotlin
class MainActivity : AppCompatActivity(), IAdInitializationListener, IAdLoadListener, IAdShowListener {  
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
            AdvSDK.INSTANCE.initialize(this.application, MY_GAME_ID,  true, this)  
        }  
  
        findViewById<View>(R.id.btnLoadRewarded).setOnClickListener {  
            AdvSDK.INSTANCE.load(AdvertiseType.REWARDED,this)  
        }  
        findViewById<View>(R.id.btnLoadInterstitial).setOnClickListener {  
            AdvSDK.INSTANCE.load(AdvertiseType.INTERSTITIAL, object : IAdLoadListener {  
                override fun onLoadComplete(id: String) {  
                    advId = id  
                    addLog("INTERSTITIAL onLoadComplete, id = $id")  
                }  
                override fun onLoadError(error: LoadErrorType, errorMessage: String, id: String) {  
                    addLog("INTERSTITIAL onLoadError, id = $id, ${error.name}, errorMessage $errorMessage")  
                }            })        }  
        findViewById<View>(R.id.btnShow).setOnClickListener {  
            advId?.let {  
                AdvSDK.INSTANCE.show(it, this)  
            }  
        }    }  
  
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
    }}
```
