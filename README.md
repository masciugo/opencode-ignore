# opencode-ignore

OpenCode plugin to restrict AI access to files and directories using `.ignore` patterns (gitignore-style).

> Requires **OpenCode V2** (plugin API v2). For migrating from V1, see the [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1).

## Usage

Install from GitHub and register it in your OpenCode configuration
(`~/.config/opencode/opencode.json` for global use, or `opencode.json(c)` in a project):

```sh
opencode plugin add github:masciugo/opencode-ignore
```

Or add it manually to the config:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "plugins": [
    "github:masciugo/opencode-ignore"
  ]
}
```

For local development you can point at a checkout instead (relative paths resolve from the config file):

```jsonc
{
  "plugins": [
    "/absolute/path/to/opencode-ignore",
    "../opencode-ignore"
  ]
}
```

> **Note**: do not use the bare npm package name `"opencode-ignore"` — the version published there
> targets the legacy OpenCode V1 plugin API and fails to load on V2
> (`Missing key at ["default"]`). This repository is not published to npm.

Create a `.ignore` file in your project root with patterns to block:

```gitignore
# Block specific directory
/secrets/**

# Block all certificate files
*.crt
*.key
*.pem

# Block environment files
.env*

# Allow exception with negation
!config.local.json
```

## `.ignore` Format

The `.ignore` file follows gitignore-style syntax powered by the [`ignore`](https://github.com/kaelzhang/node-ignore) package.

### Pattern Types

#### Absolute Paths
Block specific paths from project root:
```gitignore
/to/ignore
/somedir/toignore/**
```

#### Glob Patterns
Block files matching patterns anywhere in project:
```gitignore
**/node_modules/**
**/dist/**
**/build/**
**/.cache/**
```

#### Wildcards
Block files by extension or name pattern:
```gitignore
*.crt              # All certificate files
*.key              # All private keys
*.pem              # All PEM files
*-secret.json      # Files ending with "-secret.json"
temp*.log          # Temp log files
```

#### Negation Patterns
Allow exceptions to blocked patterns:
```gitignore
*.json              # Block all JSON files
!*.local.json       # But allow *.local.json files

**/target/**        # Block target directory
!**/target/**       # Re-allow it (negation)
```

### Directory vs File Matching

The `ignore` library automatically detects whether paths are files or directories:

```gitignore
foo      # Blocks file "foo" OR directory "foo/"
foo/     # Only blocks directory "foo/"
```

**Important**: Pattern `**/node_modules/**` blocks files *inside* the directory, not the directory itself.

## Example Patterns

See `example/.ignore` for comprehensive examples:

```gitignore
# Block specific paths
/secrets/**
/private/**
!/private/public-config.json

# Block directories anywhere
**/node_modules/**
**/dist/**
**/build/**
**/.cache/**

# Block sensitive files
*-secret.json
temp*.log
*.crt
*.key
*.pem
.env*
.env
*.env
id_rsa
id_ed25519
secrets.json
credentials.json
api-keys.json

# Allow build artifacts in specific cases (negation)
!**/dist/public/**

# Allow local development files
!*.local.json
!*.dev.json
!config.development.json
!/.local/
```

## Supported Tools

The plugin protects the following OpenCode V2 native tools:

### File Operations (Pre-execution blocking)
- `read` - Blocks reading blocked files (also used for directory listing in V2)
- `write` - Blocks writing to blocked paths
- `edit` - Blocks editing blocked files
- `patch` - Blocks patches touching blocked files (paths parsed from `patchText`)

### Search Operations (Pre-execution + Post-execution filtering)
- `glob` - Blocks searching in blocked directories, filters blocked files from structured results and text content
- `grep` - Blocks searching in blocked directories, filters matches from structured results and text content

**Protection Levels**:
- **Pre-execution**: Prevents tool from accessing blocked paths entirely (read, write, edit, patch)
- **Post-execution**: Allows search but filters blocked files from results (glob, grep)
- This two-phase approach prevents both direct access and information disclosure

**Note**: Project root (`.`) is always accessible to prevent blocking entire project. OpenCode V2 has no separate `list` tool — directory listing goes through `read`.

## How It Works

### Pre-execution Protection
1. Plugin discovers `.ignore` by walking upward from the OpenCode location directory (stopping at the project root) before tool execution
2. Tool paths are normalized to relative paths from the `.ignore` file location
3. Paths are checked against ignore patterns using the `ignore` library
4. If matched, tool execution is blocked with clear error message

### Post-execution Filtering (glob/grep)
For `glob` and `grep` tools, additional protection filters results:
1. Tool executes normally (searching allowed directories)
2. Both the structured result entries and the text content are filtered to remove anything matching `.ignore` patterns
3. Blocked files are completely removed from output (no partial data leakage); metadata counts are recomputed and truncation notes are dropped
4. Empty results returned if all matches are filtered

### Graceful Degradation
- If `.ignore` missing, all access is allowed
- Project root (`.`) is always accessible

### Error Messages

When a path is blocked:
```
Access denied: path/to/file blocked by ignore file. Do NOT try to read this. Access restricted.
```

## Path Normalization

The plugin handles various path formats:

- **Absolute paths**: Converted to relative from project root
- **Relative paths**: Used directly
- **Paths with `./`**: Prefix removed (ignore library requirement)
- **Win32 backslashes**: Auto-converted to forward slashes
- **Directory paths**: Trailing `/` added when needed

## Testing

Run tests with:
```bash
bun test
```

Tests cover:
- Absolute path patterns
- Glob patterns
- Negation patterns
- Wildcard patterns
- Directory vs file matching
- Path normalization edge cases
- All supported native tools
- Missing `.ignore` graceful degradation

## Development

```bash
# Install dependencies
bun install

# Run tests
bun test

# Build (if needed)
bun run build
```

## License

MIT

## Related

- [`ignore`](https://github.com/kaelzhang/node-ignore) - The underlying pattern matching library
- [OpenCode V2 Plugin Documentation](https://opencode.ai/v2/docs/build/plugins)
- [Migrating V1 plugins to V2](https://opencode.ai/v2/docs/migrate-v1)
