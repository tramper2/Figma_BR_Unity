# Unity Figma Bridge 프로젝트 분석

## 프로젝트 개요

**Unity Figma Bridge**는 Figma 디자인 문서를 Unity 네이티브 UI로 변환하는 패키지입니다. Figma의 Components, Assets, Prototypes를 Unity로 가져와서 게임 잼, 프로토타이핑, 디자인 구현에 활용할 수 있습니다.

- **현재 지원 Unity 버전**: Unity 2021.3+ (Unity 6.3 미지원)
- **타겟 Unity 버전**: Unity 6.3
- **버전**: 1.0.9
- **라이선스**: MIT

---

## 주요 의존성

| 패키지 | 버전 | 용도 |
|--------|------|------|
| TextMeshPro | 2.0.1 | 텍스트 렌더링 |
| Newtonsoft.Json | 2.0.1-preview.1 | JSON 직렬화 |

---

## 클래스별 기능 분석

### 1. 임포터 (Importer)

#### `UnityFigmaBridgeImporter.cs`
**메인 임포터 클래스** - Figma 문서 동기화 및 Unity 자산 생성을 관리합니다.

**주요 함수:**
- `Sync()` - Figma 문서 동기화 메인 함수
- `CheckRequirements()` - 필수 조건 확인 (설정 파일, TMP, API 토큰 등)
- `DownloadFigmaDocument()` - Figma API에서 문서 다운로드
- `ImportDocument()` - 문서 임포트 메인 프로세스

**주요 기능:**
- Figma Personal Access Token 관리 (PlayerPrefs에 저장)
- 페이지 선택적 임포트
- 서버 렌더링 이미지 일괄 처리 (최대 300개 배치)
- 컴포넌트 인스턴스화 및 프리팹 생성
- 프로토타입 플로우 빌드

---

### 2. 노드 관리 (Nodes)

#### `FigmaAssetGenerator.cs`
Unity UI 자산 생성 메인 클래스입니다.

**주요 함수:**
- `BuildFigmaFile()` - 전체 Figma 파일을 Unity UI로 빌드
- `BuildFigmaPage()` - 개별 페이지 생성
- `BuildFigmaNode()` - 재귀적으로 노드 트리 생성
- `SaveFigmaScreenAsPrefab()` - 스크린을 프리팹으로 저장
- `RegisterFigmaSection()` - Figma 섹션 등록

**주요 기능:**
- 페이지, 컴포넌트, 스크린 프리팹 생성
- 마스킹 처리
- 스크롤 콘텐츠 자동 크기 조정

#### `FigmaNodeManager.cs`
Figma 노드 속성을 Unity 컴포넌트로 적용합니다.

**주요 함수:**
- `CreateUnityComponentsForNode()` - 노드에 Unity 컴포넌트 생성
- `ApplyUnityComponentPropertiesForNode()` - 노드 속성 적용
- `SetupFill()` - 채우기 설정 (단색, 그라데이션, 이미지)
- `SetupStroke()` - 획 설정
- `SetupImageFill()` - 이미지 채우기 설정
- `NodeIsSubstitution()` - 서버 렌더링 대체 노드 확인

**지원 노드 타입:**
- FRAME, RECTANGLE, ELLIPSE, STAR, COMPONENT, INSTANCE, SECTION
- TEXT (TextMeshProUGUI 사용)

---

### 3. Figma API (FigmaApi)

#### `FigmaApiData.cs`
Figma API 응답 데이터 구조를 정의합니다.

**주요 클래스:**
- `FigmaFile` - Figma 문서 루트
- `Node` - 기본 노드 클래스 (모든 노드 타입의 기본)
- `Paint` - 채우기/획 색상 및 그라데이션
- `TypeStyle` - 텍스트 스타일
- `Effect` - 그림자, 블러 효과
- `LayoutConstraint` - 레이아웃 제약 조건

**노드 타입 enum:**
```csharp
DOCUMENT, CANVAS, FRAME, GROUP, VECTOR, BOOLEAN_OPERATION,
STAR, LINE, ELLIPSE, REGULAR_POLYGON, RECTANGLE, TEXT, SLICE,
COMPONENT, COMPONENT_SET, INSTANCE, STICKY, SHAPE_WITH_TEXT,
CONNECTOR, SECTION, TABLE, TABLE_CELL, WASHI_TAPE
```

