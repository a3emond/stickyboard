# StickyBoard — Tablet-Focused Life Management Dashboard for Android

## 📌 Overview

**StickyBoard** is a pen-first, always-on tablet dashboard that unifies notes, tasks, and events into one intuitive interface. Designed to replace a paper agenda and scattered apps, it combines a Post-it style board, an integrated calendar, and intelligent task management — all accessible from a single full-page layout on your home screen.

StickyBoard is more than a note-taking tool. It integrates with productivity services (Google Calendar, Microsoft Teams, Google Tasks, etc.), offers handwriting recognition, and organizes your personal and work life with a clean, customizable view.

---

## 🎯 Core Goals

* **Full-page dashboard** optimized for landscape tablet use.
* **Unified Post-it model**: Every item starts as a note, classified as Generic, Task, or Event.
* **Separation of views**:

  * **Note Board**: Generic notes only (no tasks/events).
  * **Task List**: All tasks, sorted by due date.
  * **Calendar**: Events and tasks with dates; optional work log view for dated notes.
* **Categories/Tags** for organizing notes into projects or themes.
* **Historical backdating** to log past work or events.
* **Secure storage** with encryption and backup.

---

## 🖼️ Layout Concept

**Dashboard (Landscape)**

```
+---------------------------------------------------------------+
| Header: Date | Weather | Quick Add (+) | Filters              |
+-----------------------+---------------------------------------+
| Calendar (month/week) | Note Board (Generic Notes only)       |
+-----------------------+---------------------------------------+
| Task List (due/overdue tasks, sorted)                         |
+---------------------------------------------------------------+
```

**Day View**

* Left: Time slots with events.
* Right: Tasks due that day + dated notes (logs).

---

## 🛠️ Key Features

* **Unified editor**: Choose note type (Generic, Task, Event) with contextual fields.
* **Intelligent parsing**: Handwriting-to-text and natural language recognition for dates/times.
* **Service integration**: Sync with Google Calendar, Google Tasks, Microsoft Teams tasks.
* **Categories/tags**: Group notes by project or topic.
* **Historical editing**: Add/edit notes for past dates.
* **Quick capture**: Bubble overlay for notes from any app.
* **Encrypted local storage** + optional cloud backup.

---

## 🤖 Intelligence Layer

* **Type suggestion** based on handwriting or typed keywords.
* **Auto-date recognition** ("tomorrow 3 PM" → sets due date).
* **Suggested reminders** from freeform notes.
* **Contextual linking**: Link related notes, tasks, and events.

---

## 📦 Data Model

```kotlin
enum class NoteType { GENERIC, TASK, EVENT }

data class Note(
    val id: UUID,
    val type: NoteType,
    val title: String?,
    val text: String?,
    val drawingUri: Uri?,
    val date: LocalDate?,
    val time: LocalTime?,
    val durationHours: Float?,
    val tags: List<String>,
    val priority: Int?,
    val createdAt: Instant,
    val updatedAt: Instant,
    val completed: Boolean
)
```

---

## 📲 Development Plan

**Phase 1 — Core Board & Tasks**

* Unified data model.
* Responsive Note Board (Generic only).
* Task List view.
* Unified editor with type selection.
* Encrypted Room database.

**Phase 2 — Calendar & Day View**

* Month/week calendar view.
* Day view with events, tasks, and optional logs.
* Backdating support.

**Phase 3 — Intelligence Layer**

* Handwriting recognition.
* Natural language date parsing.
* Auto-type suggestion.

**Phase 4 — Integrations & Sync**

* Google Calendar, Google Tasks, Microsoft Teams.
* Secure cloud backup.

**Phase 5 — Enhancements**

* Customizable themes.
* Advanced search & filters.
* Data export/import.

---

## ⚙️ Technical Notes

* **UI**: Jetpack Compose (tablet-optimized), AndroidView interop for pen input.
* **Storage**: Room (SQLCipher for encryption).
* **Reminders**: AlarmManager + WorkManager fallback.
* **Integrations**: REST/Graph APIs for Google/Microsoft.
* **Minimum SDK**: 26.

---

## 📜 License

All rights reserved — © Alexandre Emond, 2025. Unauthorized copying or redistribution of this project’s source code, in whole or in part, is strictly prohibited without express written permission.
