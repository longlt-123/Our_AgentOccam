# WebChoreArena + AgentOccam on Windows
## Run Shopping task `20004` locally

This README documents the setup used to run **one WebChoreArena Shopping task (`task_id = 20004`)** on **Windows 11** using Docker Desktop, AgentOccam, Playwright, and Gemini.

The goal is to verify that the local Shopping environment and evaluation pipeline work correctly before running a larger task set.

---

## 1. Environment

Tested setup:

- Windows 11
- PowerShell
- Docker Desktop with WSL2 backend
- Python 3.10 virtual environment
- WebChoreArena / AgentOccam
- Shopping environment only
- Shopping URL: `http://localhost:7770`
- Test task: `20004`

Example directory layout:

```text
D:\INT3011E\
├── .venv\
├── WebChoreArena\
│   └── AgentOccam\
└── webarena\
    └── webarena-images\
```

---

## 2. Clone the repository

Clone WebChoreArena:

```powershell
cd D:\INT3011E
git clone https://github.com/WebChoreArena/WebChoreArena.git
```

Move into AgentOccam:

```powershell
cd D:\INT3011E\WebChoreArena\AgentOccam
```

---

## 3. Create and activate the Python virtual environment

Create a Python 3.10 environment:

```powershell
cd D:\INT3011E
py -3.10 -m venv .venv
```

Activate it:

```powershell
.\.venv\Scripts\Activate.ps1
```

Verify that the correct Python interpreter is active:

```powershell
python -c "import sys; print(sys.executable)"
```

Expected output should point to:

```text
D:\INT3011E\.venv\Scripts\python.exe
```

Then return to AgentOccam:

```powershell
cd D:\INT3011E\WebChoreArena\AgentOccam
```

---

## 4. Install Python dependencies

Install WebArena & AgentOccam requirements:

```powershell
python -m pip install numpy==1.26.4
cd D:\INT3011E\webarena
python -m pip install -r requirements.txt
cd D:\INT3011E\WebChoreArena\AgentOccam
python -m pip install -r requirements.txt
```

The following packages were also required during this setup:

```powershell
python -m pip install --upgrade transformers
python -m pip install --upgrade openai
python -m pip install playwright
python -m playwright install chromium
```

Verify OpenAI:

```powershell
python -c "import openai; print(openai.__version__); print(openai.__file__)"
```

Verify Playwright:

```powershell
python -m pip show playwright
```

---

## 5. Download and load the Shopping Docker image

The Shopping Docker image used for this setup can be downloaded from Google Drive:

```text
https://drive.google.com/file/d/1gxXalk9O0p9eu1YkIJcmZta1nvvyAJpA/view
```

Download the file and place it somewhere convenient, for example:

```text
D:\INT3011E\webarena\webarena-images\
```

Open PowerShell in that directory:

```powershell
cd D:\INT3011E\webarena\webarena-images
```

Load the Docker image:

```powershell
docker load -i shopping_final_0712.tar
```

Check that the image exists:

```powershell
docker images
```

You should see an image named similar to:

```text
shopping_final_0712
```

Start the container:

```powershell
docker run --name shopping -p 7770:80 -d shopping_final_0712
```

If the container already exists:

```powershell
docker start shopping
```

Check the container:

```powershell
docker ps
```

Expected port mapping:

```text
0.0.0.0:7770->80/tcp
```

---

## 6. Configure Magento Shopping to use `localhost`

The original Shopping image may contain a base URL pointing to another hostname.

For local Windows use, change Magento's base URL to:

```text
http://localhost:7770
```

Run:

```powershell
docker exec shopping `
  /var/www/magento2/bin/magento `
  setup:store-config:set `
  --base-url="http://localhost:7770"
```

Then update the secure base URL directly in Magento's database:

```powershell
docker exec shopping mysql `
  -u magentouser `
  -pMyPassword `
  magentodb `
  -e "UPDATE core_config_data SET value='http://localhost:7770/' WHERE path='web/secure/base_url';"
```

Flush the Magento cache:

```powershell
docker exec shopping `
  /var/www/magento2/bin/magento `
  cache:flush
