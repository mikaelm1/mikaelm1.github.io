---
layout: post
title: Flask Factory with Click
subtitle: How to implement the Factory method in Flask with Click CLI
---

[Click](http://click.pocoo.org/5/) is a Python package for creating command line interfaces and is written by the same guy that created Flask. Click has been integrated into Flask as of version 0.11. I found the "flask" script in version 0.11 to be a bit buggy (hopefully some of its issues will be fixed soon), but Click is a great tool and I used it for my last project with Flask version 0.10. 

There are a lot of samples online about implmenting the Factory method in Flask using the Flask-Script extension, but I was unable to find any using Click. Flask-Script is great, but since Click has been integrated into Flask, I figured I would start using it instead of Flask-Script.

Let's start making our app! Actually, it will just be an empty shell of an app called "flask-factory-click", but that's all that's required to demonstrate the Factory method. Begin by setting up the folders needed for the app as shown below (You can find the final "app" [here](google.com)):

```
├── app
│   ├── __init__.py
│   │   └── __init__.py
│   ├── blueprints
│   ├── data.sqlite
│   └── tests
├── cli
│   ├── __init__.py
│       └── __init__.py
├── requirements.txt
├── setup.py
```

## 1. Virtual Invrionment

First step, after creating the above foldlers/files is to create a virtual environment. While in the project's root directory, run `virtualenv venv .`, which will create a virtual environment for the project's dependencies called venv. This assumes your default Python version is 3.4 or above. If you default is anything else, specify the version you want in the command (more [info](http://docs.python-guide.org/en/latest/dev/virtualenvs/)). Activate the virtual environment by running `source venv/bin activate`. You will know you're in the virtual environment if you see something like `(venv) ➜  flask-factory-click` in your terminal. Form here on, every command listed below is assuming you're in the virtual environment.

## 2. Dependecies

Inside `requirements.txt`, type the following:
Flask==0.10.1
Click==6.4
Flask-SQLAlchemy==2.1

Run `pip install -r requirements.txt`. This will install the three dependencies inside the requirements file.

## 3. Setup CLI with setuptools

Inside `setup.py`, type the following:
```python
from setuptools import setup 

setup(
    name='Flask-Factory-CLI',
    version='1.0',
    packages=['cli'],
    include_package_data=True,
    install_requires=[
        'click',
    ],
    entry_points="""
        [console_scripts]
        factory=cli:cli 
    """,
)
```

"setup" is a Python library for creating packages and will allow us to run the "factory" command as if it was a binary. It basically links Click to the `cli` folder where our cli commands will be stored. In order to activate the package we need to run `pip install --editable .`. This command will create a folder called `Flask_Factory_CLI.egg-info`, but don't run it just yet.

## 4. Setup App

Now that we have all of our dependencies installed, it's time to create the actual app. Inside `app/__init__.py`, type the following:
```python
import os

from flask import Flask
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

def create_app():
    app = Flask(__name__)
    app.config['DEBUG'] = True
    app.config['SECRET_KEY'] = 'ejkfhbwi242ri'
    basedir = os.path.abspath(os.path.dirname(__file__))
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///' + os.path.join(basedir, 'data.sqlite')
    app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

    db.init_app(app)

    @app.route('/')
    def index():
        return 'Hi there'

    return app
```

If you've worked with Flask before, then this pattern will be familiar to you. The `create_app()` function creates a Flask app instance, initializes a database and returns the app. This is also where you would reigster any Blueprints you might have.

## 5. Setup CLI Run

Now we're ready to create a cli script to will create our app and run it for us. Inside `cli/__init__.py`, type the following:
```python
import click

from app import create_app, db

@click.command()
def cli():
    """
    Run the application
    """
    app = create_app()
    with app.app_context():
        db.drop_all()
        db.create_all()
    app.run()
```

This is a simple Click command that calls the `create_app()` function, resets the database, and runs the app. You'll notice that the databse operations are done on the application context. The reason for this is the way we initialized the Flask-SQLAlchemy databse istance in our `app/__init__.py` file, which requires an application context to be attached to. You can read more about it [here](http://flask-sqlalchemy.pocoo.org/2.1/api/#configuration).

Now if you try running `factory` in your terminal, the app should start up. If you go to "http://localhost:5000/", you'll see "Hi there" in the browser!


