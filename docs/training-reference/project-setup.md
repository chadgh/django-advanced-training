# Setting up the training project

The training project can be found on github at [https://github.com/chadgh/django-advanced-training-project](https://github.com/chadgh/django-advanced-training-project).

Follow the following steps to setup the project:

1. Clone the project. `git clone https://github.com/chadgh/django-advanced-training-project.git`
2. Change to the project directory. `cd django-advanced-training-project`
3. Create a Python virtual environment. `python -m venv .venv && source .venv/bin/activate`
4. Install Django. `pip install django`
5. Run Django migrations. `./manage.py migrate`

# Working on the project

If you leave the projects virtual environment but you have already setup the project, you can get back into the project by the following steps:

1. Change to the project directory. `cd django-advanced-training-project`
2. Activate the virtual environment. `source .venv/bin/activate`

# Running the development server

After you have setup the project and are in the projects virtual environment you can run the development server from within the project directory with `./manage.py runserver`. The server will be hosted at [localhost:8000](http://localhost:8000).
