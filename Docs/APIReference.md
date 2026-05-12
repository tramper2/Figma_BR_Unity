# Unity Figma Bridge API 레퍼런스

## 메인 API

### UnityFigmaBridgeImporter

메인 임포터 클래스 - Figma 문서 동기화를 담당합니다.

```csharp
public static class UnityFigmaBridgeImporter
{
    // 메뉴 항목
    [MenuItem("Figma Bridge/Sync Document")]
    static void Sync();
    
    [MenuItem("Figma Bridge/Select Settings File")]
    static void SelectSettings();
    
    [MenuItem("Figma Bridge/Set Personal Access Token")]
    static void SetPersonalAccessToken();
    
    // 주요 메서드
    public static bool CheckRequirements();
    public static async Task<FigmaFile> DownloadFigmaDocument(string fileId);
    private static async Task ImportDocument(string fileId, FigmaFile figmaFile, List<Node> downloadPageNodeList);
}
```

**사용 예시:**
```csharp
// 프로그래밍 방식 임포트
if (UnityFigmaBridgeImporter.CheckRequirements())
{
    var figmaFile = await UnityFigmaBridgeImporter.DownloadFigmaDocument("fileId");
    // 처리 계속...
}
```

---

## 런타임 UI API

### PrototypeFlowController

프로토타입 화면 전환을 관리합니다.

```csharp
public class PrototypeFlowController : MonoBehaviour
{
    // 이벤트
    public UnityEvent<string, GameObject> OnScreenChanged;
    
    // 속성
    public RectTransform ScreenParentTransform { get; set; }
    public TransitionEffect TransitionEffect { get; set; }
    public GameObject CurrentScreenInstance { get; }
    public string PrototypeFlowInitialScreenId { get; set; }
    public FigmaFlowScreen StartFlowScreen { get; }
    
    // 메서드
    public void RegisterFigmaScreen(FigmaFlowScreen flowScreen);
    public void RegisterFigmaSection(FigmaSection figmaSection);
    public void ClearFigmaScreens();
    public void TransitionToScreenById(string screenNodeID);
    public void TransitionToScreenByName(string screenName);
    public void SetCurrentScreenByNodeId(string nodeId);
    public void SetScreenByName(string screenName);
}
```

**사용 예시:**
```csharp
// 화면 전환
PrototypeFlowController controller = GetComponent<PrototypeFlowController>();
controller.TransitionToScreenByName("MainMenu");

// 이벤트 리스닝
controller.OnScreenChanged.AddListener((screenName, screenObject) => {
    Debug.Log($"Screen changed to: {screenName}");
});
```

### FigmaFlowScreen

화면 데이터를 저장합니다.

```csharp
[System.Serializable]
public class FigmaFlowScreen
{
    public string FigmaNodeId;           // Figma 노드 ID
    public GameObject FigmaScreenPrefab; // 스크린 프리팹
    public string FigmaScreenName;       // 스크린 이름
    public string ParentSectionNodeId;   // 부모 섹션 ID (선택사항)
}
```

### FigmaSection

섹션 데이터를 저장합니다.

```csharp
[System.Serializable]
public class FigmaSection
{
    public string FigmaNodeId;                      // 섹션 노드 ID
    public string FigmaPrototypeFlowStartNodeId;    // 플로우 시작 노드 ID
    public string FigmaPrototypeFlowStartNodeName;  // 플로우 시작 노드 이름
    public string FigmaNodeName;                    // 섹션 이름
}
```

### FigmaImage

Figma 도형을 렌더링하는 커스텀 Image 컴포넌트입니다.

```csharp
public class FigmaImage : Image
{
    // 열거형
    public enum ShapeType { Rectangle, Ellipse, Star }
    public enum FillStyle { Solid, LinearGradient, RadialGradient }
    public enum ImageScaleMode { Fill, Fit, Stretch, Tile }
    public enum StrokePositionType { Inside, Outside, Center }
    
    // 속성
    public ShapeType Shape { get; set; }
    public Color StrokeColor { get; set; }
    public Color FillColor { get; set; }
    public float StrokeWidth { get; set; }
    public Vector4 CornerRadius { get; set; }
    public StrokePositionType StrokePosition { get; set; }
    public FillStyle Fill { get; set; }
    public ImageScaleMode ScaleMode { get; set; }
    public Gradient FillGradient { get; set; }
    public Vector2[] GradientHandlePositions { get; set; }
    public Vector3[] ImageTransform { get; set; }
    public float ImageScaleFactor { get; set; }
    public float EllipseInnerRadius { get; set; }
    public Vector2 EllipseArcAngleRange { get; set; }
}
```

