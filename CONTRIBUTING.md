<![CDATA[# Contributing to FireGuard

Thank you for your interest in contributing to FireGuard! This document provides guidelines and instructions for contributing to the project.

---

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [Branch Strategy](#branch-strategy)
- [Coding Standards](#coding-standards)
- [Commit Messages](#commit-messages)
- [Pull Request Process](#pull-request-process)
- [Reporting Issues](#reporting-issues)
- [Contributors](#contributors)

---

## Code of Conduct

This is a graduation project developed by a university team. All contributors are expected to:

- Be respectful and constructive in all communications.
- Focus on the quality and accuracy of the implementation.
- Follow the established coding conventions and architecture.
- Document all changes thoroughly.

---

## Project Structure

```
Fire Detection/
├── ai_service/          # AI detection pipeline (Python)
├── backend/             # Django REST API (Python)
├── frontend/            # Flutter mobile app (Dart)
├── docs/                # Project documentation
├── screenshots/         # App screenshots
└── archive/             # Archived files
```

See the [README.md](README.md) for a detailed directory breakdown.

---

## Getting Started

### Prerequisites

| Tool | Version | Required For |
|------|---------|-------------|
| Python | ≥ 3.10 | Backend + AI service |
| Flutter SDK | ≥ 3.9 | Mobile app |
| Git | Latest | Version control |
| Android Studio | Latest | Android emulator (optional) |

### 1. Fork and Clone

```bash
git clone https://github.com/your-username/fire-detection.git
cd fire-detection
```

### 2. Set Up Backend

```bash
cd backend
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows
pip install -r requirements.txt
cp ../.env.example .env
python manage.py migrate
python manage.py runserver
```

### 3. Set Up AI Service

```bash
cd ai_service
pip install -r requirements.txt
# Ensure best.pt model weights are present
# Ensure a webcam is connected
python camera.py
```

### 4. Set Up Flutter App

```bash
cd frontend
flutter pub get
flutter run
```

---

## Development Setup

### Environment Variables

Copy `.env.example` to the appropriate location and configure:

```bash
cp .env.example backend/.env
```

Key variables:
- `SECRET_KEY` — Django secret key (change for production)
- `FIREBASE_CREDENTIALS_PATH` — Path to Firebase service account JSON
- `ALLOWED_HOSTS` — Comma-separated list of allowed hostnames

### Firebase Configuration

1. Create a Firebase project at [console.firebase.google.com](https://console.firebase.google.com).
2. Download the service account JSON and place it at `backend/firebase_credentials.json`.
3. Run `flutterfire configure` in the `frontend/` directory to generate `firebase_options.dart`.

---

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Production-ready code |
| `develop` | Integration branch for new features |
| `feature/<name>` | Individual feature development |
| `fix/<name>` | Bug fixes |

### Workflow

1. Create a feature branch from `develop`:
   ```bash
   git checkout -b feature/your-feature-name develop
   ```

2. Make your changes and commit.

3. Push and create a pull request to `develop`.

4. After review and approval, merge to `develop`.

5. Periodically, `develop` is merged to `main` for releases.

---

## Coding Standards

### Python (Backend + AI Service)

- Follow **PEP 8** style guidelines.
- Use **4 spaces** for indentation.
- Use **snake_case** for variables, functions, and module names.
- Use **PascalCase** for class names.
- Add docstrings to all public functions and classes.
- Keep lines under **120 characters**.

### Dart (Flutter App)

- Follow the official [Dart style guide](https://dart.dev/guides/language/effective-dart/style).
- Use **2 spaces** for indentation.
- Use **camelCase** for variables and functions.
- Use **PascalCase** for class and type names.
- Organize imports alphabetically.
- Use `const` constructors where possible.

### Django REST Framework

- Use class-based views (`APIView`).
- Define explicit `permission_classes` on every view.
- Use serializers for all request/response data validation.
- Use `select_related` and `prefetch_related` to optimize queries.
- Return consistent response structures.

---

## Commit Messages

Follow the conventional commit format:

```
<type>(<scope>): <description>

[optional body]
```

### Types

| Type | Description |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes |
| `style` | Code style changes (formatting, no logic change) |
| `refactor` | Code refactoring |
| `test` | Adding or modifying tests |
| `chore` | Maintenance tasks |

### Scopes

| Scope | Description |
|-------|-------------|
| `ai` | AI service / camera.py |
| `backend` | Django backend |
| `mobile` | Flutter app |
| `docs` | Documentation |

### Examples

```
feat(backend): add event statistics endpoint
fix(ai): handle camera disconnection gracefully
docs(docs): update API endpoint table
feat(mobile): add notification preferences screen
```

---

## Pull Request Process

1. **Ensure your code follows the coding standards** outlined above.
2. **Test your changes** manually before submitting.
3. **Update documentation** if your changes affect the API, architecture, or user-facing behavior.
4. **Write a clear PR description** explaining:
   - What changes were made and why.
   - How to test the changes.
   - Any known limitations or follow-up work needed.
5. **Request review** from at least one team member.
6. **Address review feedback** promptly.

---

## Reporting Issues

When reporting a bug or requesting a feature, include:

1. **Title** — Clear, concise summary.
2. **Description** — Detailed explanation of the issue or feature.
3. **Steps to Reproduce** (for bugs) — Numbered steps to reproduce the issue.
4. **Expected Behavior** — What should happen.
5. **Actual Behavior** — What actually happens.
6. **Environment** — OS, Python version, Flutter version, device model.
7. **Screenshots/Logs** — If applicable.

---

## Contributors

This project is developed by the following team (alphabetical order):

| # | Name | Role |
|---|------|------|
| 1 | **Abdelrahman Reda Abdellah** | Backend Development |
| 2 | **Ali Khaled Abo Raya** | AI Development |
| 3 | **Mohamed Abdelfatah Attalah** | AI Development |
| 4 | **Mohamed Asaad** | AI Development |
| 5 | **Mohamed Hany El-Essawy** | UI/UX Design, Mobile Application Development |
| 6 | **Mostafa Abdo El-Gohary** | Deployment |
| 7 | **Saad Ezzat Saad** | UI/UX Design, Mobile Application Development |
| 8 | **Tarek Mostafa Kashef** | Database Development |

---

## License

By contributing to FireGuard, you agree that your contributions will be licensed under the [MIT License](LICENSE).
]]>