```

A successful cache flush should print entries similar to:

```text
Flushed cache types:
config
layout
block_html
collections
reflection
db_ddl
compiled_config
eav
customer_notification
config_integration
config_integration_api
full_page
config_webservice
translate
```

Test the website:

```powershell
curl.exe -I http://localhost:7770
```

Expected result:

```text
HTTP/1.1 200 OK
```

The site should now be available in a browser at:

```text
http://localhost:7770
```

You can also check resource usage:

```powershell
docker stats shopping
```

---

## 7. Set environment variables

Open PowerShell with the virtual environment active.

### Gemini API key

```powershell
$env:GEMINI_API_KEY="YOUR_GEMINI_API_KEY"
```

Check without printing the key:

```powershell
if ($env:GEMINI_API_KEY) {
    "GEMINI_API_KEY OK"
} else {
    "GEMINI_API_KEY MISSING"
}
```

Do not commit the real API key to GitHub.

### WebArena URLs

Set the local WebArena URLs:

```powershell
$env:SHOPPING="http://localhost:7770"
$env:SHOPPING_ADMIN="http://localhost:7780/admin"
$env:REDDIT="http://localhost:9999"
$env:GITLAB="http://localhost:8023"
$env:MAP="http://localhost:3000"
$env:WIKIPEDIA="http://localhost:8888/wikipedia_en_all_maxi_2022-05/A/User:The_other_Kiwix_guy/Landing"
$env:HOMEPAGE="http://localhost:4399"
```

For task `20004`, only the Shopping site needs to actually be running.

### Azure import workaround

The current AgentOccam dependency chain may import the Azure provider even when Gemini is used.

Set dummy values:

```powershell
$env:AZURE_OPENAI_API_KEY="dummy"
$env:AZURE_ENDPOINT="https://dummy.openai.azure.com"
```

---

## 8. Generate Shopping task files

Move to:

```powershell
cd D:\INT3011E\WebChoreArena\AgentOccam\config_files
```

Run:

```powershell
python generate_test_data_new_shopping.py
```

This should create files under:

```text
config_files/
└── new_tasks/
    └── shopping/
        ├── 20000.json
        ├── 20001.json
        ├── ...
        └── 20004.json
```

Verify that task `20004` exists:

```powershell
Test-Path .\new_tasks\shopping\20004.json
```

Expected:

```text
True
```

Return to AgentOccam:

```powershell
cd ..
```

---

## 9. Create the Shopping login state

Create the authentication directory:

```powershell
New-Item -ItemType Directory -Force .\.auth
```

Run the login helper:

```powershell
python browser_env/auto_login.py --site_list shopping
```

Verify:

```powershell
Get-ChildItem .\.auth
```

Expected:

```text
shopping_state.json
```

Do not commit this authentication state to a public repository.

Recommended `.gitignore` entry:

```gitignore
.auth/
```

---

## 10. Verify that only task `20004` will run

Check the current test configuration:

```powershell
python -c "import yaml; c=yaml.safe_load(open(r'AgentOccam/configs/AgentOccam_webchorearena_shopping_gemini38_test.yml')); print('model =', c['agent']['actor']['model']); print('task_ids =', c['env']['task_ids'])"
```

Expected:

```text
model = gemini-3.8-flash
task_ids = [20004]
```

Also confirm the task file exists:

```powershell
Test-Path .\config_files\new_tasks\shopping\20004.json
```

Expected:

```text
True
```

If `task_ids` contains only `20004`, the evaluator will execute only that task.

---

## 11. Run task `20004`

Make sure:

- `.venv` is active
- Docker Shopping is running
- `http://localhost:7770` returns HTTP 200
- `GEMINI_API_KEY` is set
- `config_files/new_tasks/shopping/20004.json` exists
- Playwright Chromium is installed
- `.auth/shopping_state.json` exists

Run:

```powershell
python eval_webchorearena.py `
  --config AgentOccam/configs/AgentOccam_webchorearena_shopping_gemini38_test.yml
```

A successful start should include:

```text
Config file: AgentOccam/configs/AgentOccam_webchorearena_shopping_gemini38_test.yml
Task 20004.
```

The agent should then begin producing actions such as:

```text
[Step 1] ...
[Step 2] ...
[Step 3] ...
```

---

## 12. Logs and output

With logging enabled, output is written under the configured trajectory directory, for example:

```text
AgentOccam-Trajectories_gemini-3.8-flash/
└── shopping/
    └── auto/
