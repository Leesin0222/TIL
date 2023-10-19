# 행복한 CPS (ContinuationPassingStyle) 이야기

직역하자면 **Continuation Passing Style == 연속 전달 스타일**이라고 할 수 있다. suspend를 구현할 때 필수로 필요한 것이 suspend 내부에서 cps가 어떻게 suspend를 실행시키는가인데 이것을 어떻게 구현할 수 있을까.. 바로 cps이다. suspend 어떻게 작동되는지 cps를 통해 알아보자.

<br/>

일단 Coroutine은 기존의 콜백 기반 비동기 코드를 순차적(Sequential)인 코드로 바꿔주기 때문에 비동기 코드를 간결히 할 수 있다.

이 점 때문에 일명 콜백 지옥에서 벗어나게 해주면서 가독성을 올려주어 비동기적인 코드를 간결히, 가독성있게 바꿔준다.

<br/>

그럼 Coroutine이 이러한 일명, ****맛있는 코드**를 작성할 수 있는 이유가 무엇일까? 바로바로 **suspend** 키워드를 보면 알 수 있는데 suspend는 **내부적으로 콜백을 생성**하고 suspned 키워드를 읽어들인 kotlin의 컴파일러는 suspend와 resume을 위한 콜백 코드를 만들어준다. 한 번 그 속 안을 보러 가보자.

<br/>

### 1. Continuation 객체가 뭔가요?

continuation객체를 사용하는 것이다. coroutine에는 CPS 패러다임이 적용되어있다고 한다. CPS는 호출되는 함수에 Continuation을 전달하고, 각 함수의 작업이 완료되는대로 전달받은 Continuation을 호출하는 패러다임을 의미한다. 이런 부분에서 Continuation을 일종의 콜백이라고 생각할 수 있다. 그럼 이 Continuation을 어떻게 전달하고, 그 이유에 대해 알아보자.

<br/>

우선 [Continuation](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt) 코드를 보자.

```kotlin
/**
 * Interface representing a continuation after a suspension point that returns a value of type `T`.
 */
@SinceKotlin("1.3")
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    /**
     * Resumes the execution of the corresponding coroutine passing a successful or failed [result] as the
     * return value of the last suspension point.
     */
    public fun resumeWith(result: Result<T>)
}
```

*출처 : Corotuine 공식문서 중 -*

<br/>

코루틴은 다음에 무슨 일을 해야 할지 담고 있는 확장된 콜백이다. 위 코드를 풀어서 보자면 Continuation 인터페이스에는 크게 context 객체와 resumeWith라는 함수가 있다.

- resumeWith 함수는 특정한 함수가 suspend 되어야 될 때 현재 함수에서 특정한 함수의 결과 값을 T로 받게 해주는 함수이다.
- context는 각 Continuation이 특정 스레드나 스레드 풀에서 실행되는 것을 허용해준다.

CPS에서는 Continuation을 전달해주면서 당장 suspend된 부분에서 resume을 가능하게 해준다. Continuation은 resume되었을 때의 동작 관리를 위한 객체로, 연속적인 상태간의 의사소통하는 역할을 맡고 있다고 할 수 있다.

<br/>

요약하면 Continuation은 호출 함수 간의 일시 중단, 재개를 위한 의사소통하는 역할이고, CPS는 함수 호출 시에 이 Continuation을 전달하는 패러다임이다. Coroutine 역시 CPS로 구현되었다.

<br/>

### 2. suspend fun에서의 signature가 변경되었다.

Continuation은 kotlin 컴파일러가 suspend 함수의 시그니처를 변경하면서 사용된다.

<br/>

1. Continuation 객체가 기존 suspend 함수의 파라미터에 추가되고 suspend 키워드가 사라진다.

2. 이 객체는 suspend 함수 계산의 결과를 호출한 Coroutine에 전달하는데 사용된다.

