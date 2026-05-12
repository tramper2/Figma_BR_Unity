# Unity Figma Bridge - Unity 6.3 마이그레이션 가이드

## 개요

이 가이드는 Unity Figma Bridge 패키지를 Unity 6.3으로 마이그레이션하는 방법을 설명합니다. Unity 6는 이전 버전(2021.3)과 비교해서 매우 큰 변경사항이 있는 메이저 버전입니다.

**주요 변경사항:**
- .NET Framework → .NET 8
- Built-in RP 제거, URP 기본
- 새 Input System 기본
- TextMeshPro 4.0+
- UI Toolkit 권장
- 셰이더 API 변경

---

## 1단계: 프로젝트 설정

### 1.1 package.json 수정

```json
{
  "name": "com.simonoliver.unityfigma",
  "displayName": "Unity Figma Bridge",
  "unity": "6.0",
  "unityRelease": "0f1",
  "version": "2.0.0",  // 메이저 버전 업데이트
  "dependencies": {
    "com.unity.ugui": "2.0.0",
    "com.unity.textmeshpro": "4.0.0-pre.6"
  }
}
```

**변경사항:**
- Unity 버전: `2021.3` → `6.0`
- 패키지 버전: `1.0.9` → `2.0.0` (Breaking changes)
- Newtonsoft.Json 의존성 제거 (System.Text.Json 사용)
- TextMeshPro 버전 업데이트

### 1.2 asmdef 파일 수정

```csharp
// UnityFigmaBridgeRuntime.asmdef
{
    "name": "UnityFigmaBridgeRuntime",
    "references": [],
    "includePlatforms": [],
    "excludePlatforms": [],
    "allowUnsafeCode": false,
    "overrideReferences": true,
    "precompiledReferences": [],
    "autoReferenced": true,
    "defineConstraints": [],
    "versionDefines": [],
    "noEngineReferences": false
}
```

---

## 2단계: 셰이더 URP 변환

### 2.1 FigmaImageShader.shader 재작성

현재 Built-in RP 기반 셰이더를 URP용으로 재작성해야 합니다.

