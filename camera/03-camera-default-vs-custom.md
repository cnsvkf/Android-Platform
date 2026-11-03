# 기본 카메라 vs 커스텀 카메라: 사진은 어떻게 앱으로 돌아올까?

> 앱에서 사진을 찍는 방법은 두 가지입니다.
> **기본 카메라**는 "카메라 앱아, 사진 한 장 찍어 줘" 하고 **부탁**하는 방식입니다.
> **커스텀 카메라**는 카메라 화면을 **내 앱 안에서 직접** 그리고 사진도 **직접** 찍는 방식입니다.
> 이 글에서는 두 방식의 흐름, 사진 파일이 전달되는 방법, 코드의 차이를 순서대로 설명합니다.

---

## 0. 먼저 알아둘 개념 2가지

### 개념 1: 앱은 다른 앱의 파일 경로를 직접 열 수 없습니다

카메라 앱은 **다른 앱**입니다. 내 앱이 "/cache/photo.jpg"라는 경로를 알려 줘도 카메라 앱은 그 파일을 쓸 권한이 없습니다.
그래서 **"이 주소의 파일에 저장해 주세요"** 라는 **Uri(주소표)** 를 건네줍니다.

### 개념 2: FileProvider는 "안전한 열쇠"입니다

Android 7.0 이상에서는 `file://` 경로를 다른 앱에 넘기면 오류가 납니다.
`FileProvider`는 내 파일을 `content://` 형식의 Uri로 바꿔 주고, **필요한 권한만 잠깐** 열어 줍니다.

```
내 앱의 파일  ──(FileProvider)──▶  content://...  Uri  ──▶  카메라 앱·다른 앱
(/cache/photos/a.jpg)              (열쇠 역할)             (권한 받아서 읽고 씀)
```

---

## 1. 한눈에 비교하기

| 구분 | 기본 카메라 | 커스텀 카메라 (CameraX) |
|---|---|---|
| 비유 | 사진관에 맡기기 | 내 스튜디오에서 직접 찍기 |
| 카메라 화면 | 시스템 카메라 앱 | 내 앱 안의 미리보기 |
| 저장을 누가 하나요? | 카메라 앱 | 내 앱 |
| 카메라 권한(`CAMERA`) | 필요 없음 | 필요함 |
| 준비 코드 양 | 적음 | 많음 |
| 화면 꾸미기 | 불가능 | 자유롭게 가능 |
| 어울리는 경우 | 사진 한 장만 필요할 때 | 가이드 선을 띄우거나 연속 촬영할 때 |

> **한 줄 정리**
> 사진 한 장이면 **기본 카메라**, 화면과 동작을 직접 설계해야 하면 **커스텀 카메라**입니다.

---

## 2. 기본 카메라: 카메라 앱에 부탁하기

### 2-1. 전체 흐름

```
① 저장할 빈 파일을 만든다 (내 앱 안)
② 그 파일의 content:// Uri를 만든다 (FileProvider)
③ 카메라 앱에 "이 Uri에 찍어 줘"라고 부탁한다 (launch)
④ 사용자가 찍고 확인을 누른다
⑤ 카메라 앱이 Uri의 파일에 사진을 쓴다
⑥ 성공 여부(true/false)가 돌아온다
⑦ 내 앱은 같은 파일을 읽어서 쓴다
```

핵심은 **⑤에서 사진이 "돌아오는" 것이 아니라 이미 파일에 저장되어 있다**는 점입니다.

### 2-2. 준비 1: FileProvider 설정

**AndroidManifest.xml**

```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

**res/xml/file_paths.xml** (공개할 폴더를 적습니다)

```xml
<paths>
    <!-- 캐시 폴더 안의 photos 폴더만 다른 앱에 공개합니다. -->
    <cache-path name="photos" path="photos/" />