#### `FigmaApiUtils.cs`
Figma API와 통신하는 유틸리티 함수입니다.

**주요 함수:**
- `GetFigmaDocument()` - 문서 다운로드
- `GetDocumentImageFillData()` - 이미지 채우기 데이터 가져오기
- `GetFigmaServerRenderData()` - 서버 렌더링 이미지 요청
- `DownloadFiles()` - 파일 다운로드 큐 처리
- `GenerateDownloadQueue()` - 다운로드 목록 생성
- `CheckExistingAssetProperties()` - 기존 자산 속성 확인 (sRGB)

---

### 4. 런타임 UI (Runtime/UI)

#### `FigmaImage.cs`
커스텀 Image 컴포넌트로 Figma 도형을 SDF 셰이더로 렌더링합니다.

**주요 속성:**
- `Shape` - 도형 타입 (Rectangle, Ellipse, Star)
- `FillColor` - 채우기 색상
- `StrokeColor` - 획 색상
- `StrokeWidth` - 획 두께
- `CornerRadius` - 모서리 반경 (Vector4)
- `Fill` - 채우기 스타일 (Solid, LinearGradient, RadialGradient)
- `ScaleMode` - 이미지 축척 모드 (Fill, Fit, Stretch, Tile)

**주요 기능:**
- SDF 셰이더를 사용한 도형 렌더링
- 그라데이션 지원 (최대 16개 그라데이션 스톱)
- 커스텀 UV 계산 (이미지 축척 모드별)
- 추가 셰이더 채널 요구 (TexCoord1)

#### `PrototypeFlowController.cs`
프로토타입 화면 전환을 관리하는 컨트롤러입니다.

**주요 함수:**
- `RegisterFigmaScreen()` - 스크린 등록
- `RegisterFigmaSection()` - 섹션 등록
- `TransitionToScreenById()` - ID로 화면 전환
- `TransitionToScreenByName()` - 이름으로 화면 전환
- `SetCurrentScreen()` - 현재 스크린 설정

**주요 기능:**
- 섹션별 활성 스크린 추적
- 전환 효과 (TransitionEffect) 지원
- OnScreenChanged 이벤트 발생

#### 기타 런타임 클래스:
- `FigmaNodeObject` - Figma 노드 ID 참조 저장
- `FigmaComponentNodeMarker` - 컴포넌트 인스턴스 마커
- `FigmaPrototypeFlowButton` - 프로토타입 버튼
- `SafeArea` - 디바이스 Safe Area 처리
- `TransitionEffect` - 화면 전환 효과 기본 클래스
- `TransitionEffectAnimationDriven` - 애니메이션 구동 전환 효과

---

### 5. 폰트 관리 (Fonts)

#### `FontManager.cs`
폰트 매핑 및 TextMeshPro 자산 생성을 관리합니다.

**주요 함수:**
- `GenerateFontMapForDocument()` - 문서의 폰트 맵 생성
- `GetClosestFont()` - Levenshtein 거리로 가장 가까운 폰트 찾기
- `GetEffectMaterialPreset()` - 텍스트 효과 머티리얼 프리셋 생성

**주요 기능:**
- Google Fonts에서 누락된 폰트 다운로드
- 텍스트 그림자 및 외곽선 머티리얼 생성
- 폰트 이름 유사성 매칭

#### `GoogleFontLibraryManager.cs`
Google Fonts 라이브러리 관리 및 폰트 다운로드를 처리합니다.

#### `TextMeshProFontUtils.cs`
TextMeshPro 폰트 자산 생성 유틸리티입니다.

---

### 6. 프로토타입 플로우 (PrototypeFlow)

#### `BehaviourBindingManager.cs`
MonoBehaviour 자동 바인딩을 관리합니다.

**주요 함수:**
- `BindBehaviours()` - 모든 컴포넌트와 스크린에 동작 바인딩
- `BindFieldsForComponent()` - 컴포넌트 필드 자동 할당
- `GetTypeByName()` - 네임스페이스로 타입 찾기

**주요 기능:**
- 이름 일치 MonoBehaviour 자동 추가
- SerializeField 자동 할당 (하위 3depth 검색)
- `[BindFigmaButtonPress]` 속성으로 버튼 이벤트 바인딩
- "SafeArea" 이름에 SafeArea 컴포넌트 자동 추가

#### `PrototypeFlowManager.cs`
프로토타입 기능을 노드에 적용합니다.

