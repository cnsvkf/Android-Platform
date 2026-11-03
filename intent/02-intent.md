# Intent 정리: 저는 이렇게 이해했습니다 (화면 이동, 외부 앱 호출, 결과 받기)

> Intent를 처음 봤을 때 저는 "이게 대체 뭐지?"라는 생각이 먼저 들었습니다.
> 여러 번 읽고 코드를 살펴보니, Intent는 **"이 일을 해 줘"라고 부탁하는 쪽지**라는 것을 알게 됐습니다.
> 같은 앱의 다른 화면에 부탁할 수도 있고, 브라우저나 카메라 같은 **다른 앱에 부탁할 수도** 있습니다.
> 이 글에는 제가 정리한 내용을 적었고, 읽고 나면 Intent 코드를 보고 "누구에게, 무엇을, 어떤 결과로" 부탁하는지 바로 읽을 수 있을 거라고 생각합니다.

---

## 0. 저는 편지 봉투에 비유해서 이해했습니다

Intent를 **편지 봉투**라고 생각하면 구조가 잘 보였습니다.

| 편지 봉투의 부분 | Intent의 부분 | 제가 든 예 |
|---|---|---|
| 받는 사람 | `component` (누구에게 갈지) | `DetailActivity` |
| 용건 | `action` (무엇을 할지) | `ACTION_VIEW` (보여 줘) |
| 보낼 물건 | `data` (어떤 대상인지) | `https://developer.android.com` |
| 첨부 메모 | `extras` (추가 정보) | `"memo_id" = 42` |
| 배달 옵션 | `flags` (어떻게 처리할지) | 읽기 권한 주기 |

> 받는 사람을 **정해서** 보내면 **명시적**, 용건만 적어서 **아무 앱이나 처리하게** 하면 **암시적**이라고 정리했습니다.

---

## 1. Intent로 할 수 있는 일들

| 하고 싶은 일 | 어떤 Intent를 쓰나요? | 예시 |
|---|---|---|
| 다른 화면 열기 | 명시적 Intent | 상세 화면으로 이동 |
| 웹 페이지 열기 | 암시적 `ACTION_VIEW` | 브라우저에서 링크 열기 |
| 전화 화면 열기 | 암시적 `ACTION_DIAL` | 번호만 채워 둔 전화 화면 |
| 글 공유하기 | 암시적 `ACTION_SEND` | 공유 창 띄우기 |
| 사진 찍기 | `ACTION_IMAGE_CAPTURE` | 카메라 앱 실행 |
| 결과 돌려받기 | Activity Result API | 파일 고르기, 촬영 결과 받기 |

---

## 2. 명시적 Intent: 같은 앱 안에서 화면 옮기기

받는 사람(화면 클래스)을 **직접 지정**하는 방식입니다.

```kotlin
// 메모 상세 화면을 열면서 메모 번호를 함께 보냅니다.
val intent = Intent(context, MemoDetailActivity::class.java).apply {
    putExtra("memo_id", 42L)      // 보낼 값: 키 이름과 값
}
context.startActivity(intent)
```

**받는 쪽(MemoDetailActivity)에서는 이렇게 꺼냅니다.**

```kotlin
// 보낼 때 쓴 키 이름("memo_id")과 같아야 값을 받을 수 있습니다.
val memoId = intent.getLongExtra("memo_id", -1L)
```

> 📌 **주의**: 키 이름의 철자가 한 글자라도 다르면 값을 받지 못하고 기본값(`-1`)이 나옵니다. 그래서 키 이름을 상수로 만들어 함께 쓰는 편이 안전합니다.

---

## 3. 암시적 Intent: 다른 앱에 일 맡기기

받는 사람은 적지 않고 **용건(action)과 대상(data)만** 적습니다. 시스템이 그 일을 할 수 있는 앱을 찾아 줍니다.

### 3-1. 웹 페이지 열기 (`ACTION_VIEW`)

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://developer.android.com"))
context.startActivity(intent)
```

- `ACTION_VIEW`는 "보여 줘"라는 뜻으로 이해했습니다.
- `Uri`에 `https://`를 넣으면 브라우저가, `geo:`를 넣으면 지도 앱이 처리한다는 점이 신기했습니다.

### 3-2. 전화 화면 열기 (`ACTION_DIAL`)

```kotlin
// 번호만 채운 전화 화면을 엽니다. 바로 거는 ACTION_CALL과 달리 권한이 필요 없습니다.
val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:01012345678"))
context.startActivity(intent)
```

### 3-3. 글 공유하기 (`ACTION_SEND`)

