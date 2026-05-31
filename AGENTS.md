# Project Agent Instructions

## Single-File Runtime

This project is an ultra-lightweight static Pomodoro app. The final runnable app must stay self-contained in `index.html`.

- Do not add committed runtime packages, build systems, bundlers, package managers, framework scaffolding, or external app source files.
- Do not require users to run `npm install`, build commands, deployment steps, or a local server.
- All app HTML, CSS, and JavaScript changes must be integrated into `index.html`.
- Users should be able to run the app by opening the hosted URL or double-clicking `index.html`.
- Temporary local tools may be used for development or verification, but remove their project files before handing off changes.

## Repository Hygiene

- Keep the app dependency-free and static.
- Preserve the existing README and LICENSE unless the user explicitly asks to change them.
- Avoid adding generated reports, dependency directories, or test artifacts to the repository.