**주요 함수:**
- `ApplyPrototypeFunctionalityToNode()` - 노드에 프로토타입 기능 적용
- `AddButtonComponent()` - 버튼 컴포넌트 추가

#### `ScreenNameCodeGenerator.cs`
스크린 이름 상수 C# 파일 생성기입니다.

---

### 7. 컴포넌트 관리 (Components)

#### `ComponentManager.cs`
Figma 컴포넌트와 인스턴스를 관리합니다.

**주요 함수:**
- `GenerateComponentAssetFromNode()` - 노드에서 컴포넌트 프리팹 생성
- `InstantiateAllComponentPrefabs()` - 모든 컴포넌트 인스턴스화
- `RemoveAllTemporaryNodeComponents()` - 임시 컴포넌트 제거

---

### 8. 기타 유틸리티

#### `NodeTransformManager.cs`
노드 변환(위치, 크기, 앵커)을 관리합니다.

#### `FigmaLayoutManager.cs`
Auto Layout 및 스크롤 처리를 담당합니다.

#### `EffectManager.cs`
Figma 효과(그림자, 블러)를 Unity에 적용합니다.

#### `FigmaDataUtils.cs`
Figma 데이터 처리 및 검색 유틸리티입니다.

#### `FigmaPaths.cs`
파일 경로 생성 및 디렉토리 관리입니다.

#### `MathUtils.cs`
수학 유틸리티 (Levenshtein 거리 등).

---

## Unity 6.3 업그레이드 TODO 작업

> **⚠️ 중요**: Unity 6.3은 Unity 2021.3에서 매우 큰 변경사항이 있는 메이저 버전 업그레이드입니다.  
> Unity 6는 2024년에 릴리스된 최신 Unity 버전으로, .NET 8, 새로운 Input System, UI Toolkit 등 많은 변화가 있습니다.

---

## 🔴 긴급 작업 (Critical)

### 1. **package.json Unity 버전 업데이트**

#### 현재 상태
```json
"unity": "2021.3",
"unityRelease": "0f1"
```

#### 필요 작업
```json
"unity": "6.0",
"unityRelease": "0f1"  // 또는 최신 Unity 6.3에 맞는 값
```

Unity 6.3은 `"unity": "6.0"` 범위에 포함됩니다.

---

### 2. **.NET 런타임 업데이트**

#### 현재 상태
- Unity 2021.3은 .NET Framework (또는 .NET Standard 2.1) 사용

#### Unity 6 변경사항
- **Unity 6는 .NET 8 런타임을 기본으로 사용**
- API 호환성 검사 필요

#### 필요 작업
```csharp
// Assembly-CSharp.csproj 또는 asmdef 설정 확인
// .NET 8 호환성 확인
```

Unity 6에서는 다음 API들이 변경되었을 수 있습니다:
- `System.Text.Json` (기본 제공되므로 Newtonsoft.Json 제거 가능)
- `async/await` 패턴 개선
- `Span<T>`, `Memory<T>` 관련 성능 향상

---

### 3. **Input System 필수 업데이트**

#### 현재 상태
```csharp
// UnityFigmaBridgeImporter.cs:286
// TODO - Allow for new input system?
existingEventSystem.gameObject.AddComponent<StandaloneInputModule>();
```

#### Unity 6 변경사항
- **새로운 Input System이 기본 시스템**
- `StandaloneInputModule`은 레거시

#### 필요 작업
```csharp
using UnityEngine.InputSystem;
using UnityEngine.InputSystem.UI;

// 기존 Input System 백업 처리
#if UNITY_INPUT_SYSTEM_AVAILABLE
    var inputSystem = existingEventSystem.GetComponent<InputSystemUIInputModule>();
    if (inputSystem == null)
    {
        inputSystem = existingEventSystem.gameObject.AddComponent<InputSystemUIInputModule>();
    }
#else
    var inputSystem = existingEventSystem.GetComponent<StandaloneInputModule>();
    if (inputSystem == null)
    {
        inputSystem = existingEventSystem.gameObject.AddComponent<StandaloneInputModule>();
    }
#endif
```

---

### 4. **TextMeshPro 4.0+ 호환성**

#### 현재 상태
```json
"com.unity.textmeshpro": "2.0.1"
```

#### Unity 6 변경사항
- **Unity 6는 TextMeshPro 4.0+ 내장**
- 셰이더 구조 변경
- Material API 변경

