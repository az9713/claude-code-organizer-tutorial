# Scope Discovery Engine

The scope discovery engine in `src/scanner.mjs` reconstructs Claude Code's configuration hierarchy by decoding the directory names under `~/.claude/projects/`, resolving them back to real filesystem paths, and building a tree of parent-child relationships. This is the foundation that every other part of CCO depends on -- without knowing what scopes exist and how they relate, there is nothing to scan or move.

## How `~/.claude/projects/` Encodes Filesystem Paths

Claude Code creates a subdirectory under `~/.claude/projects/` for each project it has been used in. The directory name encodes the project's absolute filesystem path by replacing every `/` with `-` and prepending a leading `-`.

For example, the project at `/home/user/mycompany/repo1` becomes the directory `~/.claude/projects/-home-user-mycompany-repo1/`.

The problem: directory names on the filesystem can themselves contain dashes. A directory called `my-company` produces the same dash-separated encoding as two separate directories `my` and `company`. The encoded name `-home-user-my-company-repo1` is ambiguous -- it could represent `/home/user/my-company/repo1` or `/home/user/my/company/repo1`. Resolving this ambiguity is what the greedy path decoding algorithm handles.

## The Greedy Path Decoding Algorithm

The function `resolveEncodedProjectPath()` (scanner.mjs lines 159-191) decodes an encoded directory name back to a real filesystem path using greedy longest-match-first resolution:

```javascript
async function resolveEncodedProjectPath(encoded) {
  const segments = encoded.replace(/^-/, "").split("-");
  let currentPath = "/";
  let i = 0;

  while (i < segments.length) {
    let matched = false;
    for (let end = segments.length; end > i; end--) {
      const candidate = segments.slice(i, end).join("-");
      const testPath = join(currentPath, candidate);
      if (await exists(testPath)) {
        const s = await safeStat(testPath);
        if (s && s.isDirectory()) {
          currentPath = testPath;
          i = end;
          matched = true;
          break;
        }
      }
    }
    if (!matched) {
      currentPath = join(currentPath, segments[i]);
      i++;
    }
  }

  if (await exists(currentPath)) return currentPath;
  return null;
}
```

### Walkthrough

Given the encoded name `-home-user-my-company-repo1`, the algorithm splits on dashes to get segments `["home", "user", "my", "company", "repo1"]`, then resolves left to right:

1. At `/`, try the longest candidate first: `home-user-my-company-repo1` -- does not exist. Shorten progressively until `home` matches `/home`. Advance to index 1.
2. At `/home`, try `user-my-company-repo1` -- does not exist. Eventually `user` matches `/home/user`. Advance to index 2.
3. At `/home/user`, try `my-company-repo1` -- does not exist. Try `my-company` -- it exists as a real directory. Match. Advance to index 4.
4. At `/home/user/my-company`, try `repo1` -- exists. Match. Done.
5. Result: `/home/user/my-company/repo1`.

The greedy strategy (longest match first) works because real directory names with dashes are typically longer than single path segments, so trying the longest candidate first resolves the ambiguity correctly in practice.

## Building the Scope Hierarchy

The `discoverScopes()` function (scanner.mjs lines 198-283) orchestrates the full discovery process:

1. **Create the global scope** -- a singleton representing `~/.claude/` itself, always present.
2. **Read all directories** in `~/.claude/projects/` and call `resolveEncodedProjectPath()` on each.
3. **Filter empty scopes** -- directories with no entries beyond `.DS_Store` are skipped (lines 235-244).
4. **Sort by path depth** (shorter paths first, then alphabetically) so that parent directories are processed before their children.
5. **Assign parent-child relationships** (lines 257-280): for each scope, find the deepest existing scope whose resolved `realPath` is a prefix of the current scope's path. That scope becomes the `parentId`. If no intermediate parent exists, the parent defaults to `global`.

Each scope object carries: `id` (the encoded directory name), `name` (human-readable), `type` (global, workspace, or project), `tag` (display label), `parentId`, `claudeProjectDir` (the `~/.claude/projects/<encoded>/` path), and `repoDir` (the resolved real filesystem path).

## Workspace Detection

A scope is classified as `workspace` rather than `project` when two conditions are met (lines 267-269):

1. Its parent is the `global` scope (it is a top-level entry).
2. At least one other scope's resolved path starts with this scope's path followed by `/`.

In other words, a workspace is a directory that contains child projects. For example, if both `/home/user/mycompany` and `/home/user/mycompany/repo1` appear as scopes, then `mycompany` is classified as a workspace. This distinction matters in the UI, where workspaces display as expandable tree nodes grouping their child projects.

## Why This Design Matters

The greedy path decoding algorithm is the critical piece -- without it, CCO cannot map encoded directory names back to the filesystem and the entire scope hierarchy collapses. The approach is pragmatic: it uses filesystem existence checks rather than trying to maintain a separate mapping database, which means it works correctly even as projects are created, moved, or deleted outside of CCO. The parent-child relationship builder and workspace classifier then layer a meaningful hierarchy on top, giving both the dashboard UI and the MCP server a structured view of all configuration scopes.