```glsl
Shader "Figma/FigmaImageShader"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Tint", Color) = (1,1,1,1)
        
        // Stroke properties
        _StrokeColor ("Stroke Color", Color) = (1,1,1,1)
        _StrokeWidth ("Stroke Width", Range(0, 1)) = 0
        
        // Fill properties
        _FillColor ("Fill Color", Color) = (1,1,1,1)
        
        // Corner radius
        _CornerRadius ("Corner Radius", Vector) = (0,0,0,0)
        
        // Gradient properties
        _GradientColors ("Gradient Colors", ColorArray) = (1,1,1,1)
        _GradientStops ("Gradient Stops", FloatArray) = (0,1)
        _GradientNumStops ("Gradient Num Stops", Float) = 2
        _GradientHandlePositions ("Gradient Handle Positions", VectorArray) = (0,0,0)
        
        // Arc properties
        _ArcAngleRangeInnerRadius ("Arc Angle Range / Inner Radius", Vector) = (0, 6.28, 1, 0)
    }
    
    SubShader
    {
        Tags
        {
            "RenderType"="Transparent"
            "Queue"="Transparent"
            "RenderPipeline" = "UniversalPipeline"
        }
        
        Blend SrcAlpha OneMinusSrcAlpha
        ZWrite Off
        Cull Off
        
        Pass
        {
            Name "FigmaImagePass"
            
            HLSLPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            
            // Unity 6 URP 포함
            #include "Packages/com.unity.render-pipelines.universal/ShaderLibrary/Core.hlsl"
            
            // 키워드
            #pragma multi_compile_local _ STROKE
            #pragma multi_compile_local _ LINEAR_GRADIENT RADIAL_GRADIENT
            #pragma multi_compile_local _ SHAPE_RECTANGLE SHAPE_ELLIPSE SHAPE_STAR
            #pragma multi_compile_local _ ARC_ANGLE_RANGE
            #pragma multi_compile_local _ CLAMP_TEXTURE
            
            struct Attributes
            {
                float4 positionOS : POSITION;
                float4 uv0 : TEXCOORD0;
                float4 color : COLOR;
                float4 uv1 : TEXCOORD1;  // Size in UV1
            };
            
            struct Varyings
            {
                float4 positionCS : SV_POSITION;
                float4 uv0 : TEXCOORD0;
                float4 color : COLOR;
                float4 sizeData : TEXCOORD1;  // Size in TEXCOORD1
            };
            
            TEXTURE2D(_MainTex);
            SAMPLER(sampler_MainTex);
            
            CBUFFER_START(UnityPerMaterial)
                float4 _Color;
                float4 _StrokeColor;
                float4 _FillColor;
                float4 _CornerRadius;
                float _StrokeWidth;
                float4 _GradientColors[16];
                float _GradientStops[16];
                float _GradientNumStops;
                float3 _GradientHandlePositions[3];
                float4 _ArcAngleRangeInnerRadius;
            CBUFFER_END
            
            // SDF Functions (Inigo Quilez)
            float sdBox(float2 p, float2 b)
            {
                float2 d = abs(p) - b;
                return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
            }
            
            float sdEllipse(float2 p, float2 ab)
            {
                p = abs(p);
                float t = ((ab.x * ab.x - ab.y * ab.y) * p.x * p.x + ab.y * ab.y * ab.y * ab.y) /
                         (2.0 * ab.y * ab.y * (ab.x * ab.x - ab.y * ab.y) * p.x +
                          2.0 * ab.x * ab.x * ab.y * ab.y);
                t = clamp(t, 0.0, 1.0);
                float2 s = float2(ab.x * ab.x * p.x - ab.x * ab.y * ab.y * t,
                                   -ab.y * ab.x * p.x + ab.x * ab.x * ab.y * t);
                return (length(s) - ab.x * ab.y) / sqrt(ab.x * ab.x * (1.0 - t) * (1.0 - t) +
                       ab.y * ab.y * t * t);
            }
            
            Varyings vert(Attributes input)
            {
                Varyings output;
                
                // URP TransformObjectToHClip
                output.positionCS = TransformObjectToHClip(input.positionOS.xyz);
                output.uv0 = input.uv0;
                output.color = input.color * _Color;
                output.sizeData = input.uv1;  // Size passed through UV1
                
                return output;
            }
            
            half4 frag(Varyings input) : SV_Target
            {
                float2 uv = input.uv0.xy;
                float2 size = input.sizeData.zw;
                float2 center = float2(0.5, 0.5);
                
                // Convert to signed distance field
                float2 p = (uv - center) * size;
                
                float sdf = 0;
                
                #ifdef SHAPE_RECTANGLE
                    float2 halfSize = size * 0.5;
                    #ifdef ARC_ANGLE_RANGE
                        // Ellipse with arc
                        float2 ab = halfSize.xy * float2(1.0, _ArcAngleRangeInnerRadius.z);
                        sdf = sdEllipse(p, ab);
                        
                        // Apply arc range
                        float angle = atan2(p.y, p.x);
                        float angleStart = _ArcAngleRangeInnerRadius.x;
                        float angleEnd = _ArcAngleRangeInnerRadius.y;
                        if (angle < angleStart || angle > angleEnd) sdf = -100;
                    #else
                        sdf = sdBox(p - float2(0, 0), halfSize - _CornerRadius.xy);
                        float2 corner = abs(p) - halfSize + _CornerRadius.xy;
                        sdf = min(sdf, length(max(corner, 0.0)) - min(min(_CornerRadius.x, _CornerRadius.y), 0.0));
                    #endif
                #elif defined(SHAPE_ELLIPSE)
                    sdf = length(p / size * 2.0) - 1.0;
                #elif defined(SHAPE_STAR)
                    // Simplified star SDF
                    float2 starP = abs(p);
                    float starSDF = -100.0;  // Placeholder
                    sdf = starSDF;
                #endif
                
                // Calculate alpha based on SDF
                float alpha = 1.0 - smoothstep(0.0, 1.0, sdf);
                
                // Stroke
                #ifdef STROKE
                    float strokeSDF = sdf + _StrokeWidth;
                    float strokeAlpha = 1.0 - smoothstep(0.0, 1.0, strokeSDF);
                    float4 finalColor = lerp(_FillColor * input.color, _StrokeColor * input.color, strokeAlpha - alpha);
                    alpha = max(alpha, strokeAlpha);
                #else
                    float4 finalColor = _FillColor * input.color;
                #endif
                
                // Gradient
                #ifdef LINEAR_GRADIENT
                    // Sample gradient based on position
                    float gradientPos = uv.x;
                    float4 gradientColor = _GradientColors[0];
                    for (int i = 0; i < (int)_GradientNumStops - 1; i++)
                    {
                        float t = saturate((gradientPos - _GradientStops[i]) / (_GradientStops[i + 1] - _GradientStops[i]));
                        gradientColor = lerp(_GradientColors[i], _GradientColors[i + 1], t);
                    }
                    finalColor.rgb = gradientColor.rgb;
                #elif defined(RADIAL_GRADIENT)
                    float distFromCenter = length(uv - 0.5) * 2.0;
                    float4 gradientColor = _GradientColors[0];
                    for (int i = 0; i < (int)_GradientNumStops - 1; i++)
                    {
                        float t = saturate((distFromCenter - _GradientStops[i]) / (_GradientStops[i + 1] - _GradientStops[i]));
                        gradientColor = lerp(_GradientColors[i], _GradientColors[i + 1], t);
                    }
                    finalColor.rgb = gradientColor.rgb;
                #endif
                
                // Texture sampling
                float4 texColor = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, uv);
                
                #ifdef CLAMP_TEXTURE
                    if (uv.x < 0 || uv.x > 1 || uv.y < 0 || uv.y > 1)
                    {
                        texColor.a = 0;
                    }
                #endif
                
                finalColor *= texColor;
                finalColor.a *= alpha;
                
                return finalColor;
            }
            ENDHLSL
        }
    }
}
```