**사용 예시:**
```csharp
// 런타임에 모양 변경
FigmaImage figmaImage = GetComponent<FigmaImage>();
figmaImage.Shape = FigmaImage.ShapeType.Ellipse;
figmaImage.FillColor = Color.red;
figmaImage.CornerRadius = new Vector4(10, 10, 10, 10);
```

### SafeArea

디바이스 Safe Area를 처리합니다.

```csharp
public class SafeArea : MonoBehaviour
{
    // RectTransform을 Safe Area에 맞춰 조정
    // ApplySafeArea() 메서드는 자동으로 호출됨
}
```

### TransitionEffect

화면 전환 효과의 기본 클래스입니다.

```csharp
public class TransitionEffect : MonoBehaviour
{
    public void AnimateIn(Action onComplete = null);
    public void AnimateOut(Action onComplete = null);
}
```

### TransitionEffectAnimationDriven

애니메이션 기반 전환 효과 구현입니다.

```csharp
public class TransitionEffectAnimationDriven : TransitionEffect
{
    public Animator animator;
    
    public override void AnimateIn(Action onComplete = null);
    public override void AnimateOut(Action onComplete = null);
}
```

### FigmaNodeObject

Figma 노드 ID 참조를 저장합니다.

```csharp
public class FigmaNodeObject : MonoBehaviour
{
    public string NodeId;  // Figma 노드 ID
}
```

### FigmaComponentNodeMarker

컴포넌트 인스턴스를 위한 임시 마커입니다.

```csharp
public class FigmaComponentNodeMarker : MonoBehaviour
{
    public void Initialise(string nodeId, string parentNodeId, string componentId);
}
```

---

## 에디터 API

### UnityFigmaBridgeSettings

설정 데이터 클래스입니다.

```csharp
public class UnityFigmaBridgeSettings : ScriptableObject
{
    // Figma 설정
    public string FileId;
    
    // 임포트 설정
    public bool BuildPrototypeFlow = true;
    public bool OnlyImportSelectedPages = false;
    public bool GenerateNodesMarkedForExport = false;
    public bool EnableGoogleFontsDownloads = true;
    public bool EnableAutoLayout = false;
    public bool CreateScreenNameCSharpFile = false;
    
    // 런타임 설정
    public string RunTimeAssetsScenePath;
    
    // 렌더링 설정
    public float ServerRenderImageScale = 3f;
    
    // 바인딩 설정
    public string ScreenBindingNamespace = "";
    
    // 페이지 선택
    public List<PageData> PageDataList;
    
    // 메서드
    public void RefreshForUpdatedPages(FigmaFile figmaFile);
}
```

### FontManager

폰트 관리를 담당합니다.

```csharp
public static class FontManager
{
    public static async Task<FigmaFontMap> GenerateFontMapForDocument(FigmaFile figmaFile, bool enableGoogleFontsDownload);
    public static Material GetEffectMaterialPreset(FigmaFontMapEntry fontMapEntry, bool shadow, Color shadowColor, Vector2 shadowDistance, bool outline, Color outlineColor, float outlineThickness);
}
```

### BehaviourBindingManager

동작 자동 바인딩을 관리합니다.

```csharp
public static class BehaviourBindingManager
{
    public static void BindBehaviours(FigmaImportProcessData figmaImportProcessData);
    public static void BindFieldsForComponent(GameObject gameObject, Component component);
}
```

### BindFigmaButtonPress 속성

버튼 클릭 이벤트를 자동으로 바인딩하는 속성입니다.

```csharp
public class BindFigmaButtonPress : Attribute
{
    public string TargetButtonName;
    
    public BindFigmaButtonPress(string targetButtonName);
}
```

