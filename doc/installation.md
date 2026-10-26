## How to install `delex_lite` on macOS

Follow this guide to install `delex_lite` on a Mac with Apple silicon, the currently supported platform. The installation supports both the command-line and Python examples in the [tutorial](tutorial.md).

The installer creates a Python environment in `.venv` inside the downloaded `delex_lite` folder. Install the prerequisites, Git and Python, separately as described in Step 1.

This guide was tested on a Mac with an M5 Pro chip running macOS 26.4, with Python 3.9.6 and git 2.50.1.

---

## Step 1: Check the prerequisites

`delex_lite` requires a Mac with Apple silicon (M1 or later), macOS 11 or later, and Python 3.9 or later. Git is also needed to download the repository. Check these prerequisites in Terminal before running the installer.

### The chip and the macOS version

Open **Applications → Utilities → Terminal**. Run these commands to check the processor architecture and macOS version:

```bash
uname -m
sw_vers -productVersion
```

Check that the first command reports `arm64` and the second reports macOS 11 or later. For example:

**Example output:**

```text
arm64
26.4
```

| What you see | What it means | What to do |
|---|---|---|
| `x86_64` | The Mac has an Intel processor. | `delex_lite` does not run on this Mac. |
| A version below 11, such as `10.15.7` | macOS is older than version 11. | Update macOS in System Settings > General > Software Update. |

### Git and Python

Run these commands in the same Terminal window to check Git and Python:

```bash
git --version
python3 --version
```

Confirm that Git is available and Python is version 3.9 or later. For example:

**Example output:**

```text
git version 2.50.1 (Apple Git-155)
Python 3.9.6
```

| What you see | What it means | What to do |
|---|---|---|
| A window that asks to install the command line developer tools | The Xcode Command Line Tools, which provide `git` and `python3`, are not installed. | Click **Install** and wait for the installation to finish, which often takes a few minutes. Then run both commands again. |
| `xcrun: error: invalid active developer path` | The Command Line Tools are missing, which happens after some macOS updates. | Run `xcode-select --install`, complete the installation, and run both checks again. |
| `Python 3.8.2`, or another version below 3.9 | The default `python3` is too old. | Download the macOS installer of Python 3.12 from [python.org](https://www.python.org/downloads/macos/) and run it. Then run `python3 --version` again to check which version your shell uses. |

---

## Step 2: Download `delex_lite`

In Terminal, go to the folder where you want to keep `delex_lite`, such as your home folder. Run:

```bash
git clone https://github.com/relixa-ai/delex_lite.git
```

This downloads the repository into a new `delex_lite` folder, approximately 2 MB in size. The folder contains:

**Example output:**

```text
LICENSE
README.md
data
delex_lite-0.1.0-cp39-abi3-macosx_11_0_arm64.whl
demo.ipynb
doc
install.sh
programs
```

| What you see | What it means | What to do |
|---|---|---|
| `fatal: destination path 'delex_lite' already exists and is not an empty directory.` | A folder named `delex_lite` already exists here. | Run the command in another folder, or rename or remove the existing one. |

---

## Step 3: Run the installer

Enter the downloaded folder and run the installer in Terminal:

```bash
cd delex_lite
bash install.sh
```

The installer performs four steps:

- **Checks the prerequisites:** Apple silicon and Python 3.9 or later.
- **Creates the Python environment:** a `.venv` folder inside `delex_lite`, separate from the system Python.
- **Installs the packages:** `delex_lite` from the included wheel, plus NumPy and PyArrow from the Python Package Index ([pypi.org](https://pypi.org)). It also updates `pip` and `wheel` in the environment. Running the installer again reinstalls these packages.
- **Checks the installation:** `delex-lite smoke` runs a small search to confirm that the software starts successfully.

Wait for the installer to finish. The example output below shows the installation stages. Paths and package versions may differ. The recorded installation took 22 seconds with NumPy and PyArrow already cached. Your installation time may vary.

**Example output:**

```text
[1/4] checking your machine
      OK — Apple Silicon Mac (arm64, macOS 26.4)
[2/4] checking Python
      OK — Python 3.9.6 at /usr/bin/python3
[3/4] creating .venv and installing delex_lite (+ numpy, pyarrow)
Processing ./delex_lite-0.1.0-cp39-abi3-macosx_11_0_arm64.whl
...
Successfully installed delex-lite-0.1.0 numpy-2.0.2 pyarrow-21.0.0
      OK — installed into /Users/yourname/delex_lite/.venv
[4/4] smoke test
delex-lite: smoke test
  OK — delex_lite runs on this machine.

──────────────────────────────────────────────────────────────────────
 Installation complete.

 Next, turn on the environment (do this once per terminal window):

     source .venv/bin/activate

 Then run your first blocking job:

     delex-lite programs/title_bm25.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet

 The tutorial is doc/tutorial.md.
──────────────────────────────────────────────────────────────────────
```

Confirm that the output includes `OK — delex_lite runs on this machine.` and `Installation complete.`. Then activate the environment and run the first tutorial example in the same Terminal window:

```bash
source .venv/bin/activate
delex-lite programs/title_bm25.json data/dblp_acm/table_a.parquet data/dblp_acm/table_b.parquet
```

---

## Checking the installation

To check an existing installation, run the following commands in Terminal from the `delex_lite` folder:

```bash
source .venv/bin/activate
delex-lite smoke
```

**Example output:**

```text
delex-lite: smoke test
  OK — delex_lite runs on this machine.
```

---

## Troubleshooting

| What you see | What it means | What to do |
|---|---|---|
| `ERROR: delex_lite needs an Apple Silicon Mac (macOS, arm64).` | The Mac has an Intel processor. | `delex_lite` does not run on this Mac. |
| `ERROR: python3 was not found on your machine.` | No `python3` is installed. | Install the Command Line Tools or Python as in Step 1. |
| `ERROR: delex_lite needs Python 3.9 or newer.` | The default `python3` is older than 3.9. | Install Python 3.12 as in Step 1. |
| `no such file or directory: install.sh` | Terminal is not in the `delex_lite` folder. | Execute `cd delex_lite` in the folder where you ran `git clone`, then run the installer. |
| Errors that mention the network, or `Could not find a version` | A package could not be downloaded in step [3/4]. | Check the internet connection and run the installer again. Running it twice is safe. |
| `"dl_engine.dylib" cannot be opened because the developer cannot be verified` | macOS blocked a compiled file of `delex_lite`. This happens when an installed `delex_lite` folder is copied from another Mac. | Run `bash install.sh` again in the `delex_lite` folder, which installs fresh copies of the files. |
| `command not found: delex-lite` | The environment is not active in this Terminal window. | Run `source .venv/bin/activate` in the `delex_lite` folder. The prompt then starts with `(.venv)`. |

If your problem is not listed, contact [our team](mailto:hello@relixa.ai) and include the full output of `bash install.sh`.

---

## How to uninstall

To remove the installed Python environment, run `rm -rf .venv` from the `delex_lite` folder. This also removes any packages you added to that environment. To remove the downloaded software as well, move the `delex_lite` folder to the Trash. Save any input files, results or notebooks you want to keep before removing the folder.