---

## 3단계: Input System 업데이트

### 3.1 EventSystem 수정

```csharp
// UnityFigmaBridgeImporter.cs

using UnityEngine.InputSystem;
using UnityEngine.InputSystem.UI;

private static Canvas CreateCanvas(bool createEventSystem)
{
    var canvasGameObject = new GameObject("Canvas");
    var canvas = canvasGameObject.AddComponent<Canvas>();
    canvas.renderMode = RenderMode.ScreenSpaceOverlay;
    canvasGameObject.AddComponent<GraphicRaycaster>();

    if (!createEventSystem) return canvas;

    var existingEventSystem = Object.FindObjectOfType<EventSystem>();
    if (existingEventSystem == null)
    {
        var eventSystemGameObject = new GameObject("EventSystem");
        existingEventSystem = eventSystemGameObject.AddComponent<EventSystem>();
        
        // Unity 6 Input System 사용
        #if UNITY_INPUT_SYSTEM_AVAILABLE
            eventSystemGameObject.AddComponent<InputSystemUIInputModule>();
        #else
            eventSystemGameObject.AddComponent<StandaloneInputModule>();
        #endif
    }

    return canvas;
}
```

---

## 4단계: TextMeshPro 4.0 호환성

### 4.1 Material 속성 ID 변경

```csharp
// FontManager.cs

// TextMeshPro 4.0 셰이더 프로퍼티 ID
private static readonly int s_FaceColorPropertyID = Shader.PropertyToID("_FaceColor");
private static readonly int s_OutlineColorPropertyID = Shader.PropertyToID("_OutlineColor");
private static readonly int s_OutlineWidthPropertyID = Shader.PropertyToID("_OutlineWidth");
private static readonly int s_UnderlayColorPropertyID = Shader.PropertyToID("_UnderlayColor");
private static readonly int s_UnderlayOffsetPropertyID = Shader.PropertyToID("_UnderlayOffset");
```

### 4.2 Material 생성 코드 수정

