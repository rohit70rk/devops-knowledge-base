# Git Workflow: Keep Infrastructure Files Only in `dev-test`

Use `dev-test` as the dedicated branch for development and testing, while using `main` for collaboration, pull requests, and shared integration before changes move to `prod`.

Infrastructure files kept only in `dev-test`:

* `Dockerfile`
* `.gitlab-ci.yml`
* `.dockerignore`

---

## 1. Setup the Merge Shield

Prevent infrastructure files from crossing branches using Git merge rules.

### Create `.gitattributes`

Add this on **both branches**:

```bash id="5yh9ih"
echo ".gitlab-ci.yml merge=ours" >> .gitattributes
echo "Dockerfile merge=ours" >> .gitattributes
echo ".dockerignore merge=ours" >> .gitattributes

git add .gitattributes
git commit -m "Add merge strategy for infrastructure files"
```

### Enable the `ours` merge driver

Run once per machine:

```bash id="z6q5ri"
git config merge.ours.driver true
```

This tells Git to always keep the current branch’s version of these files during merges.

---

## 2. Development Workflow (`dev-test`)

### Purpose

Use `dev-test` for:

* Active development
* Testing
* Docker setup
* GitLab CI/CD configuration
* Experimental changes

### Typical workflow

```bash id="8w1r0g"
git switch dev-test

# work on features
git add .
git commit -m "Add new feature"

git push origin dev-test
```

All infrastructure files remain available here.

---

## 3. Collaboration Workflow (`dev-test` → `main`)

### Purpose

Use `main` as the shared collaboration branch where:

* Collaborators open pull requests
* Features are reviewed
* Code is merged together
* Stable integration happens before deployment to `prod`

### Merge changes into `main`

```bash id="nq5a0v"
git switch main
git merge dev-test
git push origin main
```

### Result

`main` receives application code changes while excluding infrastructure files because of the `merge=ours` rules.

---

## 4. Sync Shared Changes Back (`main` → `dev-test`)

### Purpose

Keep `dev-test` updated with merged collaborator changes from `main` without losing infrastructure files.

### Commands

```bash id="x7r9nc"
git switch dev-test
git merge main
git push origin dev-test
```

### Result

`dev-test` stays fully configured for development and testing, including Docker and CI files.

---

## 5. Emergency Recovery

If infrastructure files are accidentally staged for deletion:

### Undo the merge

```bash id="v2f6yq"
# If merge is not committed yet
git restore --staged Dockerfile .gitlab-ci.yml .dockerignore

# If a fast-forward merge already completed
git reset --hard HEAD~1
```

### Re-merge safely

```bash id="7hr6be"
git merge main --no-commit --no-ff
```

### Restore infrastructure files

```bash id="s1f1zp"
git checkout HEAD -- Dockerfile .gitlab-ci.yml .dockerignore
```

### Finalize

```bash id="x4m2sk"
git commit -m "Merge main into dev-test and preserve infrastructure files"
```
