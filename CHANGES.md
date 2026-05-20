# Setup Architecture Changes

## Summary

Improved the project setup so the evaluator does NOT need to manually import the workflow. The workflow is now pre-configured in a persistent `.n8n/` folder.

---

## What Changed

### Files Modified

1. **docker-compose.yml**
   - Added persistent `.n8n/` volume mount
   - Workflows now persist across container restarts
   - Before: `./:/home/node/.n8n-files` (files only)
   - After: Added `./.n8n:/home/node/.n8n` (persistent data) + files mount

2. **.gitignore**
   - Updated to allow `.n8n/` workflows while excluding sensitive credentials
   - Excludes: `.n8n/credentials/`, `.n8n/.secret`, `.n8n/.n8n.lock`
   - Includes: Workflow database and settings
   - Added `.env` configuration notes (included for submission, gitignored for public repos)
   - Added `output/.gitkeep` exception

3. **docs/MEMORY_BANK.md**
   - Added note about persistent .n8n/ architecture
   - Updated key decisions to highlight "no manual import"

4. **Plan file (au10tix_ticket_triage_plan_faa963c6.plan.md)**
   - Updated README template with new run instructions
   - Updated Architecture section to mention persistent storage
   - Updated Security section to explain .n8n/ handling
   - Updated project structure diagram to include .n8n/ folder

### Files Added

5. **SETUP.md** (NEW)
   - Developer-facing setup guide
   - Explains one-time import process for developer
   - Documents what evaluator will receive
   - Security notes about .env and credentials

6. **output/.gitkeep** (NEW)
   - Ensures output directory exists in git
   - Prevents empty directory from being ignored

7. **CHANGES.md** (THIS FILE)
   - Documents architectural changes
   - Provides summary for evaluator

### Files Removed

None.

---

## How It Works Now

### Developer Workflow (One-Time Setup)

1. Create `.env` with Gemini API key
2. Run `docker-compose up -d`
3. Open http://localhost:5678
4. Complete n8n initial setup (create owner account)
5. **Import workflow once** from `n8n/workflow.json`
6. Continue building workflow (Steps 5-9)
7. **The .n8n/ folder now persists the workflow**
8. Commit .n8n/ folder to submission (credentials excluded)

### Evaluator Workflow (Zero Manual Import)

1. Receive submission package (includes `.n8n/` folder with workflow)
2. Run `docker-compose up -d`
3. Open http://localhost:5678
4. Complete n8n initial setup (create owner account) - 30 seconds
5. **Workflow is already there** - just click "Execute Workflow"
6. Check `output/tickets_triaged.csv` for results

**No workflow import step needed!**

---

## Technical Details

### Volume Mounts

```yaml
volumes:
  - ./.n8n:/home/node/.n8n              # Persistent n8n data
  - ./:/home/node/.n8n-files            # Project files (CSV, prompts)
```

### What's in .n8n/

- `database.sqlite` - Contains workflows, executions, settings
- `workflows/` - (if using file-based workflows)
- `credentials/` - **GITIGNORED** (sensitive, evaluator creates own)
- `.secret` - **GITIGNORED** (encryption key)
- `.n8n.lock` - **GITIGNORED** (lock file)

### Security Considerations

#### For Submission (Private to Evaluator)
- ✅ Include `.env` with your API key
- ✅ Include `.n8n/` folder with workflows
- ✅ Include `.n8n/database.sqlite`
- ❌ Exclude `.n8n/credentials/` (gitignored)
- ❌ Exclude `.n8n/.secret` (gitignored)

#### For Public GitHub Repos
- ❌ Gitignore `.env` (uncomment in .gitignore)
- ✅ Include `.n8n/` workflows
- ✅ Include `.env.example` with instructions
- ❌ Exclude credentials (already gitignored)

---

## Benefits

1. **Zero Manual Steps for Evaluator**
   - No workflow import
   - No file path configuration
   - Just docker-compose up and go

2. **Professional Setup**
   - Mimics production persistence patterns
   - Clean separation of data vs credentials
   - Easy to backup and version

3. **Developer-Friendly**
   - Workflows persist during development
   - Can restart Docker without losing work
   - Clear documentation in SETUP.md

4. **Simple and Clean**
   - No complex startup scripts
   - No workflow auto-import logic
   - Uses Docker's native volume mounting

---

## Migration Notes

If you already have a running n8n instance:

1. Stop your current container: `docker-compose down`
2. Your workflows are **lost** if you didn't have .n8n/ mounted
3. Re-import workflow after applying these changes
4. Workflow will now persist in `.n8n/` folder
5. Commit `.n8n/` to your submission

---

## Questions?

See `SETUP.md` for detailed developer instructions.
