# **Retrofit**

## Overview
**Retrofit** is a **REST** client for Java & Kotlin. It makes it easy to request data via a **REST** based **web service**.

## Code Example
Open the `api-retrofit` directory provided to you in **Android Studio**. 

Lets take a look at what is happening...

### build.grade

Go to **Gradle Scripts > build.grade (Module: API.app)**. You should see the following in the **dependencies** block:

```xml
...
implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.2.0'
...
```

Without this dependencies, you can **not** use **Retrofit**.

### AndroidManifest
To make a request to an **API**, you need declare the internet permission in `AndroidManifest.xml`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="op.mobile.app.dev.api">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:usesCleartextTraffic="true"
        android:theme="@style/Theme.API">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

### GitHubJobs

```kotlin
data class GitHubJobs(val id: String, val title: String)
```

### IGitHubJobs
**Retrofit** converts your **API** into a **Kotlin** interface. Annotations on an interface method & its parameters indicate how a request will be handled. There are eight different annotations: `HTTP`, `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS` & `HEAD`. The **URL** of the resource is specified in the annotation. Also, you can specify query parameters in the **URL**.

```kotlin
...

interface IGitHubJobs {
    @GET("positions.json?description=kotlin&page=1")
    suspend fun getResponse(): List<GitHubJobs>
}
```

What is a `suspend` function?

Have a look at this [video](https://www.youtube.com/watch?v=BOHK_w09pVA&t=300s) from **Google I/O`19**.

**Resource:** https://square.github.io/retrofit

### ServiceInstance
- `Retrofit.Builder()` - build a new `Retrofit`.
- `baseUrl(String baseUrl)` - set the **API** base URL.
- `addConverterFactory(Converter.Factory factory)` - converter factory for serializing & deserializing objects.
- `build()` - create the **Retrofit** instance using the configured values.

**Note:** calling the `baseUrl()` method is required before calling `build()` method. 

`Retrofit` class converts your **API** interface, i.e., `IGitHubJobs` into a callable object, i.e., `retrofitService`. 

```kotlin
...

private const val BASE_URL = "https://jobs.github.com/"

object ServiceInstance {
    private val retrofit by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    val retrofitService: IGitHubJobs by lazy {
        retrofit.create(IGitHubJobs::class.java)
    }
}
```
An `object` does not have a constructor. The instance of the `object` will be created the first time you use it, i.e., **lazy initialisation**.

**Resources:** 
- https://square.github.io/retrofit/2.x/retrofit
- https://github.com/google/gson

### ServiceStatus
```kotlin
enum class ServiceStatus {
    LOADING,
    ERROR,
    COMPLETE
}
```

### ServiceBindingAdapter
**Binding adapters** are responsible for making calls to set values. For example, setting value to a `TextView` by calling the `setText()` method or setting a click event listener to a `Button` by calling the `setOnClickListener()` method. By using **data binding**, you can create an attribute for any setter, i.e., `apiServiceStatus`. You will see how to use `apiServiceStatus` later.

```kotlin
...

@BindingAdapter("apiServiceStatus")
fun bindAPIServiceStatus(tvStatus: TextView, status: ServiceStatus?) {
    when (status) {
        ServiceStatus.LOADING -> {
            tvStatus.visibility = View.VISIBLE
            tvStatus.text = tvStatus.context.getString(R.string.loading)
        }
        ServiceStatus.ERROR -> {
            tvStatus.visibility = View.VISIBLE
            tvStatus.text = tvStatus.context.getString(R.string.connection_error)
        }
        ServiceStatus.COMPLETE -> tvStatus.visibility = View.GONE
    }
}
```

**Resource:** https://developer.android.com/topic/libraries/data-binding/binding-adapters

### GitHubJobsViewModel

**Coroutines** provide an API that enables you to write asynchronous code. You can define a `CoroutineScope`, i.e., `viewModelScope.launch`, which manages when your **coroutines** should run. 

```kotlin
...

class GitHubJobsViewModel : ViewModel() {
    private val _status = MutableLiveData<ServiceStatus>()
    val status: LiveData<ServiceStatus> get() = _status

    private val _response = MutableLiveData<List<GitHubJobs>>()
    val response: LiveData<List<GitHubJobs>> get() = _response

    init {
        viewModelScope.launch {
            _status.value = ServiceStatus.LOADING
            try {
                _response.value = retrofitService.getResponse()
                _status.value = ServiceStatus.COMPLETE
            } catch (e: Exception) {
                _response.value = ArrayList()
                _status.value = ServiceStatus.ERROR
            }
        }
    }
}
```

**Resource:** https://developer.android.com/topic/libraries/architecture/coroutines

### GitHubJobsFragment Layout

Note how `apiServiceStatus` binding adapter is being used. 

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".ui.github.GitHubJobsFragment">

    <data>

        <variable
            name="githubJobsViewModel"
            type="op.mobile.app.dev.api.ui.github.GitHubJobsViewModel" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/tv_status"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="32dp"
            android:layout_marginTop="16dp"
            android:layout_marginEnd="32dp"
            android:layout_marginBottom="16dp"
            android:gravity="center"
            android:textSize="24sp"
            app:apiServiceStatus="@{githubJobsViewModel.status}"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

        <TextView
            android:id="@+id/tv_title"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="32dp"
            android:layout_marginTop="16dp"
            android:layout_marginEnd="32dp"
            android:layout_marginBottom="16dp"
            android:gravity="center"
            android:text="@{githubJobsViewModel.response.get(0).title}"
            android:textSize="24sp"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

    </androidx.constraintlayout.widget.ConstraintLayout>

</layout>
```

### GitHubJobsFragment

This is similar to previous code examples.

```kotlin
...

class GitHubJobsFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        val binding: FragmentGithubJobsBinding = DataBindingUtil.inflate(
            inflater, R.layout.fragment_github_jobs, container, false
        )

        val viewModel = ViewModelProvider(this).get(GitHubJobsViewModel::class.java)

        binding.lifecycleOwner = viewLifecycleOwner

        binding.githubJobsViewModel = viewModel

        return binding.root
    }
}
```