```

The evaluator may generate:

- trajectory JSON files
- `summary.csv`
- a copy of the experiment config

Large trajectory folders normally should not be committed unless they are intentionally part of the experiment.

Recommended `.gitignore` entries:

```gitignore
.auth/
.env
AgentOccam-Trajectories*/
*.log
*.tar
```

---

## 13. Common errors

### `ImportError: cannot import name 'OpenAI' from 'openai'`

Upgrade the package:

```powershell
python -m pip install --upgrade openai
```

---

### Azure credential error

Example:

```text
openai.OpenAIError: Missing credentials...
```

Set:

```powershell
$env:AZURE_OPENAI_API_KEY="dummy"
$env:AZURE_ENDPOINT="https://dummy.openai.azure.com"
```

---

### Unicode error while generating tasks

Example:

```text
UnicodeDecodeError: 'charmap' codec can't decode byte ...
```

Make sure the task generation code reads and writes JSON using UTF-8.

---

### Missing task file

Example:

```text
FileNotFoundError:
config_files/new_tasks/shopping/20004.json
```

Run:

```powershell
cd config_files
python generate_test_data_new_shopping.py
cd ..
```

Then verify:

```powershell
Test-Path .\config_files\new_tasks\shopping\20004.json
```

---

### Playwright not found

Example:

```text
ModuleNotFoundError: No module named 'playwright'
```

Install it in the active virtual environment:

```powershell
python -m pip install playwright
python -m playwright install chromium
```

---

### Shopping redirects to the wrong hostname

Re-run the localhost configuration:

```powershell
docker exec shopping `
  /var/www/magento2/bin/magento `
  setup:store-config:set `
  --base-url="http://localhost:7770"
```

Then:

```powershell
docker exec shopping mysql `
  -u magentouser `
  -pMyPassword `
  magentodb `
  -e "UPDATE core_config_data SET value='http://localhost:7770/' WHERE path='web/secure/base_url';"
```

Finally:

```powershell
docker exec shopping `
  /var/www/magento2/bin/magento `
  cache:flush
```

Test again:

```powershell
curl.exe -I http://localhost:7770
```

---

## 14. Warnings seen during the run

Warnings such as the following may appear:

```text
BeartypeDecorHintPep585DeprecationWarning
```

```text
google.generativeai package has ended support
```

```text
Python 3.10 support warning from google.api_core
```

These are warnings and do not necessarily stop the evaluation.

---

## 15. Minimal rerun checklist

After the initial installation, the shortest rerun flow is:

```powershell
# Activate venv
D:\INT3011E\.venv\Scripts\Activate.ps1

# Go to AgentOccam
cd D:\INT3011E\WebChoreArena\AgentOccam

# Start Shopping
docker start shopping

# Set URLs
$env:SHOPPING="http://localhost:7770"
$env:SHOPPING_ADMIN="http://localhost:7780/admin"
$env:REDDIT="http://localhost:9999"
$env:GITLAB="http://localhost:8023"
$env:MAP="http://localhost:3000"
$env:WIKIPEDIA="http://localhost:8888/wikipedia_en_all_maxi_2022-05/A/User:The_other_Kiwix_guy/Landing"
$env:HOMEPAGE="http://localhost:4399"

# Set Gemini API key
$env:GEMINI_API_KEY="YOUR_GEMINI_API_KEY"

# Azure import workaround
$env:AZURE_OPENAI_API_KEY="dummy"
$env:AZURE_ENDPOINT="https://dummy.openai.azure.com"

# Verify Shopping
curl.exe -I http://localhost:7770

# Run only task 20004
python eval_webchorearena.py `
  --config AgentOccam/configs/AgentOccam_webchorearena_shopping_gemini38_test.yml
```

---

## 16. Before pushing to GitHub

Check:

```powershell
git status
```

Do not commit:

```text
.auth/
.env
API keys
shopping_final_0712.tar
large trajectory directories
```

Recommended `.gitignore`:

```gitignore
.auth/
.env
AgentOccam-Trajectories*/
*.log
*.tar
```

Check that the Gemini API key is not stored in tracked files:

```powershell
git grep -n "GEMINI_API_KEY"
```

Only references to the environment variable should appear, never the real key.

---

## Expected result

When everything is configured correctly, the evaluator should:

1. connect to the Shopping website at `http://localhost:7770`,
2. load `config_files/new_tasks/shopping/20004.json`,
3. load or refresh Shopping authentication,
4. launch the browser environment,
5. initialize AgentOccam,
6. call the configured Gemini model,
7. produce browser actions,
8. finish task `20004` or stop at the configured maximum step count.

The console should show:

```text
Task 20004.
```

followed by steps such as:

```text
[Step 1] ...
[Step 2] ...
[Step 3] ...
```

This confirms that the single-task Shopping pipeline is working.
