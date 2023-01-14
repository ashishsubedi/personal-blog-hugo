---
title: "Making My Life Easier with Bash/Zsh Function"
date: 2023-01-14T09:02:39+05:45
draft: false
---
### Intro

Hello everyone, it's been such a long time since my last blog. Life's been busy and have not been able to update the blog as I thought I would. Anyway, new post incoming.  

Unlike every other people on the planet, I use `pyenv virtualenv` to create virtualenv. Reasons: 
- For me is to easily switch between python version depending on the repo, as my workplace maintains repos with 2 python versions. 
- Also, I like how venv is separate from the main codebase. 

One drawback is that you have to remember the name of the venv that you need for the repo. To overcome this drawback, I started naming my venv with the name of the directory I'm currently working on.  

This worked good, but everytime I need to activate my venv in different repo (happens everytime), it gets tedious to write `pyenv activate <name>`. 

So I wrote this bash/zsh function to make my life easier. Bonus, also added initialization for [`pre-commit`](https://pre-commit.com/) if it doesn't exist, because I forget to do that more often than not.

### Working

- This function activates the virtual env based on the directory name or uses the name passed. 
- In addition, it also checks if pre-commits is installed, and installs if not. 
 
- Assumes pyenv  and pip  commands are available

- In `~/.bashrc`  or `~/.zshrc` 

```
### Activate virtual env based on the directory name
# Install precommit if not installed
#
ve() {
    # only do this in git directory
    local git_root=$(git rev-parse --show-toplevel 2>&1)
    local dir_name="$1"
    # Check if git repo
    if [[ -z "${dir_name}" && "$git_root" == *"fatal"* ]]; then
	    echo "Not a git directory, ignoring..."
	    return 0
    fi
    local dir_name=$(basename $git_root)
    local venv="${1:-$dir_name}"

    # If not already in virtualenv
    if [ -z "${VIRTUAL_ENV}" ]; then
        echo "Trying to activate virtual environment  ${venv}, activating..."
	pyenv activate $venv
    elif [[ "${VIRTUAL_ENV}" != ${venv} ]]; then
	local exists=$(pyenv virtualenvs | grep $(basename $venv))
	if [[ "$exists" ]]; then
    		echo "Activating virtual environment, ${venv}..."
		pyenv deactivate
		pyenv activate $venv
	else
		echo "Virtualenv ${venv} doesn't exist, continuing with same virtualenv..."
	fi
    else
        echo "Already in a virtual environment ${dir_name}!"
    fi
    ##
    if [[ "$git_root" == *"fatal"* ]]; then
	    echo "Not a git directory, ignoring pre-commit..."
	    return 0
    fi
    local pre_commit=$(ls $git_root/.git/hooks | grep -x "pre-commit\$")

    if [ -z "$pre_commit" ]; then
	    echo "Installing pre-commit, please wait..."
	    pip install pre-commit  > /dev/null 2>&1
	    pre-commit install
	    echo "Installation complete..."
    else
	    echo "Pre-commit already installed..."

    fi
}
```
### Usage
- Typing `ve` in terminal runs this function whenever you need to activate virtualenv based on the current directory name
- Typing `ve <venv_name>` to activate the given `<venv_name>` virtualenv.

### Limitation
- Doesn't create new virtualenv if it cannot find
- Might have some bugs  

### Ending Note
I've been using this for while now, and I find it really useful. This definitely has made my life easier and saves me 2 seconds of my time XD. Well, that's it folks, hope you have a great one. See ya!

