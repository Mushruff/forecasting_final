# Commodity Price Forecasting — Setup & Run Guide

This README covers two setup paths:

1. **Remote VM setup** — cloning the repo fresh and running the full batch pipeline on a server.
2. **Local (VS Code) setup** — running the pipeline from an existing local checkout.

Both end with the same **scoring** step.



---

## 1. Remote VM Setup

### 1.1 Go to home directory
```bash
cd ~
```

### 1.2 Delete the existing repository
⚠️ This permanently removes the current local copy — make sure nothing uncommitted is in it.
```bash
rm -rf forecasting_final
```

### 1.3 Verify it's gone
```bash
ls
```
You should no longer see `Commodity-Price-Forecasting-Kushal` in the listing.

### 1.4 Clone the latest code
```bash
git clone git@github.com:KushalThakkarBeroe/forecasting_final.git
```

### 1.5 Go to the project folder
```bash
cd ~/forecasting_final
```

### 1.6 Create a new virtual environment
```bash
python3.13 -m venv .venv
```

### 1.7 Activate it
```bash
source .venv/bin/activate
```
Your prompt should now be prefixed with `(.venv)`.

### 1.8 Upgrade pip
```bash
python -m pip install --upgrade pip
```

### 1.9 Install requirements
```bash
pip install -r requirements.txt
```

### 1.10 Verify installation
```bash
pip list
```

### 1.11 Run the pipeline (background batch job)
This launches the batch run in the background via `nohup`, logging output and exit status to timestamped files.

```bash
LOG="run_$(date +%Y%m%d_%H%M%S).log"
STATUS="${LOG%.log}.status"

nohup stdbuf -oL -eL bash -c '
python3 -c "
import sys
sys.path.insert(0, \"src\")
sys.path.insert(0, \"src/pipeline\")
sys.path.insert(0, \"src/data\")
sys.path.insert(0, \"src/features\")
sys.path.insert(0, \"src/models\")
sys.path.insert(0, \"src/scoring\")
from config_loader import load_commodities
from run_batch import run_batch
commodities = [c for c in load_commodities() if c.id in (
    \"aluminum\",
    \"atlantic_cod\",
    \"average_wages\",
    \"beef\",
    \"black_pepper\",
    \"carbon_steel\",
    \"cheese\",
    \"corrugated_boards\",
    \"cpi\",
    \"crude_oil\",
    \"electricity\",
    \"ferro_tungsten\"
)]
print(\"Starting batch run...\")
print(\"Commodities:\", [c.id for c in commodities])
result = run_batch(
    commodities=commodities,
    parallel=True,
    max_workers=4
)
print(\"results:\", len(result.results), \"errors:\", result.errors)
print(\"consolidated summary:\", result.consolidated_summary_path)
"
EXIT_CODE=$?
echo "$(date "+%Y-%m-%d %H:%M:%S") EXIT_CODE=$EXIT_CODE" >> "'"$STATUS"'"
exit $EXIT_CODE
' > "$LOG" 2>&1 &

PID=$!
echo "======================================"
echo "Batch job started"
echo "PID:    $PID"
echo "LOG:    $LOG"
echo "STATUS: $STATUS"
echo "======================================"
```

Check progress with `tail -f <LOG file>` and check completion status in the `.status` file.

---

## 2. Local Setup (VS Code)
```bash
git clone https://github.com/KushalThakkarBeroe/forecasting_final.git



cd forecasting_final

### 2.1 One-time environment setup
```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2.2 Run a batch job (foreground, subset of commodities)
```bash
python3 -c "
import sys
sys.path.insert(0, 'src')
sys.path.insert(0, 'src/pipeline')
sys.path.insert(0, 'src/data')
sys.path.insert(0, 'src/features')
sys.path.insert(0, 'src/models')
sys.path.insert(0, 'src/scoring')

from config_loader import load_commodities
from run_batch import run_batch

commodities = [c for c in load_commodities() if c.id in ('pulp','skim_milk_powder', 'gasoline', 'electricity')]
result = run_batch(commodities=commodities)

print('results:', len(result.results), 'errors:', result.errors)
print('consolidated summary:', result.consolidated_summary_path)
"
```

---

## 3. Scoring

Run this after a batch job completes (either environment) to generate short-horizon rank-1 predictions for a set of commodities:

```bash
for c in copper wheat aluminum; do
  python src/scripts/predict_from_saved_model.py --commodity "$c" --horizon short --rank 1 --months 3
done
```

---

## Prerequisites

- Python **3.13** (VM) or **Python 3** (local) — align these before relying on both paths interchangeably.
- `requirements.txt` present at the project root.
- SSH access configured for GitHub (`git@github.com:...` clone URLs require an SSH key on the machine).
- Saved model artifacts already present for the `scoring` step — it loads from a **saved model**, it does not train one.