</paths>
```

> `authorities`는 코드의 `"${packageName}.fileprovider"`와 **반드시 같아야** 합니다.

### 2-3. 준비 2: 저장할 파일과 Uri 만들기

```kotlin
fun createCaptureUri(context: Context): Uri {
    // 1) 캐시 폴더 안에 photos 폴더를 만듭니다.
    val photoDir = File(context.cacheDir, "photos").apply { mkdirs() }

    // 2) 겹치지 않는 이름의 빈 파일을 만듭니다.
    val file = File(photoDir, "capture_${UUID.randomUUID()}.jpg")
    file.createNewFile()

    // 3) 파일을 content:// Uri로 바꿉니다. 이 Uri를 카메라 앱에 줍니다.
    return FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        file,
    )
}
```

### 2-4. 촬영 요청과 결과 받기

```kotlin
@Composable
fun BasicCameraButton(onPhotoSaved: (Uri) -> Unit) {
    val context = LocalContext.current
    var pendingUri by remember { mutableStateOf<Uri?>(null) }

    // TakePicture: 카메라 앱을 실행하고, 결과는 true/false로만 돌아옵니다.
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture(),
    ) { success ->
        val uri = pendingUri
        if (success && uri != null) {
            onPhotoSaved(uri)      // 사진은 이미 uri 파일에 있습니다.
        }
    }

    Button(onClick = {
        val uri = createCaptureUri(context)
        pendingUri = uri
        launcher.launch(uri)       // 카메라 앱 실행: 이 Uri에 저장해 달라고 부탁
    }) {
        Text("카메라로 찍기")
    }
}
```

> 📌 **포인트**: `onActivityResult`의 `data.extras["data"]`로 받던 **썸네일**은 작은 이미지입니다. 원본을 받으려면 **Uri에 저장하는 방식**을 써야 합니다.

---

## 3. 커스텀 카메라: 내 앱에서 직접 찍기 (CameraX)

커스텀 카메라는 **세 단계**로 생각하면 쉽습니다.

```
① 권한 받기 (CAMERA)
② 미리보기 화면 보여 주기 (PreviewView + Preview)
③ 버튼을 누르면 사진 저장하기 (ImageCapture)
```

### 3-1. 권한 요청

**AndroidManifest.xml**

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

**Compose 코드**

```kotlin
var hasCameraPermission by remember { mutableStateOf(false) }

// 권한 요청 창을 띄우고, 허용 여부를 Boolean으로 받습니다.
val permissionLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestPermission(),
) { granted ->
    hasCameraPermission = granted
}

// 화면이 처음 열릴 때 한 번 요청합니다.
LaunchedEffect(Unit) {
    permissionLauncher.launch(Manifest.permission.CAMERA)
}
```

### 3-2. 미리보기 화면 보여 주기

```kotlin
val lifecycleOwner = LocalLifecycleOwner.current
val imageCapture = remember { ImageCapture.Builder().build() }

AndroidView(factory = { context ->
    PreviewView(context).also { previewView ->
        val future = ProcessCameraProvider.getInstance(context)
        future.addListener({
            val cameraProvider = future.get()

            // 미리보기를 화면(PreviewView)에 연결합니다.
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

            // 이전 연결을 지우고, 뒤쪽 카메라에 미리보기와 촬영 기능을 함께 붙입니다.
            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                lifecycleOwner,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageCapture,
            )
        }, ContextCompat.getMainExecutor(context))
    }
})
```

> 📌 `bindToLifecycle`에 `lifecycleOwner`를 넘기면 **화면이 꺼질 때 카메라도 자동으로 꺼집니다.**

### 3-3. 버튼을 누르면 파일로 저장하기

```kotlin
fun takePhoto(
    context: Context,
    imageCapture: ImageCapture,
    onSaved: (Uri) -> Unit,
) {
    // 1) 저장할 파일을 정합니다. 앱이 직접 쓰므로 경로를 알고 있습니다.
    val file = File(File(context.cacheDir, "photos").apply { mkdirs() }, "custom_${UUID.randomUUID()}.jpg")
    val options = ImageCapture.OutputFileOptions.Builder(file).build()

    // 2) 촬영하고 파일로 저장합니다. 결과는 콜백으로 받습니다.
    imageCapture.takePicture(
        options,
        ContextCompat.getMainExecutor(context),
        object : ImageCapture.OnImageSavedCallback {
            override fun onImageSaved(results: ImageCapture.OutputFileResults) {
                // 3) 다른 앱에 넘길 때를 위해 content:// Uri로 바꿔서 돌려줍니다.
                onSaved(FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file))
            }

            override fun onError(exception: ImageCaptureException) {
                // 실패 처리 (예: 안내 메시지 표시)
            }
        },
    )
}
```

### 3-4. 커스텀 카메라의 흐름 정리

```
권한 허용 → 미리보기 시작 → 버튼 클릭 → takePicture
   → 앱이 직접 file에 JPEG 저장 → onImageSaved 콜백 → Uri로 변환해서 사용
