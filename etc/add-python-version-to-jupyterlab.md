## Connect to JupyterLab with Python 3.11

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
