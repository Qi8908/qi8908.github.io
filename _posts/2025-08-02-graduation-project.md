---
title: "Graduation Project: The Murder at Qingliu Manor"
date: 2025-08-02 14:00:00 +0800
categories: [Game Development, C#]
tags: [Unity, Game Design, C#]
image: "/post-img/graduation-project/Cover.png"
published: true
---

# The Murder at Qingliu Manor - My Graduation Project

## Project Overview

**The Murder at Qingliu Manor** is my graduation project - a detective mystery adventure game where players take on the role of an investigator solving a puzzling murder case in the mysterious Qingliu Manor.

🎮 **Play the game**: [itch.io Page](https://pumpkincaptain.itch.io/the-murder-at-qingliu-manor-windows)

## Game Features

### 🔍 Core Gameplay
- **Detective Investigation**: Collect clues, analyze evidence, and deduce the truth
- **Interactive Exploration**: Freely explore the carefully designed manor
- **Character Dialogue**: Interact with various characters to gather crucial information
- **Logical Reasoning**: Use critical thinking to unravel the mystery

### 🎨 Technical Implementation
- **Engine**: Unity
- **Programming Language**: C#
- **Art Style**: [Describe your art style here]
- **Platform**: macOS

## Development Journey

### Project Background
[Share why you chose to create a mystery game, your inspiration sources, etc.]

### Design Philosophy
The game aims to create an immersive detective experience where players feel like real investigators. Every clue matters, and logical deduction is key to solving the case.

### Technical Challenges
During development, I encountered several challenges:
- **Dialogue System**: Creating a flexible conversation system that could handle complex branching narratives
- **Inventory Management**: Implementing an intuitive evidence collection and examination system
- **Scene Transitions**: Seamlessly connecting different areas of the manor
- **Save System**: Ensuring players could save their progress at any point

### Development Timeline
- **Pre-production** (Month 1-2): Concept design, story writing, and initial planning
- **Prototype Development** (Month 3-4): Core gameplay mechanics implementation
- **Art Production** (Month 5-6): Scene design, character art, and UI creation
- **System Integration** (Month 7-8): Feature completion and bug fixing
- **Polish & Release** (Month 9): Performance optimization and final packaging

## Project Highlights

### 🎭 Narrative Design
The story unfolds in the 1920s at the enigmatic Qingliu Manor, where a wealthy businessman has been found dead under mysterious circumstances. Players must interrogate suspects, each with their own secrets and motives.

**Key Characters:**
- The victim's family members
- Household staff
- Business associates
- Mysterious guests

### 🏛️ Environmental Design
The manor features multiple interconnected rooms, each telling part of the story:
- **Grand Library**: Where crucial documents are hidden
- **Study Room**: The crime scene with vital evidence
- **Gardens**: Outdoor areas with additional clues
- **Servant Quarters**: Revealing the household's secrets

### 🧩 Puzzle Design
The game includes various types of puzzles:
- **Logic Puzzles**: Deductive reasoning challenges
- **Hidden Object**: Finding crucial evidence in scenes
- **Code Breaking**: Deciphering encrypted messages
- **Timeline Reconstruction**: Piecing together the sequence of events

## Technical Details

### System Architecture
```
Game Architecture:
├── Scene Management System
├── Dialogue System
├── Inventory & Evidence System
├── Save/Load System
├── Audio Manager
└── UI Management
```

### Core Features Implementation
```csharp
// Example: Evidence Collection System
public class EvidenceManager : MonoBehaviour
{
    public List<Evidence> collectedEvidence;
    
    public void CollectEvidence(Evidence evidence)
    {
        if (!collectedEvidence.Contains(evidence))
        {
            collectedEvidence.Add(evidence);
            UIManager.Instance.UpdateEvidenceDisplay();
        }
    }
}
```

## Development Insights

### What I Learned
- **Narrative Design**: How to craft engaging mysteries with proper pacing
- **Unity Proficiency**: Advanced understanding of Unity's systems and workflows
- **Problem Solving**: Breaking down complex features into manageable components
- **User Experience**: The importance of intuitive controls and clear feedback

### Challenges Overcome
- **Balancing Difficulty**: Making puzzles challenging but not frustrating
- **Performance Optimization**: Ensuring smooth gameplay across different devices
- **Narrative Coherence**: Maintaining story consistency throughout development
- **Scope Management**: Learning to prioritize features within time constraints

### Future Improvements
If I were to revisit this project, I would:
- Add voice acting for key characters
- Implement multiple endings based on player choices
- Create additional case files for extended gameplay
- Enhance the visual effects and animations

## Screenshots

[Add your game screenshots here - showing key scenes, UI, and gameplay moments]

## Try It Out!

I invite you to play my graduation project and experience the mystery firsthand!

**Download**: [The Murder at Qingliu Manor on itch.io](https://pumpkincaptain.itch.io/the-murder-at-qingliu-manor-macos)

**System Requirements:**
- macOS 10.12 or later
- 2GB RAM minimum
- 500MB available storage

## Feedback Welcome

If you play the game, I'd love to hear your thoughts! Please leave comments below about:
- Your favorite puzzle or moment
- Any bugs or issues you encountered
- Suggestions for improvements
- Your overall experience

---

*This project represents my growth as a game designer and technical artist. It's my first step into the game industry, and I'm excited to continue creating engaging interactive experiences. Thank you to everyone who takes the time to play and provide feedback!*

## Contact & Connect

Feel free to reach out if you have questions about the development process or want to discuss game design!

- **Email**: [Your email]
- **LinkedIn**: [Your LinkedIn profile]
- **Portfolio**: [Your portfolio website]
