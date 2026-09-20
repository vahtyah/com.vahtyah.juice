# com.vahtyah.juice — API Reference

> Version 0.1.0. Auto-generated from the compiled public API by `VahTyah → Generate API Docs`. Do not edit by hand — edit the source or `Documentation~/Examples.md` instead.

This package ships as obfuscated DLLs. This file documents the full public surface so it can be used without decompiling.

## Usage Examples

Game-feel framework (zero-alloc) built on **LitMotion** (tween) + **UniTask** (async). Add a
`Juicer` component, compose `JuiceFeedback`s in the Inspector, and play them from code.

### Play a Juicer from code

```csharp
using VahTyah.Juice;

public class Enemy : MonoBehaviour
{
    [SerializeField] private Juicer hitJuicer; // a GameObject with a Juicer + feedbacks

    public void OnHit(Vector3 hitPoint)
    {
        // Fire-and-forget from the juicer's own transform.
        hitJuicer.Play();

        // Or pass a context (origin, global intensity, direction) and await completion.
        var ctx = new JuicePlayContext(hitPoint, intensity: 1.5f, JuiceDirection.Forward);
        hitJuicer.PlayAsync(ctx).Forget();
    }

    public void Cancel() => hitJuicer.Stop(); // cancels the chain, restores each feedback's start state
}
```

`Juicer` also exposes `PlayFeedback(int index)`, `AddFeedback(JuiceFeedback)`, `Initialize()`
(idempotent; called on Awake) and `TotalDuration`. Feedbacks run per `TimingMode`
(`AfterPrevious` = sequential, `WithPrevious` = parallel with the previous one).

### Write a custom feedback

Subclass `JuiceFeedback` (it is `[Serializable]`, not a MonoBehaviour — many live in one
`[SerializeReference] List<JuiceFeedback>`). Drive the actual tween with LitMotion and always
forward the `CancellationToken` so `Stop()`/Destroy cancels it.

```csharp
using System.Threading;
using Cysharp.Threading.Tasks;
using LitMotion;
using LitMotion.Extensions;
using UnityEngine;
using VahTyah.Juice;
using VahTyah.Inspector; // [BoxGroup], [Required]

[System.Serializable]
public class ScalePunchFeedback : JuiceFeedback
{
    [BoxGroup("Target"), Required] public Transform Target;
    [BoxGroup("Punch")] public Vector3 Strength      = Vector3.one * 0.2f;
    [BoxGroup("Punch")] public float   TweenDuration = 0.3f;
    [BoxGroup("Punch")] public int     Frequency     = 10;
    [BoxGroup("Punch")] public float   DampingRatio  = 1f;

    private Vector3 _initialScale;
    public override float Duration => TweenDuration;

    protected internal override void Initialize(Juicer owner)
    {
        if (Target == null) Target = owner.transform; // fallback to the owner
        _initialScale = Target.localScale;
    }

    protected internal override UniTask PlayAsync(JuicePlayContext context, CancellationToken token) =>
        LMotion.Punch.Create(_initialScale, Strength * context.Intensity, TweenDuration)
            .WithFrequency(Frequency)
            .WithDampingRatio(DampingRatio)
            .BindToLocalScale(Target)
            .ToUniTask(CancelBehavior.Cancel, token); // cancel → motion stops

    protected internal override void Stop()
    {
        if (Target != null) Target.localScale = _initialScale; // restore on Stop
    }
}
```

The `+` button in the `Juicer` Inspector lists every `JuiceFeedback` subclass (via `TypeCache`),
so a new feedback shows up automatically.

### Inline value vs ScriptableObject override (`JuiceParam<T>`)

A feedback's tunable values live in a `struct XxxSettings` wrapped in `JuiceParam<XxxSettings>`.
There is **no toggle** — the presence of a `JuiceConfigSO<T>` is the mode:

- No SO assigned → the `Inline` value is used (edit right on the feedback).
- A `JuiceConfigSO<XxxSettings>` assigned → the SO **overrides** it: shared across objects and
  tunable *in Play mode* (persists to the asset, applies even to prefabs instantiated at runtime).

`JuiceParam<T>.Value` is read **at Play time** (not cached), so live-tuning takes effect
immediately. Read it inside `PlayAsync` as `Settings.Value`. Scene references (Targets) always
stay inline — an SO cannot reference scene objects.

## API Reference

### namespace `VahTyah.Juice`

#### class `CameraZoomFeedback`

*: JuiceFeedback*  
Zoom camera theo kiểu pulse (in → hold → out) rồi trả về giá trị gốc — hiệu ứng "punch" cho hit-feel. Tự nhận perspective (fieldOfView) hay orthographic (orthographicSize) theo camera. Giá trị gốc đọc TẠI Play và được khôi phục khi Stop, nên compose an toàn với feedback khác đang đổi camera.  

