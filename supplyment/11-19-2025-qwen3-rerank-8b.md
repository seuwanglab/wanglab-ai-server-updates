## 🛠️ Technical Configuration Guide

### Subject: Integration of Qwen3-Reranker-8B into OpenWebUI

### Prerequisites

Ensure your OpenWebUI container is started with GPU support enabled:

```bash
--gpus all
```

### Step 1: Download the Model

Execute the following inside the container to download the model to the stable embedding directory.

```bash

```bash
# Enter the container
docker exec -it open-webui bash

# Download model
python3 << 'EOF'
from huggingface_hub import snapshot_download
import os

model_id = "Qwen/Qwen3-Reranker-8B"
cache_dir = "/app/backend/data/cache/embedding/models"

print(f"Downloading {model_id}...")

try:
    local_dir = snapshot_download(
        repo_id=model_id,
        cache_dir=cache_dir,
        resume_download=False,
        local_files_only=False,
        max_workers=8
    )
    print(f"\n✓ Download successful!")
    print(f"Location: {local_dir}")
except Exception as e:
    print(f"\n✗ Error: {e}")
EOF
```

### Step 2: Modify Configurationtion

Add the `pad_token_id` to `config.json` to ensure compatibility.

```bash
# Navigate to the snapshot directory
cd /app/backend/data/cache/embedding/models/models--Qwen--Qwen3-Reranker-8B/snapshots
NEW_SNAPSHOT=$(ls)
echo "New Snapshot ID: $NEW_SNAPSHOT"

cd $NEW_SNAPSHOT

# Backup original config
cp config.json config.json.backup

# Patch config.json
python3 << 'EOF'
import json

with open('config.json', 'r') as f:
    config = json.load(f)

# Add the missing pad_token_id
config['pad_token_id'] = 151645

with open('config.json', 'w') as f:
    json.dump(config, f, indent=2)

print("✓ config.json modified successfully: pad_token_id added.")
EOF
```