# 🎮 Full Game Improvement Plan
## Car Parking Jam Puzzle — Complete Feature Breakdown

> **READ CAREFULLY BEFORE APPLYING** — Nothing is changed yet. This is the plan. You approve → we execute.

---

## 📋 What We're Building (Summary)

| # | Feature | Status |
|---|---------|--------|
| 1 | Save/Load bug check & popup animation | Plan Ready |
| 2 | Game Over popup — animated sequence (one-by-one) | Plan Ready |
| 3 | Level Complete popup — animated sequence (one-by-one) | Plan Ready |
| 4 | Universal Animation Script (10+ animations, Singleton) | Plan Ready |
| 5 | FPS Drop analysis + fixes | Plan Ready |
| 6 | Game feel ideas (camera shake, juice) | Plan Ready |
| 7 | Visual/Lighting improvements (no perf cost) | Plan Ready |

---

## ✅ PART 1 — Save/Load Script: Bug Check

**After reading `SaveData.cs` and `GameDataLoader.cs`**, here are the bugs and issues found:

### Bugs Found:

| Bug | Location | Problem | Fix |
|-----|----------|---------|-----|
| 🔴 Double Save on level complete | `ParkingGameManager.cs` L59-63 AND `LevelManager.cs` L295-300 | Level is saved TWICE — once in `ParkingGameManager` and once in `LevelManager.CompleteLevel()` | Remove the save from `ParkingGameManager`, keep only in `LevelManager` |
| 🔴 `PowerUps.Start()` calls `SaveData.Instance.Load()` redundantly | `PowerUps.cs` L214 | `SaveData` already loads in its own `Awake()`. Calling `Load()` again in Start resets any in-session changes | Remove the `Load()` call in `PowerUps.Start()` |
| 🟡 `SpawnPassengers.Start()` calls `PassengerToSpawnBasedOnCarSpawn()` | `SpawnPassengers.cs` L56 | `LevelManager.LevelLoadSequence()` already calls this via `StartCoroutine`. Calling it again in `Start()` causes double passenger spawning on scene load | Remove the `Start()` call or add a guard flag |
| 🟡 No null check before popup show | `ParkingGameManager.cs` L83 | `UIPopupManager.Instance.ShowPopup()` can throw if UIPopupManager is null in certain load orders | Add null check |
| 🟡 `GameDataLoader` never refreshes after save | `GameDataLoader.cs` | `LoadAllData()` is only called in `Start()`. Currency/level UI won't update when rewards are collected mid-session | Call `LoadAllData()` after save events, or make it event-driven |

### Save/Load is otherwise solid ✅
- JSON persistence works correctly
- Singleton pattern is correct  
- DontDestroyOnLoad is set
- Error handling (try/catch) exists

---

## ✅ PART 2 — Game Over Popup: Animated Sequence

### Current state:
`GameOverPopup.cs` is very basic — just buttons, no animation, no sequence.

### New Design:
```
Panel appears (scale bounce) 
  ↓ 0.3s delay
"GAME OVER" title slides/fades in
  ↓ 0.4s delay  
Reward / score text pops in (punch scale)
  ↓ 0.5s delay
[Watch Video] button slides up
  ↓ 0.3s delay
[Play Again] button slides up
```

