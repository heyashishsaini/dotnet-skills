# .NET Skills

Welcome to **dotnet-skills** — a structured knowledge base and Claude plugin marketplace for mastering the .NET ecosystem, covering core fundamentals, advanced architecture, WPF, and interview preparation.

This repo is both a **reference guide** and an **installable Claude Code plugin marketplace** — you can browse the notes directly, install any skill as a plugin in Claude Code, or download the ZIP to use with claude.ai / Claude Projects.

---

## 🚀 Installation & Usage Options

Choose whichever method fits your setup:

### Option 1: Install via Claude Code CLI (Recommended for Claude Code)

Add this repo as a marketplace in your terminal:

```bash
/plugin marketplace add heyashishsaini/dotnet-skills
```

Then install the plugin:

```bash
/plugin install dotnet-skills@dotnet-skills
```

---

### Option 2: Download ZIP for claude.ai / Claude Projects / Claude Desktop

If you are using the **claude.ai** web interface, Claude Desktop app, or Claude Projects:

1. **Download the Repository ZIP:**
   Click **[Download ZIP](https://github.com/heyashishsaini/dotnet-skills/archive/refs/heads/main.zip)** directly from GitHub or click the **Code** button on GitHub and select **Download ZIP**.
2. **Extract the ZIP file:**
   Unzip the downloaded archive on your computer.
3. **Upload to Claude:**
   - **For Claude Projects:** Create or open a Project in [claude.ai](https://claude.ai), click **Add Content** / **Project Knowledge**, and drag & drop the `SKILL.md` or `references/` files from any skill folder.
   - **For Chat:** Simply attach any individual `SKILL.md` or reference guide directly into your conversation.

---

## 📂 Repository Structure

The repository is organized into four skills, each focusing on a specific pillar of .NET expertise:

### `dotnet-architect/`

Senior .NET solution architect persona — advanced design patterns, system architecture, microservices, cloud-native practices, and enterprise-level application design.

- `references/ai-integration.md`
- `references/auth-patterns.md`
- `references/clean-architecture.md`
- `references/deployment.md`

### `dotnet-main/`

Core .NET fundamentals, foundational runtime concepts, C# language features, and modern framework updates.

- `references/ai-integration.md`
- `references/aspnet-core.md`
- `references/auth.md`
- `references/azure-docker-cicd.md`
- `references/clean-architecture.md`
- `references/ef-core-sql.md`

### `dotnet-wpf/`

Desktop application development using Windows Presentation Foundation (WPF) — XAML, MVVM pattern implementation, data binding, and desktop UI/UX.

- `references/data-binding.md`
- `references/mvvm-patterns.md`
- `references/performance.md`
- `references/production-architecture.md`

### `dotnet-interview-notes/`

Curated technical questions, coding challenges, conceptual deep-dives, and preparation material for .NET developer interviews.

---

## 🧩 Using a Skill Without Installing

Each skill folder is self-contained around a `SKILL.md` entry point with a `references/` folder of deep-dive notes. You're free to just open and read the markdown directly if you don't want to install it as a plugin.

---

## 🤝 Contributing

Found a gap or an outdated pattern? PRs and issues are welcome — see individual skill folders for the topics currently covered.