3. 또한 함수의 리턴 타입 또한 androidApp에서 Unit으로 변경되었는데, 결과는 resume 함수의 매개변수를 통해 얻을 수 있게 된

   <br/>

Let’s Code!

```kotlin
suspend fun getAndroidApp(api: AndroidApi): AndroidApp {
        val sellerApp = api.fetchSeller()
        val guestApp = api.fetchGuest(sellerApp)
        val masterWeb = api.fetchMaster(guestWeb)
        val androidApplication = api.fetchAndroidApp(masterWeb)
        Log.d("드디어 안드로이드 앱을 손에 넣었다!")
        return androidApplication
}
```

이랬던 코드가

```kotlin
fun getAndroidApp(api: AndroidApi, completion: Continuation<Any?>) {
        val sellerApp = api.fetchSeller()
        val guestApp = api.fetchGuest(sellerApp)
        val masterWeb = api.fetchMaster(guestWeb)
        val androidApplication = api.fetchAndroidApp(masterWeb)
        Log.d("드디어 안드로이드 앱을 손에 넣었다!")
        completion.resume(androidApllication)
}
```

요렇게 변했어요!

<br/>

### 3. State Machine - Suspension Point를 기준으로 구분되는 코드들..

그럼 suspend, resume 연산이 어떻게 가능한 걸까?

이럴 땐… Let’s Code!

```kotlin
fun getAndroidApp(api: AndroidApi, completion: Continuation<Any?>) {
        val sellerApp = api.fetchSeller()
        val guestApp = api.fetchGuest(sellerApp)
        val masterWeb = api.fetchMaster(guestApp)
        val androidApplication = api.fetchAndroidApp(masterWeb)
        Log.d("드디어 안드로이드 앱을 손에 넣었다!")
        completion.resume(androidApllication)
}
fun getAndroidApp(api: AndroidApi, completion: Continuation<Any?>) {
        when(label) {
                0 -> // Label 0
                      val sellerApp = api.fetchSeller()// suspend
                
                1 -> // Label 1
                        val guestApp = api.fetchGuest(sellerApp)// suspend

                2 -> // Label 2
                        val masterWeb = api.fetchMaster(guestApp)// suspend

                3 -> // Label 3
                        val androidApplication = api.fetchAndroidApp(masterWeb)// suspend
                    
                4 -> { // Label 4
                        Log.d("드디어 안드로이드 앱을 손에 넣었다!")
                        completion.resume(AndroidApllication)
                }
        }
}
```

kotlin 컴파일러는 모든 중단 가능한 지점을 찾아 when으로 표현한다. Kotlin 컴파일러는 함수가 내부적으로 중단 가능 지점을 식별하고, 이 지점들에게 label을 부여하고 코드를 분리해서 각각의 코드들을 label로 인식한다. 이렇게 코드를 명확히 구분해두고, 중단 가능 지점에서 suspend되고 해당 작업이 끝나 resume되면 다시 다음 코드 구문을 실행할 수 있게된다. 이런 방식을 상태를 관리하는 하나의 방법이라 해서 state machine(상태 머신)이라고 불린다.

<br/>

즉, Kotlin 컴파일러는 suspend 함수를 컴파일 할 때 상태머신으로 표현한다. kotlin 컴파일러는 suspend 함수를 호출할 때마다 구역을 나누고, 라벨링해서 when문으로 표현한다. 여기서 라벨(label)과, suspend 함수 내부에서 선언된 변수들은 어떻게 관리할까?

<br/>

### 4. State Machine - label과 supend fun안의 내부 변수 관리

kotlin 컴파일러는 기존에 만들었던 함수 안에 클래스를 생성한다. 이 클래스는 Continuation의 자식개념인 CoroutineImpl을 구현하며, 기존의 suspend 함수에 선언된 변수들과, 실행 결과인 Result, 현재의 진행 상태인 label을 가지고 있다.

