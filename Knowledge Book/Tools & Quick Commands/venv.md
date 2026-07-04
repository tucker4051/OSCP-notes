
## Install venv Support

```bash
sudo apt update
sudo apt install python3-venv
```

## Create a Virtual Environment

```
python3 -m venv venv
```

## Activate the Virtual Environment

```
source venv/bin/activate
```

Once activated, your terminal should show something like:

```
(venv) user@kali:~/project$
```

## Install Packages Inside the venv

```
pip install requests
```

## Save Installed Packages

```
pip freeze > requirements.txt
```

## Install Packages from a Requirements File

```
pip install -r requirements.txt
```

## Deactivate the venv

```
deactivate
```

## Typical Workflow

```
mkdir my-python-projectcd my-python-projectpython3 -m venv venvsource venv/bin/activatepip install <package-name>
```
