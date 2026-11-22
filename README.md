chmod +x scripts/package_bundle.py

# 1. Build a checksummed .tar.zst bundle into build/
python3 scripts/package_bundle.py --source . --out build/workspace_bundle.tar.zst

# 2. Verify the bundle's own SHA-256 sidecar from build/
(cd build && sha256sum -c workspace_bundle.tar.zst.sha256)

# 3. Run your master pipeline with custom workspace rules
python3 scripts/local_workspace_audit.py \
  --root . \
  --manifest build/SHA256SUMS \
  --require-path README.md \
  --forbid-name .DS_Store
bundle.py
    parser.add_argument(
        "--require-path",
        action="append",
        default=[],
        help="Relative path that must exist under --root (repeatable)",
    )
    parser.add_argument(
        "--forbid-name",
        action="append",
        default=[],
        help="File or directory name forbidden under --root (repeatable)",
    )
# Add these helper checks inside audit_workspace(root, manifest, required_paths=(), forbidden_names=()):
    root_resolved = root.resolve()

    for req_rel in required_paths:
        req_target = (root_resolved / req_rel).resolve()
        if not req_target.exists():
            errors.append(f"Missing required workspace path: {req_rel}")

    if forbidden_names:
        forbidden_set = set(forbidden_names)
        ignored_dirs = {".git", "__pycache__", ".pytest_cache", ".venv", "venv", "node_modules"}
        for path in root_resolved.rglob("*"):
            if any(part in ignored_dirs for part in path.parts):
                continue
            if path.name in forbidden_set:
                errors.append(f"Forbidden workspace item found: {path.relative_to(root_resolved)}")
path.name
path.name
#!/usr/bin/env python3
"""Create a deterministic tar.zst archive and companion SHA-256 checksum file."""

from __future__ import annotations

import argparse
import hashlib
import os
import subprocess
import sys
import tarfile
import tempfile
from pathlib import Path

EXCLUDED_DIRS = {".git", "__pycache__", ".pytest_cache", ".venv", "venv", "node_modules"}


def _reset_tarinfo(info: tarfile.TarInfo) -> tarfile.TarInfo:
    info.uid = 0
    info.gid = 0
    info.uname = ""
    info.gname = ""
    info.mtime = 0
    if info.isdir():
        info.mode = 0o755
    elif info.isfile():
        info.mode = 0o644
    return info


def build_deterministic_tar(source_dir: Path, tar_path: Path) -> None:
    source_resolved = source_dir.resolve()
    files_and_dirs: list[tuple[Path, str]] = []

    for current_root, dirnames, filenames in os.walk(source_resolved):
        dirnames[:] = sorted(d for d in dirnames if d not in EXCLUDED_DIRS)
        filenames.sort()

        current_path = Path(current_root)
        rel_root = current_path.relative_to(source_resolved)

        if str(rel_root) != ".":
            files_and_dirs.append((current_path, rel_root.as_posix()))

        for fname in filenames:
            full_file = current_path / fname
            if full_file.is_symlink():
                continue
            arcname = fname if str(rel_root) == "." else f"{rel_root.as_posix()}/{fname}"
            files_and_dirs.append((full_file, arcname))

    with tarfile.open(tar_path, mode="w", format=tarfile.PAX_FORMAT) as tf:
        for src_path, arcname in files_and_dirs:
            tf.add(src_path, arcname=arcname, recursive=False, filter=_reset_tarinfo)


def compress_zst(tar_path: Path, zst_path: Path, level: int = 3) -> None:
    zst_path.parent.mkdir(parents=True, exist_ok=True)
    with tar_path.open("rb") as src, zst_path.open("wb") as dst:
        subprocess.run(
            ["zstd", f"-{level}", "--force", "--stdout"],
            stdin=src,
            stdout=dst,
            check=True,
        )