#### 필요 작업
```json
"com.unity.textmeshpro": "4.0.0-preview" 또는 버전 제약 제거
```

```csharp
// FontManager.cs의 TMP API 변경사항 확인
// TMP 4.0에서 셰이더 프로퍼티 ID 변경 가능성
private static readonly int s_StrokeColorPropertyID = Shader.PropertyToID("_OutlineColor");
private static readonly int s_FaceColorPropertyID = Shader.PropertyToID("_FaceColor");
```

---

### 5. **셰이더 키워드 API 변경**

#### 현재 상태
```csharp
// FigmaImage.cs:462
mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "STROKE"), m_StrokeWidth > 0);
```

#### Unity 6 변경사항
- `LocalKeyword` API가 변경되었을 수 있음
- 셰이더 키워드 최적화

#### 필요 작업
```csharp
// Unity 6 호환 방식
#if UNITY_6000_0_OR_NEWER
    mat.SetKeyword("STROKE", m_StrokeWidth > 0);
#else
    mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "STROKE"), m_StrokeWidth > 0);
#endif
```

---

### 6. **Shader Graph 및 URP/HDRP 셰이더**

#### 현재 상태
- 커스텀 `Figma/FigmaImageShader` 사용
- Built-in RP 기반일 가능성

#### Unity 6 변경사항
- **Built-in RP가 제거됨**
- **URP가 기본 렌더 파이프라인**
- 셰이더 코드 구조 변경

#### 필요 작업
```glsl
// FigmaImageShader.shader를 URP용으로 재작성 필요
// Shader Properties 변경
HLSLPROGRAM
// Built-in → URP 변경
#pragma vertex vert
#pragma fragment frag
#include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
ENDHLSL
```

---

## 🟡 중요 작업 (High Priority)

### 7. **UI Toolkit으로 에디터 마이그레이션**

#### 현재 상태
- IMGUI 기반 `UnityFigmaBridgeSettingsEditor.cs`

#### Unity 6 변경사항
- **UI Toolkit이 에디터 UI의 권장 방식**
- IMGUI는 레거시

#### 필요 작업
```csharp
// UnityFigmaBridgeSettingsEditor.cs를 UI Toolkit으로 재작성
using UnityEditor.UIElements;

public class UnityFigmaBridgeSettingsEditor : UnityEditor.Editor
{
    private VisualElement rootElement;
    
    public override VisualElement CreateInspectorGUI()
    {
        rootElement = new VisualElement();
        
        // UXML 파일로드 또는 코드 기반 UI 생성
        var fileIdField = new TextField("Figma Document URL");
        rootElement.Add(fileIdField);
        
        return rootElement;
    }
}
```

---

### 8. **PrefabUtility API 변경**

#### 현재 상태
```csharp
PrefabUtility.SaveAsPrefabAssetAndConnect()
PrefabUtility.LoadPrefabContents()
PrefabUtility.UnloadPrefabContents()
```

#### Unity 6 변경사항
- Prefab API가 개선됨
- `PrefabStage` 관련 API 변경

#### 필요 작업
```csharp
// Unity 6에서 동작 확인
// 아래 API가 변경되었을 수 있음:
- PrefabUtility.SaveAsPrefabAssetAndConnect → PrefabUtility.SaveAsPrefabAsset
- PrefabUtility.LoadPrefabContents → PrefabUtility.LoadPrefabContents 확인 필요
```

---

### 9. **UnityWebRequest 비동기 패턴**

#### 현재 상태
- `UnityWebRequestAwaiter.cs` 사용

#### Unity 6 변경사항
- `SendWebRequest()`가 `AsyncOperation`을 직접 반환
- `await webRequest.SendWebRequest()` 패턴 지원

#### 필요 작업
```csharp
// Unity 6 간단한 비동기 패턴
using UnityWebRequest = UnityEngine.Networking.UnityWebRequest;

public static async Task<FigmaFile> GetFigmaDocument(string fileId, string accessToken)
{
    using var request = UnityWebRequest.Get($"https://api.figma.com/v1/files/{fileId}");
    request.SetRequestHeader("X-Figma-Token", accessToken);
    
    await request.SendWebRequest();  // Unity 6에서 직접 await 가능
    
    if (request.result == UnityWebRequest.Result.Success)
    {
        return JsonConvert.DeserializeObject<FigmaFile>(request.downloadHandler.text);
    }
    throw new Exception(request.error);
}
```