```csharp
public CameraZoomFeedback();
public JuiceParam<CameraZoomSettings> Settings;
public Camera Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `CameraZoomSettings`

Bộ value tune được của CameraZoom — dùng cho cả inline lẫn SO. Một "pulse" zoom: zoom vào tới đích, giữ, rồi trả về giá trị gốc.  

```csharp
public JuiceEasing Easing;
public float HoldDuration;
public bool Relative;
public float Target;
public float ZoomInDuration;
public float ZoomOutDuration;
public static CameraZoomSettings Default { get; }
```

- `Target` — Giá trị zoom đích. Perspective: field of view (độ, nhỏ hơn = zoom vào). Orthographic: orthographic size (nhỏ hơn = zoom vào). Relative = cộng vào giá trị hiện tại.

#### class `DestinationFeedback`

*: JuiceFeedback*  
Di chuyển Target (position world) tới vị trí của DestinationTarget trong Duration. Cả hai là scene-ref nên để inline (không có SO config). Đích lấy tại thời điểm Play (nếu đích di chuyển sau đó thì không bám theo).  

```csharp
public DestinationFeedback();
public Transform DestinationTarget;
public float Duration;
public Ease Ease;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `FadeFeedback`

*: JuiceFeedback*  
Fade alpha của Renderer (2D/3D) — appear/disappear, ghost, panel fade. SpriteRenderer đổi .color.a; Renderer khác đổi alpha qua MaterialPropertyBlock (material cần trong suốt mới thấy). Giữ nguyên RGB.  

```csharp
public FadeFeedback();
public string ColorProperty;
public JuiceParam<FadeSettings> Settings;
public Renderer Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

- `ColorProperty` — Slot màu khi Target KHÔNG phải SpriteRenderer. Trống/không có → tự dò _BaseColor rồi _Color.

#### struct `FadeSettings`

Bộ value tune được của Fade (alpha): tween alpha From → To, giữ nguyên RGB. Default 0 → 1 (fade-in).  

```csharp
public float Duration;
public JuiceEasing1 Easing;
public float From;
public float To;
public static FadeSettings Default { get; }
```

#### struct `JuiceCurve3`

Bộ 3 AnimationCurve (X/Y/Z) — hình animation per-axis giống Juice gốc, vẽ kiểu Particle System (xem JuiceCurve3Drawer). Curve nhận progress 0..1, output là hệ số lerp (có thể vượt 0..1 để overshoot/bounce).  

```csharp
public AnimationCurve X;
public AnimationCurve Y;
public AnimationCurve Z;
public static JuiceCurve3 Linear { get; }
public Vector3 Evaluate(float p);
public Vector3 Lerp(Vector3 a, Vector3 b, float p);
```

- `Evaluate` — Hệ số mỗi trục tại progress p (0..1).
- `Lerp` — Lerp (unclamped) từ a→b theo curve mỗi trục tại progress p — dùng cho move/rotate/scale-to.

#### enum `JuiceDirection`

Hướng chơi. v0 chỉ dùng Forward; Reverse để dành cho bản sau (feedback tự quyết cách đảo).  

```csharp
enum JuiceDirection : int
{
    Forward = 0,
    Reverse = 1,
}
```

#### struct `JuiceEasing`

Chọn Ease (preset LitMotion, áp cho cả 3 trục) HOẶC Curve (AnimationCurve per-axis) — giống dropdown mode của Particle System. Feedback tween-to đọc Lerp để ra giá trị tại progress p.  

```csharp
public JuiceCurve3 Curves;
public JuiceEasingMode Mode;
public Ease Preset;
public static JuiceEasing Default { get; }
public Vector3 Lerp(Vector3 a, Vector3 b, float p);
```

- `Lerp` — Lerp (unclamped) a→b tại progress p theo mode hiện tại.

#### struct `JuiceEasing1`

Easing 1-chiều (scalar) — chọn Ease preset (LitMotion) HOẶC một AnimationCurve duy nhất. Dùng cho feedback scalar: color, image fill/alpha, audio volume, light intensity… Khác JuiceEasing (3 trục X/Y/Z) vốn dành riêng cho Transform tween Vector3. Tái dùng enum JuiceEasingMode (Ease / Curve).  

```csharp
public AnimationCurve Curve;
public JuiceEasingMode Mode;
public Ease Preset;
public static JuiceEasing1 Default { get; }
public float Ease01(float p);
```

- `Ease01` — Hệ số eased 0..1 tại progress p (Ease preset, hoặc Curve — fallback linear nếu rỗng).

#### enum `JuiceEasingMode`

Cách nội suy: preset Ease (đồng nhất mọi trục) hay AnimationCurve per-axis.  

```csharp
enum JuiceEasingMode : int
{
    Ease = 0,
    Curve = 1,
}
```

#### abstract class `JuiceFeedback`

Base cho mọi hiệu ứng feel. Là class [Serializable] (không phải MonoBehaviour) nên nhiều feedback đa hình sống chung trong một List<JuiceFeedback> qua [SerializeReference]. Logic chạy bằng UniTask; tween thực tế nên dùng LitMotion bên trong PlayAsync.  

```csharp
protected JuiceFeedback();
public bool Active;
public float Delay;
public bool IsPlaying;
public string Label;
public float StartedAt;
public JuiceTimingMode TimingMode;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal abstract UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