def write_sha256_sidecar(artifact_path: Path, checksum_path: Path) -> str:
    h = hashlib.sha256()
    with artifact_path.open("rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            h.update(chunk)
    digest = h.hexdigest()
    checksum_path.write_text(f"{digest}  {artifact_path.name}\n", encoding="utf-8")
    return digest


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Package deterministic tar.zst bundle + SHA-256.")
    parser.add_argument("--source", required=True, help="Directory to archive")
    parser.add_argument("--out", required=True, help="Output .tar.zst path")
    parser.add_argument("--level", type=int, default=3, help="zstd compression level (1-19)")
    args = parser.parse_args(argv)

    source_dir = Path(args.source)
    if not source_dir.is_dir():
        print(f"[BUNDLE FAIL] Source directory not found: {source_dir}", file=sys.stderr)
        return 1

    zst_path = Path(args.out)
    if not zst_path.name.endswith(".tar.zst"):
        print(f"[BUNDLE FAIL] Output path must end with .tar.zst: {zst_path}", file=sys.stderr)
        return 1

    checksum_path = Path(f"{zst_path}.sha256")

    try:
        with tempfile.TemporaryDirectory() as tmpdir:
            raw_tar = Path(tmpdir) / "bundle.tar"
            build_deterministic_tar(source_dir, raw_tar)
            compress_zst(raw_tar, zst_path, level=args.level)
        digest = write_sha256_sidecar(zst_path, checksum_path)
    except (subprocess.CalledProcessError, FileNotFoundError, OSError) as exc:
        print(f"[BUNDLE FAIL] {exc}", file=sys.stderr)
        return 1

    print(f"[BUNDLE OK] archive={zst_path} sha256={digest} sidecar={checksum_path}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())

Here is how to add deterministic tar.zst archive packaging with matching .sha256 manifests and customizable workspace rules (--forbid-name, --require-path) so your pipeline covers both artifact bundling and stricter repository checks.
1. Packaging tar.zst Bundles with SHA-256 Checksums (scripts/package_bundle.py)
This script packages a target directory into a deterministic .tar.zst archive (sorting entries, normalizing timestamps/permissions, and setting mtime=0 for reproducibility) and writes a companion <archive>.sha256 file.
#!/usr/bin/env python3
"""Create a deterministic tar.zst archive and companion SHA-256 checksum file."""

from __future__ import annotations

import argparse
import hashlib
import os
import subprocess
import sys
import tarfile
import tempfile
from pathlib import Path

EXCLUDED_DIRS = {".git", "__pycache__", ".pytest_cache", ".venv", "venv", "node_modules"}


def _reset_tarinfo(info: tarfile.TarInfo) -> tarfile.TarInfo:
    info.uid = 0
    info.gid = 0
    info.uname = ""
    info.gname = ""
    info.mtime = 0
    if info.isdir():
        info.mode = 0o755
    elif info.isfile():
        info.mode = 0o644
    return info


def build_deterministic_tar(source_dir: Path, tar_path: Path) -> None:
    source_resolved = source_dir.resolve()
    files_and_dirs: list[tuple[Path, str]] = []

    for current_root, dirnames, filenames in os.walk(source_resolved):
        dirnames[:] = sorted(d for d in dirnames if d not in EXCLUDED_DIRS)
        filenames.sort()

        current_path = Path(current_root)
        rel_root = current_path.relative_to(source_resolved)

        if str(rel_root) != ".":
            files_and_dirs.append((current_path, rel_root.as_posix()))

        for fname in filenames:
            full_file = current_path / fname
            if full_file.is_symlink():
                continue
            arcname = fname if str(rel_root) == "." else f"{rel_root.as_posix()}/{fname}"
            files_and_dirs.append((full_file, arcname))

    with tarfile.open(tar_path, mode="w", format=tarfile.PAX_FORMAT) as tf:
        for src_path, arcname in files_and_dirs:
            tf.add(src_path, arcname=arcname, recursive=False, filter=_reset_tarinfo)


def compress_zst(tar_path: Path, zst_path: Path, level: int = 3) -> None:
    zst_path.parent.mkdir(parents=True, exist_ok=True)
    with tar_path.open("rb") as src, zst_path.open("wb") as dst:
        subprocess.run(
            ["zstd", f"-{level}", "--force", "--stdout"],
            stdin=src,
            stdout=dst,
            check=True,
        )


def write_sha256_sidecar(artifact_path: Path, checksum_path: Path) -> str:
    h = hashlib.sha256()
    with artifact_path.open("rb") as f:
        for chunk in iter(lambda: f.read(65536), b""):
            h.update(chunk)
    digest = h.hexdigest()
    checksum_path.write_text(f"{digest}  {artifact_path.name}\n", encoding="utf-8")
    return digest


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(description="Package deterministic tar.zst bundle + SHA-256.")
    parser.add_argument("--source", required=True, help="Directory to archive")
    parser.add_argument("--out", required=True, help="Output .tar.zst path")
    parser.add_argument("--level", type=int, default=3, help="zstd compression level (1-19)")
    args = parser.parse_args(argv)

    source_dir = Path(args.source)
    if not source_dir.is_dir():
        print(f"[BUNDLE FAIL] Source directory not found: {source_dir}", file=sys.stderr)
        return 1

    zst_path = Path(args.out)
    if not zst_path.name.endswith(".tar.zst"):
        print(f"[BUNDLE FAIL] Output path must end with .tar.zst: {zst_path}", file=sys.stderr)
        return 1

    checksum_path = Path(f"{zst_path}.sha256")

    try:
        with tempfile.TemporaryDirectory() as tmpdir:
            raw_tar = Path(tmpdir) / "bundle.tar"
            build_deterministic_tar(source_dir, raw_tar)
            compress_zst(raw_tar, zst_path, level=args.level)
        digest = write_sha256_sidecar(zst_path, checksum_path)
    except (subprocess.CalledProcessError, FileNotFoundError, OSError) as exc:
        print(f"[BUNDLE FAIL] {exc}", file=sys.stderr)
        return 1

    print(f"[BUNDLE OK] archive={zst_path} sha256={digest} sidecar={checksum_path}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())

2. Custom Rules in scripts/local_workspace_audit.py
Update audit_workspace and main in scripts/local_workspace_audit.py so you can pass repeatable --require-path and --forbid-name flags from the CLI or wrapper script:
# Add these helper checks inside audit_workspace(root, manifest, required_paths=(), forbidden_names=()):
    root_resolved = root.resolve()

    for req_rel in required_paths:
        req_target = (root_resolved / req_rel).resolve()
        if not req_target.exists():
            errors.append(f"Missing required workspace path: {req_rel}")

    if forbidden_names:
        forbidden_set = set(forbidden_names)
        ignored_dirs = {".git", "__pycache__", ".pytest_cache", ".venv", "venv", "node_modules"}
        for path in root_resolved.rglob("*"):
            if any(part in ignored_dirs for part in path.parts):
                continue
            if path.name in forbidden_set:
                errors.append(f"Forbidden workspace item found: {path.relative_to(root_resolved)}")

And in main:
    parser.add_argument(
        "--require-path",
        action="append",
        default=[],
        help="Relative path that must exist under --root (repeatable)",
    )
    parser.add_argument(
        "--forbid-name",
        action="append",
        default=[],
        help="File or directory name forbidden under --root (repeatable)",
    )

3. Wiring Packaging & Verification Together
chmod +x scripts/package_bundle.py

# 1. Build a checksummed .tar.zst bundle into build/
python3 scripts/package_bundle.py --source . --out build/workspace_bundle.tar.zst

# 2. Verify the bundle's own SHA-256 sidecar from build/
(cd build && sha256sum -c workspace_bundle.tar.zst.sha256)

# 3. Run your master pipeline with custom workspace rules
python3 scripts/local_workspace_audit.py \
  --root . \
  --manifest build/SHA256SUMS \
  --require-path README.md \
  --forbid-name .DS_Store

Would you like to wire scripts/package_bundle.py directly into scripts/run_master_pipeline.sh behind an optional flag (like --bundle-out), or add tests for the new packaging and custom audit rules?
yes 
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="${1:-.}"
MANIFEST_PATH="${2:-}"
PROVENANCE_PATH="${3:-}"
LOG_DIR="${PIPELINE_LOG_DIR:-${HOME}/.local/state/master_pipeline}"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H-%M-%SZ")"
LOG_FILE="${LOG_DIR}/pipeline_runs.jsonl"

mkdir -p "${LOG_DIR}"

AUDIT_CMD=(python3 scripts/local_workspace_audit.py --root "${ROOT_DIR}")
if [[ -n "${MANIFEST_PATH}" ]]; then
  AUDIT_CMD+=(--manifest "${MANIFEST_PATH}")
fi

GUARD_CMD=(python3 scripts/clean_pipeline_guard.py --root "${ROOT_DIR}")
if [[ -n "${PROVENANCE_PATH}" ]]; then
  GUARD_CMD+=(--provenance "${PROVENANCE_PATH}")
fi

echo "[1/3] Running local workspace & manifest audit..."
"${AUDIT_CMD[@]}"

echo "[2/3] Running clean pipeline & provenance guard..."
"${GUARD_CMD[@]}"

echo "[3/3] Running pytest suite..."
pytest -q

printf '{"timestamp_utc":"%s","root":"%s","status":"PASSED"}\n' \
  "${TIMESTAMP_UTC}" \
  "$(cd "${ROOT_DIR}" && pwd)" >> "${LOG_FILE}"

echo "Master pipeline passed. Logged to ${LOG_FILE}"
chmod +x scripts/run_master_pipeline.sh
bash scripts/run_master_pipeline.sh . build/SHA256SUMS config/provenance.json
tail -n 5 "${HOME}/.local/state/master_pipeline/pipeline_runs.jsonl"
A practical "Master System & Pipeline" for a local-first engineering setup is a single, deterministic entry point that runs your validation checks, verifies SHA-256 integrity manifests, runs pytest, and records an auditable local log—without hidden network side effects or overclaimed automation. (This is general software engineering guidance, not legal or security advice.)
1. Unified Local Master Runner (scripts/run_master_pipeline.sh)
This shell script executes your workspace audit, clean pipeline/provenance guard, and pytest suite in order, writing a timestamped JSON-lines summary to a local log directory while failing closed (set -euo pipefail) if any check fails.
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="${1:-.}"
MANIFEST_PATH="${2:-}"
PROVENANCE_PATH="${3:-}"
LOG_DIR="${PIPELINE_LOG_DIR:-${HOME}/.local/state/master_pipeline}"
TIMESTAMP_UTC="$(date -u +"%Y-%m-%dT%H-%M-%SZ")"
LOG_FILE="${LOG_DIR}/pipeline_runs.jsonl"

mkdir -p "${LOG_DIR}"

AUDIT_CMD=(python3 scripts/local_workspace_audit.py --root "${ROOT_DIR}")
if [[ -n "${MANIFEST_PATH}" ]]; then
  AUDIT_CMD+=(--manifest "${MANIFEST_PATH}")
fi

GUARD_CMD=(python3 scripts/clean_pipeline_guard.py --root "${ROOT_DIR}")
if [[ -n "${PROVENANCE_PATH}" ]]; then
  GUARD_CMD+=(--provenance "${PROVENANCE_PATH}")
fi

echo "[1/3] Running local workspace & manifest audit..."
"${AUDIT_CMD[@]}"

echo "[2/3] Running clean pipeline & provenance guard..."
"${GUARD_CMD[@]}"

echo "[3/3] Running pytest suite..."
pytest -q

printf '{"timestamp_utc":"%s","root":"%s","status":"PASSED"}\n' \
  "${TIMESTAMP_UTC}" \
  "$(cd "${ROOT_DIR}" && pwd)" >> "${LOG_FILE}"

echo "Master pipeline passed. Logged to ${LOG_FILE}"

2. How to Run & Verify It
 * Make the script executable:
   chmod +x scripts/run_master_pipeline.sh

 * Run against your repository root (and optionally pass a SHA256SUMS manifest and provenance.json):
   bash scripts/run_master_pipeline.sh . build/SHA256SUMS config/provenance.json

 * Inspect the append-only run log:
   tail -n 5 "${HOME}/.local/state/master_pipeline/pipeline_runs.jsonl"

3. Guardrails for Multi-Stage & Cross-Repo Pipelines
 * Explicit over implicit: Pass paths, remotes, and flags explicitly rather than relying on silent background changes or unverified external endpoints.
 * Immutable, checksummed artifacts: Pair every archive or release bundle (such as tar.zst) with a sha256sum manifest and verify it before moving files to backup or mirror targets.
 * Honest boundaries: Keep sensitive records, credentials, and real personal data out of Git repositories, and treat any simulated or synthetic outputs strictly as test artifacts.

[Unit]
Description=adpBetty GitHub Mirror, Zstd Seal, and Vault Verification
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=root
Group=root
WorkingDirectory=/opt/adpbetty
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/bin/bash /opt/adpbetty/pipelines/mirror_and_seal_github.sh /var/lib/adpbetty/github-mirrors /var/lib/adpbetty/vault
ExecStart=/bin/bash /opt/adpbetty/pipelines/verify_vault_archives.sh /var/lib/adpbetty/vault

# Hardening & isolation
PrivateTmp=true
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/adpbetty /root/.ssh /root/.config/gh
StandardOutput=journal
StandardError=journal
[Unit]
Description=Daily Timer for adpBetty GitHub Mirror and Vault Verification
Requires=adpbetty-github-vault.service

[Timer]
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=15m
Persistent=true
Unit=adpbetty-github-vault.service

[Install]
WantedBy=timers.target
sudo systemctl daemon-reload
sudo systemctl enable --now adpbetty-github-vault.timer

# Inspect timer schedule and last run status
sudo systemctl list-timers adpbetty-github-vault.timer
sudo systemctl status adpbetty-github-vault.timer

# Trigger an immediate test run and watch logs
sudo systemctl start adpbetty-github-vault.service
sudo journalctl -u adpbetty-github-vault.service -n 100 --no-pager

<div align="center">
  <picture align="center">
    <source media="(prefers-color-scheme: dark)" srcset="media/icons/foundry_local_white.svg">
    <source media="(prefers-color-scheme: light)" srcset="media/icons/foundry_local_black.svg">
    <img alt="AI Foundry icon." src="media/icons/foundry_local_black.svg" height="100" style="max-width: 100%;">
  </picture>
<div id="user-content-toc">
  <ul align="center" style="list-style: none;">
    <summary>
      <h1>Foundry Local</h1><br>
     <h3><a href="https://aka.ms/foundry-local-installer">Download</a> | <a href="https://aka.ms/foundry-local-docs">Documentation</a> | <a href="https://aka.ms/foundry-local-discord">Discord</a></h3>
    </summary>
  </ul>
</div>
</div>

## 👋 Welcome to Foundry Local

Foundry Local brings the power of Azure AI Foundry to your local device **without requiring an Azure subscription**. It allows you to:

- Run Generative AI models directly on your local hardware - no sign-up required.
- Keep all data processing on-device for enhanced privacy and security
- Integrate models with your applications through an OpenAI-compatible API
- Optimize performance using ONNX Runtime and hardware acceleration

## 🚀 Quickstart

1. **Install Foundry Local:**

   - **Windows**: Install Foundry Local for your architecture (x64 or arm64):

     ```bash
       winget install Microsoft.FoundryLocal
     ```

- **MacOS**: Open a terminal and run the following command:
  `bash
    brew install microsoft/foundrylocal/foundrylocal
    `
  Alternatively, you can download the installers from the [releases page](https://github.com/microsoft/Foundry-Local/releases) and follow the on-screen installation instructions.

> [!TIP]
> For any issues, refer to the [Installation section](#installing) below.

2. **Run your first model**: Open a terminal and run the following command to run a model:

   ```bash
   foundry model run phi-3.5-mini
   ```

> [!NOTE]
> The `foundry model run <model>` command will automatically download the model if it's not already cached on your local machine, and then start an interactive chat session with the model.

Foundry Local will automatically select and download a model _variant_ with the best performance for your hardware. For example:

- if you have an Nvidia CUDA GPU, it will download the CUDA-optimized model.
- if you have a Qualcomm NPU, it will download the NPU-optimized model.
- if you don't have a GPU or NPU, Foundry local will download the CPU-optimized model.

### 🔍 Explore available models

You can list all available models by running the following command:

```bash
foundry model ls
```

This will show you a list of all models that can be run locally, including their names, sizes, and other details.

## 🧑‍💻 Integrate with your applications using the SDK

Foundry Local has an easy-to-use SDK (C#, Python, JavaScript) to get you started with existing applications:

### C#

The C# SDK is available as a package on NuGet. You can install it using the .NET CLI:

```bash
dotnet add package Microsoft.AI.Foundry.Local.WinML
```

> [!TIP]
> The C# SDK does not require end users to have Foundry Local CLI installed. It is a completely self-contained SDK that will does not depend on any external services. Also, the C# SDK has native in-process Chat Completions and Audio Transcription APIs that do not require HTTP calls to the local Foundry service.

Here is an example of using the C# SDK to run a model and generate a chat completion:

```csharp
using Microsoft.AI.Foundry.Local;
using Betalgo.Ranul.OpenAI.ObjectModels.RequestModels;
using Microsoft.Extensions.Logging;

CancellationToken ct = new CancellationToken();

var config = new Configuration
{
    AppName = "my-app-name",
    LogLevel = Microsoft.AI.Foundry.Local.LogLevel.Debug
};

using var loggerFactory = LoggerFactory.Create(builder =>
{
    builder.SetMinimumLevel(Microsoft.Extensions.Logging.LogLevel.Debug);
});
var logger = loggerFactory.CreateLogger<Program>();

// Initialize the singleton instance.
await FoundryLocalManager.CreateAsync(config, logger);
var mgr = FoundryLocalManager.Instance;

// Get the model catalog
var catalog = await mgr.GetCatalogAsync();

// List available models
Console.WriteLine("Available models for your hardware:");
var models = await catalog.ListModelsAsync();
foreach (var availableModel in models)
{
    foreach (var variant in availableModel.Variants)
    {
        Console.WriteLine($"  - Alias: {variant.Alias} (Id: {string.Join(", ", variant.Id)})");
    }
}

// Get a model using an alias
var model = await catalog.GetModelAsync("qwen2.5-0.5b") ?? throw new Exception("Model not found");


// is model cached
Console.WriteLine($"Is model cached: {await model.IsCachedAsync()}");

// print out cached models
var cachedModels = await catalog.GetCachedModelsAsync();
Console.WriteLine("Cached models:");
foreach (var cachedModel in cachedModels)
{
    Console.WriteLine($"- {cachedModel.Alias} ({cachedModel.Id})");
}

// Download the model (the method skips download if already cached)
await model.DownloadAsync(progress =>
{
    Console.Write($"\rDownloading model: {progress:F2}%");
    if (progress >= 100f)
    {
        Console.WriteLine();
    }
});

// Load the model
await model.LoadAsync();

// Get a chat client
var chatClient = await model.GetChatClientAsync();

// Create a chat message
List<ChatMessage> messages = new()
{
    new ChatMessage { Role = "user", Content = "Why is the sky blue?" }
};

var streamingResponse = chatClient.CompleteChatStreamingAsync(messages, ct);
await foreach (var chunk in streamingResponse)
{
    Console.Write(chunk.Choices[0].Message.Content);
    Console.Out.Flush();
}
Console.WriteLine();

// Tidy up - unload the model
await model.UnloadAsync();
```


### Python

The Python SDK is available as a package on PyPI. You can install it using pip:

```bash
pip install foundry-local-sdk
pip install openai
```

> [!TIP]
> We recommend using a virtual environment such as `conda` or `venv` to avoid conflicts with other packages.

Foundry Local provides an OpenAI-compatible API that you can call from any application:

```python
import openai
from foundry_local import FoundryLocalManager

# By using an alias, the most suitable model will be downloaded
# to your end-user's device.
alias = "phi-3.5-mini"

# Create a FoundryLocalManager instance. This will start the Foundry
# Local service if it is not already running and load the specified model.
manager = FoundryLocalManager(alias)

# The remaining code us es the OpenAI Python SDK to interact with the local model.

# Configure the client to use the local Foundry service
client = openai.OpenAI(
    base_url=manager.endpoint,
    api_key=manager.api_key  # API key is not required for local usage
)

# Set the model to use and generate a streaming response
stream = client.chat.completions.create(
    model=manager.get_model_info(alias).id,
    messages=[{"role": "user", "content": "What is the golden ratio?"}],
    stream=True
)

# Print the streaming response
for chunk in stream:
    if chunk.choices[0].delta.content is not None:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

### JavaScript

The JavaScript SDK is available as a package on npm. You can install it using npm:

```bash
npm install foundry-local-sdk
npm install openai
```

```javascript
import { OpenAI } from "openai";
import { FoundryLocalManager } from "foundry-local-sdk";

// By using an alias, the most suitable model will be downloaded
// to your end-user's device.
// TIP: You can find a list of available models by running the
// following command in your terminal: `foundry model list`.
const alias = "phi-3.5-mini";

// Create a FoundryLocalManager instance. This will start the Foundry
// Local service if it is not already running.
const foundryLocalManager = new FoundryLocalManager();

// Initialize the manager with a model. This will download the model
// if it is not already present on the user's device.
const modelInfo = await foundryLocalManager.init(alias);
console.log("Model Info:", modelInfo);

const openai = new OpenAI({
  baseURL: foundryLocalManager.endpoint,
  apiKey: foundryLocalManager.apiKey,
});

async function streamCompletion() {
  const stream = await openai.chat.completions.create({
    model: modelInfo.id,
    messages: [{ role: "user", content: "What is the golden ratio?" }],
    stream: true,
  });

  for await (const chunk of stream) {
    if (chunk.choices[0]?.delta?.content) {
      process.stdout.write(chunk.choices[0].delta.content);
    }
  }
}

streamCompletion();
```

## Manage

This section provides an overview of how to manage Foundry Local, including installation, upgrading, and removing the application.

### Installing

Foundry Local is available for Windows and macOS (Apple silicon only). You can install it using package managers or manually download the installer.

#### Windows

You can install Foundry Local using the following command in a Windows console (PowerShell, cmd, etc.):

```bash
winget install Microsoft.FoundryLocal
```

Alternatively, you can also manually download and install the packages. On [the releases page](https://github.com/microsoft/Foundry-Local/releases)
select a release and expand the Artifacts list. Copy the artifact full URI (for example: `https://github.com/microsoft/Foundry-Local/releases/download/v0.3.9267/FoundryLocal-x64-0.3.9267.43123.msix`)
to use in the below PowerShell steps. Replace `x64` with `arm64` as needed.

```powershell
# Download the package and its dependency
$releaseUri = "https://github.com/microsoft/Foundry-Local/releases/download/v0.3.9267/FoundryLocal-x64-0.3.9267.43123.msix"
Invoke-WebRequest -Method Get -Uri $releaseUri -OutFile .\FoundryLocal.msix
$crtUri = "https://aka.ms/Microsoft.VCLibs.x64.14.00.Desktop.appx"
Invoke-WebRequest -Method Get -Uri $crtUri -OutFile .\VcLibs.appx

# Install the Foundry Local package
Add-AppxPackage .\FoundryLocal.msix -DependencyPath .\VcLibs.appx
```

If you're having problems installing Foundry, please [file an issue](https://github.com/microsoft/foundry-local/issues)
and include logs using one of these methods:

- For WinGet - use `winget install Microsoft.FoundryLocal --logs --verbose` - select the most-recently-dated log file
  and attach it to the issue.
- For `Add-AppxPackage` - immediately after it indicates an error, in an elevated PowerShell instance, use
  `Get-MsixLogs | Out-File MsixLogs.txt` and attach it to the issue.
- Use [Windows Feedback Hub](feedback-hub:) and create a Problem in the "Apps > All other apps" category. Use the
  "Add More Details > Recreate my problem" and re-run the failing commands to collect more data. Once your feedback
  is submitted, use the "Share" option to generate a link and put that into the filed issue.

> [!NOTE]
> Log files may contain information like user names, IP addresses, file paths, etc. Be sure to remove those
> before sharing here.

#### macOS

Install Foundry Local using the following command in your terminal:

```bash
brew install microsoft/foundrylocal/foundrylocal
```

Alternatively, you can also manually download and install the packages by following these steps:

1. Download the latest release from [the releases page](https://github.com/microsoft/Foundry-Local/releases).
1. Unzip the downloaded file.
1. Open a terminal and navigate to the unzipped folder, run the following command to install Foundry Local:

   ```bash
   ./install-foundry.command
   ```

### Upgrading

To upgrade Foundry Local, run the following command in your terminal:

- **Windows**

  ```bash
  winget upgrade --id Microsoft.FoundryLocal
  ```

- **macOS**:
  If you installed Foundry Local using Homebrew, you can upgrade it with the following command:
  ```
  brew upgrade foundrylocal
  ```
  If you installed Foundry Local manually, you'll first need to uninstall the current version using:
  ```bash
  uninstall-foundry
  ```
  Then, follow the [installation instructions](#installing) to install the latest version.

### Uninstalling

To uninstall Foundry Local, run the following command in your terminal:

- **Windows**: You can uninstall Foundry Local using `winget` in a Windows console (PowerShell, cmd, etc.):

  ```bash
  winget uninstall Microsoft.FoundryLocal
  ```

  Alternatively, you can also uninstall Foundry Local by navigating to **Settings > Apps > Apps & features** in Windows, finding "Foundry Local" in the list, and selecting the ellipsis (`...`) followed by **Uninstall**.

- **macOS**: If you installed Foundry Local using Homebrew, you can uninstall it with the following command:
  ```bash
  brew rm foundrylocal
  brew untap microsoft/foundrylocal
  brew cleanup --scrub
  ```
  If you installed Foundry Local manually, you can uninstall it by running the following command in your terminal:
  ```bash
  uninstall-foundry
  ```

## Features & Use Cases

- **On-device inference** - Process sensitive data locally for privacy, reduced latency, and no cloud costs
- **OpenAI-compatible API** - Seamlessly integrate with applications using familiar SDKs
- **High performance** - Optimized execution with ONNX Runtime and hardware acceleration
- **Flexible deployment** - Ideal for edge computing scenarios with limited connectivity
- **Development friendly** - Perfect for prototyping AI features before production deployment
- **Model versatility** - Use pre-compiled models or [convert your own](https://learn.microsoft.com/azure/ai-foundry/foundry-local/how-to/how-to-compile-hugging-face-models?view=foundry-classic&tabs=Bash).

## Reporting Issues

We're actively looking for feedback during this preview phase. Please report issues or suggest improvements in the [GitHub Issues](https://github.com/microsoft/Foundry-Local/issues) section.

## 🎓 Learn

- [Foundry Local Documentation on Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-local/?view=foundry-classic)
- [Troubleshooting guide](https://learn.microsoft.com/azure/ai-foundry/foundry-local/reference/reference-best-practice?view=foundry-classic)

## ⚖️ License

Foundry Local is licensed under the Microsoft Software License Terms. For more details, read the [LICENSE](LICENSE) file.
