# Lab 3: Git Branching – Creating, Pushing, Merging & Deleting a Branch

---

## Step 1: Create a New Branch Named `test`

```bash
git branch test
```

---

## Step 2: List All Local Branches

```bash
git branch
```

---

## Step 3: Switch to the `test` Branch

```bash
git checkout test
```

---

## Step 4: Verify Branch, Add a New File & Commit

```bash
git branch                            # Confirms you're on 'test' branch
touch mycode                          # Creates a new file named 'mycode'
git add mycode                        # Stages the new file
ls                                    # Lists the file to confirm it's created
git commit -m "adding to test branch" # Commits the file
```

---

## Step 5: Push the `test` Branch to Remote

```bash
git push origin test
```

---

## Step 6: Merge `test` Into `main` and Delete It

```bash
git checkout main              # Switch back to main branch
git merge test                 # Merge 'test' branch changes into 'main'
git branch -d test             # Delete local 'test' branch
git push origin --delete test  # Delete remote 'test' branch
git push origin main           # Push the updated main branch
```

---

> 💡 **Tip:** Always switch back to `main` before merging, and delete the branch both **locally** and **remotely** to keep your repository clean.
