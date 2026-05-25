## install JupyterLab
1. go to [https://github.com/jupyterlab/jupyterlab-desktop](https://github.com/jupyterlab/jupyterlab-desktop)
2. select and download "arm64 Installer (Apple silicon)"

## Add new kernel(Python 3.11) to JupyterLab

1. find the path of the python3.11
```
which python3.11
python3.11 is /opt/homebrew/bin/python3.11
```

2. add python3.11 in the JupyterLab
```
/opt/homebrew/bin/python3.11 -m pip install ipykernel
/opt/homebrew/bin/python3.11 -m ipykernel install --user --name python3-11 --display-name "Python 3.11"
```


3. restart JupyterLab
