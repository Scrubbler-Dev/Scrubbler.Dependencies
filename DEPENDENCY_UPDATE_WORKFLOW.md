# Dependency Update Workflow

`Scrubbler.Dependencies` is the shared dependency catalog and build-target source for the Scrubbler repositories. Consumer repositories include it as a git submodule, import its central package versions, and then receive updates through tag-triggered pull requests.

## What This Repository Owns

- `Directory.Packages.props` is the central NuGet version catalog used by the app, plugin, and shared library repositories.
- `global.json` pins shared MSBuild SDK settings, especially `Uno.Sdk`.
- `Scrubbler.Build.targets` copies the dependency repo's `global.json` into the consumer build root before non-test projects build.
- `Scrubbler.Plugins.targets` copies Debug plugin build outputs into `Scrubbler-2/Scrubbler/DebugPlugins` for local plugin development.
- `Catalog/Catalog.csproj` references the catalog packages so dependency changes can be restored and built in one small project.
- `.github/workflows/bump-consumers.yml` fans out a tagged dependency update to the repositories that consume this submodule.

Uno package versions are mostly controlled by the `Uno.Sdk` version in `global.json`. When updating Uno, change `global.json` rather than adding explicit Uno package versions unless the package is intentionally outside the SDK's implicit package set.

## Consumer Wiring

Most consumers use this repository as a submodule at `deps`:

```text
[submodule "deps"]
path = deps
url = https://github.com/Scrubbler-Dev/Scrubbler.Dependencies
```

`Scrubbler-2` uses `Scrubbler/deps`.

Consumers opt in through repo-local imports:

```xml
<Import Project="deps/Directory.Packages.props" />
<Import Project="deps/Scrubbler.Build.targets" />
<Import Project="deps/Scrubbler.Plugins.targets" />
```

The main app and shared base libraries import only the targets they need. Plugin repositories import both `Scrubbler.Build.targets` and `Scrubbler.Plugins.targets`.

## Normal Update Flow

1. Edit dependency inputs in `Scrubbler.Dependencies`.

   Common changes:

   - Update NuGet versions in `Directory.Packages.props`.
   - Update `Uno.Sdk` in `global.json`.
   - Update shared build behavior in `Scrubbler.Build.targets` or `Scrubbler.Plugins.targets`.

2. Validate the central catalog locally.

   ```powershell
   dotnet restore .\Catalog.slnx
   dotnet build .\Catalog.slnx -c Release
   ```

3. If the change affects a Scrubbler-owned package, publish that package from its own repository.

   `Scrubbler.PluginBase` and `Scrubbler.MediaPlayerScrobblerBase` consume this shared dependency baseline, but their NuGet versions are pinned by the repos that reference them, not by `Scrubbler.Dependencies`. Their release workflows should fan out version bumps to their own consumers after publishing.

4. Commit and tag the dependency update.

   ```powershell
   git add Directory.Packages.props global.json Scrubbler.Build.targets Scrubbler.Plugins.targets Catalog
   git commit -m "Update dependencies"
   git tag vX.Y.Z
   git push origin main
   git push origin vX.Y.Z
   ```

   Tags matching `v*` or `deps-*` trigger the fan-out workflow.

5. Let `bump-consumers.yml` open consumer PRs.

   The workflow:

   - Computes the pushed tag and the dependency commit behind it.
   - Lists non-archived, non-fork repositories under `Scrubbler-Dev`.
   - Skips repositories without `.gitmodules`.
   - Finds the submodule whose URL contains `Scrubbler.Dependencies`.
   - Creates or resets `chore/bump-deps-<tag>` from the consumer default branch.
   - Checks out the dependency submodule to the tagged commit.
   - Copies the dependency `global.json` to the consumer repository root when present.
   - Commits the submodule pointer and copied `global.json`.
   - Pushes the branch and opens a PR named `chore: bump Scrubbler.Dependencies to <tag>`.

   The workflow uses the `SCRUBBLER_PLUGINS_PAT` secret as `GH_TOKEN`, because it needs to clone, push branches, and create PRs across repositories.

6. Review and merge the consumer PRs after CI passes.

   Consumer test workflows checkout submodules recursively, install `.NET 10.0.x`, then run:

   ```bash
   dotnet build -c Release
   dotnet test -c Release --no-build --verbosity normal
   ```

   `Scrubbler-2` runs those commands from its `Scrubbler` subdirectory.

7. Refresh the local multi-repo workspace after merges.

   From the parent workspace:

   ```powershell
   .\pull-all-repos.ps1 -IncludeSubmodules
   ```

## Bulk Merge And Cleanup Helpers

`Scrubbler-2` contains optional GitHub CLI helper scripts for the PR fan-out branches:

- `merge-dependency-branches.sh` scans `Scrubbler-Dev/Scrubbler*` repositories for `chore/bump-deps-*` branches. It merges open PRs only when they are not draft, have `CLEAN` merge state, and have no pending or failing checks.
- `cleanup-dependency-branches.sh` scans the same branch pattern, closes matching open PRs, and deletes the remote branches.

Both scripts require `gh` authentication with enough repository permissions. `merge-dependency-branches.sh` also requires `jq`.

## Manual Consumer Bump

Use this when the fan-out workflow cannot update a repository automatically.

For a normal plugin or library consumer:

```powershell
git checkout main
git pull --ff-only
git checkout -B chore/bump-deps-vX.Y.Z
git submodule update --init --recursive

Push-Location deps
git fetch --all --tags
git checkout vX.Y.Z
Pop-Location

Copy-Item .\deps\global.json .\global.json -Force
git add .\deps .\global.json
git commit -m "chore: bump Scrubbler.Dependencies to vX.Y.Z"
git push -u origin chore/bump-deps-vX.Y.Z
```

For `Scrubbler-2`, use `Scrubbler/deps` as the submodule path:

```powershell
Push-Location .\Scrubbler\deps
git fetch --all --tags
git checkout vX.Y.Z
Pop-Location

Copy-Item .\Scrubbler\deps\global.json .\global.json -Force
git add .\Scrubbler\deps .\global.json
```

Open a PR against the consumer default branch and let its normal test workflow validate the bump.

## Things To Watch

- A dependency tag is the release signal. Pushing `main` alone updates `Scrubbler.Dependencies` but does not fan out to consumers.
- Scrubbler-owned packages are intentionally not pinned in this shared catalog. Keep `Scrubbler.PluginBase` and `Scrubbler.MediaPlayerScrobblerBase` versions in the repositories that reference them.
- The fan-out workflow copies `global.json` to the consumer repository root. `Scrubbler.Build.targets` also copies `../deps/global.json` to the project parent directory during non-test builds, so SDK pins should remain aligned even in repos with a nested solution root.
- The workflow skips repositories with no `.gitmodules` file or no submodule URL containing `Scrubbler.Dependencies`.
- If a consumer is already at the tagged submodule commit and copied `global.json`, no PR is opened.
- Release workflows for the app, plugins, and shared NuGet packages checkout submodules recursively, so merged dependency bumps affect future release builds automatically.
- `Scrubbler.Dependencies` currently has no separate PR validation workflow; use the catalog restore/build commands before tagging.