- `Delay` — Chờ (giây) trước khi feedback này bắt đầu, tính từ lúc batch của nó khởi động.
- `IsPlaying` — Runtime: feedback này có đang chạy không (để editor hiện nút Play/Stop từng cái).
- `StartedAt` — Runtime: mốc thời gian (scaled Time.time, khớp clock của LitMotion) lúc bắt đầu chạy — để editor vẽ thanh duration khi play.
- `PlaybackDuration` — Thời lượng dự kiến để player/editor ước tính tổng duration (không điều khiển motion thật). 0 = tức thời. Đặt tên khác "Duration" để feedback có thể có field serialize tên Duration mà không đụng property này.
- `Initialize` — Gọi một lần khi Juicer khởi tạo. Cache component ở đây để runtime không alloc/tìm lại.
- `PlayAsync` — Logic chính. Dùng LitMotion bên trong và await bằng UniTask, truyền token để huỷ khi Stop.
- `Stop` — Được gọi khi player Stop. Token đã bị huỷ nên motion tự dừng; override để dọn/đưa về trạng thái gốc.

#### sealed class `JuiceFeedbackMenuAttribute`

*: Attribute*  
Đường dẫn hiển thị trong menu "Add feedback" (dùng "/" để tạo submenu), ví dụ "Transform/Position". Không gắn thì editor fallback về tên type đã nicify.  

```csharp
public JuiceFeedbackMenuAttribute(string path);
public string Path { get; }
```

#### class `JuiceLoopStartFeedback`

*: JuiceFeedback*  
Mốc "đầu vòng lặp" — feedback RỖNG (không hiệu ứng), chỉ đánh dấu vị trí để JuiceLooperFeedback tua head về đây. Là marker neo theo nội dung nên sống sót qua reorder/insert/delete trong editor (khác với lưu index cứng). Nếu không có LoopStart active nào phía trên looper, head tua về đầu chuỗi (index 0 = lặp toàn bộ).  

```csharp
public JuiceLoopStartFeedback();
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `JuiceLooperFeedback`

*: JuiceFeedback*  
Control-point kiểu "head-rewind" (giống MMF_Looper của Feel): khi head chạy tới đây và còn lượt lặp, head TUA về JuiceLoopStartFeedback active gần nhất phía trên (hoặc đầu chuỗi) để phát lại đoạn đó. Toàn bộ logic loop nằm gọn trong class này qua 2 interface: IJuiceSequenceStart (reset/cache đầu Play) và IJuiceControlFlow (quyết định head). Juicer chỉ nói chuyện qua interface, KHÔNG biết về loop; base class không đụng gì. Đặt ở AfterPrevious.  

```csharp
public JuiceLooperFeedback();
public bool InfiniteLoop;
public int NumberOfLoops;
public float PlaybackDuration { get; }
public int GetNextHead(int index, List<JuiceFeedback> feedbacks);
public void OnSequenceStart(int index, List<JuiceFeedback> feedbacks);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

- `NumberOfLoops` — Số lần TUA lại đoạn lặp — đoạn phát tổng cộng NumberOfLoops + 1 lần. Bỏ qua khi InfiniteLoop.
- `OnSequenceStart` — Đầu mỗi lần Play: cache mốc tua (LoopStart active gần nhất) và nạp lại bộ đếm lượt lặp.
- `GetNextHead` — Còn lượt lặp → trả head về mốc đã cache (và arm lại các looper lồng bên trong); hết lượt → index + 1.

#### class `JuiceParam<T>`

Bộ value của một feedback: mặc định dùng Inline (chỉnh ngay trên feedback, tiện). Gán một Config (JuiceConfigSO<T>) thì SO **override** — dùng chung cho nhiều object và tune ngay trong Play mode (lưu vào asset nên không mất khi thoát play, áp cả prefab instantiate runtime). Không có SO thì fallback về Inline. Không có toggle — sự hiện diện của SO chính là "mode". Feedback đọc Value TẠI Play (không cache). T là struct → đọc Value zero-alloc.  

```csharp
public JuiceParam();
public T Inline;
public bool UsingConfig { get; }
public T Value { get; }
```

- `Value` — Có SO → SO thắng; không thì dùng Inline.

#### struct `JuicePlayContext`

Dữ liệu cho một lần chơi feedback. readonly struct, truyền by-value để không cấp phát.  

```csharp
public JuicePlayContext(Vector3 position, float intensity = 1, JuiceDirection direction = JuiceDirection.Forward);
public readonly JuiceDirection Direction;
public readonly float Intensity;
public readonly Vector3 Position;
```

- `Position` — Vị trí gốc của lần chơi (vd điểm va chạm). Feedback có thể dùng hoặc bỏ qua.
- `Intensity` — Hệ số cường độ toàn cục nhân vào output của feedback (1 = mặc định).
- `Direction` — Hướng chơi.

#### enum `JuiceTimingMode`

Cách một feedback xếp thời gian so với feedback đứng ngay trước nó trong list.  