**사용 예시:**
```csharp
public class MainMenuScreen : MonoBehaviour
{
    // 필드 자동 바인딩 (이름 일치)
    public TextMeshProUGUI TitleText;
    public Button PlayButton;
    
    // 버튼 이벤트 자동 바인딩
    [BindFigmaButtonPress("PlayButton")]
    void OnPlayClicked()
    {
        Debug.Log("Play button clicked!");
    }
    
    [BindFigmaButtonPress("SettingsButton")]
    void OnSettingsClicked()
    {
        // 설정 화면으로 전환
    }
}
```

---

## Figma API 데이터 모델

### FigmaFile

```csharp
public class FigmaFile
{
    public Node document;
    public Dictionary<string, Component> components;
    public int schemaVersion;
    public Dictionary<string, Style> styles;
    public string name;
    public string lastModified;
    public string version;
}
```

### Node (주요 속성)

```csharp
public class Node
{
    public string id;
    public string name;
    public NodeType type;
    public bool visible;
    public Node[] children;
    
    // 프레임 속성
    public Paint[] fills;
    public Paint[] strokes;
    public float strokeWeight;
    public float cornerRadius;
    public float[] rectangleCornerRadii;
    public LayoutConstraint constraints;
    public float opacity;
    public Rectangle absoluteBoundingBox;
    public Vector size;
    public bool clipsContent;
    
    // 오토 레이아웃
    public LayoutMode layoutMode;
    public PrimaryAxisSizingMode primaryAxisSizingMode;
    public CounterAxisSizingMode counterAxisSizingMode;
    public PrimaryAxisAlignItems primaryAxisAlignItems;
    public CounterAxisAlignItems counterAxisAlignItems;
    public float paddingLeft, paddingRight, paddingTop, paddingBottom;
    public float itemSpacing;
    public OverflowDirection overflowDirection;
    
    // 텍스트
    public string characters;
    public TypeStyle style;
    
    // 효과
    public Effect[] effects;
    public bool isMask;
    
    // 인스턴스
    public string componentId;
    
    // 프로토타입
    public string transitionNodeID;
    public float transitionDuration;
    public EasingType transitionEasing;
    
    // 타원
    public ArcData arcData;
}
```

### Paint (채우기/획)

```csharp
public class Paint
{
    public PaintType type;  // SOLID, GRADIENT_LINEAR, GRADIENT_RADIAL, IMAGE, etc.
    public bool visible;
    public float opacity;
    public Color color;
    public BlendMode blendMode;
    public Vector[] gradientHandlePositions;
    public ColorStop[] gradientStops;
    public ScaleMode scaleMode;
    public string imageRef;
    public float scalingFactor;
}
```

### TypeStyle (텍스트 스타일)

```csharp
public class TypeStyle
{
    public string fontFamily;
    public int fontWeight;
    public float fontSize;
    public bool italic;
    public TextCase textCase;
    public TextDecoration textDecoration;
    public TextAutoResize textAutoResize;
    public TextAlignHorizontal textAlignHorizontal;
    public TextAlignVertical textAlignVertical;
    public float letterSpacing;
    public float lineHeightPx;
    public Paint[] fills;
}
```

### Effect (효과)

```csharp
public class Effect
{
    public EffectType type;  // INNER_SHADOW, DROP_SHADOW, LAYER_BLUR, BACKGROUND_BLUR
    public bool visible;
    public float radius;
    public Color color;
    public BlendMode blendMode;
    public Vector offset;
    public float spread;
}
```

### LayoutConstraint (레이아웃 제약)

```csharp
public class LayoutConstraint
{
    public VerticalLayoutConstraint vertical;   // TOP, BOTTOM, CENTER, TOP_BOTTOM, SCALE
    public HorizontalLayoutConstraint horizontal; // LEFT, RIGHT, CENTER, LEFT_RIGHT, SCALE
}
```

---

## 유틸리티 API

### FigmaDataUtils

Figma 데이터 처리 유틸리티입니다.

```csharp
public static class FigmaDataUtils
{
    public static void FindAllNodesOfType(Node rootNode, NodeType nodeType, List<Node> foundNodes, int currentDepth);
    public static List<Node> GetPageNodes(FigmaFile figmaFile);
    public static bool IsScreenNode(Node node, Node parentNode);
    public static Node GetFigmaNodeInChildren(Node node, string nodeId);
    public static string FindPrototypeFlowStartScreenId(FigmaFile figmaFile);
    public static List<string> GetAllPrototypeFlowStartingPoints(FigmaFile figmaFile);
    public static Color GetUnityFillColor(Paint paint);
    public static Gradient ToUnityGradient(Paint paint);
    public static Dictionary<string, Node> BuildNodeLookupDictionary(FigmaFile figmaFile);
}
```

