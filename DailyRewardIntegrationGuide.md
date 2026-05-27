# Daily Reward Integration Guide

## 1️⃣  Script – `DailyRewardManager.cs`

```csharp
using System;
using UnityEngine;

/// <summary>
/// Handles a 7‑day rotating daily reward that can be claimed once every 24 h.
/// Attach this script to any GameObject (e.g., a UI manager or the SaveManager).
/// </summary>
public class DailyRewardManager : MonoBehaviour
{
    // -----------------------------------------------------------------
    // 1️⃣  Define the rewards for each of the 7 days.
    //    You can edit the values in the Inspector or directly here.
    // -----------------------------------------------------------------
    [Serializable]
    public struct DailyReward
    {
        public string description;   // e.g. "100 Coins"
        public int    coinAmount;   // amount of coins to give
        public string powerupKey;    // must match a key in SaveData.powerupCounts
        public int    powerupAmount; // how many of that power‑up
    }

    // Example week‑long reward cycle – feel free to change the numbers.
    public DailyReward[] rewards = new DailyReward[7]
    {
        new DailyReward{ description="100 Coins",        coinAmount=100, powerupKey="", powerupAmount=0 },
        new DailyReward{ description="200 Coins",        coinAmount=200, powerupKey="", powerupAmount=0 },
        new DailyReward{ description="Fill‑Slot Power‑up",coinAmount=0,   powerupKey="FillSlot",    powerupAmount=1 },
        new DailyReward{ description="Rearrange Power‑up",coinAmount=0,   powerupKey="Rearrange", powerupAmount=1 },
        new DailyReward{ description="Helicopter Power‑up",coinAmount=0, powerupKey="Helicopter", powerupAmount=1 },
        new DailyReward{ description="300 Coins",        coinAmount=300, powerupKey="", powerupAmount=0 },
        new DailyReward{ description="500 Coins",        coinAmount=500, powerupKey="", powerupAmount=0 }
    };
    // -----------------------------------------------------------------

    private SaveData _save;   // reference to the SaveData component in the scene

    private void Awake()
    {
        _save = FindObjectOfType<SaveData>();
        if (_save == null)
        {
            Debug.LogError("[DailyReward] SaveData not found – disabling manager.");
            enabled = false;
            return;
        }
        if (rewards == null || rewards.Length != 7)
        {
            Debug.LogError("[DailyReward] You must define exactly 7 daily rewards.");
            enabled = false;
            return;
        }
        // Ensure the stored index/date are valid on startup
        UpdateDayIfNeeded();
    }

    // -----------------------------------------------------------------
    // Public method – call from a UI button "Collect Daily Reward"
    // -----------------------------------------------------------------
    public void TryCollectReward()
    {
        string today = DateTime.UtcNow.ToString("yyyy-MM-dd");
        // Has the player already taken today’s reward?
        if (_save.HasCollectedDailyReward(today))
        {
            Debug.Log("[DailyReward] Reward already claimed today.");
            return;
        }
        // Grab today’s reward based on the stored index (0‑6)
        int idx = _save.GetDailyRewardIndex();
        DailyReward reward = rewards[idx];
        // Grant the reward
        if (reward.coinAmount > 0) _save.AddCurrency(reward.coinAmount);
        if (!string.IsNullOrEmpty(reward.powerupKey))
            _save.AddPowerup(reward.powerupKey, reward.powerupAmount);
        Debug.Log($"[DailyReward] Granted: {reward.description}");
        // Remember that today’s reward was taken
        _save.CollectDailyReward(today);
    }

    // -----------------------------------------------------------------
    // Internal – checks if the calendar day changed and advances the index.
    // -----------------------------------------------------------------
    private void UpdateDayIfNeeded()
    {
        string today = DateTime.UtcNow.ToString("yyyy-MM-dd");
        string last  = _save.GetLastRewardDate(); // empty on first launch
        if (string.IsNullOrEmpty(last))
        {
            _save.SetLastRewardDate(today);
            _save.SetDailyRewardIndex(0);
            return;
        }
        if (!today.Equals(last, StringComparison.Ordinal))
        {
            int newIdx = (_save.GetDailyRewardIndex() + 1) % 7;
            _save.SetDailyRewardIndex(newIdx);
            _save.SetLastRewardDate(today);
            Debug.Log($"[DailyReward] New day – reward index now {newIdx}.");
        }
    }

    // -----------------------------------------------------------------
    // OPTIONAL: auto‑advance while the app stays open across midnight.
    // Uncomment if you need it.
    // -----------------------------------------------------------------
    // private float _timer = 0f;
    // private const float CheckInterval = 60f; // seconds
    // private void Update()
    // {
    //     _timer += Time.unscaledDeltaTime;
    //     if (_timer >= CheckInterval)
    //     {
    //         _timer = 0f;
    //         UpdateDayIfNeeded();
    //     }
    // }
}
```

