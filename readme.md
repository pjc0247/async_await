Async-Await
====

`async`, `await`는 __C#__의 강력한 비동기 작업 모델입니다.<br>
이는 프로그래머가 쓰레드에 대한 지식 없이도 간단하게 무거운 작업을 백그라운드 스레드로 분배하고, 다시 메인 스레드로 스위치 시켜서 작업을 이어갈 수 있도록 해줍니다. (스레드의 고수준 추상화라고 생각합니다.)<br>

그동안 비동기 작업을 만들려면 알아야 했던 것들
----
* 단일 쓰레드 스폰 / 그리고 pre-spawn 된 쓰레드 풀
* 크리티컬 섹션
* 컨디셔널 변수
* 큐 (주로 컨디셔널 변수와 같이)
* 잡 디스패쳐 (애플의 GCD 같은..)

메소드를 async로 선언하기
----
메소드의 반환형 __왼쪽__에 `async` 키워드를 넣으면 자동으로 __대기 가능__한 메소드로 변신합니다.
```cs
async void MyFirstAsyncMethod() {
  Console.WriteLine("Hello Async");
}
```
`async`가 붙은 메소드에서는 `await`문을 사용할 수 있습니다.<br>
여기서 주의할 점은 `async`를 붙였다고 해서 메소드가 자동으로 비동기 실행으로 변경되지는 않는다는 것 입니다. `async`는 단순히 이 메소드가 __대기 가능__임을 나타내며, __대기__에 의해서 메소드가 비동기로 동작하게 될 수 는 있습니다. 이는 나중에 자세히 다루도록 합니다.<br>
<br>
메소드가 __대기 가능__ㅅ상태가 되었으니 이제 실제로 __대기__를 시켜 보겠습니다.<br>
메소드의 코드를 아래와 같이 변경한 후 실행시킵니다.
```cs
async void MyFirstAsyncMethod() {
  Console.WriteLine("Hello Async");
  await Task.Delay(1000);
  Console.WriteLine("After Delay");
}
```
코드를 실행시키면 예상과는 다른 결과가 나타날 것입니다.<br>
실제로 `After Delay` 부분은 전혀 출력되지 않은 채 프로그램이 종료됩니다. 심지어 1초동안 `Delay` 하는 코드조차도 실행되지 않은것처럼 보입니다.<br>
<br>
만약 `Threading`에 대해 다뤄본적이 있다면 `Join` 메소드를 생각해보세요.<br>
문제는 간단합니다. `Main` 메소드는 우리의 `MyFirstAsyncMethod`가 실행을 끝마치기도 전에 먼저 종료되어 프로세스가 끝난 상태가 되어버린 것입니다.<br>
<br>
문제를 해결하기 위해서 `Main` 메소드가 `MyFirstAsyncMethod`가 완전히 끝날때까지 대기하도록 합니다.
```cs
void Main(string[] args) {
  MyFirstAsyncMethod();

  System.Threading.Thread.Sleep(2000);
}
```
이제 정상적으로 1초 후에 `After Delay` 메세지가 출력되는것을 볼 수 있습니다.

반환값이 있는 대기 가능 메소드 만들기
----
```cs
async Task<int> SumAsync(int a,int b) {
  await Task.Delay(100);
  return a + b;
}
```

진짜로 비동기로 동작하는 메소드 만들기
----
앞에서 설명한것과 같이 메소드에 `async`를 붙이는것만으로 메소드는 비동기로 동작하지 않습니다.<br>
만약 메소드를 완전히 다른 스레드에서 실행되도록 하고 싶다면 `Task.Run` 메소드를 이용합니다.
```cs
Task FooAsync() {
  return Task.Run(() => {
    /* 여기에 원래 하려던걸 적습니다. */
  });
}
```
`Task.Run` 은 `Task`를 리턴합니다. `FooAsync` 메소드는 이 반환값을 그대로 리턴함으로써 호출자가 이 메소드가 끝날때까지 대기하도록 할 수 있습니다.<br>
재차 말하는 사항이지만, 이곳에 `async` 키워드를 적지 마세요. `async`는 비동기라는 뜻이 아닌, __대기 가능__을 나타냅니다. 이 메소드는 대기하지 않으므로 `async` 키워드 또한 필요 없습니다.

작업 분배기 만들기
----
__COM__의 `STA`와 처럼, 특정 실행 콘텍스트들을 하나의 스레드에 전부 묶어버려야 하는 경우가 있습니다.<br>
`WinForm` 프로그램에서는 이러한 기본적으로 이러한 고민이 적용되어 있기 때문에 개발자가 아무생각없이 `async`, `await`를 남발해도 사용상에 아무런 문제가 되지 않지만,
직접 처음부터 프레임워크를 구축하는 상황에서는 이를 반드시 고려해야 합니다.<br>
<br>
무슨 말이냐 하면, 윈폼 개발자들에게 아래와같은 코드는 굉장히 익숙할것입니다.
```cs
label.Text = "Downloading....";
var data = await UpdateDataFromHttpAsync();
label.Text = "Done";
```
위와같은 코드가 탈나지 않고 무사히 동작하는 이유는, 윈폼 프레임워크가 작업간의 교통정리를 알아서 잘 해주기 때문에 메인 스레드에서 대기된 코드는 다시 메인스레드에서 스케쥴되기 때문입니다.<br>
만약 콘솔 응용 프로그램을 만들고 위와 `비슷한` 코드를 작성하면 어떻게 될까요?
```cs
async void Foo() {
  Console.WriteLine(System.Threading.Thread.CurrentThread.ThreadId);
  await Task.Delay(1000);
  Console.WriteLine(System.Threading.Thread.CurrentThread.ThreadId);
}
```
안타깝게도, 서로 찍히는값이 다릅니다.<br>
첫번째 출력에서는 호출자와 같은(메인 스레드) 스레드 아이디가 찍히지만, 두번째는 그렇지 않습니다.<br>
분명 윈폼 개발환경에서는 호출 전과 후의 스레드가 동일하게 UI 스레드임이 보장되었지만, 콘솔 개발환경에서는 그렇지 않은 것으로 보입니다. 이것은 왜그럴까여??????????????????????????????????????