```csharp
// FontManager.cs

public static Material GetEffectMaterialPreset(FigmaFontMapEntry fontMapEntry, 
    bool shadow, Color shadowColor, Vector2 shadowDistance, 
    bool outline, Color outlineColor, float outlineThickness)
{
    var newMaterialPreset = new Material(fontMapEntry.FontAsset.material);
    
    // TextMeshPro 4.0 셰이더
    newMaterialPreset.shader = Shader.Find("TextMeshPro/Distance Field");
    
    // Shadow (Underlay in TMP 4.0)
    newMaterialPreset.SetFloat("_UnderlayEnabled", shadow ? 1 : 0);
    if (shadow)
    {
        newMaterialPreset.SetColor(s_UnderlayColorPropertyID, shadowColor);
        newMaterialPreset.SetFloat(s_UnderlayOffsetPropertyID, shadowDistance.y);
    }
    
    // Outline
    newMaterialPreset.SetFloat("_OutlineWidth", outline ? outlineThickness : 0);
    if (outline)
    {
        newMaterialPreset.SetColor(s_OutlineColorPropertyID, outlineColor);
    }
    
    // AssetDatabase.CreateAsset...
    return newMaterialPreset;
}
```

---

## 5단계: 셰이더 키워드 API 수정

```csharp
// FigmaImage.cs

protected override void OnEnable()
{
    EnsureCanvasHasChannelsForFigmaImage();
    base.OnEnable();
}

public override Material GetModifiedMaterial(Material baseMaterial)
{
    var mat = base.GetModifiedMaterial(baseMaterial);
    
    // Unity 6 셰이더 키워드 API
    #if UNITY_6000_0_OR_NEWER
        mat.SetKeyword("STROKE", m_StrokeWidth > 0);
        mat.SetKeyword("LINEAR_GRADIENT", m_Fill == FillStyle.LinearGradient);
        mat.SetKeyword("RADIAL_GRADIENT", m_Fill == FillStyle.RadialGradient);
        mat.SetKeyword("SHAPE_RECTANGLE", m_Shape == ShapeType.Rectangle);
        mat.SetKeyword("SHAPE_ELLIPSE", m_Shape == ShapeType.Ellipse);
        mat.SetKeyword("SHAPE_STAR", m_Shape == ShapeType.Star);
        mat.SetKeyword("ARC_ANGLE_RANGE", m_EllipseArcAngleRange.y < Mathf.PI * 2.0f);
        mat.SetKeyword("CLAMP_TEXTURE", m_ImageScaleMode == ImageScaleMode.Fit);
    #else
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "STROKE"), m_StrokeWidth > 0);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "LINEAR_GRADIENT"), m_Fill == FillStyle.LinearGradient);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "RADIAL_GRADIENT"), m_Fill == FillStyle.RadialGradient);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "SHAPE_RECTANGLE"), m_Shape == ShapeType.Rectangle);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "SHAPE_ELLIPSE"), m_Shape == ShapeType.Ellipse);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "SHAPE_STAR"), m_Shape == ShapeType.Star);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "ARC_ANGLE_RANGE"), m_EllipseArcAngleRange.y < Mathf.PI * 2.0f);
        mat.SetKeyword(new LocalKeyword(baseMaterial.shader, "CLAMP_TEXTURE"), m_ImageScaleMode == ImageScaleMode.Fit);
    #endif
    
    // 기존 속성 설정 코드...
    return mat;
}
```

---

## 6단계: System.Text.Json 전환

### 6.1 Newtonsoft.Json 제거

```json
// package.json - 의존성 제거
"dependencies": {
    "com.unity.ugui": "2.0.0",
    "com.unity.textmeshpro": "4.0.0-pre.6"
    // "com.unity.nuget.newtonsoft-json" 제거
}
```

### 6.2 FigmaApiUtils 수정

```csharp
// FigmaApiUtils.cs

using System.Text.Json;
using System.Text.Json.Serialization;

public static class FigmaApiUtils
{
    private static readonly JsonSerializerOptions s_jsonOptions = new()
    {
        PropertyNameCaseInsensitive = true,
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        Converters = { new JsonStringEnumConverter() }
    };
    
    public static async Task<FigmaFile> GetFigmaDocument(string fileId, string accessToken, bool deep = false)
    {
        string url = $"https://api.figma.com/v1/files/{fileId}";
        if (deep) url += "?deep=true";
        
        using var request = UnityWebRequest.Get(url);
        request.SetRequestHeader("X-Figma-Token", accessToken);
        
        await request.SendWebRequest();
        
        if (request.result == UnityWebRequest.Result.Success)
        {
            var json = request.downloadHandler.text;
            return JsonSerializer.Deserialize<FigmaFile>(json, s_jsonOptions);
        }
        
        throw new Exception($"Error fetching Figma document: {request.error}");
    }
}
```