```csharp
enum JuiceTimingMode : int
{
    AfterPrevious = 0,
    WithPrevious = 1,
}
```

#### class `Juicer`

*: MonoBehaviour*  
Nhạc trưởng: giữ danh sách feedback và chơi chúng theo model timing (AfterPrevious / WithPrevious). Component-based như MMF_Player nhưng chạy trên UniTask (không coroutine) + LitMotion (zero-alloc). Inspector do com.vahtyah.inspector vẽ tự động — không cần custom editor.  

```csharp
public Juicer();
public bool CancelPreviousOnPlay;
public List<JuiceFeedback> Feedbacks;
public bool IsPlaying;
public bool PlayOnStart;
public float PlayStartedAt;
public float TotalDuration { get; }
public void AddFeedback(JuiceFeedback feedback);
public void Initialize();
public void Play();
public UniTask PlayAsync(JuicePlayContext context);
public void PlayFeedback(int index);
public UniTask PlayFeedbackAsync(int index, JuicePlayContext context);
public void Stop();
public void StopFeedback(int index);
```

- `CancelPreviousOnPlay` — Nếu true, gọi Play khi đang chơi sẽ huỷ lần chơi cũ trước khi bắt đầu lại.
- `PlayStartedAt` — Runtime: mốc bắt đầu lần play tổng (scaled Time.time, khớp clock của LitMotion) — để editor vẽ thanh duration tổng.
- `TotalDuration` — Tổng thời lượng cả chuỗi theo model timing (batch WithPrevious lấy max, các batch cộng dồn).
- `Initialize` — Khởi tạo một lần: cache destroy token và cho từng feedback cache reference của nó.
- `AddFeedback` — Thêm feedback lúc runtime. Nếu player đã Initialize thì khởi tạo feedback ngay.
- `Play` — Chơi bằng context mặc định (vị trí của transform này). Fire-and-forget, không alloc Task.
- `PlayAsync` — Chơi toàn bộ feedback và await tới khi xong (hoặc bị huỷ bởi Stop/OnDestroy).
- `Stop` — Dừng ngay: huỷ token (motion LitMotion tự dừng) và cho từng feedback dọn trạng thái.
- `PlayFeedback` — Chơi riêng một feedback theo index (dùng cho nút Play từng cái trong Inspector).
- `PlayFeedbackAsync` — Chơi riêng một feedback theo index và await tới khi xong — biến thể async của PlayFeedback. Huỷ lần chơi riêng trước đó của chính feedback này (nếu có) rồi mới bắt đầu.
- `StopFeedback` — Dừng một feedback đang chơi riêng.

#### class `LookAtFeedback`

*: JuiceFeedback*  
Xoay Target (rotation world) để nhìn về LookAtTarget trong Duration. Cả hai đều là scene-ref nên để inline (không có SO config).  

```csharp
public LookAtFeedback();
public float Duration;
public Ease Ease;
public Transform LookAtTarget;
public Transform Target;
public Vector3 Up;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `MaterialColorFeedback`

*: JuiceFeedback*  
Tween một shader property COLOR theo tên (Property) qua MaterialPropertyBlock — thường là _EmissionColor (glow/bloom, cần material bật emission), _OutlineColor, _RimColor… Khác Renderer Color: slot chính xác (không sprite path, không fallback) + HdrIntensity cho emission.  

```csharp
public MaterialColorFeedback();
public string Property;
public JuiceParam<MaterialColorSettings> Settings;
public Renderer Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### struct `MaterialColorSettings`

Bộ value tune được của Material Color. HdrIntensity nhân RGB đích để làm glow (emission HDR). ToAndBack = true: tới rồi về (pulse). false: giữ.  

```csharp
public Color Destination;
public float Duration;
public JuiceEasing1 Easing;
public float HdrIntensity;
public bool ToAndBack;
public static MaterialColorSettings Default { get; }
```

#### class `MaterialFloatFeedback`

*: JuiceFeedback*  
Tween một shader property FLOAT theo tên (Property) qua MaterialPropertyBlock — dissolve, fill amount, glow, outline width, progress… From → To (cả hai tune được). Cần material có property.  

```csharp
public MaterialFloatFeedback();
public string Property;
public JuiceParam<MaterialFloatSettings> Settings;
public Renderer Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### struct `MaterialFloatSettings`

Bộ value tune được của Material Float: tween From → To. ToAndBack = true: tới To rồi về From (pulse). false: giữ ở To.  

```csharp
public float Duration;
public JuiceEasing1 Easing;
public float From;
public float To;
public bool ToAndBack;
public static MaterialFloatSettings Default { get; }
```

#### class `PauseFeedback`

*: JuiceFeedback*  
Chỉ chờ Duration giây rồi kết thúc — tạo khoảng nghỉ tường minh trong chuỗi. Đặt ở AfterPrevious để chèn gap giữa các bước; không có Target, không tween. Lưu ý: chờ bằng UniTask.Delay (player loop) nên chỉ advance khi Play, không chạy ở preview edit mode.  

```csharp
public PauseFeedback();
public float Duration;
public float PlaybackDuration { get; }
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `PositionFeedback`

