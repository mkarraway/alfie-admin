# Git LFS setup guide

How to set up Git LFS for images in this repo, plus the rule Claude follows.

## 1. Install Git LFS

- Mac: `brew install git-lfs`
- Windows: comes with Git for Windows

## 2. Turn it on (once per computer)

```bash
git lfs install
```

## 3. Track image types in the repo

Run inside the repo folder:

```bash
git lfs track "*.png" "*.jpg" "*.jpeg" "*.gif" "*.webp" "*.heic" "*.psd" "*.tif" "*.tiff"
git add .gitattributes
git commit -m "Track images with Git LFS"
git push
```

## 4. Check it works

Add an image, commit it, then run `git lfs ls-files`. The image should be listed.

> **Images committed before this** stay in normal git history. `git lfs migrate import --include="*.png,*.jpg" --everything` moves them to LFS, but it rewrites history and needs a force push. Check with anyone else using the repo first.

## 5. Add this to CLAUDE.md

Paste this block into the `CLAUDE.md` file at the root of the repo:

```markdown
## Images and binary files: Git LFS only

- Never commit images (png, jpg, jpeg, gif, webp, heic, psd, tif, tiff) as regular git files.
- Images go into the repo only through Git LFS.
- Before committing any image, run `git check-attr filter -- <file>`. It must say `filter: lfs`.
  If it doesn't, stop. Add the pattern with `git lfs track "*.ext"`, commit `.gitattributes`, and only then add the image.
- After committing, check with `git lfs ls-files` that the image is listed.
- If `git lfs` is not installed, do not commit images. Tell me instead.
- Never run `git lfs migrate` or force push without asking me first.
- Setup steps for Git LFS are in `GIT_LFS_SETUP.md`.
```

SVG files are text, so they can stay in normal git unless they are very large.
