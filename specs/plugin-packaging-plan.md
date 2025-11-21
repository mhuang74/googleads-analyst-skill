# Packaging Google Ads Analyst Skill as a Claude Code Plugin

## Current State

The skill is a **markdown-based Claude Skill** consisting of:
- `SKILL.md` - Main skill definition with frontmatter
- 7 reference documents (`mcc_gaql_reference.md`, etc.)
- Depends on the external `mcc-gaql` CLI tool

## Recommended Plugin Structure

To package this as a distributable Claude Code plugin, create this structure:

```
googleads-analyst-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── skills/
│   └── google-ads-analyst/
│       ├── SKILL.md             # Your existing skill
│       ├── mcc_gaql_reference.md
│       ├── derived_metrics_reference.md
│       ├── appendix_reference.md
│       ├── common_errors_reference.md
│       ├── common_performance_patterns_reference.md
│       ├── contextual_analysis_rules_reference.md
│       └── identify_potential_causes.md
├── hooks/
│   └── hooks.json               # SessionStart hook for installation
├── scripts/
│   └── install-mcc-gaql.sh      # Installs mcc-gaql binary
└── README.md                    # Installation docs
```

## plugin.json

```json
{
  "name": "google-ads-analyst",
  "description": "Generate Google Ads performance reports using natural language queries",
  "version": "1.0.0",
  "author": {
    "name": "mhuang74"
  }
}
```

## hooks.json (SessionStart Hook)

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup",
      "hooks": [{
        "type": "command",
        "command": "${PLUGIN_DIR}/scripts/install-mcc-gaql.sh"
      }]
    }]
  }
}
```

## install-mcc-gaql.sh

```bash
#!/bin/bash

MCC_GAQL_VERSION="v0.13.0"

# Check if mcc-gaql is already installed
if command -v mcc-gaql &> /dev/null; then
    echo "mcc-gaql is already installed: $(mcc-gaql --version)"
    exit 0
fi

# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

case "$OS-$ARCH" in
    darwin-arm64)
        BINARY_URL="https://github.com/mhuang74/mcc-gaql-rs/releases/download/${MCC_GAQL_VERSION}/mcc-gaql-${MCC_GAQL_VERSION}-macos-aarch64.tar.gz"
        ;;
    darwin-x86_64)
        BINARY_URL="https://github.com/mhuang74/mcc-gaql-rs/releases/download/${MCC_GAQL_VERSION}/mcc-gaql-${MCC_GAQL_VERSION}-macos-x86_64.tar.gz"
        ;;
    linux-x86_64)
        BINARY_URL="https://github.com/mhuang74/mcc-gaql-rs/releases/download/${MCC_GAQL_VERSION}/mcc-gaql-${MCC_GAQL_VERSION}-linux-x86_64.tar.gz"
        ;;
    *)
        echo "Unsupported platform: $OS-$ARCH"
        echo "Please install mcc-gaql manually from: https://github.com/mhuang74/mcc-gaql-rs/releases"
        exit 1
        ;;
esac

echo "Installing mcc-gaql ${MCC_GAQL_VERSION}..."
curl -L "$BINARY_URL" | tar xz
sudo mv mcc-gaql /usr/local/bin/ || mv mcc-gaql ~/.local/bin/
echo "mcc-gaql installed successfully!"

# Prompt for setup
echo ""
echo "Run 'mcc-gaql --setup' to configure your Google Ads credentials."
```

## Distribution Options

### Option 1: Plugin Marketplace (Recommended)

Create a marketplace repository:

```
googleads-marketplace/
├── .claude-plugin/
│   └── marketplace.json
└── google-ads-analyst/
    └── (plugin contents)
```

**marketplace.json:**
```json
{
  "name": "Google Ads Tools",
  "description": "Plugins for Google Ads analysis and reporting",
  "plugins": ["google-ads-analyst"]
}
```

**User installation:**
```
/plugin marketplace add mhuang74/googleads-marketplace
/plugin install google-ads-analyst
```

### Option 2: Direct Git Install

Users can install directly from a Git repository without a marketplace.

## Key Considerations

| Aspect | Recommendation |
|--------|----------------|
| **Binary distribution** | Use SessionStart hook to download from GitHub Releases |
| **Cross-platform** | Support macOS (ARM64/x86), Linux (x86_64) |
| **Credential setup** | Guide user to run `mcc-gaql --setup` post-install |
| **Version pinning** | Pin mcc-gaql version in install script |
| **Fallback** | Provide manual install instructions if auto-install fails |

## User Experience Flow

1. User runs `/plugin marketplace add mhuang74/googleads-marketplace`
2. User runs `/plugin install google-ads-analyst`
3. On first session start, hook downloads and installs `mcc-gaql` binary
4. User runs `mcc-gaql --setup` to configure Google Ads credentials
5. User can now invoke the skill: "Generate a performance report for my Google Ads account"

## mcc-gaql Source

- **Repository**: https://github.com/mhuang74/mcc-gaql-rs
- **Latest version**: v0.13.0
- **Platforms**: macOS (ARM64, x86_64), Linux (x86_64)
- **Installation**: Pre-built binaries available in GitHub Releases

## Next Steps

1. Restructure repository into plugin format
2. Create `plugin.json` metadata
3. Create `hooks.json` with SessionStart installation script
4. Write cross-platform install script
5. Create marketplace repository (optional)
6. Add comprehensive README with manual setup instructions
7. Test installation on fresh systems