*: JuiceFeedback*  
Tween localPosition từ vị trí hiện tại tới đích (move-to) và GIỮ ở đó — không tự trả về. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public PositionFeedback();
public JuiceParam<PositionSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `PositionPunchFeedback`

*: JuiceFeedback*  
"Punch" localPosition của một Transform rồi trả về vị trí gốc — hiệu ứng nảy/đẩy. Target luôn inline (scene-ref); thông số punch qua JuiceParam (Direct hoặc Shared SO).  

```csharp
public PositionPunchFeedback();
public JuiceParam<PositionPunchSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `PositionPunchSettings`

Bộ value tune được của PositionPunch — dùng cho cả inline (Direct) lẫn SO (Shared).  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static PositionPunchSettings Default { get; }
```

#### struct `PositionSettings`

Bộ value tune được của Position (move-to) — dùng cho cả inline lẫn SO.  

```csharp
public Vector3 Destination;
public float Duration;
public JuiceEasing Easing;
public bool Relative;
public static PositionSettings Default { get; }
```

#### class `PositionShakeFeedback`

*: JuiceFeedback*  
"Shake" localPosition của một Transform rồi trả về vị trí gốc — hiệu ứng rung/lắc vị trí. Target luôn inline (scene-ref); thông số shake qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public PositionShakeFeedback();
public JuiceParam<PositionShakeSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `PositionShakeSettings`

Bộ value tune được của PositionShake — dùng cho cả inline lẫn SO.  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static PositionShakeSettings Default { get; }
```

#### class `PositionSpringFeedback`

*: JuiceFeedback*  
"Spring" localPosition: bump theo Strength rồi nảy tắt dần về vị trí gốc (damped harmonic, closed-form). Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public PositionSpringFeedback();
public JuiceParam<PositionSpringSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `PositionSpringSettings`

Bộ value tune được của PositionSpring — dùng cho cả inline lẫn SO.  

```csharp
public float DampingRatio;
public float Duration;
public float Frequency;
public Vector3 Strength;
public static PositionSpringSettings Default { get; }
```

#### class `RendererColorFeedback`

*: JuiceFeedback*  
Tween màu của Renderer (2D hoặc 3D) tới Destination — flash về gốc, hoặc giữ. SpriteRenderer đổi .color; Renderer khác đổi qua MaterialPropertyBlock (xem JuiceRendererColor). context.Intensity lerp mức áp màu. Target auto-bind Renderer của owner nếu để trống.  

```csharp
public RendererColorFeedback();
public string ColorProperty;
public JuiceParam<RendererColorSettings> Settings;
public Renderer Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

- `ColorProperty` — Shader property khi Target KHÔNG phải SpriteRenderer. Trống/không có → tự dò _BaseColor rồi _Color.

#### struct `RendererColorSettings`

Bộ value tune được của Renderer Color — dùng cho cả inline lẫn SO. ToAndBack = true: nháy tới Destination rồi về màu gốc trong Duration (flash). false: tween tới và GIỮ.  

```csharp
public Color Destination;
public float Duration;
public JuiceEasing1 Easing;
public bool ToAndBack;
public static RendererColorSettings Default { get; }
```

#### class `RendererFlickerFeedback`

*: JuiceFeedback*  
Nháy màu Renderer (2D hoặc 3D) giữa màu gốc ↔ FlickerColor (square-wave, kết thúc ở màu gốc). SpriteRenderer đổi .color; Renderer khác qua MaterialPropertyBlock. context.Intensity lerp độ đậm FlickerColor. Target auto-bind Renderer của owner nếu để trống.  

```csharp
public RendererFlickerFeedback();
public string ColorProperty;
public JuiceParam<RendererFlickerSettings> Settings;
public Renderer Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

- `ColorProperty` — Shader property khi Target KHÔNG phải SpriteRenderer. Trống/không có → tự dò _BaseColor rồi _Color.

#### struct `RendererFlickerSettings`

Bộ value tune được của Renderer Flicker — nháy qua lại giữa màu gốc và FlickerColor Count lần (square-wave) trong Duration.  

```csharp
public int Count;
public float Duration;
public Color FlickerColor;
public static RendererFlickerSettings Default { get; }
```

#### class `RotatePositionAroundFeedback`

*: JuiceFeedback*  
Quay localPosition của Target quanh một Pivot theo Axis một góc Angle (theo thời gian). 360° = về chỗ cũ. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public RotatePositionAroundFeedback();
public JuiceParam<RotatePositionAroundSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `RotatePositionAroundSettings`

Bộ value tune được của RotatePositionAround — dùng cho cả inline lẫn SO.  

```csharp
public float Angle;
public Vector3 Axis;
public float Duration;
public Ease Ease;
public Vector3 Pivot;
public static RotatePositionAroundSettings Default { get; }
```

#### class `RotationFeedback`

*: JuiceFeedback*  
Tween localEulerAngles từ góc hiện tại tới đích (rotate-to) và GIỮ ở đó — không tự trả về. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public RotationFeedback();
public JuiceParam<RotationSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `RotationPunchFeedback`

*: JuiceFeedback*  
"Punch" localEulerAngles của một Transform rồi trả về góc gốc — hiệu ứng giật/nghiêng. Target luôn inline (scene-ref); thông số punch qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public RotationPunchFeedback();
public JuiceParam<RotationPunchSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `RotationPunchSettings`

Bộ value tune được của RotationPunch — dùng cho cả inline lẫn SO. Strength tính bằng độ (localEulerAngles).  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static RotationPunchSettings Default { get; }
```

