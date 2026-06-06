# svn-to-azure-ado


# SVN to Azure DevOps (ADO) Git Migration Guide

Tested on: **CommonComponents** SVN → ADO (May 2026)  
Next target: **3rdParty** SVN → ADO  
Environment: Windows 11 + WSL2 (Ubuntu)

---

## Prerequisites

Run once in WSL to verify tools:
```bash
git --version          # need 2.x
git svn --version      # need git-svn bundled
svn --version --quiet  # need 1.x
```

Fix credential manager for WSL (avoids broken Windows GCM path):
```bash
git config --global credential.helper store
```

---

## Migration Steps

### STEP 1 — Create working directory

```bash
# Set these variables for your migration
SVN_URL="https://gullstrand.zeiss.org/svn/3rdParty"
ADO_URL="https://ZEISSgroup-MED@dev.azure.com/ZEISSgroup-MED/BLR_REF_COVERY_SW_PLATFORM/_git/czm-thirdparty"
WORK_DIR="/mnt/c/svn-thirdparty"
CLONE_DIR="$WORK_DIR/ThirdParty-git"
SVN_USER="x6rgunna"

mkdir -p "$WORK_DIR"
cd "$WORK_DIR"
```

---

### STEP 2 — Check SVN repo structure

```bash
svn info "$SVN_URL" --username "$SVN_USER"
svn list "$SVN_URL" --username "$SVN_USER" | head -30
```

> **Note:** If output shows `trunk/`, `branches/`, `tags/` subdirectories → standard layout.  
> If output shows project folders directly → **flat layout** (use `--no-standard-layout` approach below).

---

### STEP 3 — Extract SVN authors list

```bash
# Get raw list of all commit authors
svn log "$SVN_URL" --username "$SVN_USER" -q \
  | grep "^r" \
  | awk '{print $3}' \
  | sort -u \
  > "$WORK_DIR/authors_raw.txt"

cat "$WORK_DIR/authors_raw.txt"
```

---

### STEP 4 — Create authors.txt mapping file

Format: `svn_username = Full Name <email@zeiss.com>`

```bash
cat > "$WORK_DIR/authors.txt" << 'EOF'
inavinay = Vinayak Ajay <ajay.vinayak@zeiss.com>
insnagyi = Nagaiyan Sathishkumar <sathishkumar.nagaiyan@zeiss.com>
m1hkoe = Koebe Hardy <hardy.koebe@zeiss.com>
ogabey = Unknown User <unknown@zeiss.com>
ogako = Gebauer Andreas <andreas.gebauer@zeiss.com>
ogamu = Pauliks Achim <achim.pauliks@zeiss.com>
ogaputsc = Putsche Alexander <alexander.putsche@zeiss.com>
ogawe = Waldheim Axel <axel.waldheim@zeiss.com>
ogbuild = Unknown User <unknown@zeiss.com>
ogbuild_confidential = Unknown User <unknown@zeiss.com>
ogmar = Unknown User <unknown@zeiss.com>
ogmblo = Bloos Marco <marco.bloos@zeiss.com>
ogmle = Lehnort Marco <marco.lehnort@zeiss.com>
ogrkoehl = Koehler Rainer <rainer.koehler@zeiss.com>
ogrod = Roeder Gerd <gerd.roeder@zeiss.com>
ogsweisl = Weisleder Stefan <stefan.weisleder@zeiss.com>
ogupe = Peterlein Ulf <ulf.peterlein@zeiss.com>
x1amahmo = Unknown User <unknown@zeiss.com>
x1hquanz = Unknown User <unknown@zeiss.com>
x1omuell = Unknown User <unknown@zeiss.com>
x1tblumh = Unknown User <unknown@zeiss.com>
x6hgupta = Gupta Himanshu [ext] <himanshu.gupta.ext@zeiss.com>
x6hsodha = sodha Hima [ext] <hima.sodha.ext@zeiss.com>
x6mmanja = Manjari Mallikarjun [ext] <mallikarjun.manjari.ext@zeiss.com>
x6mpanch = Panchal Mayank [ext] <mayank.panchal.ext@zeiss.com>
x6rajaks = S Rajakumar [ext] <rajakumar.s.ext@zeiss.com>
x6rtamar = Tamarala Raju [ext] <raju.tamarala.ext@zeiss.com>
xoazil = Unknown User <unknown@zeiss.com>
y3fil = Fiedler Lars <lars.fiedler@zeiss.com>
EOF
```

> **Add any new authors** found in `authors_raw.txt` that are not in the list above.  
> If `git svn` hits an unknown author during fetch it will abort — add them and re-run.

