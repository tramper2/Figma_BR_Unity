# Unity Figma Bridge 아키텍처

## 시스템 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        UNITY FIGMA BRIDGE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐      ┌──────────────────┐      ┌─────────────────────┐  │
│  │   Figma API  │─────▶│  Import Process  │─────▶│  Unity UI Hierarchy │  │
│  │              │      │                  │      │                     │  │
│  │ - Document   │      │ - Download       │      │ - Canvases          │  │
│  │ - Images     │      │ - Parse          │      │ - Screens           │  │
│  │ - Fonts      │      │ - Generate       │      │ - Components        │  │
│  └──────────────┘      └──────────────────┘      └─────────────────────┘  │
│                                                             │               │
│                                                             ▼               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        RUNTIME UI LAYER                              │  │
│  │                                                                      │  │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐  │  │
│  │  │  FigmaImage     │  │ PrototypeFlow    │  │   SafeArea         │  │  │
│  │  │  (SDF Shader)   │  │ Controller       │  │                    │  │  │
│  │  └─────────────────┘  └──────────────────┘  └────────────────────┘  │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 임포트 프로세스 흐름

```
                    ┌─────────────────┐
                    │   User Click    │
                    │  "Sync Document"│
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │  Check Requirements │
                    │  - Settings      │
                    │  - TMP           │
                    │  - Access Token  │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Download Figma  │
                    │ Document        │
                    └────────┬────────┘
                             │
                             ▼
        ┌────────────────────────────────────┐
        │    Process Selected Pages          │
        │  (if "Only Import Selected Pages") │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Find Server Render Nodes          │
        │  - Vector shapes                   │
        │  - "render" named objects          │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Download All Assets               │
        │  - Server rendered images          │
        │  - Image fills                     │
        │  - Fonts (from Google Fonts)       │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Generate Font Map                 │
        │  - Match fonts in document         │
        │  - Download missing fonts          │
        │  - Find closest fallback           │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Build Figma File                  │
        │  - Create pages                    │
        │  - Build nodes recursively         │
        │  - Apply transforms                │
        │  - Add components                  │
        │  - Apply effects                   │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Save Prefabs                      │
        │  - Screens                         │
        │  - Components                      │
        │  - Pages                           │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Bind Behaviours                   │
        │  - Auto-add MonoBehaviours         │
        │  - Bind fields                     │
        │  - Bind button events              │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │  Setup Prototype Flow (if enabled) │
        │  - Register screens                │
        │  - Register sections               │
        │  - Instantiate default screen      │
        └────────────┬───────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────────┐
        │        Complete!                   │
        └────────────────────────────────────┘
```

---

## 노드 생성 프로세스

```
BuildFigmaNode(node, parent, depth, ...)
        │
        ▼
┌───────────────────────────────────┐
│  Create GameObject with RectTransform │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Apply Transform                   │
│  - Position, Size, Rotation        │
│  - Anchors, Pivot                  │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Check Node Type                  │
│                                   │
│  ┌─────────────────────────────┐  │
│  │ INSTANCE?                   │  │
│  │ → Add Component Marker      │  │
│  │ → Return (handle later)     │  │
│  └─────────────────────────────┘  │
│                                   │
│  ┌─────────────────────────────┐  │
│  │ Server Render Candidate?    │  │
│  │ → Add Image component       │  │
│  │ → Load server image         │  │
│  └─────────────────────────────┘  │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Create Unity Components          │
│  - Text → TextMeshProUGUI         │
│  - Frame/Rect/Ellipse → FigmaImage│
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Apply Properties                 │
│  - Fill colors/gradients          │
│  - Stroke settings                │
│  - Corner radius                  │
│  - Text content and style         │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Apply Effects                    │
│  - Drop shadows                   │
│  - Layer blur                     │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Apply Layout                     │
│  - Auto layout groups             │
│  - Scroll rect                    │
│  - Content size fitter            │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Build Children (Recursive)       │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Apply Prototype Functionality    │
│  - Button components              │
│  - Navigation links               │
└───────────────┬───────────────────┘
                │
                ▼
┌───────────────────────────────────┐
│  Save as Prefab (if needed)       │
│  - Screen                         │
│  - Component                      │
│  - Section                        │
└───────────────────────────────────┘
```

---

## 데이터 흐름

### Figma → Unity 데이터 매핑

| Figma 요소 | Unity 매핑 | 생성 경로 |
|-----------|-----------|----------|
| Frame (root) | Screen Prefab | `Assets/FigmaBridge/[Document]/Screens/` |
| Component | Component Prefab | `Assets/FigmaBridge/[Document]/Components/` |
| Page | Page Prefab | `Assets/FigmaBridge/[Document]/Pages/` |
| Image Fill | Sprite | `Assets/FigmaBridge/[Document]/ImageFills/` |
| Server Render | PNG Image | `Assets/FigmaBridge/[Document]/ServerRender/` |
| Font | TMP Font Asset | `Assets/FigmaBridge/Fonts/` |
| Font Material | Material | `Assets/FigmaBridge/FontMaterialPresets/` |

### 컴포넌트 인스턴스화

```
Component Definition (Figma)
        │
        ▼
┌─────────────────────────────────────┐
│  Component Prefab                   │
│  (Generated during first pass)      │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  Component Instance (Figma)         │
│  → Component Marker (temp)          │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  After all components generated     │
│  → Replace markers with prefabs     │
│  → Apply property overrides         │
└─────────────────────────────────────┘
```

---

## 클래스 의존성 그래프