#### struct `RotationSettings`

Bộ value tune được của Rotation (rotate-to) — dùng cho cả inline lẫn SO. Đơn vị độ (localEulerAngles).  

```csharp
public Vector3 Destination;
public float Duration;
public JuiceEasing Easing;
public bool Relative;
public static RotationSettings Default { get; }
```

#### class `RotationShakeFeedback`

*: JuiceFeedback*  
"Shake" localEulerAngles của một Transform rồi trả về góc gốc — hiệu ứng rung xoay. Target luôn inline (scene-ref); thông số shake qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public RotationShakeFeedback();
public JuiceParam<RotationShakeSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `RotationShakeSettings`

Bộ value tune được của RotationShake — dùng cho cả inline lẫn SO. Strength tính bằng độ (localEulerAngles).  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static RotationShakeSettings Default { get; }
```

#### class `RotationSpringFeedback`

*: JuiceFeedback*  
"Spring" localEulerAngles: bump theo Strength rồi nảy tắt dần về góc gốc (damped harmonic, closed-form). Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public RotationSpringFeedback();
public JuiceParam<RotationSpringSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `RotationSpringSettings`

Bộ value tune được của RotationSpring — dùng cho cả inline lẫn SO. Đơn vị độ (localEulerAngles).  

```csharp
public float DampingRatio;
public float Duration;
public float Frequency;
public Vector3 Strength;
public static RotationSpringSettings Default { get; }
```

#### class `ScaleFeedback`

*: JuiceFeedback*  
Tween localScale từ scale hiện tại tới đích (scale-to) và GIỮ ở đó — không tự trả về. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public ScaleFeedback();
public JuiceParam<ScaleSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `ScalePunchFeedback`

*: JuiceFeedback*  
"Punch" localScale của một Transform rồi trả về scale gốc — hiệu ứng nảy cơ bản. Target luôn inline (scene-ref); thông số punch qua JuiceParam (Direct hoặc Shared SO).  

```csharp
public ScalePunchFeedback();
public JuiceParam<ScalePunchSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `ScalePunchSettings`

Bộ value tune được của ScalePunch — dùng cho cả inline (Direct) lẫn SO (Shared).  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static ScalePunchSettings Default { get; }
```

#### struct `ScaleSettings`

Bộ value tune được của Scale (scale-to) — dùng cho cả inline lẫn SO.  

```csharp
public Vector3 Destination;
public float Duration;
public JuiceEasing Easing;
public bool Relative;
public static ScaleSettings Default { get; }
```

#### class `ScaleShakeFeedback`

*: JuiceFeedback*  
"Shake" localScale của một Transform rồi trả về scale gốc — hiệu ứng rung theo scale. Target luôn inline (scene-ref); thông số shake qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public ScaleShakeFeedback();
public JuiceParam<ScaleShakeSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `ScaleShakeSettings`

Bộ value tune được của ScaleShake — dùng cho cả inline lẫn SO.  

```csharp
public float DampingRatio;
public float Duration;
public int Frequency;
public Vector3 Strength;
public static ScaleShakeSettings Default { get; }
```

#### class `ScaleSpringFeedback`

*: JuiceFeedback*  
"Spring" localScale: bump theo Strength rồi nảy tắt dần về scale gốc (damped harmonic, closed-form). Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public ScaleSpringFeedback();
public JuiceParam<ScaleSpringSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `ScaleSpringSettings`

Bộ value tune được của ScaleSpring — dùng cho cả inline lẫn SO.  

```csharp
public float DampingRatio;
public float Duration;
public float Frequency;
public Vector3 Strength;
public static ScaleSpringSettings Default { get; }
```

#### class `SetParentFeedback`

*: JuiceFeedback*  
Đổi parent của Target sang NewParent (tức thời, Duration = 0). Scene-ref nên để inline (không có SO config). NewParent = null → tách khỏi parent (đưa ra root).  

```csharp
public SetParentFeedback();
public Transform NewParent;
public Transform Target;
public bool WorldPositionStays;
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
```

#### class `ShatterFeedback`

*: JuiceFeedback*  
"Vỡ" một model PRE-FRACTURED: gom mọi mesh con của Root rồi văng tất cả ra theo quỹ đạo ballistic (văng từ tâm hình học + vòng cung gravity + xoay quanh trục ngẫu nhiên), tuỳ chọn co nhỏ ở cuối. Chạy zero-alloc: mảng per-piece cấp phát 1 lần ở Initialize; mỗi Play chỉ tính lại vận tốc/trục xoay vào mảng có sẵn và lái CẢ cụm bằng MỘT LitMotion motion (state struct + lambda static). Mảnh KHÔNG bị reparent — animate localPosition/localRotation/localScale dưới Root nên reset trivial và không tốn chuyển hệ toạ độ. Đầu mỗi Play và khi Stop() đều reassemble về pose gốc → model replay được. Up (bias) và Gravity đều khai báo ở WORLD và độc lập nhau — bias đẩy theo +Up, Gravity là vector gia tốc tự mang hướng+độ lớn — nên game lấy trục "xuống" là gì cũng chỉnh được (không khoá Y), Root xoay vẫn đúng (cả hai convert sang local; Up normalize, Gravity giữ nguyên độ lớn). Giả định: Root là parent của các mảnh (mỗi mảnh 1 Transform con có MeshRenderer). Fade per-piece / trả pool chưa làm ở bản này — xem ghi chú cuối.  

```csharp
public ShatterFeedback();
public Transform Root;
public JuiceParam<ShatterSettings> Settings;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `ShatterSettings`