---
ramesh@IN05N001GB:/mnt/c/svn-thirdparty/ThirdParty-git$ svn info https://gullstrand.zeiss.org/svn/3rdParty --username x6rgunna | grep "^Revision:"
Authentication realm: <https://gullstrand.zeiss.org:443> SVN Repositories
Password for 'x6rgunna': ****************

Revision: 5152
## If disconnected will retry after 15sec
cd /mnt/c/svn-thirdparty/ThirdParty-git
while true; do
  git svn fetch && echo "DONE!" && break
  echo "Disconnected. Retrying in 15s..."
  sleep 15
done

### STEP 5 — Clone the SVN repo into Git

**For flat layout repos (no trunk/branches/tags) — use this:**
```bash
git svn clone \
  "$SVN_URL" \
  --no-metadata \
  --authors-file="$WORK_DIR/authors.txt" \
  --username "$SVN_USER" \
  "$CLONE_DIR"
```

**For standard layout repos (has trunk/branches/tags) — use this instead:**
```bash
git svn clone \
  "$SVN_URL" \
  --stdlayout \
  --no-metadata \
  --authors-file="$WORK_DIR/authors.txt" \
  --username "$SVN_USER" \
  "$CLONE_DIR"
```

> Enter your network password when prompted.  
> This replays every SVN revision as a git commit — **expect 30–120 minutes** for large repos.

---

### STEP 6 — Handle clone crash (signal 6 / heap corruption)

If `git svn clone` crashes with `error: git-svn died of signal 6`, fetch in batches:

```bash
cd "$CLONE_DIR"

cd /mnt/c/svn-thirdparty/ThirdParty-git

git config http.postBuffer 524288000

# Get all commits in order (oldest first)
COMMITS=$(git rev-list --reverse master)
TOTAL=$(echo "$COMMITS" | wc -l)
BATCH=200

echo "Total commits: $TOTAL"

i=0
for sha in $COMMITS; do
  i=$((i + 1))
  if [ $((i % BATCH)) -eq 0 ] || [ $i -eq $TOTAL ]; then
    echo "=== Pushing commit $i/$TOTAL: $sha ==="
    git push origin $sha:refs/heads/master --force
    if [ $? -ne 0 ]; then
      echo "FAILED at commit $i: $sha"
      echo "Retrying with smaller batch..."
      git push origin $sha:refs/heads/master --force
      if [ $? -ne 0 ]; then
        echo "Still failing. This commit may be too large."
        echo "Large files in this commit:"
        git diff-tree -r --no-commit-id $sha | sort -k4 -n -r | head -5
        break
      fi
    fi
  fi
done

---

### STEP 7 — Verify local clone

```bash
cd "$CLONE_DIR"
echo "Total commits: $(git log --oneline | wc -l)"
git log --oneline -5
git branch -a
```

Expected output: commits on `master` branch, `remotes/git-svn` tracking ref.

---
### Identify how many remotes done sofor
ramesh@IN05N001GB:/mnt/c/svn-thirdparty/ThirdParty-git$ git log --oneline refs/remotes/git-svn 2>/dev/null | wc -l

4319
### Identify total clones
ramesh@IN05N001GB:/mnt/c/svn-thirdparty$ svn info https://gullstrand.zeiss.org/svn/3rdParty --username x6rgunna | grep "^Revision:"
Authentication realm: <https://gullstrand.zeiss.org:443> SVN Repositories
Password for 'x6rgunna': ****************

Revision: 5152

### STEP 8 — Add ADO remote and push

```bash
cd "$CLONE_DIR"

# Add ADO as remote (skip if already set)
git remote add origin "$ADO_URL"

