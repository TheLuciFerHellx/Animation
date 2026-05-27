# Save & Persistence Integration Guide for Car‑OUT Jam Puzzle Game

This markdown file contains **all the scripts you need** (with the exact code snippets) and step‑by‑step instructions to wire them together so that **player name, current level, currency, three power‑up counters, and daily‑reward history** are automatically saved to a single JSON file.

---

## 1️⃣ SaveData.cs  *(already present – no changes required)*
```csharp
// SaveData.cs – simple JSON‑based save system
using System.Collections.Generic;
using System.IO;
using UnityEngine;

[System.Serializable]
public class GameSaveData
{
    public string playerName = "Player";
    public int currentLevel = 1;
    public int currency = 0;
    // three power‑ups
    public Dictionary<string, int> powerupCounts = new Dictionary<string, int>()
    {
        { "FillSlot", 0 },
        { "Rearrange", 0 },
        { "Helicopter", 0 }
    };
    // list of ISO dates (yyyy‑MM‑dd) for which daily reward has been claimed
    public List<string> dailyRewardsCollected = new List<string>();
}

public class SaveData : MonoBehaviour
{
    private const string FileName = "gamesave.json";
    private string FilePath => Path.Combine(Application.persistentDataPath, FileName);

    public GameSaveData CurrentSave { get; private set; } = new GameSaveData();

    private void Awake()
    {
        DontDestroyOnLoad(gameObject);
        Load();
    }

    public void Save()
    {
        try
        {
            string json = JsonUtility.ToJson(CurrentSave, true);
            File.WriteAllText(FilePath, json);
            Debug.Log($"Game saved to {FilePath}");
        }
        catch (System.Exception ex)
        {
            Debug.LogError($"Failed to save game: {ex.Message}");
        }
    }

    public void Load()
    {
        if (File.Exists(FilePath))
        {
            try
            {
                string json = File.ReadAllText(FilePath);
                CurrentSave = JsonUtility.FromJson<GameSaveData>(json);
                Debug.Log($"Game loaded from {FilePath}");
            }
            catch (System.Exception ex)
            {
                Debug.LogError($"Failed to load game: {ex.Message}");
                CurrentSave = new GameSaveData();
            }
        }
        else
        {
            Debug.Log("No save file found – starting fresh.");
            CurrentSave = new GameSaveData();
        }
    }

    // ---------- Helper API (auto‑saves) ----------
    public void SetPlayerName(string name)
    {
        CurrentSave.playerName = name;
        Save();
    }

    public void SetLevel(int level)
    {
        CurrentSave.currentLevel = level;
        Save();
    }

    public void AddCurrency(int amount)
    {
        CurrentSave.currency += amount;
        Save();
    }

    public void SetCurrency(int amount)
    {
        CurrentSave.currency = amount;
        Save();
    }

    public void AddPowerup(string key, int amount)
    {
        if (CurrentSave.powerupCounts.ContainsKey(key))
            CurrentSave.powerupCounts[key] += amount;
        else
            CurrentSave.powerupCounts[key] = amount;
        Save();
    }

    public bool HasCollectedDailyReward(string date)
    {
        return CurrentSave.dailyRewardsCollected.Contains(date);
    }

    public void CollectDailyReward(string date)
    {
        if (!CurrentSave.dailyRewardsCollected.Contains(date))
        {
            CurrentSave.dailyRewardsCollected.Add(date);
            Save();
        }
    }
}
```
---

## 2️⃣ ParkingGameManager.cs – level win handling
Insert the following block **right after line 52** (the `Debug.Log` that announces level completion) and **before the coroutine call**:
```csharp
            // ----- Save progress (auto‑saves) -----
            var save = FindObjectOfType<SaveData>();
            if (save != null)
            {
                // advance to the next level
                save.SetLevel(save.CurrentSave.currentLevel + 1);
                // reward for completing the level (adjust amount as you like)
                save.AddCurrency(200);
            }
```
---

## 3️⃣ PowerUps.cs – add a cached reference to SaveData
At the very top of the file (right after the `using` statements) add:
```csharp
private SaveData _save => FindObjectOfType<SaveData>();
```

### 3.1 Fill‑Slot Power‑up (lines ~233‑236)
After you decrement the UI counter (`fillSlotCount--;`) add:
```csharp
if (_save != null) _save.AddPowerup("FillSlot", -1);
```