## 2️⃣  How to set it up in your Unity project

1. **Create the script file**
   - In the Unity **Project** window go to `Assets/Script`.
   - Right‑click → **Create → C# Script** → name it **`DailyRewardManager`**.
   - Double‑click the file to open it in your IDE and replace its contents with the code block above.

2. **Add a GameObject to hold the manager**
   - In the **Hierarchy**, Create → **Empty** and name it **`DailyRewardManager`**.
   - Drag the newly created `DailyRewardManager` script onto this GameObject.
   - (Optional) Keep the GameObject active for the whole session – it does not need to be destroyed on load.

3. **Make sure the Save system exists**
   - You already have `SaveData.cs`. Ensure there is a **persist‑across‑scenes GameObject** (commonly called **SaveManager**) with the `SaveData` component attached. It must be present in the first scene.
   - `SaveData.Awake()` already calls `DontDestroyOnLoad`, so it will survive scene changes.

4. **Create a UI button for the player**
   - In your Canvas, add a **Button** and set its label to something like **"Collect Daily Reward"**.
   - In the **Button (Script)** component, click **+** under **On Click ()**.
   - Drag the **DailyRewardManager** GameObject from the Hierarchy into the object field.
   - From the dropdown select **DailyRewardManager → TryCollectReward**.

5. **Optional – show the current reward on the UI**
   - You can create a Text element that reads the reward description:
   ```csharp
   var dr = FindObjectOfType<DailyRewardManager>();
   int idx = dr ? dr.rewards[dr.GetComponent<DailyRewardManager>().GetComponent<DailyRewardManager>().GetDailyRewardIndex()] : 0;
   rewardText.text = dr.rewards[idx].description;
   ```
   - Run this code in a simple `RewardUI` script attached to the Text.

6. **Test the flow**
   - Play the game, press the **Collect Daily Reward** button – you should see a log entry with the granted reward and the corresponding coin/power‑up changes in the UI.
   - Stop Play mode, close the editor, reopen – the reward for that day will be *disabled* (the script will refuse another claim). 
   - Change your system clock or wait until the next day; the reward index will advance automatically (you’ll see a log message indicating the new index).

## 3️⃣  What the JSON (`gamesave.json`) now contains
```json
{
  "playerName":"Player",
  "currentLevel":1,
  "currency":0,
  "powerupCounts":{
    "FillSlot":0,
    "Rearrange":0,
    "Helicopter":0
  },
  "dailyRewardsCollected":["2026-05-26"],
  "dailyRewardIndex":3,
  "lastRewardDate":"2026-05-27"
}
```
- `dailyRewardsCollected` stops double‑claims on the same day.
- `dailyRewardIndex` tells which of the 7 rewards is **active today**.
- `lastRewardDate` is used to detect when a new calendar day starts and to rotate the index.

---

**You now have**:
1. The **full C# script** (`DailyRewardManager.cs`).
2. A **step‑by‑step guide** to add it to the project, wire a UI button, and make sure `SaveData` is present.
3. An explanation of the JSON format so you can verify the data on disk.

Drop the script in, hook the button, and the 7‑day daily‑reward loop will work automatically! 🎉
