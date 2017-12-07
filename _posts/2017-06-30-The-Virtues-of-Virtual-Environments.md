---
category: Setup-Notes
---
# Setting up a Virtual Environment with Anaconda
Previously, I had a couple of python versions installed on my machine, and it was a hassle to keep track of which packages were installed on which version of python. After hearing a lot about 'virtual environments' I decided it was time to give it a try.

## Prerequisite

### Anaconda
In one of my previous posts, I walked through the installation of [Anaconda](https://black-tea.github.io/data%20analysis/2017/04/12/Windows-Data-Analysis-Setup.html). You don't need Anaconda for virtual environments, but it makes it easy to set it up, so I would recommend going that route.

After you install Anaconda, make sure to update everything. Skipping this step ended up costing me an hour trying to figure out why I could not activate my virtual environment after I had already set it up. In the command prompt (Windows), type:
```
conda update --all
```

## Setting Up the Virtual Environment
From here, it really only takes a few lines of code to get setup. Begin with:
```
conda create -n [env_name] python=[python_version]
```
You don't need to specify the python version, but the assumption is that the reason you are setting up a virtual environment is to work with multiple versions of python. Once you run the previous command to create the virtual environment, Windows will append the path to the virtualenv activate file in your %PATH% environmental variable. Now you may activate it by simply typing:
```
activate [env_name]
```
Now you can install all the packages from pip or packages from anaconda. They will all be installed on this version of python.
```
conda install [package_name]
pip install [package_name]
```
When you want to exit the virtual environment, you can at any time simply type:
```
deactivate
```
## Using a virtualenv in an IPython notebook
Now after installing the different virtual environments (and different python versions), you will notice that when you run the Jupyter Notebook, you won't have the kernel for the virtualenv pre-loaded. Getting it to recognize the virtual environments just takes a few additional [steps](https://help.pythonanywhere.com/pages/IPythonNotebookVirtualenvs/):
1. Activate your virtualenv and install the ipython kernle module* 
```
activate [env_name] 
pip install ipykernel
```
2. Now run the kernel "self-install" script:
```
python -m ipykernel install --user --name=[env_name]
```
3. You should now be able to see your kernel in the IPython notebook menu: Kernel -> Change kernel and be able to switch to it (you may need to refresh the page)
4. There you go!

_*ipykernel will require Microsoft Visual C++ Compiler for Python. If you don't have it, you can get it from [here](http://aka.ms/vcpython27)_