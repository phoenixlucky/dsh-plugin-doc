# DeepSeek Harness Plugin Authoring and Installation Guide

[简体中文](README.md) | [English](README.en.md)

This guide is for projects that develop, distribute, and install plugins for DeepSeek Harness (DSH). A “plugin package” here means an npm package that can be installed by the DSH profile manager. A bundle is usually the distribution boundary: it declares a configuration patch and can combine one or more Cordis plugins.

## 1. Choose the Plugin Type

- **Installable bundle:** A complete feature package distributed to users. Its `package.json` declares `dsh.bundle.patch`, and the patch inserts or adjusts Cordis configuration entries. Users install the bundle through the DSH profile manager.
- **Cordis plugin:** A code unit that provides a service or feature. A bundle can reference and combine it; it is suitable for distribution to Harness users as a standalone npm package only after it has been packaged as an installable bundle.
- **Client plugin:** When browser-side code is needed, declare its entry point, platform, and injected dependencies through the DSH `dsh.client` configuration. Package browser resources using the resource-loading mechanism provided by the host; do not depend on absolute paths from a development machine.

Before development, confirm the target DSH version, runtime surface (such as Web or Desktop), required host services, and the user-visible behavior after installation. Do not infer Harness compatibility from a dependency version alone.

## 2. Package Name, Version, and Compatibility

`package.json` is the single source of truth for the plugin package identity and publish scope.

```json
{
  "name": "@scope/dsh-example",
  "version": "1.0.0",
  "type": "module",
  "engines": {
    "dsh": ">=0.2.1-rc.3 <0.3.0"
  },
  "dsh": {
    "bundle": {
      "patch": "./cordis.patch.yml"
    }
  }
}
```

- Use a stable, unique npm package name, and keep it consistent in `cordis.patch.yml`, entry metadata, marketplace indexes, the README, and installation commands. Renaming a package does not automatically create an alias for its old name.
- `version` is the plugin's own SemVer version; `engines.dsh` defines the compatible Harness version range. A dependency's version must not be mistaken for the plugin version.
- Increment the plugin version for every release. A published version must not be overwritten; fixes require a new version number.
- Declare only DSH version ranges that have been verified. The ranges of DSH APIs and peer dependencies must match that range.
- Put runtime imports in `dependencies`. Put APIs provided by Harness in `peerDependencies`, and also list build-time peers in `devDependencies`.

## 3. Code and Lifecycle

- Follow Cordis plugin conventions: use explicit `name`, `inject`, and configuration declarations, along with an `apply` entry point. Inject required services explicitly.
- Clean up listeners, timers, DOM nodes, stylesheets, and registrations created when the plugin is enabled when the plugin is unloaded. Repeated loading must not leave duplicate state behind.
- Scope client styles and resources to the plugin's own theme or ID. Use the theme variables and extension points documented by Harness; do not depend on generated CSS module class names or unrelated pages' internal structure.
- Preserve the host's existing settings, persistence, and interaction behavior. Respect `prefers-reduced-motion` for animation and set reasonable layout constraints for narrow windows.
- Put distributable resources in the project's resource directory, use semantic filenames, and keep file sizes under control. Include resources in the npm tarball and reference them at runtime through the host's resource loader.
- When referring to the package name in documentation, configuration, or patches, use the value in `package.json` to avoid leaving an old ID behind after a rename.

## 4. Complete the Publishing Checklist

Check `package.json` fields `files`, `exports`, and entry points, along with `dsh.bundle.patch`:

- `main`, `exports`, and `types` point to files that exist and are intended for publication.
- Every target in `exports` is included in the tarball. If source files are not intended for distribution, do not export `src/*`.
- The tarball contains the bundle patch, runtime entry point, type declarations, locales, icons, and all runtime resources referenced by the code.
- Do not rely on files on a development machine, private paths inside the Harness installation, or undeclared transitive dependencies.
- `publishConfig.access: "public"` declares the publication access level. It does not publish the package automatically or guarantee that the specified version exists in a registry.

