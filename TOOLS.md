# TOOLS.md - Atlas

## Your Environment

### Workspace
- **Location:** ~/work/atlas/
- **Git Identity:** Atlas AI Architect <atlas-ai-architect@users.noreply.github.com>
- **SSH Key:** ~/.ssh/id_atlas

### GitHub Access
- **Host Alias:** github-atlas
- **Account:** atlas-ai-architect

## How to Clone Repos

```bash
cd ~/work/atlas/
git clone git@github-atlas:owner/repo.git
```

Or use HTTPS (auto-rewritten to SSH):
```bash
cd ~/work/atlas/
git clone https://github.com/owner/repo.git
```

## Using gh CLI

Atlas has an isolated GitHub CLI configuration to avoid conflicts with other team members.

**Config location:** `~/.config/gh-atlas/`

**Usage:**
```bash
export GH_CONFIG_DIR=~/.config/gh-atlas
cd ~/work/atlas/my-repo
gh issue list
gh pr create --title "Design: Add architecture diagram"
```

**Forking a repo:**
```bash
export GH_CONFIG_DIR=~/.config/gh-atlas
gh repo fork owner/repo --remote
```

If you need to work on a repo outside ~/work/atlas/, specify the repo explicitly:
```bash
export GH_CONFIG_DIR=~/.config/gh-atlas
gh repo view owner/repo --web
```

Note: Always set `GH_CONFIG_DIR` before using `gh` commands to ensure you're using Atlas's credentials.

## Your Tools

- **Architecture diagrams:** Use canvas or draw.io exports
- **Documentation:** Markdown in the repo's docs/ folder
- **Design decisions:** Record in ADRs (Architecture Decision Records)

## Collaboration

- **With Forge:** Provide clear specifications, review implementation plans
- **With Sentinel:** Design for testability, document edge cases
- **With Maxi:** Report blockers, escalate architectural concerns
