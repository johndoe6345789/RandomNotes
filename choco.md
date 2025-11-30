# Private Chocolatey Repository on GitHub

## Simple GitHub Pages Feed

This document describes how to host your own private Chocolatey package repository on GitHub using only GitHub Pages and `.nupkg` files. No server, no NuGet backend, no API keys. Chocolatey supports simple static file feeds, which makes the entire workflow seamless.

---

## 1. Create Repository

Create a new GitHub repository, for example:

```text
my-choco-repo
```

Add a folder to hold your packages:

```text
packages/
```

Your directory structure:

```text
my-choco-repo/
  packages/
```

Commit and push.

---

## 2. Enable GitHub Pages

In the repo:

1. Go to Settings → Pages.
2. Choose:

   * Source: Deploy from a branch
   * Branch: `main` (or `master`)
   * Folder: `/root` (or `/docs`)

After activation, your repository gets a public URL similar to:

```text
https://<username>.github.io/my-choco-repo/
```

Your packages folder will be:

```text
https://<username>.github.io/my-choco-repo/packages/
```

This URL is now a valid Chocolatey feed.

---

## 3. Create and Package Chocolatey Packages

Generate a package skeleton:

```powershell
choco new mytool
```

Edit `mytool.nuspec` and add metadata such as `id`, `version`, `authors`, and description.

Add installation logic under:

```text
tools/chocolateyinstall.ps1
tools/chocolateyuninstall.ps1
```

Pack the package:

```powershell
choco pack mytool.nuspec
```

This produces:

```text
mytool.1.0.0.nupkg
```

---

## 4. Publish Packages to GitHub

Copy your `.nupkg` files into the `packages/` directory in your repo:

```text
my-choco-repo/
  packages/
    mytool.1.0.0.nupkg
```

Commit and push:

```bash
git add packages/mytool.1.0.0.nupkg
git commit -m "Add mytool 1.0.0"
git push
```

Verify in a browser that the `.nupkg` is visible and downloadable.

---

## 5. Add Your GitHub Repo as a Chocolatey Source

On any Windows machine where you want to consume this feed:

```powershell
choco source add `
  -n="mygithub" `
  -s="https://<username>.github.io/my-choco-repo/packages/"
```

Verify sources:

```powershell
choco source list
```

You should see `mygithub` listed.

---

## 6. Installing Packages

Install from your private feed by source name:

```powershell
choco install mytool -y -s mygithub
```

Or using the raw URL directly:

```powershell
choco install mytool -y -s "https://<username>.github.io/my-choco-repo/packages/"
```

Upgrade a package:

```powershell
choco upgrade mytool -y -s mygithub
```

Chocolatey will select the latest version found in your feed.

---

## 7. Using the Source in Scripts / CI

Example GitHub Actions step:

```yaml
- name: Use private choco feed
  shell: pwsh
  run: |
    choco source add -n=mygithub -s="https://<username>.github.io/my-choco-repo/packages/"
    choco install mytool -y -s mygithub
```

Example PowerShell script snippet:

```powershell
$PrivateChocoSource = "https://<username>.github.io/my-choco-repo/packages/"

choco source add -n="mygithub" -s=$PrivateChocoSource -y
choco install mytool -y -s mygithub
```

---

## 8. Disable Public Chocolatey Repo (Optional)

If you want to restrict machines so they *only* use your private repo:

```powershell
choco source disable -n=chocolatey
```

You can re-enable it later:

```powershell
choco source enable -n=chocolatey
```

---

## 9. Updating Packages

To publish a new version:

1. Bump the version in your `.nuspec`.

2. Re-pack:

   ```powershell
   choco pack mytool.nuspec
   ```

   Produces e.g. `mytool.1.1.0.nupkg`.

3. Copy the new `.nupkg` into `packages/` and commit:

   ```bash
   cp path/to/mytool.1.1.0.nupkg packages/
   git add packages/mytool.1.1.0.nupkg
   git commit -m "mytool 1.1.0"
   git push
   ```

Clients can then update with:

```powershell
choco upgrade mytool -y -s mygithub
```

Chocolatey will see all available versions and pick the latest according to semantic versioning.
