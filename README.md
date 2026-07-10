# Task Flow

A premium, glassmorphic task management application built using **Vue 3**, **TypeScript**, and **Vite**. Task Flow is designed to streamline your daily workflow, combining productivity tracking, analytics, category management, and a Pomodoro focus timer into a single cohesive experience.

---

## Key Features

- **📂 Tabbed Page Routing**: Organized into four clean and spacious workspaces:
  - **Tasks**: Create, edit, search, sort, filter, and track subtask checklist items.
  - **Dashboard**: Track overall task completion rates with a circular progress widget and status overview cards.
  - **Focus Timer**: A dedicated Pomodoro workspace to select a pending task and start focused intervals.
  - **Categories**: Directly create and customize color-coded tags to group your tasks.
- **✨ Premium UI/UX**: Custom glassmorphic styling utilizing CSS variables and OKLCH color palettes, ensuring high visual quality in both Dark and Light themes.
- **🌓 Theme Pinned Control**: Toggle between automatic system preferences and pinned Light/Dark options seamlessly.
- **📋 Subtask Progress**: Break tasks down into smaller steps with a checklist and visual progress bar indicators.
- **⏱️ Interactive Pomodoro**: Includes Focus (25 min), Short Break (5 min), and Long Break (15 min) intervals, browser tab time-tracking, and synthesizer audio alerts (using the Web Audio API).
- **💾 Local Storage Sync**: Automatically persists tasks, categories, theme settings, and focus preferences on the fly.
- **📱 Responsive Layout**: Adaptive layouts optimized for desktops, tablets, and mobile devices using CSS Container Queries and Flexbox/Grid structures.

---

## Tech Stack

- **Framework**: [Vue 3](https://vuejs.org/) (Composition API, `<script setup>`)
- **Build Tool**: [Vite](https://vite.dev/)
- **Programming Language**: [TypeScript](https://www.typescriptlang.org/)
- **Styles**: Vanilla CSS with custom properties (OKLCH, Container Queries)
- **Icons**: Clean inline SVG vector graphics

---

## Getting Started

### Prerequisites

Make sure you have Node.js (version 18 or above recommended) installed on your system.

### Installation

1. Clone or download the repository.
2. Navigate to the project directory:
   ```bash
   cd todo_list
   ```
3. Install the dependencies:
   ```bash
   npm install
   ```

### Development Server

Run the development server locally:
```bash
npm run dev
```
Open your browser and navigate to `http://localhost:5173/` to view the application.

### Build and Production

Compile and bundle the application for production:
```bash
npm run build
```
The output files will be generated inside the `dist` directory. You can preview the production bundle locally with:
```bash
npm run preview
```

---

## Project Structure

```text
├── src/
│   ├── components/
│   │   ├── Dashboard.vue     # Analytics progress ring, search, filter and sorting controls
│   │   ├── PomodoroTimer.vue # Focus Timer with audio alarms and task selectors
│   │   ├── TagsManager.vue   # Dynamic category tags editor (modal & inline view modes)
│   │   ├── TaskForm.vue      # Modal form drawer for creating and editing tasks
│   │   └── TaskItem.vue      # Individual task card containing checklist items and actions
│   ├── App.vue               # Main app layout, tab controller, global state management
│   ├── main.ts               # App entrypoint
│   └── style.css             # Core design tokens, gradients, theme variables and resets
├── index.html                # App template
├── package.json              # Project scripts & dependencies
└── tsconfig.json             # TypeScript configuration
```
