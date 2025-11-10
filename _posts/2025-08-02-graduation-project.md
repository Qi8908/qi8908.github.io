---
title: "Unity AVG Game: The Murder at Qingliu Manor"
date: 2025-08-02 14:00:00 +0800
categories: [Game Development, C#]
tags: [Unity, Game Design, C#]
image: "/post-img/graduation-project/Cover.png"
published: true
---

# The Murder at Qingliu Manor - My Graduation Project

## Project Overview

**The Murder at Qingliu Manor** is my graduation project - a detective mystery adventure game where players take on the role of an investigator solving a puzzling murder case in the mysterious Qingliu Manor.

**Play the game**: <br />
Windows: [itch.io Page](https://pumpkincaptain.itch.io/the-murder-at-qingliu-manor-windows)<br />
MacOS: [itch.io Page](https://pumpkincaptain.itch.io/the-murder-at-qingliu-manor-macos)

## Game Features

### Core Gameplay
- **Detective Investigation**: Collect clues, analyze evidence, and deduce the truth
- **Interactive Exploration**: Freely explore the carefully designed manor
- **Character Dialogue**: Interact with various characters to gather crucial information
- **Logical Reasoning**: Use critical thinking to unravel the mystery

### Technical Implementation
- **Engine**: Unity
- **Programming Language**: C#
- **Platform**: macOS Windows

## Project Highlights

### Narrative Design
The story unfolds in the 1920s at the enigmatic Qingliu Manor, where a wealthy businessman has been found dead under mysterious circumstances. Players must interrogate suspects, each with their own secrets and motives.

### Environmental Design
The manor features multiple interconnected rooms, each telling part of the story:
- **Grand Library**: Where crucial documents are hidden
- **Study Room**: The crime scene with vital evidence
- **Gardens**: Outdoor areas with additional clues
- **Servant Quarters**: Revealing the household's secrets

### Puzzle Design
The game includes various types of puzzles:
- **Logic Puzzles**: Deductive reasoning challenges
- **Hidden Object**: Finding crucial evidence in scenes
- **Code Breaking**: Deciphering encrypted messages
- **Timeline Reconstruction**: Piecing together the sequence of events

## Technical Details

### Code Architecture
```
Code Architecture:
├── VNManager.cs
├── Constants.cs
├── ExcelReader.cs
├── TypewriterEffect.cs
├── MenuManager.cs
├── GalleryManager.cs
├── HistoryManager.cs
├── MapManager.cs
├── SuspectManager.cs
├── SaveLoadManager.cs
├── ButtonHighlighter.cs
├── ScreenShotter.cs
└── UI Management
```

### Core Features Implementation
```csharp
// Choice System - Dynamic dialogue branching
void ShowChoices()
{
    // Clear existing choice buttons
    foreach (var button in currentChoiceButtons)
    {
        Destroy(button.gameObject);
    }
    currentChoiceButtons.Clear();

    choicePanel.SetActive(true);
    SetGameButtonsInteractable(false);

    // Parse choice data from Excel
    List<string> choiceTexts = new List<string>();
    List<string> jumpFiles = new List<string>();

    int lineIndex = currentLine;
    while (lineIndex < storyData.Count)
    {
        var data = storyData[lineIndex];
        
        if (lineIndex > currentLine && !string.IsNullOrEmpty(data.speakerName))
            break;

        if (!string.IsNullOrEmpty(data.speakingContent) && !string.IsNullOrEmpty(data.avatarImageFileName))
        {
            choiceTexts.Add(data.speakingContent);
            jumpFiles.Add(data.avatarImageFileName);
        }
        lineIndex++;
    }

    // Create interactive choice buttons
    for (int i = 0; i < choiceTexts.Count; i++)
    {
        int choiceIndex = i;
        Button newButton = Instantiate(choiceButtonPrefab, choicePanel.transform);
        newButton.GetComponentInChildren<TextMeshProUGUI>().text = choiceTexts[i];
        newButton.onClick.AddListener(() =>
        {
            choicePanel.SetActive(false);
            SetGameButtonsInteractable(true);
            InitializeAndLoadStory(jumpFiles[choiceIndex], Constants.DEFAULT_START_LINE);
        });
        currentChoiceButtons.Add(newButton);
    }
}
```
```csharp
// Investigation System - Core detective gameplay
void ShowInvestigation()
{
    SetGameButtonsInteractable(false);
    
    List<string> buttonTexts = new List<string>();
    List<string> jumpTargets = new List<string>();

    // Parse investigation options from data
    int lineIndex = currentLine;
    while (lineIndex < storyData.Count)
    {
        var data = storyData[lineIndex];
        
        if (lineIndex > currentLine && !string.IsNullOrEmpty(data.speakerName))
            break;

        if (!string.IsNullOrEmpty(data.speakingContent) && !string.IsNullOrEmpty(data.avatarImageFileName))
        {
            buttonTexts.Add(data.speakingContent);
            jumpTargets.Add(data.avatarImageFileName);
        }
        lineIndex++;
    }

    currentLine = lineIndex;

    // Setup investigation buttons dynamically
    for (int i = 0; i < investigationButtons.Length; i++)
    {
        if (i < buttonTexts.Count)
        {
            investigationButtons[i].gameObject.SetActive(true);
            int index = i;
            investigationButtons[i].GetComponentInChildren<TextMeshProUGUI>().text = buttonTexts[i];
            investigationButtons[i].onClick.RemoveAllListeners();
            investigationButtons[i].onClick.AddListener(() =>
            {
                SetGameButtonsInteractable(true);
                InitializeAndLoadStory(jumpTargets[index], Constants.DEFAULT_START_LINE);
            });
        }
        else
        {
            investigationButtons[i].gameObject.SetActive(false);
        }
    }

    investigatePanel.SetActive(true);
}
```
```csharp
// Save/Load UI Management - Dynamic slot management
private void UpdateSaveLoadButtons(Button button, int index)
{
    button.gameObject.SetActive(true);
    
    var savePath = GenerateDataPath(index);
    var fileExists = File.Exists(savePath);
    
    var textComponents = button.GetComponentsInChildren<TextMeshProUGUI>();
    var image = button.GetComponentInChildren<RawImage>();
    
    if (fileExists)
    {
        // Existing save: prepare for content loading
        button.interactable = true;
        textComponents[0].text = "";
        textComponents[1].text = (index + 1) + Constants.COLON + Constants.EMPTY_SLOT;
        image.texture = null;
    }
    else
    {
        // Empty slot: show placeholder
        button.interactable = true;
        textComponents[0].text = "Empty Slot";
        textComponents[1].text = "";
        
        // Load placeholder image from Resources
        Texture2D placeholder = Resources.Load<Texture2D>(Constants.THUMBNAIL_PATH + Constants.SAVE_PLACEHOLDER);
        if (placeholder != null)
        {
            image.texture = placeholder;
        }
    }
    
    // Setup click event
    button.onClick.RemoveAllListeners();
    button.onClick.AddListener(() => OnButtonClick(button, index));
}
```
```csharp
// Save Preview System - Screenshot and content display
private void LoadStorylineAndScreenshots(Button button, int index)
{
    var savePath = GenerateDataPath(index);
    if (File.Exists(savePath))
    {
        string json = File.ReadAllText(savePath);
        var saveData = JsonConvert.DeserializeObject<VNManager.SaveData>(json);
        
        // Load and display screenshot thumbnail
        if (saveData.screenShotData != null)
        {
            Texture2D screenShot = new Texture2D(2, 2);
            screenShot.LoadImage(saveData.screenShotData);
            button.GetComponentInChildren<RawImage>().texture = screenShot;
        }
        
        // Process and display save content preview
        if (saveData.savedSpeakingContent != null)
        {
            var textComponents = button.GetComponentsInChildren<TextMeshProUGUI>();
            
            // Remove color tags and create preview text
            string cleanedContent = System.Text.RegularExpressions.Regex.Replace(
                saveData.savedSpeakingContent,
                @"<color=.*?>|</color>",
                ""
            );
            
            string shortContent = cleanedContent.Length > 27
                                    ? cleanedContent.Substring(0, 27) + "..."
                                    : cleanedContent;
            
            textComponents[0].text = shortContent;
            textComponents[1].text = File.GetLastWriteTime(savePath).ToString("G");
        }
    }
}

private string GenerateDataPath(int index)
{
    return Path.Combine(Application.persistentDataPath, Constants.SAVE_FILE_PATH, index + Constants.SAVE_FILE_EXTENSION);
}
```

## Screenshots

![Capture1](/post-img/graduation-project/Cap1.png){: width="80%"} <br />
![Capture2](/post-img/graduation-project/Cap2.png){: width="80%"} <br />
![Capture3](/post-img/graduation-project/Cap3.png){: width="80%"} <br />
![Capture4](/post-img/graduation-project/Cap4.png){: width="80%"} <br />
![Capture5](/post-img/graduation-project/Cap5.png){: width="80%"} <br />