Bộ value tune được của Shatter — dùng cho cả inline lẫn SO.  

```csharp
public bool DisableOnComplete;
public float Drag;
public float Duration;
public float Force;
public float ForceRandomness;
public Vector3 Gravity;
public float Randomness;
public bool Shrink;
public float ShrinkStart;
public float SpinSpeed;
public Vector3 Up;
public float UpwardBias;
public bool UseContactCenter;
public static ShatterSettings Default { get; }
```

#### class `SquashAndStretchFeedback`

*: JuiceFeedback*  
"Punch" localScale theo kiểu squash-and-stretch rồi trả về scale gốc. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public SquashAndStretchFeedback();
public JuiceParam<SquashAndStretchSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `SquashAndStretchSettings`

Bộ value tune được của SquashAndStretch — dùng cho cả inline lẫn SO. Kéo dãn theo Axis, ép 2 trục còn lại một nửa để gần như giữ thể tích.  

```csharp
public float Amount;
public Vector3 Axis;
public float DampingRatio;
public float Duration;
public int Frequency;
public static SquashAndStretchSettings Default { get; }
public Vector3 Delta();
```

- `Delta` — Delta scale: +Amount theo Axis, -Amount/2 ở 2 trục vuông góc (giữ thể tích xấp xỉ).

#### class `SquashAndStretchSpringFeedback`

*: JuiceFeedback*  
"Spring" localScale kiểu squash-and-stretch: bump rồi nảy tắt dần về gốc (damped harmonic, closed-form). Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public SquashAndStretchSpringFeedback();
public JuiceParam<SquashAndStretchSpringSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `SquashAndStretchSpringSettings`

Bộ value tune được của SquashAndStretchSpring — dùng cho cả inline lẫn SO.  

```csharp
public float Amount;
public Vector3 Axis;
public float DampingRatio;
public float Duration;
public float Frequency;
public static SquashAndStretchSpringSettings Default { get; }
public Vector3 Delta();
```

- `Delta` — Delta scale: +Amount theo Axis, -Amount/2 ở 2 trục vuông góc (giữ thể tích xấp xỉ).

#### class `WiggleFeedback`

*: JuiceFeedback*  
Wiggle localPosition (dao động sin mượt) quanh vị trí gốc trong Duration rồi trả về gốc. Target luôn inline (scene-ref); thông số qua JuiceParam (Inline hoặc Shared SO).  

```csharp
public WiggleFeedback();
public JuiceParam<WiggleSettings> Settings;
public Transform Target;
public float PlaybackDuration { get; }
protected internal virtual void Initialize(Juicer owner);
protected internal virtual UniTask PlayAsync(JuicePlayContext context, CancellationToken token);
protected internal virtual void Stop();
```

#### struct `WiggleSettings`

Bộ value tune được của Wiggle — dùng cho cả inline lẫn SO. Dao động sin mượt quanh vị trí gốc, lệch pha 120° giữa 3 trục để trông tự nhiên.  

```csharp
public Vector3 Amplitude;
public float Duration;
public float Frequency;
public static WiggleSettings Default { get; }
```

### namespace `VahTyah.JuiceEditor`

#### class `JuiceCurve3Drawer`

*: PropertyDrawer*  
Drawer cho JuiceCurve3 — vẽ 1 dòng gọn kiểu Particle System: X [curve] Y [curve] Z [curve], mỗi trục một màu (X đỏ, Y xanh lá, Z xanh dương). Cột chia đúng công thức multi-field của Unity nên thẳng hàng Vector3 field. Phần vẽ tách ra DrawCurves để JuiceEasingDrawer tái dùng (không double-prefix).  

```csharp
public JuiceCurve3Drawer();
public static void DrawCurves(Rect body, SerializedProperty curve3Property);
public virtual float GetPropertyHeight(SerializedProperty property, GUIContent label);
public virtual void OnGUI(Rect position, SerializedProperty property, GUIContent label);
```

- `DrawCurves` — Vẽ 3 cột X/Y/Z (curve màu) trong rect body cho sẵn — dùng lại bởi JuiceEasingDrawer.

