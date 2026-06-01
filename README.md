# DCMTK fork for Insight Toolkit (ITK)

This project is a fork of the DCMTK source hosted at https://github.com/DCMTK/dcmtk.git.

This project contains the changes required to embed DCMTK into ITK. These are
primarily build-system changes that let DCMTK be built as an ITK
`ExternalProject` and link ITK's own bundled image codecs (PNG, TIFF, ZLIB,
JPEG), plus the WASI/Emscripten and GSPS/CSPS/plastimatch enabling patches that
ITK and ITK-wasm carry on top of an upstream DCMTK release.

# What is the branch naming convention?

Each fork branch is named with the pattern `for/itk-dcmtk-X.Y.Z-SHA{7}`

where:
- `X.Y.Z` is the upstream DCMTK release the ITK overlay is built on (e.g. `3.7.0`)
- `SHA{7}` is the 7-character commit hash of that upstream `DCMTK-X.Y.Z` release tag

For example, `for/itk-dcmtk-3.7.0-ccfd10b` carries the ITK overlay on top of
upstream tag `DCMTK-3.7.0` (commit `ccfd10b`).

The default branch of this fork is `welcome`, an orphan branch holding only this
document. ITK pins a specific commit of a `for/itk-dcmtk-*` branch in
`Modules/ThirdParty/DCMTK/DCMTKGitTag.cmake` (the `DCMTK_GIT_TAG` variable).

Additional documentation on developing ITK:
https://docs.itk.org/en/latest/contributing/index.html

# Workflow for Updating

## Setup repositories

1. Clone (or reuse) the fork and ensure the tree is clean:

```
git clone git@github.com:InsightSoftwareConsortium/DCMTK.git
cd DCMTK
```

2. Wire the upstream remote:

```
git remote add upstream https://github.com/DCMTK/dcmtk.git
```

3. Fetch:

```
git fetch origin
git fetch upstream
```

## Create a new branch following the convention

4. Fetch the new upstream DCMTK release tag and define the workflow variables.
   DCMTK encodes the version in the tag name (`DCMTK-X.Y.Z`), so derive it from
   there and peel the annotated tag to its commit for the branch-name SHA:

```
# Set the upstream DCMTK release tag to update to (e.g. 3.7.0)
ITK_DCMTK_TARGET_VERSION=3.7.0
ITK_DCMTK_TARGET_TAG=DCMTK-${ITK_DCMTK_TARGET_VERSION}

git fetch upstream refs/tags/${ITK_DCMTK_TARGET_TAG}:refs/tags/${ITK_DCMTK_TARGET_TAG}

# Peel the annotated tag to the commit it points to (NOT the tag object).
SHA=$(git rev-parse --short refs/tags/${ITK_DCMTK_TARGET_TAG}^{commit})
echo "SHA [${SHA}]"

NEW_BRANCH="for/itk-dcmtk-${ITK_DCMTK_TARGET_VERSION}-${SHA}"
echo "NEW_BRANCH [${NEW_BRANCH}]"
```

5. Fetch the prior ITK fork branch and inspect the overlay commits to replay:

```
# Set the current ITK fork branch (the commit pinned in DCMTKGitTag.cmake).
# (e.g. for/itk-dcmtk-3.6.0-abcdef1)
ITK_CURRENT_BRANCH=for/itk-dcmtk-3.7.0-ccfd10b

git fetch origin ${ITK_CURRENT_BRANCH}:${ITK_CURRENT_BRANCH}
git log --oneline refs/tags/${ITK_DCMTK_TARGET_TAG}..${ITK_CURRENT_BRANCH}
```

6. Determine the base commit where the ITK overlay patches begin, then branch and
   rebase the overlay onto the new upstream release (resolving conflicts as
   needed). Use `--rebase-merges` to preserve any merge topology in the overlay:

```
# Find where ITK_CURRENT_BRANCH diverged from its upstream base.
REBASE_BASE=$(git merge-base ${ITK_CURRENT_BRANCH} refs/tags/${ITK_DCMTK_TARGET_TAG})
echo "REBASE_BASE [${REBASE_BASE}]"

git checkout -b ${NEW_BRANCH} ${ITK_CURRENT_BRANCH}
git rebase --rebase-merges --onto refs/tags/${ITK_DCMTK_TARGET_TAG} ${REBASE_BASE} ${NEW_BRANCH}
```

   If the rebase encounters conflicts that are difficult to resolve, abort it and
   cherry-pick the overlay commits individually instead:

```
git rebase --abort

# List the overlay commits (oldest-first) from step 5 and cherry-pick them onto
# the new upstream base:
git checkout -b ${NEW_BRANCH} refs/tags/${ITK_DCMTK_TARGET_TAG}
git cherry-pick <oldest-overlay-sha> ... <newest-overlay-sha>
```

7. Verify the overlay replayed cleanly:

```
git log  --oneline refs/tags/${ITK_DCMTK_TARGET_TAG}..HEAD
git diff --stat    refs/tags/${ITK_DCMTK_TARGET_TAG}..HEAD
```

8. Publish the branch:

```
git push origin ${NEW_BRANCH}:${NEW_BRANCH}
```

9. In ITK, update `Modules/ThirdParty/DCMTK/DCMTKGitTag.cmake`: set `DCMTK_GIT_TAG`
   to the tip commit of `${NEW_BRANCH}`, update the trailing branch-name comment,
   and refresh the patch-list comment (`git shortlog refs/tags/${ITK_DCMTK_TARGET_TAG}..HEAD`).
   Build with `Module_ITKIODCMTK=ON`, `ITK_USE_SYSTEM_DCMTK=OFF`, and confirm the
   `ITKIODCMTK` and `GDCM` test labels pass before opening the ITK pull request.