### FigmaPaths

파일 경로 유틸리티입니다.

```csharp
public static class FigmaPaths
{
    public static void CreateRequiredDirectories();
    public static string GetPathForImageFill(string imageRef);
    public static string GetPathForScreenPrefab(Node node, int count);
    public static string GetPathForComponentPrefab(Node node, int count);
    public static string GetPathForPagePrefab(Node node, int count);
    public static string GetPathForServerRenderedImage(string nodeId, List<FigmaServerRenderNode> serverRenderNodes);
    public static string GetFileNameForNode(Node node, int count);
}
```

### MathUtils

수학 유틸리티입니다.

```csharp
public static class MathUtils
{
    public static int LeventshteinStringDistance(string a, string b);
}
```

---

## 노드 타입 열거형

```csharp
public enum NodeType
{
    DOCUMENT,
    CANVAS,
    FRAME,
    GROUP,
    VECTOR,
    BOOLEAN_OPERATION,
    STAR,
    LINE,
    ELLIPSE,
    REGULAR_POLYGON,
    RECTANGLE,
    TEXT,
    SLICE,
    COMPONENT,
    COMPONENT_SET,
    INSTANCE,
    STICKY,
    SHAPE_WITH_TEXT,
    CONNECTOR,
    SECTION,
    TABLE,
    TABLE_CELL,
    WASHI_TAPE
}
```

---

## 주요 상수

### UnityFigmaBridgeImporter

```csharp
public const string PROGRESS_BOX_TITLE = "Importing Figma Document";
private const int MAX_SERVER_RENDER_IMAGE_BATCH_SIZE = 300;
private const string FIGMA_PERSONAL_ACCESS_TOKEN_PREF_KEY = "FIGMA_PERSONAL_ACCESS_TOKEN";
```

### BehaviourBindingManager

```csharp
private const int MAX_SEARCH_DEPTH_FOR_TRANSFORMS = 3;
```

### FigmaImage

```csharp
private const int MAX_GRADIENT_STOPS = 16;
private const string FIGMA_SHADER_NAME = "Figma/FigmaImageShader";
```

---

## 이벤트 흐름

### Sync Document 이벤트 흐름

```
1. User clicks "Sync Document"
   ↓
2. CheckRequirements()
   - Validates settings
   - Checks TMP installation
   - Verifies API token
   ↓
3. DownloadFigmaDocument()
   - Fetches from Figma API
   ↓
4. Process selected pages (if enabled)
   ↓
5. Find server render nodes
   ↓
6. Download all assets
   - Server render images
   - Image fills
   ↓
7. GenerateFontMapForDocument()
   - Match fonts
   - Download from Google Fonts
   ↓
8. BuildFigmaFile()
   - Create pages
   - Build nodes recursively
   - Apply components
   - Apply effects
   ↓
9. Save prefabs
   ↓
10. BindBehaviours()
    - Auto-add MonoBehaviours
    - Bind fields
    - Bind button events
    ↓
11. Setup prototype flow (if enabled)
    ↓
12. Complete!
```

---

## 확장 포인트

### 커스텀 노드 처리

`FigmaNodeManager` 확장:

```csharp
// CreateUnityComponentsForNode()에 새 노드 타입 추가
case NodeType.CUSTOM_TYPE:
    nodeGameObject.AddComponent<CustomComponent>();
    break;
```

### 커스텀 효과

`EffectManager` 확장:

```csharp
// ApplyAllFigmaEffectsToUnityNode()에 새 효과 추가
if (effect.type == Effect.EffectType.CUSTOM_EFFECT)
{
    ApplyCustomEffect(nodeGameObject, effect);
}
```

### 커스텀 전환 효과

```csharp
public class CustomTransitionEffect : TransitionEffect
{
    public override void AnimateIn(Action onComplete = null)
    {
        // 커스텀 애니메이션 구현
    }
    
    public override void AnimateOut(Action onComplete = null)
    {
        // 커스텀 애니메이션 구현
    }
}
```