#### class `JuiceEasing1Drawer`

*: PropertyDrawer*  
Drawer cho JuiceEasing1 (scalar) — nút chọn mode (Ease / Curve) đặt ở CUỐI CỘT LABEL, vùng value giữ full-width. Ease → popup enum Ease của LitMotion. Curve → MỘT AnimationCurve. Mirror JuiceEasingDrawer nhưng 1 curve thay vì 3.  

```csharp
public JuiceEasing1Drawer();
public virtual float GetPropertyHeight(SerializedProperty property, GUIContent label);
public virtual void OnGUI(Rect position, SerializedProperty property, GUIContent label);
```

#### class `JuiceEasingDrawer`

*: PropertyDrawer*  
Drawer cho JuiceEasing — kiểu Particle System nhưng nút chọn mode (Ease / Curve) đặt ở CUỐI CỘT LABEL (khoảng trống sau chữ label), nên vùng value giữ full-width và thẳng hàng Vector3 field ở trên. Ease → popup enum Ease của LitMotion. Curve → 3 curve X/Y/Z (JuiceCurve3Drawer.DrawCurves).  

```csharp
public JuiceEasingDrawer();
public virtual float GetPropertyHeight(SerializedProperty property, GUIContent label);
public virtual void OnGUI(Rect position, SerializedProperty property, GUIContent label);
```

#### struct `JuiceFeedbackSetupContext`

Ngữ cảnh truyền cho provider: player, instance feedback, SerializedProperty của phần tử, và index.  

```csharp
public JuiceFeedbackSetupContext(Juicer owner, JuiceFeedback feedback, SerializedProperty element, int index);
public readonly SerializedProperty Element;
public readonly JuiceFeedback Feedback;
public readonly int Index;
public readonly Juicer Owner;
```

#### abstract class `JuiceFeedbackSetupProvider`

Cung cấp UI "setup" (nút mở EditorWindow, thao tác author-time…) cho MỘT loại feedback — sống hoàn toàn trong Editor nên feedback Runtime không mang theo code authoring. JuicerEditor gom mọi provider qua TypeCache (key theo FeedbackType) và gọi OnSetupGUI ngay dưới nội dung của feedback tương ứng khi nó đang mở. Thêm setup cho feedback mới = viết một subclass ngắn, không đụng Runtime.  

```csharp
protected JuiceFeedbackSetupProvider();
public Type FeedbackType { get; }
public abstract void OnSetupGUI(in JuiceFeedbackSetupContext context);
```

- `FeedbackType` — Loại feedback mà provider này phục vụ (vd typeof(ShatterFeedback)).
- `OnSetupGUI` — Vẽ UI setup (thường là một nút mở EditorWindow). Được gọi trong layout Inspector của Juicer.

#### class `JuiceParamDrawer`

*: PropertyDrawer*  
Drawer chung cho MỌI JuiceParam<T> (Unity 2020.1+ áp drawer generic qua typeof(JuiceParam<>)). Không có toggle: hàng đầu là object field Config (SO). Chưa gán SO → vẽ field Inline (chỉnh ngay). Đã gán SO → vẽ thẳng các field Value của SO (edit ghi vào asset, thấy live khi play). Dùng làm fallback khi JuiceParam KHÔNG nằm trong BoxGroup (group thì JuicerEditor gắn SO lên header).  

```csharp
public JuiceParamDrawer();
public virtual float GetPropertyHeight(SerializedProperty property, GUIContent label);
public virtual void OnGUI(Rect position, SerializedProperty property, GUIContent label);
```

#### class `JuicerEditor`

*: DeferredEditor*  
Custom editor cho Juicer. Dùng lại UI BoxGroup của com.vahtyah.inspector giống ModuleConfig: mỗi feedback là một box (nền bo góc + header band), accent màu theo type, mở ra viền + overlay accent. Thao tác đổi mảng được hoãn (_deferred) chạy sau khi layout đóng. Play/Stop tái dùng ButtonDrawer.  

```csharp
public JuicerEditor();
public virtual void OnInspectorGUI();
public virtual bool RequiresConstantRepaint();
```

#### class `ShatterFractureWindow`

*: EditorWindow*  
Cửa sổ fracture cho ShatterFeedback (route 1): cắt mesh nguyên ở Root thành các mesh con Shatter-ready. Thuật toán Voronoi theo tam giác — sinh K seed ngẫu nhiên trong bounds, gán mỗi tam giác cho seed gần nhất (theo centroid), gom thành K mesh con. Pivot mỗi mảnh đặt tại centroid nên ghép lại TRÙNG mesh gốc và Shatter văng theo hướng từ tâm ra. Mesh gốc trên Root bị ẩn (renderer.enabled = false) — pieces thay thế. Toàn bộ nằm trong Editor: feedback Runtime không mang code authoring. Mọi thao tác đều qua Undo.  

```csharp
public ShatterFractureWindow();
public static void Open(Transform root);
```

- `Open` — Mở cửa sổ fracture cho root cho trước (điền sẵn ô Root).