---

## 7단계: UI Toolkit 에디터

### 7.1 Settings 에디터 UXML

```xml
<!-- UnityFigmaBridgeSettingsEditor.uxml -->
<UIElements xmlns="UnityEngine.UIElements" xmlns="ue="UnityEditor.UIElements">
    <VisualElement class="container">
        <Label text="Unity Figma Bridge Settings" class="title" />
        
        <PropertyField binding-path="FileId" label="Figma Document URL" />
        <PropertyField binding-path="BuildPrototypeFlow" label="Build Prototype Flow" />
        <PropertyField binding-path="OnlyImportSelectedPages" label="Only Import Selected Pages" />
        <PropertyField binding-path="EnableGoogleFontsDownloads" label="Enable Google Fonts Downloads" />
        <PropertyField binding-path="EnableAutoLayout" label="Enable Auto Layout" />
        <PropertyField binding-path="ServerRenderImageScale" label="Server Render Image Scale" />
        <PropertyField binding-path="ScreenBindingNamespace" label="Screen Binding Namespace" />
        
        <Button name="syncButton" text="Sync Document" class="sync-button" />
    </VisualElement>
</UIElements>
```

### 7.2 Settings 에디터 C#

```csharp
// UnityFigmaBridgeSettingsEditor.cs

using UnityEditor;
using UnityEngine;
using UnityEngine.UIElements;

[CustomEditor(typeof(UnityFigmaBridgeSettings))]
public class UnityFigmaBridgeSettingsEditor : Editor
{
    public override VisualElement CreateInspectorGUI()
    {
        var visualTree = AssetDatabase.LoadAssetAtPath<VisualTreeAsset>(
            "Packages/com.simonoliver.unityfigma/Editor/Settings/UnityFigmaBridgeSettingsEditor.uxml");
        
        var root = visualTree.Instantiate();
        root.Bind(serializedObject);
        
        var syncButton = root.Q<Button>("syncButton");
        syncButton.clicked += OnSyncClicked;
        
        return root;
    }
    
    private void OnSyncClicked()
    {
        UnityFigmaBridgeImporter.Sync();
    }
}
```

---

## 8단계: 컴파일 및 테스트

### 8.1 컴파일 체크리스트

- [ ] 모든 스크립트가 에러 없이 컴파일됨
- [ ] 셰이더가 URP에서 정상 작동함
- [ ] Input System이 정상 작동함
- [ ] TextMeshPro Material이 정상 적용됨
- [ ] 에디터가 UI Toolkit으로 표시됨

### 8.2 기능 테스트

1. **기본 임포트**
   - Figma 문서 동기화
   - 페이지 생성
   - 컴포넌트 생성

2. **UI 렌더링**
   - 도형 (Rectangle, Ellipse, Star)
   - 채우기 (단색, 그라데이션)
   - 획
   - 이미지

3. **텍스트**
   - TextMeshPro 렌더링
   - 폰트 로딩
   - 효과 (그림자, 외곽선)

4. **프로토타입**
   - 화면 전환
   - 버튼 동작
   - 이벤트 바인딩

---

## 알려진 문제 및 해결 방안

### 문제 1: 셰이더가 Built-in RP에서만 작동함

**해결:** URP용으로 셰이더 재작성 (위 참조)

### 문제 2: Input System 오류

**해결:** Package Manager에서 Input System 설치 및 Active Input Setting 변경

### 문제 3: TextMeshPro Material 오류

**해결:** 셰이더 프로퍼티 ID를 TextMeshPro 4.0에 맞게 변경

### 문제 4: UI Toolkit 에디터가 표시되지 않음

**해결:** UXML 파일 경로 확인 및 USS 스타일 적용

---

## 지원 및 리소스

- [Unity 6 릴리스 노트](https://unity.com/releases/editor/whats-new/6000.0)
- [URP 셰이더 문서](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@latest)
- [Input System 문서](https://docs.unity3d.com/Packages/com.unity.inputsystem@latest)
- [UI Toolkit 문서](https://docs.unity3d.com/Manual/UIElements.html)
- [TextMeshPro 포럼](https://forum.unity.com/forums/textmeshpro.231/)
