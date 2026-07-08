# Solution

DVC (Data Version Control) versions large files — datasets and models — *alongside* Git, without bloating the Git repository. `dvc init` layers DVC onto an existing Git repo: it creates a `.dvc/` directory (DVC's config and bookkeeping) and a `.dvcignore` file, much like `git init` does for Git. Because these are small text files, they are committed to Git so every collaborator shares the same DVC setup; the actual data is later pushed to remote storage, leaving only tiny pointer files in Git.

> As an MLOps engineer, you lay the versioning foundation so datasets and models can be tracked alongside code — you are not building or evaluating a model here; the project is a stand-in.

#### Follow the steps below

##### 1. Move into the project directory.
DVC must be initialised from inside the working tree of the Git repository it is meant to layer on top of.
```
cd /root/code/fraud-detection
```

##### 2. Initialise DVC.
The `dvc init` command creates a hidden `.dvc/` directory containing DVC's configuration and bookkeeping, plus a `.dvcignore` file at the project root. The pattern mirrors `git init` for Git.
```
dvc init
```

##### 3. Commit the initialisation files to Git.
The files DVC has just generated are small text files that hold the team's DVC configuration. Committing them to Git ensures every collaborator picks up the same DVC setup the moment they clone the repository.
```
git add .dvc .dvcignore
git commit -m "Initialize DVC"
```

##### 4. Verify.
Confirm the DVC scaffolding is present, the commit was recorded, and the editor surfaces the new workspace.
```
ls -la .dvc/
git log --oneline | head -3
dvc version
```
You should also see a **DVC TRACKED** section appear at the bottom of the EXPLORER panel and a `DVC` indicator in the editor's bottom status bar — both are added by the DVC extension when it detects the newly-initialised `.dvc/` directory.

---

**References:**
- [DVC — Get Started](https://dvc.org/doc/start)
- [DVC — `dvc init`](https://dvc.org/doc/command-reference/init)