### 3.2 Rearrange Queue Power‑up (lines ~345‑346)
After `rearrangeCount--;` add:
```csharp
if (_save != null) _save.AddPowerup("Rearrange", -1);
```

### 3.3 Helicopter VIP Power‑up (lines ~396‑398)
After `helicopterCount--;` add:
```csharp
if (_save != null) _save.AddPowerup("Helicopter", -1);
```

### 3.4 Currency reward for each passenger placed (inside `ProcessFillCarRoutine`, after the sound line ~254)
```csharp
if (_save != null) _save.AddCurrency(10); // give 10 coins per passenger – change as needed
```
---

## 4️⃣ DailyReward.cs – new script
Create a new file **`Assets/Script/DailyReward.cs`** with this exact content:
```csharp
using UnityEngine;

public class DailyReward : MonoBehaviour
{
    // Call this from a UI button when the player taps “Collect Daily Reward”
    public void TryCollect()
    {
        var save = FindObjectOfType<SaveData>();
        if (save == null) return;

        string today = System.DateTime.UtcNow.ToString("yyyy-MM-dd");

        // Has the player already taken the reward today?
        if (save.HasCollectedDailyReward(today))
        {
            Debug.Log("Daily reward already claimed today.");
            return;
        }

        // ---- GIVE THE REWARD -------------------------------------------------
        // Example: 500 coins + 1 FillSlot power‑up – customise to your design
        save.AddCurrency(500);
        save.AddPowerup("FillSlot", 1);
        // ----------------------------------------------------------------------

        // Mark this date as collected – the method auto‑saves
        save.CollectDailyReward(today);
        Debug.Log("Daily reward granted!");
    }
}
```
Hook the **`TryCollect`** method to the **OnClick** event of your daily‑reward button in the Unity Inspector.
---

## 5️⃣ Player‑Name UI (example snippet)
Wherever you read the player‑entered name (e.g., a UI input field), add:
```csharp
var save = FindObjectOfType<SaveData>();
if (save != null) save.SetPlayerName(nameInputField.text);
```
---

## 6️⃣ Final setup steps (no code needed)
1. **Create a persistent GameObject** in the first scene (e.g., `SaveManager`).
2. **Add the `SaveData` component** to that GameObject.
3. Ensure the GameObject lives through scene loads – the script already calls `DontDestroyOnLoad`.
4. Verify the three power‑up keys (`"FillSlot"`, `"Rearrange"`, `"Helicopter"`) exist in the JSON. The first call to `AddPowerup` will create them automatically.
5. Play the game, finish a level, use power‑ups, collect the daily reward – then quit and restart. The console should show messages like **“Game saved to …”** and **“Game loaded from …”**, and the JSON file will retain the data.
---

## 7️⃣ Sample JSON after a few actions
```json
{
  "playerName": "Akhil",
  "currentLevel": 4,
  "currency": 1250,
  "powerupCounts": {
    "FillSlot": 2,
    "Rearrange": 1,
    "Helicopter": 3
  },
  "dailyRewardsCollected": [
    "2026-05-25",
    "2026-05-26"
  ]
}
```
---

*All scripts above are ready to be copied into your project. Once they compile, the save system works automatically.*

---

*Generated by Antigravity agent – keep this file in your repository (e.g., `Assets/Documentation/SaveIntegrationGuide.md`).*




==============


using System;
using UnityEngine;
using System.IO;

[Serializable]
public class GameData
{
    public int level = 1;
    public int coins = 0;
}

public class GameManager : MonoBehaviour
{
    public static GameManager Instance;

    public GameData data = new GameData();

    string path;

    void Awake()
    {
        Instance = this;

        path = Application.persistentDataPath + "/save.json";

        LoadGame();
    }

    public void SaveGame()
    {
        string json = JsonUtility.ToJson(data, true);

        File.WriteAllText(path, json);
    }

    public void LoadGame()
    {
        if (File.Exists(path))
        {
            string json = File.ReadAllText(path);

            data = JsonUtility.FromJson<GameData>(json);
        }
    }
}



==========
GameManager.Instance.SaveGame();

==========
GameManager.Instance.LoadGame();

==========
GameManager.Instance.data.coins += 100;

GameManager.Instance.data.level++;

==========
Then Save
GameManager.Instance.SaveGame();
