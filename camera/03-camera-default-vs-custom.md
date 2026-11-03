# [Android] 기본 카메라 vs 커스텀 카메라: 사용법과 파일 전달 방식 비교

> 사진을 찍는 방법은 크게 두 가지입니다. 시스템 카메라 앱을 호출하는 **기본 카메라**와, 카메라 화면을 직접 그리는 **커스텀 카메라**(CameraX)입니다.
> 이 글에서는 두 방식의 사용법, 촬영한 파일이 앱으로 전달되는 방식, 코드 차이를 정리합니다.

---

## 1. 한눈에 보기

| 구분 | 기본 카메라 | 커스텀 카메라 (CameraX) |
|---|---|---|
| 카메라 화면 | 시스템 카메라 앱 | 내 앱의 `PreviewView` |
| 구현량 | 적음 | 많음 |
| 권한 | 대부분 필요 없음 (촬영은 시스템 앱이 함) | `CAMERA` 권한 필요 |
| 파일 전달 | 지정한 Uri에 시스템 앱이 저장 | `OutputFileOptions`로 앱이 저장 |
| 화면 커스터마이징 | 불가 | 자유롭게 가능 |
| 적합한 경우 | 사진 한 장 찍기 | 가이드 오버레이, 연속 촬영, 자체 UI |

> **한 줄 요약**
> 사진 한 장만 필요하면 기본 카메라, 화면과 동작을 직접 설계해야 하면 커스텀 카메라를 씁니다.

---

## 2. 기본 카메라: 시스템 앱에 맡기기

기본 카메라는 **시스템 카메라 앱을 Intent로 호출**하는 방식입니다. Compose에서는 `TakePicture` 계약을 쓰면 됩니다.

### 2-1. 촬영 결과를 받을 파일과 Uri 준비

시스템 카메라 앱은 **앱 밖에서 실행**되므로, 저장할 파일 경로를 직접 넘길 수 없습니다. 대신 `FileProvider`가 만든 `content://` Uri를 넘깁니다.

```kotlin
// 캐시 폴더 안에 촬영용 파일을 만들고, FileProvider로 Uri를 얻는다
fun createCaptureUri(context: Context): Uri {
    val photoDir = File(context.cacheDir, "photos").apply { mkdirs() }
    val file = File(photoDir, "capture_${UUID.randomUUID()}.jpg")
    file.createNewFile()

    return FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider", // 매니페스트의 authorities와 같아야 한다
        file,
    )
}
```

`FileProvider`는 매니페스트와 `file_paths.xml`에 공개할 경로를 적어야 합니다.

```xml
<!-- AndroidManifest.xml -->
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

```xml
<!-- res/xml/file_paths.xml: 캐시 폴더의 photos 하위만 공개 -->
<paths>
    <cache-path name="photos" path="photos/" />
</paths>
```

### 2-2. Compose에서 촬영 요청

```kotlin
@Composable
fun BasicCameraButton(onPhotoSaved: (Uri) -> Unit) {
    val context = LocalContext.current
    var pendingUri by remember { mutableStateOf<Uri?>(null) }

    // 촬영이 끝나면 Boolean으로 성공 여부가 돌아온다. 사진 데이터는 Uri의 파일에 이미 저장돼 있다.
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.TakePicture(),
    ) { success ->
        val uri = pendingUri
        if (success && uri != null) onPhotoSaved(uri)
    }

    Button(onClick = {
        val uri = createCaptureUri(context)
        pendingUri = uri
        launcher.launch(uri) // 시스템 카메라 앱 실행, 결과를 uri에 저장
    }) {
        Text("카메라로 촬영")
    }
}
```

### 2-3. 기본 카메라의 파일 흐름

```
앱: Uri 생성 (FileProvider, content://)
 └→ 시스템 카메라 앱 실행 (Uri 전달, 읽기/쓰기 권한 임시 부여)
     └→ 카메라 앱이 Uri에 JPEG 저장
         └→ 앱에 결과(true/false) 반환
앱: 같은 Uri로 파일을 읽어 사용
```

> 이전 방식인 `onActivityResult`의 `data.extras["data"]`에 담긴 **썸네일**은 작은 이미지입니다. 원본 크기가 필요하면 `EXTRA_OUTPUT`으로 저장 위치를 지정해야 합니다.

---

## 3. 커스텀 카메라: CameraX로 직접 촬영하기

커스텀 카메라는 앱 안에서 카메라 미리보기를 그리고, 버튼을 누르면 `ImageCapture`로 파일을 저장합니다.

### 3-1. 권한 요청

```kotlin
// 카메라 권한이 없으면 요청한다. 거부되면 화면을 막는다.
val permissionLauncher = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.RequestPermission(),
) { granted ->
    hasCameraPermission = granted
}