```

---

## 4. 두 방식의 차이 한눈에 보기

| 항목 | 기본 카메라 | 커스텀 카메라 |
|---|---|---|
| 촬영을 시작하는 코드 | `launcher.launch(uri)` | `imageCapture.takePicture(...)` |
| 저장하는 주체 | 카메라 앱 | 내 앱 |
| 저장 위치를 정하는 방법 | FileProvider Uri를 넘김 | `OutputFileOptions(file)`에 파일을 넘김 |
| 결과를 확인하는 방법 | `success: Boolean` | `onImageSaved` / `onError` 콜백 |
| 필요한 권한 | 없음 | `CAMERA` |
| 다른 앱에 넘길 때 | 이미 `content://` Uri | `FileProvider`로 Uri 변환 필요 |
| 실패했을 때 정리 | 취소되면 빈 파일 삭제 | 실패하면 파일 삭제 |

---

## 5. 사진을 다른 앱(메신저 등)에 보내기

파일 경로(`file://`)를 그대로 보내면 안 됩니다. 반드시 **content:// Uri**를 보내고 **읽기 권한 flag**를 함께 줍니다.

```kotlin
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpeg"                                   // 사진 파일
    putExtra(Intent.EXTRA_STREAM, photoUri)               // content:// Uri
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)       // 받는 앱이 읽을 수 있게 허락
}
context.startActivity(Intent.createChooser(shareIntent, "사진 공유"))
```

> 자세한 Intent 사용법은 **2편(Intent 완전 정리)** 을 참고하세요.

---

## 6. 어떤 방식을 골라야 할까?

```
사진 한 장이면 충분한가?
├─ 예 → 기본 카메라 (TakePicture)
│       · 권한 필요 없음 · 준비 코드 적음
└─ 아니오 → 화면을 직접 꾸미거나 연속 촬영이 필요한가?
            └─ 예 → 커스텀 카메라 (CameraX)
```

---

## 7. 자주 하는 실수

| 실수 | 해결 |
|---|---|
| `authorities`와 코드의 문자열이 다름 | 둘 다 `${packageName}.fileprovider`로 맞춤 |
| `file_paths.xml`에 폴더를 안 적음 | 사진을 저장하는 폴더를 `cache-path`로 추가 |
| 파일 경로를 다른 앱에 그대로 넘김 | FileProvider로 `content://` Uri를 만듦 |
| 커스텀 카메라에서 권한 확인을 안 함 | 권한 허용 후에만 `bindToLifecycle` 호출 |
| 찍은 파일을 캐시에 두고 계속 쌓임 | 쓰지 않는 파일은 삭제 |

---

## 참고 자료

- [Take photos with the camera app (Android Developers)](https://developer.android.com/training/camera/photobasics)
- [Take a photo with CameraX (Android Developers)](https://developer.android.com/media/camera/camerax/take-photo)
- [Package visibility (Android Developers)](https://developer.android.com/training/package-visibility)

> 조사 시점: 2026-11-02. CameraX 예제는 공식 문서의 `OutputFileOptions`와 `OnImageSavedCallback` 흐름을 기준으로 했습니다.