Before publishing, build from a clean checkout and inspect the actual package contents:

```sh
pnpm install --frozen-lockfile
pnpm run build
pnpm pack --dry-run
```

Check each item in the output, including entry points, the patch, dependency metadata, and resources. Confirm that `exports` does not point to files missing from the tarball. After changing the package, regenerate the tarball and query the exact package name and version in the target registry.

Example query:

```sh
pnpm view @scope/dsh-example@1.0.0 version --registry=https://registry.npmjs.org/
```

If the registry returns `404`, first check the package name, version, and registry, and confirm that the version was actually published. Switching mirrors will not fix a typo, a rename, or a version that has not been published. If authentication is requested, check registry permissions separately; do not paste tokens into logs or issues.

## 5. Publish to a Registry

1. Confirm the target registry, scope permissions, package name, and version to be released.
2. Before publishing, inspect the tarball file list and confirm it contains no secrets, `.npmrc`, caches, private configuration, or unrelated source files.
3. Publish to the registry configured by the user. Public scoped packages need the appropriate public access setting.
4. From another environment or a clean profile, verify that the exact version can be queried, installed, and enabled. A successful publish command alone does not prove that users can install it.
5. Record the package name, version, supported DSH range, and installation method, then update the relevant README or marketplace index.

## 6. Install, Enable, and Uninstall

Use the DSH plugin manager to manage bundles in a profile:

```sh
# Install an exact version from a registry
dsh plugin --profile <profile> add @scope/dsh-example@1.0.0

# Install from a local directory; use an absolute path on Windows
dsh plugin --profile <profile> add E:\work\dsh-example

# Install from Git; publishing documentation should prefer a fixed commit or release tag
dsh plugin --profile <profile> add git+https://github.com/example/dsh-example.git#<commit>

# Uninstall
dsh plugin --profile <profile> remove @scope/dsh-example
```

Running `pnpm add` directly in a profile directory only changes npm dependencies and may not synchronize the DSH bundle selection and compatibility management. User-facing installation instructions should use `dsh plugin`. During development, use a local absolute path or `file:` spec, and avoid writing a machine-specific path as a general installation method.

After installation, confirm that the bundle appears in the target profile's plugin list, is enabled, and that the plugin's actual feature works. After uninstalling, confirm that registrations and resources created by the plugin have been removed, and document whether user data is retained or deleted.

## 7. Troubleshoot Installation Failures

| Symptom | Check first |
|---|---|
| Registry `404` | `package.json.name`, the requested version, whether it was published, and whether the current registry is correct. Do not start by repeatedly switching mirrors. |
| `401` / `403` | Scope login status, the registry token, and package access permissions. Hide or rotate any leaked token. |
| Compatibility rejection | The actual DSH version, `engines.dsh`, the Harness peer dependency range, and the plugin release notes. |
| Installation succeeds but the plugin does not load | Whether it was managed with `dsh plugin`, whether the correct profile was used, whether the bundle patch and entry point are in the package, and whether dependencies are complete. |
| Works on one machine but fails on another | Recheck the tarball contents for missing generated files; check for local paths, unpublished dependencies, or unstable Git content on the default branch. |

When reporting an issue to the maintainer, include the full package name and version, DSH version, installation method, registry hostname, error code, and logs with credentials removed. Do not send an Authorization header, npm token, or complete `.npmrc` file.

## 8. Pre-Release Checklist

- [ ] The package name matches across the manifest, patch, code, README, and marketplace metadata.
- [ ] The plugin version, `engines.dsh`, and dependency versions are each correct.
- [ ] The build succeeds; the tarball contains all exported files, the patch, and runtime resources.
- [ ] `exports` does not point to unpublished files; installation does not depend on development-machine state.
- [ ] The target registry can query the exact `name@version`.
- [ ] Installation, enablement, and uninstallation have been verified with the Harness plugin manager in a clean profile.
- [ ] Release notes and installation documentation are updated; error logs contain no credentials.
