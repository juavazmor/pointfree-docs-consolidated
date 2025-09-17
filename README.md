# Point-Free Documentation Consolidated

Auto-generated consolidated documentation for Point-Free's Swift libraries. This repository exists because AI coding assistants struggle with Point-Free's DocC format, so we flatten everything into simple Markdown files.

## What's This For?

When working with Point-Free libraries (Composable Architecture, Swift Navigation, etc), AI assistants often can't parse the official DocC documentation. This repository solves that by:

- Cloning each Point-Free repository
- Extracting all `.md` files from their documentation folders
- Consolidating them into single files per library

## Available Documentation

| Library | Consolidated File | Original Repository |
|---------|------------------|---------------------|
| Swift Composable Architecture | [swift-composable-architecture-docs.md](./swift-composable-architecture-docs.md) | [pointfreeco/swift-composable-architecture](https://github.com/pointfreeco/swift-composable-architecture) |
| Swift Navigation | [swift-navigation-docs.md](./swift-navigation-docs.md) | [pointfreeco/swift-navigation](https://github.com/pointfreeco/swift-navigation) |
| Swift Dependencies | [swift-dependencies-docs.md](./swift-dependencies-docs.md) | [pointfreeco/swift-dependencies](https://github.com/pointfreeco/swift-dependencies) |
| Swift Case Paths | [swift-case-paths-docs.md](./swift-case-paths-docs.md) | [pointfreeco/swift-case-paths](https://github.com/pointfreeco/swift-case-paths) |

## How to Use with AI

Simply provide your AI assistant with the raw GitHub URL of the documentation you need:

```
https://raw.githubusercontent.com/juavazmor/pointfree-docs-consolidated/main/swift-composable-architecture-docs.md
```

Replace `juavazmor` with your username if you forked this repository.

## How It Works

A GitHub Actions workflow runs and:

1. Clones each Point-Free repository
2. Finds all `.md` files in their documentation folders
3. Concatenates them with proper headers and structure
4. Commits the updated consolidated files

See [`.github/workflows/sync-pointfree-docs.yml`](./.github/workflows/sync-pointfree-docs.yml) for the complete implementation.

## Manual Updates

You can trigger updates manually from the Actions tab, or run the local script if you want to test changes locally.

## Contributing

If Point-Free adds new libraries or changes their documentation structure, simply update the repository list in the workflow file. The script is flexible and handles multiple documentation folders per repository.

## Supported Libraries

Currently tracking documentation for:
- **Composable Architecture** - State management and architecture
- **Swift Navigation** - Declarative navigation tools
- **Swift Dependencies** - Dependency management
- **Swift Case Paths** - Key path utilities for enums

---

*Generated documentation files are automatically updated and may overwrite manual changes.*
