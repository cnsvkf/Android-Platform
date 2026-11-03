# Intent 완전 정리: 화면 이동, 외부 앱 호출, 결과 받기

> Intent는 한마디로 **"이 일을 해 줘"라고 부탁하는 쪽지**입니다.
> 같은 앱의 다른 화면에 부탁할 수도 있고, 웹 브라우저·지도·카메라 같은 **다른 앱에 부탁할 수도** 있습니다.
> 이 글을 읽고 나면 Intent 코드를 보고 "누구에게, 무엇을, 어떤 결과로" 부탁하는지 바로 읽을 수 있게 됩니다.

---

## 0. 편지 봉투로 이해하기

Intent를 **편지 봉투**라고 생각해 보세요.

| 편지 봉투의 부분 | Intent의 부분 | 예시 |
|---|---|---|
| 받는 사람 | `component` (누구에게 갈지) | `DetailActivity` |
| 용건 | `action` (무엇을 할지) | `ACTION_VIEW` (보여 줘) |
| 보낼 물건 | `data` (어떤 대상인지) | `https://developer.android.com` |
| 첨부 메모 | `extras` (추가 정보) | `"memo_id" = 42` |
| 배달 옵션 | `flags` (어떻게 처리할지) | 읽기 권한 주기 |

> 받는 사람을 **정해서** 보내면 **명시적**, 용건만 적어 **아무나 처리하게** 하면 **암시적**입니다.

---

## 1. Intent로 할 수 있는 일

| 하고 싶은 일 | 어떤 Intent를 쓰나요? | 예시 |
|---|---|---|
| 다른 화면 열기 | 명시적 Intent | 상세 화면으로 이동 |
| 웹 페이지 열기 | 암시적 `ACTION_VIEW` | 브라우저에서 링크 열기 |
| 전화 걸기 화면 열기 | 암시적 `ACTION_DIAL` | 번호만 채워 둔 전화 화면 |
| 글 공유하기 | 암시적 `ACTION_SEND` | 공유 창 띄우기 |
| 사진 찍기 | `ACTION_IMAGE_CAPTURE` | 카메라 앱 실행 |
| 결과 돌려받기 | Activity Result API | 파일 고르기, 촬영 결과 받기 |

---

## 2. 명시적 Intent: 같은 앱 안에서 화면 이동

받는 사람(화면 클래스)을 **직접 지정**합니다.

```kotlin
// 메모 상세 화면을 열면서 메모 번호를 함께 보냅니다.
val intent = Intent(context, MemoDetailActivity::class.java).apply {
    putExtra("memo_id", 42L)      // 보낼 값: 키 이름과 값
}
context.startActivity(intent)
```

**받는 쪽(MemoDetailActivity)에서는 이렇게 꺼냅니다.**

```kotlin
// 보낼 때 쓴 키 이름("memo_id")과 같아야 합니다.
val memoId = intent.getLongExtra("memo_id", -1L)
```

> 📌 **실수 포인트**: 키 이름이 한 글자라도 다르면 값을 받지 못합니다. 보통 키 이름은 상수로 만들어 두고 함께 씁니다.

---

## 3. 암시적 Intent: 다른 앱에 일 맡기기

받는 사람을 적지 않고 **용건(action)과 대상(data)만** 적습니다. 시스템이 그 일을 할 수 있는 앱을 찾아 줍니다.

### 3-1. 웹 페이지 열기 (`ACTION_VIEW`)

```kotlin
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://developer.android.com"))
context.startActivity(intent)
```

- `ACTION_VIEW`는 "보여 줘"라는 뜻입니다.
- `Uri`에 `https://`를 넣으면 브라우저가, `geo:`를 넣으면 지도 앱이 처리합니다.

### 3-2. 전화 화면 열기 (`ACTION_DIAL`)

```kotlin
// 번호만 채운 전화 화면을 엽니다. 전화를 바로 거는 것(ACTION_CALL)과 다르게 권한이 필요 없습니다.
val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:01012345678"))
context.startActivity(intent)
```

### 3-3. 글 공유하기 (`ACTION_SEND`)

```kotlin
val sendIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"                                  // 보내는 데이터의 종류
    putExtra(Intent.EXTRA_TEXT, "오늘의 메모를 공유합니다.")
}
// createChooser: 공유 앱 선택 창을 한 번 띄워 줍니다.
context.startActivity(Intent.createChooser(sendIntent, "공유하기"))
```

### 3-4. 처리할 앱이 있는지 먼저 확인하기

처리할 앱이 없으면 `ActivityNotFoundException`이 납니다. 미리 확인하거나 감싸서 처리합니다.

```kotlin
fun openWebPage(context: Context, url: String) {
    val intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
    try {
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        // 브라우저가 없을 때 안내
    }
}
```

> Android 11(API 30) 이상에서는 다른 앱을 **조회**하려면 매니페스트에 `<queries>`를 선언해야 합니다. 조회 없이 `startActivity`만 호출하면 보통 필요 없습니다.

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

"사진을 고르고 그 결과를 받아 오고 싶다"처럼 **돌아오는 값**이 필요할 때 씁니다.
예전의 `startActivityForResult`와 `onActivityResult`는 지금 권장되지 않습니다.

### 4-1. 흐름 이해하기

```
내 화면                    다른 앱(또는 화면)
  │  launch(intent) ──────▶  실행
  │                          사용자가 작업
  │  ◀────── 결과 전달 ───────
  └─ 콜백에서 결과 처리
```

### 4-2. Compose에서 쓰는 예제 (파일 고르기)

```kotlin
@Composable
fun PickImageButton(onPicked: (Uri) -> Unit) {
    // 1) 어떤 계약(Contract)으로 부탁할지 정합니다. GetContent: 파일 한 개 고르기
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent(),
    ) { uri: Uri? ->
        // 2) 결과가 돌아오면 이 콜백이 실행됩니다. 취소하면 null입니다.
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

> 카메라 촬영은 **3편(기본 카메라 vs 커스텀 카메라)** 에서 자세히 다룹니다.

---

## 5. Intent의 flags 한눈에 보기

| flag | 뜻 | 언제 쓰나요? |
|---|---|---|
| `FLAG_ACTIVITY_NEW_TASK` | 새 작업(화면 묶음)에서 열기 | Activity가 아닌 곳(예: 서비스)에서 화면을 열 때 |
| `FLAG_GRANT_READ_URI_PERMISSION` | 상대 앱에 파일 읽기 권한 잠깐 주기 | `content://` 파일을 다른 앱에 넘길 때 |

```kotlin
// 파일(Uri)을 다른 앱에 보낼 때는 읽기 권한을 함께 줘야 합니다.
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpeg"
    putExtra(Intent.EXTRA_STREAM, photoUri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
```

---

## 6. 정리 체크리스트

- [ ] 받는 화면이 정해져 있으면 **명시적**, 아니면 **암시적**을 씁니다.
- [ ] 암시적 Intent는 처리할 앱이 없을 때를 대비합니다. (`ActivityNotFoundException`)
- [ ] 결과가 필요하면 **Activity Result API**를 씁니다.
- [ ] 파일을 넘길 때는 `content://` Uri와 `FLAG_GRANT_READ_URI_PERMISSION`을 씁니다.
- [ ] Android 11 이상에서 다른 앱을 조회하면 `<queries>`를 선언합니다.

---

## 참고 자료

- [Intents and intent filters (Android Developers)](https://developer.android.com/guide/components/intents-filters)
- [Package visibility (Android Developers)](https://developer.android.com/training/package-visibility)

> 조사 시점: 2026-11-02.
