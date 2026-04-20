# apex-v2

A desktop learning platform built with Tauri (Svelte + Rust). Uses Claude Code to generate and manage courses stored locally on your machine.

## Features

- AI-generated courses with markdown lessons and quizzes
- Progress tracking per course
- Local-first: all course data stored in your system's app data directory

## Tech Stack

- **Frontend**: Svelte
- **Backend**: Rust (Tauri)

## Development

```bash
pnpm install
pnpm tauri dev
```

## Build

```bash
pnpm tauri build
```

## How It Works

On first launch, the app initializes a data directory and a `CLAUDE.md` file that instructs Claude on how to create courses. Courses are structured as:

```
data-dir/
  course-name/
    index.json          # ordered list of sections
    progress.json       # auto-managed progress tracker
    section-name/
      index.json        # ordered list of content files
      lesson.md         # markdown lesson
      quiz.json         # Q&A pairs
```

In the top-right corner, there is a "Copy Claude Command" button - copy and paste into your terminal. This will open a Claude Code window, where you can request a new course, change a section, add more quiz questions, etc.

A CLAUDE.md file exists in the directory, so don't worry about passing in context regarding file/folder structure.
