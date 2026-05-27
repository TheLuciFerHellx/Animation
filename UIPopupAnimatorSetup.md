# UI Popup Animator – DOTween Integration

## Overview
This guide bundles the **`UIPopupAnimator.cs`** script and all required setup steps to animate only the *car* UI element inside each popup (Profile, Settings, DailyReward, Shop, Pause, LevelComplete, GameOver). The background (black overlay) remains static while the car slides in/out with a smooth squash‑stretch effect.

---

## 1️⃣ Script (copy this into a new C# file)

```csharp
using UnityEngine;
using UnityEngine.UI;
using DG.Tweening;

/// <summary>
/// Handles opening/closing animations for UI pop‑ups.
/// Only the car UI element (assigned via the inspector) is animated.
/// The black background stays static.
/// </summary>
[RequireComponent(typeof(CanvasGroup))]
public class UIPopupAnimator : MonoBehaviour
{
    // -------------------------------------------------------------
    // Inspector fields – configure per popup
    // -------------------------------------------------------------
    [Header("Car UI Element (the thing that moves")] 
    [Tooltip("Assign the RectTransform of the car image inside the popup.")]
    public RectTransform carRect;

    [Header("Animation Settings")]
    [Tooltip("How far (in UI units) the car starts off‑screen on the left.")]
    public float offscreenOffset = 500f; // can be tweaked per popup

    [Tooltip("Duration of the opening animation.")]
    public float openDuration = 0.6f;

    [Tooltip("Duration of the closing animation.")]
    public float closeDuration = 0.4f;

    [Tooltip("Amount of vertical squash when the car lands (1 = no squash).")]
    [Range(0.5f, 1f)]
    public float squashScale = 0.9f;

    // -------------------------------------------------------------
    // Private state
    // -------------------------------------------------------------
    private CanvasGroup canvasGroup;
    private Vector3 _originalPos;
    private bool _isOpen = false;

    private void Awake()
    {
        canvasGroup = GetComponent<CanvasGroup>();
        if (carRect == null)
        {
            Debug.LogError("[UIPopupAnimator] Car RectTransform not assigned.");
            enabled = false;
            return;
        }
        // Store the position where the car should finally rest (centre of popup)
        _originalPos = carRect.anchoredPosition;
        // Ensure the popup starts hidden
        canvasGroup.alpha = 0;
        carRect.anchoredPosition = new Vector2(-offscreenOffset, _originalPos.y);
    }

    // -------------------------------------------------------------
    // Public API – hook these up to UI Buttons
    // -------------------------------------------------------------
    public void Open()
    {
        if (_isOpen) return;
        _isOpen = true;
        canvasGroup.blocksRaycasts = true;
        // Fade in background (black overlay)
        canvasGroup.DOFade(1f, openDuration * 0.5f).SetEase(Ease.OutQuad);
        // Slide the car from left → centre
        carRect.DOAnchorPosX(_originalPos.x, openDuration)
                .SetEase(Ease.OutBack, 2) // gives a slight overshoot
                .OnComplete(() =>
                {
                    // Slight squash‑stretch when the car lands
                    Sequence seq = DOTween.Sequence();
                    seq.Append(carRect.DOScaleY(squashScale, openDuration * 0.2f))
                       .Join(carRect.DOScaleX(1f / squashScale, openDuration * 0.2f))
                       .Append(carRect.DOScaleY(1f, openDuration * 0.2f))
                       .Join(carRect.DOScaleX(1f, openDuration * 0.2f));
                });
    }

    public void Close()
    {
        if (!_isOpen) return;
        _isOpen = false;
        canvasGroup.blocksRaycasts = false;
        // Fade out background quickly
        canvasGroup.DOFade(0f, closeDuration * 0.5f).SetEase(Ease.InQuad);
        // Slide the car out to the right side of the screen
        float exitPosX = Screen.width + offscreenOffset; // far enough to disappear
        carRect.DOAnchorPosX(exitPosX, closeDuration)
                .SetEase(Ease.InBack);
    }

    // Optional convenience – toggle open/close with a single call
    public void Toggle()
    {
        if (_isOpen) Close(); else Open();
    }
}
```

---

## 2️⃣ DOTween Installation
1. **Open Unity Package Manager** → **Add package from git URL**.
2. Paste: `https://github.com/Demigiant/dotween.git#2.1.21` (or the latest stable tag).
3. Click **Add** and let Unity download the package.
4. After import, go to **Tools → DOTween Utility Panel → Setup DOTween** and click **Setup DOTween** (this generates the required scripts).

---

## 3️⃣ Adding the Animator to a Popup
For each popup (Profile, Settings, DailyReward, Shop, Pause, LevelComplete, GameOver):
1. **Select the root GameObject** of the popup in the Hierarchy.
2. Press **Add Component → UIPopupAnimator**.
3. Drag the **Car** child (the UI element you want to animate) onto the **Car Rect** field in the inspector.
4. Adjust the following per‑popup if needed:
   - `Offscreen Offset` – how far the car starts left of the screen (default 500).
   - `Open Duration` / `Close Duration` – speed of the slide.
   - `Squash Scale` – how much vertical squash occurs on landing (0.9 looks nice).
5. Ensure the popup has a **CanvasGroup** component (the script adds it automatically via `RequireComponent`). This component controls the fade of the black overlay.

---

## 4️⃣ Hooking UI Buttons
1. **Open Button** (or any button that should show the popup):
   - In the Button component’s **On Click ()** list, drag the popup root object.
   - Choose **UIPopupAnimator → Open**.
2. **Close Button** (or the X at the top‑right):
   - Same steps, but select **Close**.
3. (Optional) **Toggle Button** – use the **Toggle** method if you want a single button to open/close.

---

## 5️⃣ Testing the Animation
1. Enter **Play Mode**.
2. Click the Open button → the car slides in from the left, overshoots slightly, then squashes and settles.
3. Click the Close button → the car slides out to the right and the background fades out.
4. Verify that the black overlay never moves – only the car moves.
5. Tweak the inspector values for each popup to achieve the visual feel you want (speed, offset, squash).

---

## 6️⃣ Common Pitfalls & Fixes
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Car doesn’t move, only fades | `carRect` not assigned | Drag the Car’s RectTransform onto the **Car Rect** field |
| Animation is jittery | Off‑screen offset too small, causing the car to start inside the view | Increase `offscreenOffset` (e.g., 600‑800) |
| Popup stays visible after Close | CanvasGroup blocksRaycasts still true | Ensure `Close()` sets `blocksRaycasts = false` (script does this) |
| DOTween errors in console | DOTween not set up | Run **Tools → DOTween Utility Panel → Setup DOTween** again |

---

## 7️⃣ Full Project Integration Checklist
- [ ] DOTween package installed and set up.
- [ ] `UIPopupAnimator.cs` placed under **Assets/Script**.
- [ ] Each popup GameObject has **UIPopupAnimator** attached.
- [ ] Car UI element assigned for every popup.
- [ ] Buttons wired to `Open`/`Close` (or `Toggle`).
- [ ] Play‑test all popups, adjust values as needed.
- [ ] Commit the new script and updated prefabs/scenes to version control.

---

### 🎉 You’re ready!
With the guide above you have **one markdown file** that contains both the **code** and the **full setup instructions** for animating every UI popup in your game using DOTween. Just follow the checklist and you’ll see the car UI glide in/out while the background stays perfectly still.

---

*If you need any further customisation (different easing, extra sound effects, chaining multiple UI elements, etc.), just let me know!*   

---

*Generated by Antigravity – your AI coding assistant.*   

---