```kotlin
val sendIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"                                  // 보내는 데이터의 종류
    putExtra(Intent.EXTRA_TEXT, "오늘의 메모를 공유합니다.")
}
// createChooser를 쓰면 공유할 앱을 고르는 창이 한 번 뜹니다.
context.startActivity(Intent.createChooser(sendIntent, "공유하기"))
```

### 3-4. 처리할 앱이 없을 때 막는 방법

처리할 앱이 없으면 `ActivityNotFoundException`이 납니다. 이 예외를 잡아서 처리할 수 있습니다.

```kotlin
fun openWebPage(context: Context, url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    try {
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        // 브라우저가 없을 때 안내를 보여 줍니다.
    }
}
```

> Android 11(API 30) 이상에서는 다른 앱을 **조회**하려면 매니페스트에 `<queries>`를 적어야 한다는 점을 기억해 두세요. 조회 없이 `startActivity`만 호출하면 보통 필요 없습니다.

```xml
<!-- AndroidManifest.xml: "https 주소를 여는 앱이 있는지" 확인할 때 필요합니다. -->
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" />
    </intent>
</queries>
```

---

## 4. 결과 돌려받기: Activity Result API

"사진을 고르고 그 결과를 받고 싶다"처럼 **돌아오는 값**이 필요할 때 이 방식을 씁니다. 예전의 `startActivityForResult`와 `onActivityResult`는 지금은 권장되지 않으므로, 여기서는 새 방식을 기준으로 정리합니다.

### 4-1. 제가 이해한 흐름

```
내 화면                    다른 앱(또는 화면)
  │  launch(intent) ──────▶  실행
  │                          사용자가 작업
  │  ◀────── 결과 전달 ───────
  └─ 콜백에서 결과를 처리
```

### 4-2. Compose에서 파일 고르기

```kotlin
@Composable
fun PickImageButton(onPicked: (Uri) -> Unit) {
    // 1) 어떤 계약(Contract)으로 부탁할지 정합니다. GetContent는 파일 한 개를 고르는 계약입니다.
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent(),
    ) { uri: Uri? ->
        // 2) 결과가 돌아오면 이 콜백이 실행됩니다. 취소하면 null이 옵니다.
        uri?.let(onPicked)
    }

    Button(onClick = {
        // 3) 이미지 종류만 고르도록 요청합니다.
        launcher.launch("image/*")
    }) {
        Text("사진 고르기")
    }
}
```

### 4-3. 자주 쓰는 계약(Contract)

| 계약 | 하는 일 | 돌아오는 값 |
|---|---|---|
| `StartActivityForResult` | 임의의 Intent 실행 | `ActivityResult` (`resultCode`, `data`) |
| `GetContent` | 파일 한 개 고르기 | `Uri?` |
| `PickVisualMedia` | 사진·영상 선택기 열기 | `Uri?` |
| `TakePicture` | 카메라로 찍어 지정한 Uri에 저장 | `Boolean` |
| `RequestPermission` | 권한 요청 | `Boolean` |

> 카메라 촬영은 **3편(기본 카메라 vs 커스텀 카메라)** 에서 더 자세히 적었습니다.

---

## 5. Intent의 flags

| flag | 뜻 | 언제 쓰나요? |
|---|---|---|
| `FLAG_ACTIVITY_NEW_TASK` | 새 작업(화면 묶음)에서 화면을 엽니다 | Activity가 아닌 곳(예: 서비스)에서 화면을 열 때 |
| `FLAG_GRANT_READ_URI_PERMISSION` | 상대 앱에게 파일 읽기 권한을 잠깐 줍니다 | `content://` 파일을 다른 앱에 넘길 때 |

```kotlin
// 파일(Uri)을 다른 앱에 보낼 때는 읽기 권한을 함께 줘야 합니다.
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpeg"
    putExtra(Intent.EXTRA_STREAM, photoUri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```

---

## 6. 체크리스트

- [ ] 받는 화면이 정해져 있으면 **명시적**, 아니면 **암시적** Intent를 씁니다.
- [ ] 암시적 Intent는 처리할 앱이 없을 때를 대비합니다. (`ActivityNotFoundException`)
- [ ] 결과가 필요하면 **Activity Result API**를 씁니다.
- [ ] 파일을 넘길 때는 `content://` Uri와 `FLAG_GRANT_READ_URI_PERMISSION`을 씁니다.
- [ ] Android 11 이상에서 다른 앱을 조회하면 `<queries>`를 선언합니다.

---

## 참고 자료

- [Intents and intent filters (Android Developers)](https://developer.android.com/guide/components/intents-filters)
- [Package visibility (Android Developers)](https://developer.android.com/training/package-visibility)

> 조사 시점: 2026-11-02.
