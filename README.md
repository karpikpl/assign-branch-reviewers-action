# Branch Reviewers GitHub Action

A reusable GitHub Action that automatically assigns reviewers to pull requests based on the target branch and enforces minimum approval requirements.

## Features

- ✅ **Automatic Reviewer Assignment**: Automatically assigns individual reviewers and teams based on target branch
- 🎯 **Branch Pattern Matching**: Supports both exact branch names and simple glob patterns
- 👥 **Team Support**: Full support for GitHub team reviewers with membership verification
- 🔢 **Minimum Approvals**: Enforces minimum approval requirements before merge
- 💬 **PR Comments**: Adds informative comments when reviewers are automatically assigned
- 🔍 **Smart Detection**: Avoids duplicate assignments and excludes PR authors from review assignments

## Usage

### Basic Workflow

Add this to your `.github/workflows/` directory:

```yaml
name: Auto Assign Reviewers

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  assign-reviewers:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Assign Branch Reviewers
        uses: your-org/github-reviewers-action@v1
```

### With Custom PAT (Required for Teams)

For team reviewers, you need a PAT with appropriate permissions:

```yaml
      - name: Assign Branch Reviewers
        uses: your-org/github-reviewers-action@v1
        with:
          configuration-file: '.github/branch-reviewers.json'
          github-pat: ${{ secrets.REVIEWER_PAT }}
          fail-on-missing-approvals: 'true'
```

## Configuration

### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `configuration-file` | Path to the JSON configuration file | No | `.github/branch-reviewers.json` |
| `github-pat` | GitHub token for API access | No | `${{ github.token }}` |
| `fail-on-missing-approvals` | Whether to fail if required approvals aren't met | No | `false` |

### Outputs

| Output | Description |
|--------|-------------|
| `added-reviewers` | Boolean indicating if reviewers were added |
| `added-team-reviewers` | JSON array of team reviewers that were added |

## Configuration File Format

Create a JSON file (default: `.github/branch-reviewers.json`) with your reviewer configuration:

```json
{
  "branches": {
    "main": {
      "teams": ["acme-corp/senior-developers", "acme-corp/security-team"],
      "require_minimum": 2
    },
    "develop": {
      "reviewers": ["john-doe", "jane-smith"],
      "teams": ["acme-corp/developers"],
      "require_minimum": 1
    },
    "^feature/.*": {
      "reviewers": ["lead-dev"],
      "teams": ["acme-corp/developers"],
      "require_minimum": 1
    },
    "^hotfix/.*": {
      "teams": ["acme-corp/senior-developers", "acme-corp/ops-team"],
      "require_minimum": 2
    },
    "^release/v\\d+\\.\\d+": {
      "reviewers": ["release-manager"],
      "teams": ["acme-corp/qa-team"],
      "require_minimum": 2
    }
  }
}
```

### Configuration Options

#### Branch Matching
- **Exact Match**: Use the exact branch name (e.g., `"main"`, `"develop"`)
- **Wildcard Pattern**: Use `*` for simple pattern matching (e.g., `"feature/*"`, `"*hotfix"`, `"release*"`)

#### Wildcard Pattern Syntax

| Pattern | Description | Example Matches | Example Non-Matches |
|---------|-------------|-----------------|-------------------|
| `prefix*` | Starts with prefix | `feature*` matches `feature/login`, `feature-auth` | Won't match `my-feature` |
| `*suffix` | Ends with suffix | `*hotfix` matches `urgent-hotfix`, `fix-hotfix` | Won't match `hotfix-v1` |
| `*substring*` | Contains substring | `*release*` matches `pre-release-v1`, `my-release` | Won't match `deploy-v1` |
| `prefix*suffix` | Starts with prefix and ends with suffix | `v*-rc` matches `v1.0-rc`, `v2.1-rc` | Won't match `v1.0` or `beta-rc` |

**Examples:**
- `feature/*` - Matches `feature/login`, `feature/auth`, `feature/ui`
- `*hotfix` - Matches `urgent-hotfix`, `security-hotfix`
- `release*` - Matches `release/v1.0`, `release-candidate`
- `v*-rc` - Matches `v1.0-rc`, `v2.1-rc`