```
UnityFigmaBridgeImporter (Main Entry)
        │
        ├──▶ FigmaApiUtils (API Communication)
        │       │
        │       ├──▶ FigmaApiData (Data Models)
        │       └──▶ FigmaDataUtils (Data Processing)
        │
        ├──▶ FontManager (Font Processing)
        │       ├──▶ GoogleFontLibraryManager
        │       └──▶ TextMeshProFontUtils
        │
        ├──▶ FigmaAssetGenerator (Asset Creation)
        │       ├──▶ FigmaNodeManager (Node Properties)
        │       ├──▶ NodeTransformManager (Transforms)
        │       ├──▶ EffectManager (Effects)
        │       ├──▶ FigmaLayoutManager (Layout)
        │       └──▶ ComponentManager (Components)
        │
        └──▶ PrototypeFlowManager (Prototyping)
                └──▶ BehaviourBindingManager (Auto-Binding)
```

---

## SDF 셰이더 아키텍처

### FigmaImage 컴포넌트

```
GameObject
    │
    ├──▶ RectTransform (UI positioning)
    │
    └──▶ FigmaImage : Image
            │
            ├──▶ Custom Material (Figma/FigmaImageShader)
            │       │
            │       ├──▶ Shape Keywords
            │       │   - SHAPE_RECTANGLE
            │       │   - SHAPE_ELLIPSE
            │       │   - SHAPE_STAR
            │       │
            │       ├──▶ Fill Keywords
            │       │   - SOLID
            │       │   - LINEAR_GRADIENT
            │       │   - RADIAL_GRADIENT
            │       │
            │       └──▶ Stroke Keywords
            │           - STROKE
            │
            └──▶ OnPopulateMesh (Custom UV generation)
```

### 셰이더 속성

| 속성 | 타입 | 설명 |
|-----|-----|-----|
| `_StrokeColor` | Color | 획 색상 |
| `_FillColor` | Color | 채우기 색상 |
| `_StrokeWidth` | Float | 획 두께 |
| `_CornerRadius` | Vector4 | 모서리 반경 (TL, TR, BR, BL) |
| `_GradientColors` | ColorArray | 그라데이션 색상 (최대 16개) |
| `_GradientStops` | FloatArray | 그라데이션 위치 (최대 16개) |
| `_GradientHandlePositions` | FloatArray | 그라데이션 핸들 위치 |
| `_ArcAngleRangeInnerRadius` | Vector4 | 타원 호 각도 및 내부 반경 |

---

## 프로토타입 플로우 시스템

### 화면 전환 흐름

```
User Interaction (Button Click)
        │
        ▼
FigmaPrototypeFlowButton
        │
        ├──▶ Get Target Screen ID
        │
        └──▶ PrototypeFlowController.TransitionToScreenById()
                │
                ├──▶ TransitionEffect.AnimateOut()
                │       │
                │       └──▶ Animation/Transition
                │
                ├──▶ Destroy Current Screen
                │
                ├──▶ Instantiate New Screen
                │
                ├──▶ Setup Screen Layout
                │
                └──▶ TransitionEffect.AnimateIn()
                        │
                        └──▶ Animation/Transition
```

### 섹션 시스템

```
Figma Section
        │
        ├──▶ Contains multiple screens
        │
        ├──▶ Has a default flow start
        │
        └──▶ Remembers active screen per section
                │
                └──▶ m_CurrentScreenForSection Dictionary
```

---

## 폰트 관리 시스템

### 폰트 매핑 프로세스

```
Text Node in Figma Document
        │
        ├──▶ Font Family + Weight
        │
        ▼
┌─────────────────────────────────────┐
│  Check Local Google Fonts Library   │
│  (Already downloaded?)              │
└───────────────┬─────────────────────┘
                │ No
                ▼
┌─────────────────────────────────────┐
│  Check Google Fonts Download        │
│  (Available for download?)          │
└───────────────┬─────────────────────┘
                │ Yes
                ▼
┌─────────────────────────────────────┐
│  Download Font from Google Fonts    │
│  - Download TTF                     │
│  - Import to Unity                  │
│  - Create TMP Font Asset            │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  Find Closest Font in Project       │
│  (Levenshtein distance matching)    │
└───────────────┬─────────────────────┘
                │
                ▼
┌─────────────────────────────────────┐
│  Create Font Map Entry              │
│  - Font asset reference             │
│  - Material variations (effects)    │
└─────────────────────────────────────┘
```

### 텍스트 효과 머티리얼

```
TextMeshProUGUI
        │
        └──▶ Font Material (Variant)
                │
                ├──▶ Outline Effects
                │   - Color
                │   - Thickness
                │
                └──▶ Shadow Effects
                    - Color
                    - Distance
```

---

## 확장 포인트

### 1. 커스텀 노드 타입 지원

`FigmaNodeManager.CreateUnityComponentsForNode()` 확장

### 2. 커스텀 효과

`EffectManager.ApplyAllFigmaEffectsToUnityNode()` 확장

### 3. 커스텀 레이아웃

`FigmaLayoutManager.ApplyLayoutPropertiesForNode()` 확장

### 4. 커스텀 전환 효과

`TransitionEffect` 상속 클래스 구현

### 5. 커스텀 동작 바인딩

`BehaviourBindingManager` 네임스페이스 규칙 확장

---

## 성능 고려사항

### 다운로드 최적화
- 서버 렌더링 이미지 배치 처리 (300개씩)
- 선택된 페이지만 처리 옵션

### 메모리 관리
- 임시 GameObject 적시 파괴
- 프리팹 로드 후 언로드

### 에디터 성능
- 진행률 표시로 에디터 프리징 방지
- 비동기 다운로드 처리