### New Script: `GameOverPopup.cs` (REPLACE existing)

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class GameOverPopup : MonoBehaviour
{
    [Header("Panel")]
    public RectTransform panel;

    [Header("Elements (assign in order)")]
    public RectTransform titleText;       // "GAME OVER"
    public RectTransform rewardText;      // "Better luck next time!" or score
    public RectTransform watchVideoBtn;   // Watch Video button
    public RectTransform playAgainBtn;    // Play Again / Next button

    [Header("Timing")]
    public float panelDelay    = 0.0f;
    public float titleDelay    = 0.3f;
    public float rewardDelay   = 0.6f;
    public float videoDelay    = 1.0f;
    public float playDelay     = 1.35f;

    private void OnEnable()
    {
        // Reset all elements to hidden state
        HideAll();
        StartCoroutine(PlaySequence());
    }

    private void HideAll()
    {
        if (panel)        { panel.localScale = Vector3.zero; }
        if (titleText)    { titleText.localScale = Vector3.zero; }
        if (rewardText)   { rewardText.localScale = Vector3.zero; }
        if (watchVideoBtn){ watchVideoBtn.anchoredPosition += new Vector2(0, -200f); watchVideoBtn.localScale = Vector3.zero; }
        if (playAgainBtn) { playAgainBtn.anchoredPosition += new Vector2(0, -200f);  playAgainBtn.localScale = Vector3.zero; }
    }

    private IEnumerator PlaySequence()
    {
        // Panel bounce in
        yield return new WaitForSecondsRealtime(panelDelay);
        if (panel) panel.DOScale(1f, 0.45f).SetEase(Ease.OutBack).SetUpdate(true);

        // Title pop
        yield return new WaitForSecondsRealtime(titleDelay);
        if (titleText) titleText.DOScale(1f, 0.4f).SetEase(Ease.OutBack).SetUpdate(true);

        // Reward text pop
        yield return new WaitForSecondsRealtime(rewardDelay - titleDelay);
        if (rewardText) rewardText.DOScale(1f, 0.35f).SetEase(Ease.OutBack).SetUpdate(true);

        // Watch Video button slide up
        yield return new WaitForSecondsRealtime(videoDelay - rewardDelay);
        if (watchVideoBtn)
        {
            watchVideoBtn.DOScale(1f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
            watchVideoBtn.DOAnchorPosY(watchVideoBtn.anchoredPosition.y + 200f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
        }

        // Play Again button slide up
        yield return new WaitForSecondsRealtime(playDelay - videoDelay);
        if (playAgainBtn)
        {
            playAgainBtn.DOScale(1f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
            playAgainBtn.DOAnchorPosY(playAgainBtn.anchoredPosition.y + 200f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
        }
    }

    // === BUTTONS ===
    public void OnPlayAgain()
    {
        SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);
        Time.timeScale = 1;
        UIPopupManager.Instance.ClosePopup();
        LevelManager.Instance.RestartLevel();
    }

    public void OnWatchVideo()
    {
        SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);
        // TODO: Show rewarded ad here
        // On ad complete → call OnPlayAgain()
        Debug.Log("Watch video ad triggered");
    }

    public void QuitGame()
    {
        Application.Quit();
    }
}
```

### Unity Setup:
1. On your Game Over Canvas, create this hierarchy:
   ```
   GameOverCanvas
   └── Panel (add RectTransform, Image)
       ├── TitleText (TextMeshPro) — "GAME OVER"
       ├── RewardText (TextMeshPro) — "Try again!"
       ├── WatchVideoButton (Button)
       └── PlayAgainButton (Button)
   ```
2. Attach `GameOverPopup.cs` to the **Canvas root**
3. Drag each UI element into the inspector slots
4. Make sure `UIBase.cs` is also on this Canvas (for Show/Hide via UIPopupManager)

---

## ✅ PART 3 — Level Complete Popup: Animated Sequence

### New Design:
```
Panel bounces in
  ↓
"LEVEL COMPLETE!" title pops (with star emoji ⭐)
  ↓
"Reward: +100 coins 🪙" counter animates
  ↓
[2x Reward — Watch Video] button slides up
  ↓
[Next Level] button slides up
```

### New Script: `LevelCompletePopup.cs` (REPLACE existing)

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;
using TMPro;
using DG.Tweening;

public class LevelCompletePopup : MonoBehaviour
{
    [Header("Panel")]
    public RectTransform panel;

    [Header("Elements")]
    public RectTransform titleText;          // "LEVEL COMPLETE!"
    public RectTransform rewardText;         // "+100 Coins"
    public TextMeshProUGUI rewardCountTxt;   // Shows coin count
    public RectTransform doubleRewardBtn;    // "2x Reward - Watch Video"
    public RectTransform nextBtn;            // "Next Level"

    [Header("Rewards")]
    public int coinReward = 100;

    [Header("Timing")]
    public float titleDelay   = 0.3f;
    public float rewardDelay  = 0.65f;
    public float btn1Delay    = 1.05f;
    public float btn2Delay    = 1.4f;

    private void OnEnable()
    {
        HideAll();
        StartCoroutine(PlaySequence());
    }

    private void HideAll()
    {
        if (panel)           panel.localScale = Vector3.zero;
        if (titleText)       titleText.localScale = Vector3.zero;
        if (rewardText)      rewardText.localScale = Vector3.zero;
        if (doubleRewardBtn) { doubleRewardBtn.localScale = Vector3.zero; doubleRewardBtn.anchoredPosition += new Vector2(0, -200f); }
        if (nextBtn)         { nextBtn.localScale = Vector3.zero;         nextBtn.anchoredPosition += new Vector2(0, -200f); }
    }

    private IEnumerator PlaySequence()
    {
        // Panel
        if (panel) panel.DOScale(1f, 0.45f).SetEase(Ease.OutBack).SetUpdate(true);

        // Title
        yield return new WaitForSecondsRealtime(titleDelay);
        if (titleText) titleText.DOScale(1f, 0.4f).SetEase(Ease.OutBack).SetUpdate(true);

        // Reward counter
        yield return new WaitForSecondsRealtime(rewardDelay - titleDelay);
        if (rewardText) rewardText.DOScale(1f, 0.35f).SetEase(Ease.OutBack).SetUpdate(true);
        if (rewardCountTxt) AnimateCoinCount(0, coinReward, 0.8f);

        // Double reward button
        yield return new WaitForSecondsRealtime(btn1Delay - rewardDelay);
        if (doubleRewardBtn)
        {
            doubleRewardBtn.DOScale(1f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
            doubleRewardBtn.DOAnchorPosY(doubleRewardBtn.anchoredPosition.y + 200f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
        }

        // Next button
        yield return new WaitForSecondsRealtime(btn2Delay - btn1Delay);
        if (nextBtn)
        {
            nextBtn.DOScale(1f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
            nextBtn.DOAnchorPosY(nextBtn.anchoredPosition.y + 200f, 0.3f).SetEase(Ease.OutBack).SetUpdate(true);
        }
    }

    private void AnimateCoinCount(int from, int to, float duration)
    {
        int current = from;
        float elapsed = 0f;
        StartCoroutine(CountUp());

        IEnumerator CountUp()
        {
            while (elapsed < duration)
            {
                elapsed += Time.unscaledDeltaTime;
                current = (int)Mathf.Lerp(from, to, elapsed / duration);
                if (rewardCountTxt) rewardCountTxt.text = "+" + current + " 🪙";
                yield return null;
            }
            if (rewardCountTxt) rewardCountTxt.text = "+" + to + " 🪙";
        }
    }

    // === BUTTONS ===
    public void OnNextLevel()
    {
        SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);
        SaveData.Instance.AddCurrency(coinReward);
        Time.timeScale = 1;
        UIPopupManager.Instance.ClosePopup();
        LevelManager.Instance.CompleteLevel();
    }

    public void OnDoubleReward()
    {
        SoundManager.Instance.PlaySound(SoundManager.SoundName.Click);
        // TODO: Show rewarded ad → on complete give 2x coins
        // SaveData.Instance.AddCurrency(coinReward * 2);
        Debug.Log("Double reward ad triggered");
    }
}
```

### Unity Setup (same structure):
```
LevelCompleteCanvas
└── Panel
    ├── TitleText — "LEVEL COMPLETE! ⭐"
    ├── RewardText — "+100 Coins"
    ├── DoubleRewardButton — "2× Watch Video"
    └── NextButton — "Next Level ▶"
```

---

## ✅ PART 4 — Universal Animation Script (10+ Animations)

### New File: `GameAnimator.cs`

This is a **Singleton** script. Add it once to any persistent GameObject. Call it from anywhere.

```csharp
using UnityEngine;
using UnityEngine.UI;
using DG.Tweening;
using System.Collections;

/// <summary>
/// Universal Animation Script — 10+ animations.
/// Singleton. Add once to scene. Call from any script via GameAnimator.Instance
/// Also works as a Component: add to any GameObject and call methods directly.
/// </summary>
public class GameAnimator : MonoBehaviour
{
    public static GameAnimator Instance { get; private set; }

    void Awake()
    {
        if (Instance == null) { Instance = this; DontDestroyOnLoad(gameObject); }
        else Destroy(gameObject);
    }

    // ─────────────────────────────────────────────
    // 1. BUTTON JIGGLE (perfect for idle buttons)
    // ─────────────────────────────────────────────
    public void ButtonJiggle(Transform t, float strength = 15f, float duration = 0.5f)
    {
        t.DOKill();
        t.DOShakeRotation(duration, new Vector3(0, 0, strength), 10, 90f, false)
         .SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 2. WOBBLE (rubbery scale pulse)
    // ─────────────────────────────────────────────
    public void Wobble(Transform t, float amount = 0.15f, float duration = 0.4f)
    {
        t.DOKill();
        Vector3 original = t.localScale;
        Sequence s = DOTween.Sequence().SetUpdate(true);
        s.Append(t.DOScale(original * (1f + amount), duration * 0.3f).SetEase(Ease.OutQuad));
        s.Append(t.DOScale(original * (1f - amount * 0.5f), duration * 0.3f).SetEase(Ease.OutQuad));
        s.Append(t.DOScale(original, duration * 0.4f).SetEase(Ease.OutElastic));
    }

    // ─────────────────────────────────────────────
    // 3. PUNCH SCALE (tap feedback)
    // ─────────────────────────────────────────────
    public void PunchScale(Transform t, float punch = 0.3f, float duration = 0.35f)
    {
        t.DOKill();
        t.DOPunchScale(Vector3.one * punch, duration, 5, 0.5f).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 4. CONTINUOUS ROTATION (loading, coins, stars)
    // ─────────────────────────────────────────────
    public Tweener StartSpin(Transform t, float speed = 360f, bool clockwise = true)
    {
        t.DOKill();
        float dir = clockwise ? -speed : speed;
        return t.DORotate(new Vector3(0, 0, dir), 1f, RotateMode.FastBeyond360)
                .SetEase(Ease.Linear)
                .SetLoops(-1, LoopType.Restart)
                .SetUpdate(true);
    }

    public void StopSpin(Transform t) => t.DOKill();

    // ─────────────────────────────────────────────
    // 5. FLOAT UP-DOWN (idle hover for icons/rewards)
    // ─────────────────────────────────────────────
    public Tweener StartFloat(Transform t, float height = 12f, float speed = 1.2f)
    {
        t.DOKill();
        return t.DOLocalMoveY(t.localPosition.y + height, speed)
                .SetEase(Ease.InOutSine)
                .SetLoops(-1, LoopType.Yoyo)
                .SetUpdate(true);
    }

    public void StopFloat(Transform t) => t.DOKill();

    // ─────────────────────────────────────────────
    // 6. FADE IN (smooth alpha in)
    // ─────────────────────────────────────────────
    public void FadeIn(CanvasGroup cg, float duration = 0.4f)
    {
        cg.alpha = 0;
        cg.DOFade(1f, duration).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 7. FADE OUT (smooth alpha out)
    // ─────────────────────────────────────────────
    public void FadeOut(CanvasGroup cg, float duration = 0.3f, System.Action onComplete = null)
    {
        cg.DOFade(0f, duration).SetUpdate(true).OnComplete(() => onComplete?.Invoke());
    }

    // ─────────────────────────────────────────────
    // 8. SHAKE (camera or UI shake — great for game over/hit)
    // ─────────────────────────────────────────────
    public void ShakeUI(RectTransform rt, float strength = 20f, float duration = 0.4f)
    {
        rt.DOKill();
        rt.DOShakeAnchorPos(duration, strength, 20, 90f).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 9. BOUNCE IN (popup entrance)
    // ─────────────────────────────────────────────
    public void BounceIn(Transform t, float duration = 0.45f)
    {
        t.localScale = Vector3.zero;
        t.DOScale(1f, duration).SetEase(Ease.OutBack).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 10. BOUNCE OUT (popup exit)
    // ─────────────────────────────────────────────
    public void BounceOut(Transform t, float duration = 0.3f, System.Action onComplete = null)
    {
        t.DOScale(0f, duration).SetEase(Ease.InBack).SetUpdate(true)
         .OnComplete(() => onComplete?.Invoke());
    }

    // ─────────────────────────────────────────────
    // 11. COIN COLLECT ARC (coin flies to wallet icon)
    // ─────────────────────────────────────────────
    public void CoinFlyTo(Transform coin, Vector3 target, float duration = 0.6f, System.Action onComplete = null)
    {
        coin.DOMove(target, duration).SetEase(Ease.InBack).SetUpdate(true)
            .OnComplete(() => { onComplete?.Invoke(); Destroy(coin.gameObject); });
    }

    // ─────────────────────────────────────────────
    // 12. RAINBOW / COLOR PULSE (for rewards, stars)
    // ─────────────────────────────────────────────
    public void ColorPulse(Graphic graphic, Color colorA, Color colorB, float speed = 0.5f)
    {
        graphic.DOKill();
        graphic.DOColor(colorB, speed).SetEase(Ease.InOutSine)
               .SetLoops(-1, LoopType.Yoyo).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 13. SLIDE IN FROM BOTTOM
    // ─────────────────────────────────────────────
    public void SlideInFromBottom(RectTransform rt, float distance = 300f, float duration = 0.4f)
    {
        rt.anchoredPosition += new Vector2(0, -distance);
        rt.DOAnchorPosY(rt.anchoredPosition.y + distance, duration)
          .SetEase(Ease.OutBack).SetUpdate(true);
    }

    // ─────────────────────────────────────────────
    // 14. NUMBER COUNT UP (score, coins)
    // ─────────────────────────────────────────────
    public void CountUp(TMPro.TextMeshProUGUI txt, int from, int to, float duration, string prefix = "", string suffix = "")
    {
        DOTween.To(() => from, x => { from = x; txt.text = prefix + x + suffix; }, to, duration)
               .SetEase(Ease.OutQuart).SetUpdate(true);
    }
}
```

### How to Set Up in Unity:
1. Create an **empty GameObject** in your scene called `GameAnimator`
2. Attach `GameAnimator.cs` to it
3. That's it! It's a Singleton with `DontDestroyOnLoad`

### How to Use on any Button:
```csharp
// On button click:
GameAnimator.Instance.PunchScale(transform);
GameAnimator.Instance.ButtonJiggle(transform);

// For idle floating animation on a reward icon:
GameAnimator.Instance.StartFloat(rewardIcon.transform);

// For spinning coin:
GameAnimator.Instance.StartSpin(coinIcon.transform);
```

---

## ✅ PART 5 — FPS Drop Analysis

### Scripts Causing FPS Issues:

| Priority | Script | Problem | Fix |
|----------|--------|---------|-----|
| 🔴 HIGH | `ParkingGameManager.cs` Line 32-44 | **`Update()` runs a full for-loop on ALL passengers EVERY FRAME** using `Vector3.MoveTowards` with string update | Move passenger lerp to a dedicated coroutine, skip if passenger is already at target |
| 🔴 HIGH | `ParkingGameManager.cs` Line 34-37 | **`TotalPassengerCount.text` is set EVERY FRAME** even when count hasn't changed | Cache the last value and only update text when it changes |
| 🟡 MED | `SpawnCars.cs` `OnDrawGizmos()` | Gizmo loop runs every editor frame with double nested loop — fine in Editor, but leave as-is | Keep as-is (only Editor) |
| 🟡 MED | `CarMover.cs` `MoveRoutine()` | Every car's path uses `SpawnCars.Instance.GetSlotAt()` which does a **LINQ `.Find()`** on every step | Cache the result — call `GetSlotAt` once at the start, not per frame |
| 🟡 MED | `PowerUps.cs` `ProcessFillCarRoutine()` | Calls `GetComponent<CarMover>()` repeatedly inside coroutine loop | Cache reference before loop |
| 🟢 LOW | `SpawnCars.cs` Line 606 | `.OrderBy()` LINQ on every level load | Fine — only on level load, not per frame |

### The Main FPS Killer — Fix:

**Current Code (BAD):**
```csharp
void Update()
{
    // THIS RUNS EVERY FRAME FOR EVERY PASSENGER
    TotalPassengerCount.text = waitingPassengers.Count.ToString(); // string alloc every frame!
    
    for (int i = 0; i < waitingPassengers.Count; i++)
    {
        Vector3 nextPos = lineStart + (Vector3.right * (i * stepSize));
        waitingPassengers[i].transform.position = Vector3.MoveTowards(...);
    }
    // ...
}
```

**Fixed Code (GOOD):**
```csharp
private int _lastPassengerCount = -1;

void Update()
{
    // Only update text when count changes
    if (waitingPassengers.Count != _lastPassengerCount)
    {
        _lastPassengerCount = waitingPassengers.Count;
        if (TotalPassengerCount != null)
            TotalPassengerCount.text = waitingPassengers.Count.ToString();
    }
    
    // Only loop if passengers exist AND not loading
    if (gameOver || isLevelLoading || waitingPassengers.Count == 0) return;
    
    for (int i = 0; i < waitingPassengers.Count; i++)
    {
        if (waitingPassengers[i] == null) continue;
        Vector3 nextPos = lineStart + (Vector3.right * (i * stepSize));
        // Early exit if already close enough
        if (Vector3.SqrMagnitude(waitingPassengers[i].transform.position - nextPos) < 0.001f) continue;
        waitingPassengers[i].transform.position = Vector3.MoveTowards(
            waitingPassengers[i].transform.position, nextPos, Time.deltaTime * walkSpeedOfPassengers);
    }
    
    // Win check — only when NOT already over
    if (waitingPassengers.Count == 0)
    {
        gameOver = true;
        // ...rest of win logic
    }
}
```

### Additional Quick Fixes:
- **Disable Debug.Log in production** — every `Debug.Log()` in `SaveData`, `GameDataLoader`, `LevelManager` creates garbage. Comment them out in build.
- **Object Pool is already used ✅** — Good, you're not instantiating/destroying every frame
- **DOTween.SetRecyclable(true)** — Already used in `UIPopupAnimator` ✅

---

## ✅ PART 6 — Game Feel Ideas (Make It "One More Level!")

Your game already has: camera, car movement, passenger walk. Here's what will make it addictive:

| Idea | How It Works | Impact |
|------|-------------|--------|
| 🎯 **Camera Shake on car park** | When a car parks, shake camera slightly (0.1s, small) | BIG feel |
| 🎯 **Camera Shake on game over** | Strong shake + red flash overlay | Feels dramatic |
| 💥 **Particle burst when passenger boards** | Small confetti/sparkle at passenger position | Satisfying |
| ⭐ **Star rating on level complete** | Show 1/2/3 stars based on moves or time | Replayability |
| 🪙 **Coin fly animation** | Coins visually fly to the wallet UI on level complete | Juicy |
| 📳 **Haptic feedback** | `Handheld.Vibrate()` on park/game over | Mobile feel |
| 🔊 **Sound variation** | Pitch randomize `±0.1f` on passenger board sound | No repetition |
| 🎨 **Idle button pulse** | Power-up buttons gently pulse when player is stuck >5s | Hints player |
| 🌟 **Combo streak** | If 3+ passengers board rapidly, show "COMBO!" text burst | Excitement |
| 🚗 **Car arrival animation** | When a new car parks, it does a slight bounce settle | Polish |

### Camera Shake — Ready to Use Script:

```csharp
// Add this to your main Camera
using UnityEngine;
using DG.Tweening;

public class CameraShake : MonoBehaviour
{
    public static CameraShake Instance;
    
    void Awake()
    {
        Instance = this;
    }
    
    public void Shake(float duration = 0.3f, float strength = 0.3f, int vibrato = 10)
    {
        transform.DOShakePosition(duration, strength, vibrato, 90f, false, true);
    }
    
    // Presets
    public void ShakeLight()  => Shake(0.15f, 0.1f, 8);   // car parks
    public void ShakeHard()   => Shake(0.4f,  0.5f, 15);  // game over
    public void ShakeMedium() => Shake(0.25f, 0.25f, 10); // power-up
}
```

---

## ✅ PART 7 — Visual/Lighting Improvements (No FPS Cost)

Your game uses **URP (Universal Render Pipeline)** — confirmed by `New Universal Render Pipeline Asset.asset`.

### Why It Looks "Cheap" Right Now:
1. Default URP settings with no post-processing
2. Flat ambient lighting
3. No depth of field or bloom
4. Cars likely have unlit or default lit materials

### Fixes — All Zero FPS Cost:

#### A. Post Processing (URP Volume)
1. In your scene, add a **Global Volume** (GameObject → Volume → Global Volume)
2. Create a new **Volume Profile** asset
3. Add these overrides:
   - **Bloom**: Intensity `0.3`, Threshold `0.8`, Scatter `0.6` → gives glow to lights
   - **Color Adjustments**: Saturation `+15`, Contrast `+10` → punchier colors
   - **Vignette**: Intensity `0.2` → subtle edge darkening, cinematic
   - **Depth of Field** (Gaussian): Focus Distance `30`, Far Range `5` → background blur

#### B. Lighting Settings
- Set **Ambient Mode** to `Gradient` (Window → Rendering → Lighting)
- Sky color: `#87CEEB` (sky blue), Ground color: `#4A7C59` (grass green)
- This gives automatic color bounce onto all objects

#### C. Shadow Quality (URP Asset)
In `New Universal Render Pipeline Asset.asset`:
- Shadow Distance: `50` (reduce from default 1000 — massive perf win)
- Shadow Cascades: `2` (was probably 4)
- Soft Shadows: `ON` → looks much better, minimal cost in URP

#### D. Car Materials
- Give cars a **URP Lit material** with:
  - Metallic: `0.4`, Smoothness: `0.7`
  - This gives them the shiny car look automatically

#### E. Ambient Occlusion (SSAO)
In URP Renderer → Add Renderer Feature → `Screen Space Ambient Occlusion`
- Intensity: `0.5`, Radius: `0.5`
- Makes objects look grounded, removes "floating" look — very cheap in URP

---

## 📂 New Files to Create

| File | Location | Purpose |
|------|----------|---------|
| `GameAnimator.cs` | `Assets/Script/AnimationScript/` | Universal animation singleton |
| `CameraShake.cs` | `Assets/Script/` | Camera shake for game feel |
| `GameOverPopup.cs` | `Assets/Script/UIManager/` | Replaces existing (full rewrite) |
| `LevelCompletePopup.cs` | `Assets/Script/UIManager/` | Replaces existing (full rewrite) |

## 📝 Files to Modify (Bug Fixes Only)

| File | Change |
|------|--------|
| `ParkingGameManager.cs` | Fix FPS: cache text update, add null checks, remove duplicate level save |
| `PowerUps.cs` | Remove redundant `SaveData.Instance.Load()` in Start |
| `SpawnPassengers.cs` | Remove duplicate `PassengerToSpawnBasedOnCarSpawn()` call in Start |

---

## ❓ Open Questions (Please Answer Before We Start)

> [!IMPORTANT]
> **Q1**: Should the "Watch Video" button on Game Over actually call an ad SDK (like Unity Ads / AdMob), or just simulate it for now (print a log)?

> [!IMPORTANT]
> **Q2**: For Level Complete — do you want a **star rating** (1/2/3 stars) shown in the popup based on anything (moves used, time, powerups used)?

> [!NOTE]
> **Q3**: The current `LevelCompletePopup` gives 5 coins. The new design uses 100 coins. Which amount do you want?

> [!NOTE]
> **Q4**: Camera shake — do you want the 3D camera to shake, or is your camera a fixed overhead view? (This affects the shake script slightly)

> [!NOTE]
> **Q5**: Should `GameAnimator` and `CameraShake` be on the same GameObject, or separate?