#### Branch Configuration Properties

| Property | Type | Description | Required |
|----------|------|-------------|----------|
| `reviewers` | Array of strings | Individual GitHub usernames to assign as reviewers | No |
| `teams` | Array of strings | GitHub teams to assign as reviewers (format: `org/team-name`) | No |
| `require_minimum` | Number | Minimum number of approvals required before merge | Yes |

### Example Configurations

#### Individual Reviewers Only
```json
{
  "branches": {
    "main": {
      "reviewers": ["alice", "bob", "charlie"],
      "require_minimum": 2
    }
  }
}
```

#### Teams Only
```json
{
  "branches": {
    "main": {
      "teams": ["acme-corp/frontend-team", "acme-corp/backend-team"],
      "require_minimum": 1
    }
  }
}
```

#### Mixed Configuration
```json
{
  "branches": {
    "main": {
      "reviewers": ["tech-lead"],
      "teams": ["acme-corp/senior-developers"],
      "require_minimum": 2
    }
  }
}
```

#### Wildcard Pattern Examples
```json
{
  "branches": {
    "feature/*": {
      "teams": ["acme-corp/developers"],
      "require_minimum": 1
    },
    "hotfix/*": {
      "reviewers": ["on-call-engineer"],
      "teams": ["acme-corp/ops-team"],
      "require_minimum": 2
    },
    "*hotfix": {
      "reviewers": ["on-call-engineer"],
      "require_minimum": 1
    },
    "release*": {
      "teams": ["acme-corp/release-team"],
      "require_minimum": 1
    },
    "v*-rc": {
      "teams": ["acme-corp/qa-team"],
      "require_minimum": 2
    }
  }
}
```

## Permissions

### Basic Usage (Individual Reviewers)
The default `GITHUB_TOKEN` has sufficient permissions for individual reviewer assignment.

### Team Reviewers
For team reviewer functionality, create a Personal Access Token (PAT) with these permissions:
- **Repository permissions**: Contents (read), Pull requests (write)
- **Organization permissions**: Members (read)

Add the PAT as a repository secret and reference it in your workflow:

```yaml
with:
  github-pat: ${{ secrets.REVIEWER_PAT }}
```

## How It Works

1. **Branch Detection**: Identifies the target branch of the pull request
2. **Configuration Matching**: Finds matching configuration using exact match first, then glob patterns
3. **Reviewer Assignment**: Automatically assigns missing reviewers and teams (excluding PR author)
4. **Approval Verification**: Checks current approvals against requirements
5. **Enforcement**: Optionally fails the workflow if minimum approvals aren't met

## Behavior Details

### Smart Assignment
- Skips reviewers who are already assigned
- Excludes the PR author from being assigned as a reviewer
- Only assigns missing reviewers/teams to avoid duplicates

### Team Membership Verification
- Verifies that approvers are actually members of required teams
- Uses GitHub API to check team membership
- Counts team member approvals toward minimum requirements

### Pattern Matching Priority
1. **Exact Match**: Tries exact branch name match first
2. **Wildcard Patterns**: Falls back to testing wildcard patterns in configuration order
3. **First Match Wins**: Uses the first matching pattern found

## Troubleshooting

### Common Issues

**Teams not being assigned**
- Ensure your PAT has organization member read permissions
- Verify team names are in the format `org/team-name`
- Check that the team exists and has the correct permissions

**Approvals not being counted**
- Verify team members have actually approved (not just commented)
- Check that the approver is a member of the required team
- Ensure the latest review from each user is an approval

**Wildcard patterns not working**
- Make sure to use `*` as a wildcard character
- `prefix*` matches branches starting with "prefix"
- `*suffix` matches branches ending with "suffix"
- `*substring*` matches branches containing "substring"
- `prefix*suffix` matches branches starting with "prefix" and ending with "suffix"

### Debug Information

The action provides detailed logging:
- Configuration loading and branch matching
- Reviewer assignment status
- Approval verification with team membership checks
- Progress toward minimum requirements

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## License

[MIT License](LICENSE)