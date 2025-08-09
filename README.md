# StickyBoard — Persistent Post-it & Calendar Widget for Android

## 📌 Overview

**StickyBoard** is an always-available, pen-friendly **home screen widget** that combines a responsive Post-it board with a powerful calendar view.
It’s designed for **zero-friction note-taking** and **quick appointment management** — no app opening, no waiting — just unlock your device and write.

The widget fills an entire home screen page, adapting its layout based on the number of notes or calendar entries. All editing happens in an **instant overlay** that feels like it’s inside the widget.

---

## 🎯 Core Goals

* **Full-page widget** that adapts dynamically:

  * 1 note → fills the entire space.
  * Many notes → mini Post-its in a responsive grid.
* **Two main modes**:

  1. **Post-it Board** — freeform notes, sorted by most recent or pinned.
  2. **Calendar** — each date as a clickable cell, with daily grids for events.
* **Pen support**: Handwriting or text for each note.
* **Instant access**: Overlay opens instantly with no visible app switch.
* **Project & priority management**: Color coding, tags, filters.
* **Reminders & notifications**: Local alarms + optional push to phone.

---

## 🖼️ Widget Interaction Model

The widget supports multiple click zones with different actions:

| Click Target                 | Action                                                |
| ---------------------------- | ----------------------------------------------------- |
| Single Post-it card          | Open that note in full-screen overlay for editing.    |
| Board background/header      | Open scrollable Board List (all notes with filters).  |
| Calendar day cell            | Open Day View (hour slots, add events, reminders).    |
| **+** button on Board        | Create new note.                                      |
| **+** button on Calendar     | Create new event for that day.                        |
| Long-press (where supported) | Quick actions: pin/unpin, change color, move project. |

---

## 🛠️ Features

### Post-it Board

* **Responsive grid layout** (Glance widget, adaptive columns).
* **Mini previews** of handwritten/text notes.
* Color-coded notes with tags & priorities.
* Tap to expand, edit, and collapse back.

### Calendar View

* Month grid with day badges.
* Hour-ordered events in Day View.
* Multiple reminders per event.
* Snooze/done actions from notifications.

### Editing Overlay

* Zero-chrome, transparent Activity (feels like widget editing).
* Text editor + handwriting canvas (pressure-sensitive).
* Autosave on close.
* Color selector, pin toggle, tag picker.

### Pen & Touch

* Palm rejection during pen use.
* Undo/redo for drawings.
* Convert handwriting to text (future OCR integration).

### Notifications & Reminders

* Exact alarms (where supported).
* Cross-device push sync via FCM (optional).

### Organization Tools

* Tag-based project organization.
* Priority flags.
* Filter chips in board/day lists.

---

## 🏗️ Architecture

### Packages

```
com.stickyboard
│
├── data
│   ├── db
│   │   ├── NoteEntity.kt
│   │   ├── EventEntity.kt
│   │   ├── AppDatabase.kt
│   │   └── DaoInterfaces.kt
│   └── model
│       ├── Note.kt
│       └── Event.kt
│
├── ui
│   ├── widget
│   │   ├── StickyBoardWidget.kt       # Glance-based widget
│   │   └── CalendarWidget.kt          # Optional split layout
│   ├── overlay
│   │   ├── OverlayActivity.kt         # Zero-chrome transparent editor
│   │   ├── BoardListScreen.kt
│   │   ├── DayViewScreen.kt
│   │   └── NoteEditorScreen.kt
│   └── components
│       ├── PostItCard.kt
│       ├── CalendarDayCell.kt
│       └── HandwritingCanvasView.kt
│
├── util
│   ├── DeepLinks.kt
│   ├── WidgetUtils.kt
│   └── DateTimeUtils.kt
│
└── service
    ├── ReminderScheduler.kt
    └── PushSyncService.kt
```

---

## 📦 Data Models

### Note

```kotlin
data class Note(
    val id: UUID = UUID.randomUUID(),
    val title: String? = null,
    val text: String? = null,
    val drawingUri: Uri? = null,
    val color: NoteColor = NoteColor.Yellow,
    val tags: List<String> = emptyList(),
    val priority: Int = 0,
    val createdAt: Instant = Instant.now(),
    val updatedAt: Instant = Instant.now(),
    val pinned: Boolean = false
)
```

### Event

```kotlin
data class Event(
    val id: UUID = UUID.randomUUID(),
    val localDate: LocalDate,
    val time: LocalTime? = null,
    val title: String,
    val noteId: UUID? = null,
    val reminders: List<Instant> = emptyList()
)
```

---

## 📲 Development Plan

### Phase 1 — Post-it Board MVP

* Full-page responsive board widget.
* Overlay editor (text + handwriting).
* Board list overlay with search/filter.
* Local storage via Room.
* Basic color/tags/pin support.

### Phase 2 — Calendar Integration

* Month grid widget panel.
* Day View overlay with reminders.
* Notifications with snooze/done.

### Phase 3 — Enhancements

* Cross-device push sync.
* OCR for handwriting search.
* Customizable widget themes.
* Data export/import.

---

## ⚙️ Technical Notes

* **Widget framework**: Jetpack Glance (Compose for widgets).
* **UI**: Jetpack Compose + AndroidView interop for drawing.
* **Storage**: Room (encrypted).
* **Reminders**: AlarmManager (exact) + WorkManager fallback.
* **Push sync**: Firebase Cloud Messaging (optional).
* **Minimum SDK**: 26 (Android 8.0).
* **Target SDK**: Latest stable.

---

## 🚀 Getting Started

### Prerequisites

* Android Studio Giraffe+ with Compose support.
* Kotlin 1.9+.
* Gradle 8+.

### Clone & Build

```bash
git clone https://github.com/YOUR_USERNAME/stickyboard.git
cd stickyboard
./gradlew assembleDebug
```

### Run

* Deploy to a tablet/emulator.
* Add the **StickyBoard** widget to a home screen page.
* Resize to full page.
* Tap + to create your first note.

---

## 📜 License

All rights reserved — © Alexandre Emond, 2025. Unauthorized copying or redistribution of this project’s source code, in whole or in part, is strictly prohibited without express written permission.

---

## 🤝 Contributing

Pull requests are welcome!
For major changes, please open an issue first to discuss what you’d like to change.

---

## 🗺️ Roadmap

* [ ] MVP Post-it board.
* [ ] Handwriting + pen support.
* [ ] Calendar integration.
* [ ] Reminder notifications.
* [ ] Push sync to phone.
* [ ] OCR handwriting search.
* [ ] Custom widget themes.
