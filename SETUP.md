# Setup Instructions for Developer

This guide is for YOU (the developer) to complete the initial setup. The evaluator will receive a ready-to-run package.

## First-Time Setup (Developer Only)

### 1. Create `.env` file
```bash
cp .env.example .env
# Edit .env and add your Gemini API key
```

### 2. Start n8n
```bash
docker-compose up -d
```

### 3. Import Workflow (One-Time)
1. Open http://localhost:5678
2. Complete n8n initial setup (create owner account)
3. Click **Workflows** → **Import from File**
4. Select `n8n/ticket_triage_csv_ingestion.json`
5. Save the workflow

### 4. Verify Setup
- The workflow is now stored in `.n8n/` folder
- This folder will persist across restarts
- The `.n8n/` folder (minus credentials) should be committed to your submission

### 5. Complete the Workflow
Continue with Steps 5-9 from the plan to build the complete workflow:
- Add Gemini API integration
- Add validation layer
- Add fraud keyword override
- Test with all 50 tickets
- Export final workflow to `n8n/workflow.json`

## After Development

### Prepare for Submission
1. Ensure `.n8n/` folder contains your complete workflow
2. Export latest workflow to `n8n/workflow.json` (backup)
3. Verify `.env` is included with your API key
4. Test the evaluator flow:
   ```bash
   docker-compose down
   docker-compose up -d
   # Open http://localhost:5678 - workflow should be there
   ```

### What to Include in Submission
- `.env` file with your Gemini API key
- `.n8n/` folder (workflows and settings, but no credentials)
- All project files (prompts, input CSV, etc.)
- Documentation (README.md, answers.md)

## Evaluator Flow (What They'll Do)

The evaluator receives your complete package and runs:
```bash
docker-compose up -d
# Open http://localhost:5678
# Complete n8n setup (create account) - takes 30 seconds
# Workflow is already imported and visible
# Click "Execute Workflow"
# Check output/tickets_triaged.csv
```

**No manual workflow import needed for evaluator!**

## Security Notes

- `.env` is included in submission (contains API key for evaluator)
- `.n8n/credentials/` is gitignored (if you store additional credentials)
- For public GitHub repos, gitignore the `.env` file
- For assignment submission, `.env` is included for convenience
