# Solution

JupyterLab reads its runtime settings from a Python config file (`jupyter_lab_config.py`), where `c.ServerApp.*` options control the bind address (`ip`), the listening `port`, and the notebook `root_dir`. In this task you diagnose a broken teammate config — an unroutable bind address, the wrong port, and a non-existent root directory — correct each setting so the server binds on `0.0.0.0:8888` with a root of `/root/notebooks/`, create the missing directory, and start the server so the lab's Jupyter UI button reaches it.

> As an MLOps engineer, you get a shared notebook server correctly bound and reachable behind the lab proxy — you are not authoring notebooks or analysing data here.

#### Follow the steps below

**About the JupyterLab config:** JupyterLab reads its settings from a Python config file (`jupyter_lab_config.py`) where `c.ServerApp.*` options control how the server runs — `ip` is the address it binds to, `port` is where it listens, and `root_dir` is the root folder it opens. Behind a lab proxy the server must bind on `0.0.0.0` (all interfaces), not a loopback or unroutable address, or the proxy cannot reach it.

##### 1. Activate the virtual environment.
JupyterLab is already installed in the `ml-env` virtual environment. Activate it so the `jupyter` command resolves to the binary inside the sandbox.
```
source /root/code/ml-env/bin/activate
```

##### 2. Inspect the existing configuration.
Open `/root/code/jupyter_lab_config.py` in the VS Code editor, or print it to the terminal:
```
cat /root/code/jupyter_lab_config.py
```
Three of the team overrides are incorrect:
- `c.ServerApp.root_dir = '/root/wrong-path'` — the directory does not exist;
- `c.ServerApp.port = 8000` — the platform button expects port `8888`;
- `c.ServerApp.ip = '1.1.1.1'` — an unroutable bind address the lab proxy cannot reach; JupyterLab must bind on `0.0.0.0` (all interfaces).

##### 3. Correct the three settings.
Edit the file directly, or replace the override block with the values below:
```
sed -i "s|c.ServerApp.root_dir = '/root/wrong-path'|c.ServerApp.root_dir = '/root/notebooks'|" /root/code/jupyter_lab_config.py
sed -i "s|c.ServerApp.port = 8000|c.ServerApp.port = 8888|" /root/code/jupyter_lab_config.py
sed -i "s|c.ServerApp.ip = '1.1.1.1'|c.ServerApp.ip = '0.0.0.0'|" /root/code/jupyter_lab_config.py
```

##### 4. Create the notebooks directory.
The notebook root directory must exist before JupyterLab starts, otherwise the server refuses to launch.
```
mkdir -p /root/notebooks
```

##### 5. Start JupyterLab in the background.
The `--config` flag points Jupyter at the file under `/root/code/`. The `--allow-root` flag is required because the lab runs as root; `--no-browser` prevents Jupyter from attempting to open a browser on the headless host.
```
nohup jupyter lab --config=/root/code/jupyter_lab_config.py --allow-root --no-browser > /tmp/jupyter.log 2>&1 &
```
Allow a few seconds for the server to initialise before opening the **Jupyter UI** button at the top of the lab.

##### 6. Verify.
Confirm that the server is listening on port `8888` and that the config changes are in place.
```
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8888/lab
grep -E "root_dir|ServerApp.ip|ServerApp.port" /root/code/jupyter_lab_config.py
ls /root/notebooks/
```

---

**References:**
- [JupyterLab — Starting JupyterLab](https://jupyterlab.readthedocs.io/en/stable/getting_started/starting.html) — running the `jupyter lab` command (full flag list: `jupyter lab --help`).
- [Jupyter Server — configuration file & command line](https://jupyter-server.readthedocs.io/en/latest/users/configuration.html) — how `jupyter_server_config.py` is loaded and overridden on the CLI.
- [Jupyter Server — full configuration reference](https://jupyter-server.readthedocs.io/en/latest/other/full-config.html) — every `ServerApp` option this task touches (`ip`, `port`, `root_dir`, `allow_root`, `open_browser`).