# Verify remote
git remote -v
```

Push in batches of 50 commits to avoid ADO's 5 GB per-push limit  
(repos with large binaries like .ova, .exe, .rpm will exceed the limit):

```bash
mapfile -t COMMITS < <(git log --reverse --format="%H" master)
TOTAL=${#COMMITS[@]}
BATCH=50

echo "Total commits to push: $TOTAL"

for ((i=BATCH-1; i<TOTAL; i+=BATCH)); do
  SHA=${COMMITS[$i]}
  echo "=== Pushing commits up to $((i+1))/$TOTAL: ${SHA:0:7} ==="
  git push origin ${SHA}:refs/heads/master
done

echo "=== Final push ==="
git push origin master
```

> **If a batch fails with auth error:** Run the resume script below.  
> **If a batch fails with 5 GB size error:** Reduce `BATCH=50` to `BATCH=10` and restart.
### split and push 

cd /mnt/c/svn-thirdparty/ThirdParty-git

# Resume from commit 601, push every 50 commits instead of 200
COMMITS=$(git rev-list --reverse master)
TOTAL=$(echo "$COMMITS" | wc -l)
BATCH=50
SKIP=600  # already pushed up to 600

i=0
for sha in $COMMITS; do
  i=$((i + 1))
  [ $i -le $SKIP ] && continue
  if [ $((i % BATCH)) -eq 0 ] || [ $i -eq $TOTAL ]; then
    echo "=== Pushing commit $i/$TOTAL: $sha ==="
    while true; do
      git push origin $sha:refs/heads/master --force && break
      echo "Failed. Retrying in 10s..."
      sleep 10
    done
  fi
done
echo "=== ALL DONE ==="

---

### STEP 9 — Resume after authentication failure

```bash
cd "$CLONE_DIR"

# Find where ADO currently is
REMOTE_SHA=$(git ls-remote origin refs/heads/master | awk '{print $1}')
echo "ADO is at: $REMOTE_SHA"

# Find resume index
mapfile -t COMMITS < <(git log --reverse --format="%H" master)
TOTAL=${#COMMITS[@]}
RESUME=0
for i in "${!COMMITS[@]}"; do
  if [[ "${COMMITS[$i]}" == "$REMOTE_SHA" ]]; then
    RESUME=$((i+1))
    echo "Resuming from commit $((RESUME+1))/$TOTAL"
    break
  fi
done

# Push one commit at a time from resume point
for ((i=RESUME; i<TOTAL; i++)); do
  SHA=${COMMITS[$i]}
  echo "=== Pushing commit $((i+1))/$TOTAL: ${SHA:0:7} ==="
  git push origin ${SHA}:refs/heads/master
  if [ $? -ne 0 ]; then
    echo "FAILED at commit $((i+1)): $SHA"
    echo "Large files in this commit:"
    git ls-tree -r --long $SHA | sort -k4 -rn | head -5
    break
  fi
done

echo "=== Final push ==="
git push origin master
```

---

### STEP 10 — Fix default branch visibility in ADO

After pushing, the files may not be visible if ADO's default branch is `main` but you pushed to `master`.

**Check:**
```bash
git ls-remote origin
# Look for HEAD → if it points to refs/heads/main, files won't show
```

**Fix Option A — Change default branch in ADO portal (recommended):**
1. Go to: `https://dev.azure.com/ZEISSgroup-MED/BLR_REF_COVERY_SW_PLATFORM/_settings/repositories`
2. Click the target repo → **Default branch** → change to `master`

**Fix Option B — Push master onto main branch:**
```bash
git push origin master:main --force
```

---

### STEP 11 — Final verification

```bash
cd "$CLONE_DIR"

echo "--- Local commits ---"
git log --oneline | wc -l

echo "--- Latest 3 commits ---"
git log --oneline -3

echo "--- ADO HEAD ---"
git ls-remote origin refs/heads/master
```

Local and ADO SHAs must match. Then open the ADO repo URL in browser to confirm files are visible.

---

## Completed Migrations

| SVN Source | ADO Target | Commits | Date |
|---|---|---|---|
| `https://gullstrand.zeiss.org/svn/binary_pool/RefractiveLasers/CommonComponents/` | `czm-dummy` | 397 | May 2026 |

## Pending Migrations

| SVN Source | ADO Target | SVN HEAD Rev |
|---|---|---|
| `https://gullstrand.zeiss.org/svn/3rdParty/` | `czm-thirdparty` | 5150 |

---

## Known Issues & Fixes

| Issue | Cause | Fix |
|---|---|---|
| `git-svn died of signal 6` | Heap corruption on large repos | Fetch in batches: `git svn fetch -r START:END` |
| `TF402462: push size > 5120 MB` | Binary files (.ova, .exe, .rpm) in commits | Push in batches of 10–50 commits |
| `Authentication failed` during batch push | PAT expired mid-session | Use resume script (Step 9); run `git config --global credential.helper store` |
| No files visible in ADO portal | ADO default branch is `main`, pushed to `master` | Change default branch in ADO settings or push `master:main --force` |
| `git-credential-manager.exe: not found` | Windows GCM path has spaces, breaks WSL | Run `git config --global credential.helper store` |
| `URL non-existent in revision X` | Repo has no `trunk/` folder (flat layout) | Use `git svn clone` without `--stdlayout` or `--trunk` flags |
