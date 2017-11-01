---
category: Blog
---
# My Chromebook Setup
I just finished setting up the data analysis environment on my chromebook. I'll retrace my steps here in case anyone else wants to do the same.

## Step 1: Install Ubuntu
The very first step is to install ubuntu, so you can then install python and all other parts to your data kit. I folowed [this guide](https://www.linux.com/learn/how-easily-install-ubuntu-chromebook-crouton%20), using the Xfce desktop environment. Switch to the Linux environment.

I then used [this guide](https://www.linuxbabe.com/ubuntu/install-google-chrome-ubuntu-16-04-lts) to install the chrome browser (installing within the linux environment). I also installed a the default Ubuntu GUI text editor Gedit via [these instructions](https://help.ubuntu.com/community/gedit).

I later decided I also wanted to install the sublime text editor. It is really quite easy and explained well [here](http://monkeyhacks.com/post/how-to-install-sublime-text-2-on-ubuntu-14-04). However, after getting this message:

```
sudo: add-apt-repository: command not found
```

I needed to install the sudo-apt-repository command, which I did following [these](http://lifeonubuntu.com/ubuntu-missing-add-apt-repository-command/) instructions.

## Step 2: Install Anaconda
Installing Anaconda is the easiest and quickest way to get the data analytics environment up and running. Within the installed chrome browser, I downloaded [Anaconda for Linux](https://www.continuum.io/downloads), navigated to the Downloads folder in the terminal window, and then installed using the bash command in the instructions.

When I installed Anaconda, the Anaconda directory was not added to the bash shell PATH variable, so (using the Gedit text editor) I added it to the file:

```
gedit ~/.bashrc
```

which opened the .bashrc file, navigated to the appropriate line and added

```
export PATH="/home/tcblack/anaconda2/bin:$PATH"
```

(make sure to replace 'tcblack' with your username) and edit the path if it is different from this one.

After editing the path variable, I can now open the jupyter notebook just by typing in

```
jupyter notebook
```

into the terminal.

## Step 3: Install Git and Configure Github
The final step is to install git and connect to my online Github account. I followed [these](https://www.howtoforge.com/tutorial/install-git-and-github-on-ubuntu-14.04/) instructions. For those unfamiliar with using Git / Github, a good place to start would be [here](http://readwrite.com/2013/09/30/understanding-github-a-journey-for-beginners-part-1/a).
