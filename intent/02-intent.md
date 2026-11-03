# [Android] Intent 정리: 명시적·암시적 Intent와 Activity Result

> Intent는 "이런 일을 해 줘"라고 다른 컴포넌트(Activity, 외부 앱)에 요청하는 메시지 객체입니다.
> 이 글에서는 Intent의 구성 요소, 두 가지 종류, 결과를 받는 방법을 정리합니다.

---

## 1. Intent로 할 수 있는 일

| 용도 | 메서드 | 예시 |
|---|---|---|
| 화면(Activity) 시작 | `startActivity` | 상세 화면으로 이동 |
| 결과를 돌려받기 | Activity Result API | 사진 촬영, 파일 선택 |
| 백그라운드 작업 시작 | `startService` 등 | 현재는 WorkManager 권장 |
| 방송 전달 | `sendBroadcast` | 시스템 이벤트 수신 |

> 이 글은 화면 시작과 결과 받기에 집중합니다.

---

## 2. Intent의 구성 요소

| 속성 | 역할 | 예시 |
|---|---|---|
| `action` | 무엇을 할지 | `Intent.ACTION_VIEW` |
| `data` (Uri) | 대상 데이터 | `https://...` |
| `type` (MIME) | 데이터 종류 | `image/jpeg` |
| `extras` | 부가 데이터 (키-값) | `Intent.EXTRA_TEXT` |
| `flags` | 실행 방식 | `FLAG_GRANT_READ_URI_PERMISSION` |
| `component` | 실행할 대상을 직접 지정 | `ComponentName(...)` |

---

## 3. 명시적 Intent vs 암시적 Intent

| 구분 | 명시적 (Explicit) | 암시적 (Implicit) |
|---|---|---|
| 대상 지정 | 클래스/컴포넌트를 직접 지정 | action·data·type으로 "조건"만 지정 |
| 사용처 | 같은 앱 안의 화면 이동 | 다른 앱에 일을 맡길 때 |
| 대상 확인 | 필요 없음 | 여러 앱이 받을 수 있음 (선택창 표시 가능) |

### 3-1. 명시적 Intent (같은 앱 화면 이동)

```kotlin
// 같은 앱 안에서 특정 Activity를 직접 연다
val intent = Intent(context, MemoDetailActivity::class.java).apply {
    putExtra("memo_id", 42L) // 화면에 넘길 값
}
startActivity(intent)
```

### 3-2. 암시적 Intent (다른 앱에 맡기기)

```kotlin
// 웹 페이지 열기: 이 Intent를 처리할 수 있는 앱을 시스템이 찾는다
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://developer.android.com"))
startActivity(intent)
```

**주의**: 처리할 앱이 없으면 `ActivityNotFoundException`이 납니다. 호출 전에 확인하거나 `try-catch`로 감쌉니다.

```kotlin
// 처리할 앱이 있는지 먼저 확인한다 (Android 11 이상은 <queries> 선언 필요)
if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
} else {
    // 처리할 앱이 없을 때의 안내
}
```

---

## 4. 자주 쓰는 암시적 Intent 예시

| 목적 | action | 핵심 데이터 |
|---|---|---|
| 웹 열기 | `ACTION_VIEW` | `Uri` (http/https) |
| 전화 걸기 | `ACTION_DIAL` | `Uri("tel:...")` |
| 텍스트 공유 | `ACTION_SEND` | `EXTRA_TEXT`, type `text/plain` |
| 사진 촬영 | `ACTION_IMAGE_CAPTURE` | `EXTRA_OUTPUT` (저장할 Uri) |

```kotlin
// 텍스트 공유 (공유 시트 표시)
val sendIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "오늘의 메모를 공유합니다.")
}
startActivity(Intent.createChooser(sendIntent, "공유하기"))
```

> `createChooser`를 쓰면 매번 앱 선택창을 보여 줍니다.

---

## 5. 결과 받기: Activity Result API

예전에는 `startActivityForResult`와 `onActivityResult`를 썼습니다. 지금은 **Activity Result API**를 씁니다.

### 5-1. Compose에서 쓰는 방법

```kotlin
@Composable
fun ShareButton() {
    // 계약(Contract)을 정하고, 결과를 받을 콜백을 등록한다
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent(),
    ) { uri: Uri? ->
        // 선택한 파일의 Uri를 받는다 (취소하면 null)
        uri?.let { handlePickedFile(it) }
    }

    Button(onClick = { launcher.launch("image/*") }) {
        Text("이미지 선택")
    }
}
```

### 5-2. 주요 계약(Contract)

| 계약 | 하는 일 | 결과 타입 |
|---|---|---|
| `StartActivityForResult` | 임의의 Intent 실행 | `ActivityResult` |
| `GetContent` | 파일 선택 (MIME 필터) | `Uri?` |
| `PickVisualMedia` | 사진·영상 선택기 | `Uri?` |
| `TakePicture` | 카메라로 촬영 후 지정한 Uri에 저장 | `Boolean` |
| `RequestPermission` | 권한 요청 | `Boolean` |

> 카메라 촬영 방식은 다음 글(기본 카메라 vs 커스텀 카메라)에서 자세히 다룹니다.

---

## 6. 정리

- **명시적 Intent**: 같은 앱 화면 이동
- **암시적 Intent**: action과 data로 "이 일을 할 수 있는 앱"을 찾아 맡김
- 결과가 필요하면 `startActivityForResult` 대신 **Activity Result API**를 사용
- Android 11 이상에서 다른 앱을 조회·확인하려면 매니페스트에 `<queries>`를 선언

```xml
<!-- AndroidManifest.xml: 웹 브라우저 처리 앱을 확인하려면 선언이 필요하다 -->
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" />
    </intent>
</queries>
```

---

## 참고 자료

- [Intents and intent filters (Android Developers)](https://developer.android.com/guide/components/intents-filters)
- [Package visibility (Android Developers)](https://developer.android.com/training/package-visibility)

> 조사 시점: 2026-11-02.