LaunchedEffect(Unit) {
    if (!hasCameraPermission) permissionLauncher.launch(Manifest.permission.CAMERA)
}
```

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.CAMERA" />
```

### 3-2. 미리보기와 촬영 설정

```kotlin
// 미리보기 화면과 카메라 수명을 묶는다
val imageCapture = remember { ImageCapture.Builder().build() }

AndroidView(factory = { context ->
    PreviewView(context).also { previewView ->
        val future = ProcessCameraProvider.getInstance(context)
        future.addListener({
            val cameraProvider = future.get()
            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }
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

### 3-3. 촬영하고 파일로 저장하기

```kotlin
fun takePhoto(
    context: Context,
    imageCapture: ImageCapture,
    onSaved: (Uri) -> Unit,
) {
    val file = File(File(context.cacheDir, "photos").apply { mkdirs() }, "custom_${UUID.randomUUID()}.jpg")
    val options = ImageCapture.OutputFileOptions.Builder(file).build()

    imageCapture.takePicture(
        options,
        ContextCompat.getMainExecutor(context),
        object : ImageCapture.OnImageSavedCallback {
            override fun onImageSaved(results: ImageCapture.OutputFileResults) {
                // 앱이 직접 저장했으므로 파일 경로를 알고 있다
                onSaved(FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file))
            }

            override fun onError(exception: ImageCaptureException) {
                // 촬영 실패 처리
            }
        },
    )
}
```

### 3-4. 커스텀 카메라의 파일 흐름

```
앱: 미리보기 시작 (PreviewView + ImageCapture 바인딩)
앱: takePicture(OutputFileOptions(file))
 └→ CameraX가 앱 안에서 JPEG를 file에 저장
     └→ onImageSaved 콜백
앱: 같은 file을 읽거나, 필요하면 FileProvider Uri로 변환해 전달
```

> `ImageCapture`는 `OutputFileOptions`로 파일에 저장하는 방식 말고도, 메모리 버퍼로 받는 `takePicture(executor, OnImageCapturedCallback)`도 지원합니다. 실시간 분석에 유용합니다.

---

## 4. 두 방식의 코드 차이

| 항목 | 기본 카메라 | 커스텀 카메라 |
|---|---|---|
| 호출 방식 | `launcher.launch(uri)` | `imageCapture.takePicture(...)` |
| 저장 주체 | 시스템 카메라 앱 | 내 앱 (CameraX) |
| 저장 경로 지정 | `FileProvider` Uri를 전달 | `OutputFileOptions(file)` |
| 결과 확인 | `success: Boolean` | 콜백의 `onImageSaved` / `onError` |
| 권한 | 불필요 | `CAMERA` 필요 |
| 파일을 다른 앱에 전달 | 이미 `content://` Uri | `FileProvider`로 Uri 변환 필요 |
| 임시 파일 정리 | 취소 시 빈 파일 삭제 | 실패 시 파일 삭제 |

---

## 5. 파일을 다른 앱에 보낼 때 공통 주의점

- 다른 앱에 파일 경로(`file://`)를 넘기면 **Android 7.0 이상에서 예외**가 납니다. 반드시 `content://` Uri를 씁니다.
- Uri를 넘길 때는 `Intent.FLAG_GRANT_READ_URI_PERMISSION`을 함께 설정합니다.
- 촬영 후 쓰지 않는 임시 파일은 캐시에서 지웁니다. 캐시는 시스템이 정리할 수 있지만, 앱에서 직접 지우는 것이 안전합니다.

```kotlin
// 다른 앱(예: 메신저)에 사진 파일을 공유하는 예시
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpeg"
    putExtra(Intent.EXTRA_STREAM, photoUri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
startActivity(Intent.createChooser(shareIntent, "사진 공유"))
```

---

## 6. 어떤 방식을 고를까?

```
사진 한 장만 필요하고 UI는 시스템에 맡겨도 되는가?
├─ 예 → 기본 카메라 (TakePicture)
└─ 아니오
   └─ 가이드 오버레이, 연속 촬영, 자체 디자인이 필요한가? → 커스텀 카메라 (CameraX)
```

---

## 참고 자료

- [Take photos with the camera app (Android Developers)](https://developer.android.com/training/camera/photobasics)
- [Take a photo with CameraX (Android Developers)](https://developer.android.com/media/camera/camerax/take-photo)
- [Package visibility (Android Developers)](https://developer.android.com/training/package-visibility)

> 조사 시점: 2026-11-02. CameraX 예제는 공식 문서의 `OutputFileOptions`와 `OnImageSavedCallback` 흐름을 기준으로 했습니다.