---

### 10. **Color Space 및 그래픽스 API**

#### 현재 상태
```csharp
// FigmaImage.cs:416
// If the player settings is set to linear, we need to convert the vertex color to gamma space
Color32 color32 = color;
```

#### Unity 6 변경사항
- 색상 공간 처리가 개선됨
- `Graphics.activeColorSpace` API 변경

#### 필요 작업
```csharp
// Unity 6 색상 공간 처리
#if UNITY_6000_0_OR_NEWER
    // Unity 6에서 색상 변환이 자동으로 처리되는지 확인
    if (QualitySettings.activeColorSpace == ColorSpace.Linear)
    {
        color32 = Graphics.ConvertToLinearSpace(color);
    }
#else
    // 기존 방식 유지
#endif
```

---

## 🟢 권장 작업 (Recommended)

### 11. **Newtonsoft.Json 제거**

#### 현재 상태
```json
"com.unity.nuget.newtonsoft-json": "2.0.1-preview.1"
```

#### Unity 6 대안
- **.NET 8의 `System.Text.Json` 사용** (기본 제공)

#### 필요 작업
```csharp
// FigmaApiUtils.cs
using System.Text.Json;

// Newtonsoft.Json → System.Text.Json 변환
var figmaFile = JsonSerializer.Deserialize<FigmaFile>(jsonText, new JsonSerializerOptions
{
    PropertyNameCaseInsensitive = true,
    Converters = { new JsonStringEnumConverter() }
});
```

---

### 12. **Localization API**

#### 현재 상태
- 하드코딩된 영어 문자열

#### Unity 6 기능
- **Localization 패키지 개선**

#### 필요 작업
```csharp
// 다국어 지원 고려
using UnityEngine.Localization;
using UnityEngine.Localization.Settings;

private static string Localize(string key)
{
    return LocalizationSettings.StringDatabase.GetLocalizedStringAsync("UI", key).Result;
}
```

---

### 13. **Burst Compiler 활용**

#### Unity 6 기능
- **Burst Compiler 성능 향상**

#### 적용 가능 영역
- `MathUtils.LeventshteinStringDistance()` 등 수학 계산
- 셰이더 관련 계산

```csharp
using Unity.Burst;
using Unity.Mathematics;

[BurstCompile]
public static class MathUtils
{
    [BurstCompile]
    public static int LevenshteinDistance(ReadOnlySpan<char> a, ReadOnlySpan<char> b)
    {
        // Burst 컴파일로 최적화
    }
}
```

---

### 14. **Entity Component System (ECS) 고려**

#### Unity 6 기능
- **ECS가 더욱 안정화됨**

#### 고려사항
- UI 시스템은 GameObject 기반이 여전히 권장
- 하지만 데이터 처리는 DOTS 패턴 고려 가능

---

### 15. **RenderPipelineAsset API**

#### 현재 상태
- 특정 RP 가정 없이 작성

#### Unity 6 변경사항
- URP가 기본이지만 HDRP 지원도 필요

#### 필요 작업
```csharp
// 현재 렌더 파이프라인 확인
#if UNITY_6000_0_OR_NEWER
    var renderPipelineAsset = GraphicsSettings.currentRenderPipeline;
    if (renderPipelineAsset is UniversalRenderPipelineAsset)
    {
        // URP 관련 처리
    }
    else if (renderPipelineAsset is HighDefinitionRenderPipelineAsset)
    {
        // HDRP 관련 처리
    }
#endif
```

---

## 📋 API 변경사항 체크리스트

### 에디터 API
- [ ] `AssetDatabase` - `ImportAsset`, `Refresh`, `LoadAssetAtPath` 등
- [ ] `PrefabUtility` - `SaveAsPrefabAsset`, `LoadPrefabContents` 등
- [ ] `EditorUtility` - `DisplayProgressBar`, `ClearProgressBar`
- [ ] `EditorGUILayout` - UI Toolkit으로 대체 권장
- [ ] `EditorGUI` - UI Toolkit으로 대체 권장
- [ ] `Selection` - `activeObject` 변경사항 확인

### 런타임 API
- [ ] `GraphicRaycaster` - Input System 연동 확인
- [ ] `Canvas` - `additionalShaderChannels` API 확인
- [ ] `RectTransform` - 앵커/피벗 API 변경사항
- [ ] `Color` - 색상 공간 처리 변경사항
- [ ] `Material` - `SetKeyword`, `SetVector` 등 API 확인

