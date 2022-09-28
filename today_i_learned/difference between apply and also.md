## apply
람다를 호출한 수신자(receiver) 객체에 직접 접근한다.

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit) : T {
  // ... 
  block()
  return this
}
```

## also
사용자 지정 이름(암시적 이름)으로 람다를 호출한 수신자(receiver) 객체를 참조할 수 있다.

```kotlin
public inline fun <T> T.also(block: (T) -> Unit) : T {
  // ...
  block(T)
  return this
}
```

## 사용예시

### apply

```kotlin
xxx.apply {
  this.name = "영호"
}
```

### also

```kotlin
xxx.also { data -> 
  data.name = "영호"
}
```