```kotlin
fun getAndroidApp(api: AndroidApi, completion: Continuation<Any?>) {
        class GetAndroidAppStateMachine(completion: Continuation<Any?>): CoroutineImpl(completion) {
                //기존의 함수 내부에 선언된 변수들들들
                var sellerApp : SellerApp? = null
                var guestApp : GuestApp? = null
                var masterWeb = MasterWeb? = null
                var androidApplication : AndroidApplication? = null

                var result: Any? = null
                var label: Int = 0

                override fun invokeSuspend(result: Any?) {
                        this.result = result
                        getAndroidApp(api, this)
                }
        }
} 
```

그럼 이 클래스가 어떻게 사용되는지 알아보러 Let’s Code!

```kotlin
fun getAndroid(api: AndroidApi, completion: Continuation<Any?>) {
        val continuation = completion as? GetAndroidStateMachine ?: GetAndroidStateMashine
        (completion)

        when (continuation.label) {
                0 -> {
                        throwOnFailure(continuation.result)
                        continuation.label = 1
                        api.fetchSeller(continuation) //suspend
                }
                1 -> {
                        throwOnFailue(continuation.result)
                        continuation.sellerApp = continuation.result as SellerApp
                        continuation.label = 2
                        api.fetchGuest(continuation.sellerApp, continuation) //suspend
                }
                2 -> {
                        throwOnFailue(continuation.result)
                        continuation.guestApp = continuation.result as GuestApp
                        continuation.label = 3
                        api.fetchMaster(continuation.guestApp, continuation) //suspend
                }
                3 -> {
                        throwOnFailue(continuation.result)
                        continuation.masterWeb = continuation.result as 
                        continuation.label = 4
                        api.fetchAndroidApp(continuation.masterWeb, continuation) //suspend
                }
                4 -> {
                        throwOnFailue(continuation.result)
                        continuation.androidApplication = continuation.result as AndroidApplication
                        Log.d("드디어 안드로이드 앱을 손에 넣었다!")
                        completion.resume(continuation.androidApllication)
                }
                else -> throw IllegalStateException
        }
}       
        
```

아까 3번째 단계에서 일시 중단 가능한 지점을 나눈 이유가 이 것 때문인데, Kotlin 컴파일러가 생성해준 GetAndroidAppStateMachine의 label이 when 안에서 사용되는데, 일시 중단한 지점에 따라 코드를 나누고 label로 그 코드를 구분했으니, 이제 resume하기 위해 label을 사용할 차례이다. 각 label분기에서 label의 값을 증분 시켜서 다음 실행할 지점을 가리키고, 이러한 절차를 거쳐 resume이 된다.

<br/>

한편 GetAndroidAppStateMachine을 살펴보면 기존에 suspend 함수 내부에 선언된 변수들이 null로 초기화된 모습을 볼 수가 있는데, 이 또한 when 절안에서 resume시 suspend되고 실행된 작업의 결과가 continuation.result로 받아오는 모습도 확인할 수가 있다. continuation.result를 각 타입으로 캐스팅해서 함수 내부에서 선언된 변수들을 관리하고 있다. 이렇게 각 코드 블록을 하나하나 실행할 때마다 함수의 상태가 점점 누적되고 마지막 resume을 통해 최종 값이 전달되게 된다. 이렇게 하면 하나의 Coroutine 코드가 끝나게 된다.

<br/>

이 글로써 Kotlin 컴파일러가 suspend 키워드를 동작하는 과정에 대해 알아보았다. 머리가 아프고 완벽히 이해하진 못했지만 이런 흐름이구나.. 하고 이해하고 다음에 더 자세히 알아봐야겠다…..suspend가 이렇게 복잡한 과정을 거친다…. 이제부터 suspned 쓸 때마다 JetBrain본사를 향해 절 한 번씩 하고 쓰길 바란다…

🤸‍♀️ ← 그랜절하는 중..
