# System Information

This system runs **Fedora 43** (Linux). Use `dnf` (version 5) for package management, not `apt`.

# Custom Skills

**Always invoke skills via the Skill tool before acting.** The skill content will guide your approach.

| Skill | Invoke When |
|-------|-------------|
| `deep-research` | User requests research requiring multiple sources, comprehensive analysis, or synthesis across topics - technical research, domain knowledge gathering, market analysis, or learning about complex subjects |
| `exa-search` | Semantic/conceptual search, "find similar to...", research discovery, user mentions exa |
| `yt-transcribe` | User asks about YouTube video content (web tools only return metadata, not what's actually said) |
| `using-python-lsp` | Working with Python code and pyright-lsp plugin is enabled - for finding references, checking types, navigating definitions, or verifying type correctness |
| `using-typescript-lsp` | Working with TypeScript or JavaScript code and typescript-lsp plugin is enabled - for finding references, checking types, navigating definitions, or verifying type correctness |
| `playwright-testing` | Writing, debugging, or reviewing Playwright tests for web apps; before writing test code; when tests are flaky, slow, or brittle; when seeing timeout errors, element not found, or race conditions |
| `designing-gnome-ui` | Designing, implementing, or modifying UI for GNOME apps; before writing UI code; when reviewing existing UI for HIG compliance; when working with GTK 4/libadwaita or styling Qt/PySide6 for GNOME |
| `developing-gtk-apps` | Building GTK 4/libadwaita applications; before writing app boilerplate; when debugging threading, signals, or lifecycle issues; when setting up GSettings, resources, or packaging; delegates UI/widget decisions to designing-gnome-ui skill |
| `working-with-aspire` | Building distributed apps with Aspire 13+; orchestrating .NET, JavaScript, Python, or polyglot services; when environment variables or service discovery aren't working; when migrating from .NET Aspire 9 or Community Toolkit |

**Note:** Web search/fetch cannot access YouTube video content - only `yt-transcribe` can retrieve what's actually said in a video.