### 네트워크 API
- [ ] `UnityWebRequest` - `SendWebRequest` await 패턴
- [ ] `UnityWebRequestAsyncOperation` - 제거될 수 있음
- [ ] `DownloadHandler` - 버퍼 처리 API 변경사항

---

## 🔄 마이그레이션 순서

### 1단계: 기본 호환성 (1-2일)
1. `package.json` Unity 버전 업데이트
2. .NET 런타임 설정 확인
3. 컴파일 에러 해결
4. 기본 기능 테스트

### 2단계: 필수 API 변경 (3-5일)
1. Input System 업데이트
2. TextMeshPro 4.0 호환성
3. 셰이더 URP 변환
4. 셰이더 키워드 API 수정

### 3단계: 에디터 개선 (2-3일)
1. UI Toolkit 마이그레이션
2. Prefab API 확인
3. 에디터 스크립트 수정

### 4단계: 성능 및 최적화 (2-3일)
1. UnityWebRequest 비동기 패턴 개선
2. Newtonsoft.Json 제거 (System.Text.Json 전환)
3. Burst Compiler 적용 검토

### 5단계: 테스트 및 검증 (3-5일)
1. URP/HDRP 호환성 테스트
2. 색상 공간 테스트
3. 레거시 Input System 백업 테스트
4. 전체 기능 회귀 테스트

**총 예상 소요 시간: 2-3주**

---

## 비지원 기능 (README 기준)

다음 기능은 현재 구현되지 않았습니다:

1. 사용자 정의 자산 저장 위치
2. 이미지 트윽 (노출, 대비)
3. 그림자 블러
4. 타원 sweep angles 및 fill ratio
5. 대부분의 효과 (내부 그림자, 레이어 블러, 배경 블러)
6. 단일 객체에 여러 채우기
7. 단색 이외의 획 스타일
8. 도형의 획 위치 (외부/중앙)
9. 텍스트의 획 위치 (내부/중앙)
10. 동적 장치 폰트 생성
11. 비디오 채우기
12. 별 모양 5개 꼭짓점 및 기본 반경 제한
13. 다각형 도형
14. 불리언 연산
15. 선/화살표
16. 일관된 UUID
17. "Scale" 제약 조건 지원

---

## 프로젝트 구조

```
UnityFigmaBridge/
├── Editor/
│   ├── Components/          # 컴포넌트 관리
│   ├── FigmaApi/            # Figma API 데이터 및 유틸리티
│   ├── Fonts/               # 폰트 관리
│   ├── Nodes/               # 노드 생성 및 관리
│   ├── PrototypeFlow/       # 프로토타입 플로우
│   ├── Settings/            # 설정 및 에디터
│   ├── UnityComponentEditors/ # 커스텀 에디터
│   └── Utils/               # 유틸리티
├── Runtime/
│   └── UI/                  # 런타임 UI 컴포넌트
└── Assets/                  # 셰이더, 프리팹 등
```

---

## 결론

Unity Figma Bridge는 Figma 디자인을 Unity로 가져오는 강력한 도구입니다. **Unity 6.3으로 업그레이드 시 매우 큰 변경사항**이 있으며, 다음 사항들을 특히 주의해야 합니다:

### 🔴 가장 중요한 변경사항

1. **.NET 8 런타임** - 코드 호환성 검사 필요
2. **Built-in RP 제거** - 셰이더를 URP용으로 재작성 필수
3. **새 Input System 기본** - Input SystemUIInputModule으로 업데이트 필요
4. **TextMeshPro 4.0+** - Material 및 셰이더 프로퍼티 API 변경

### 예상 작업량

- **긴급 작업**: 4-5일 (기본 호환성, Input System, TMP, 셰이더)
- **중요 작업**: 3-4일 (UI Toolkit, Prefab API, 비동기 패턴)
- **권장 작업**: 2-3일 (Newtonsoft.Json 제거, 최적화)
- **테스트**: 3-5일 (전체 기능 테스트)

**총 예상 소요 시간: 2-3주**

### 추가 고려사항

- Unity 6.3은 2024년 기준 매우 최신 버전으로, 관련 자료가 제한적일 수 있음
- URP 셰이더 작성 경험이 필요함
- Input System에 대한 이해가 필요함
- UI Toolkit에 대한 학습이 필요함
