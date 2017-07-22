# Setting up the training project

The training project can be found on github at [https://github.com/chadgh/django-advanced-training-project](https://github.com/chadgh/django-advanced-training-project).

Follow the steps below to setup the project:

1. Clone the project. `git clone https://github.com/chadgh/django-advanced-training-project.git`
2. Change to the project directory. `cd django-advanced-training-project`
3. Create a Python virtual environment. `python -m venv env && source env/bin/activate`
4. Install Django. `pip install django`
5. Run Django migrations. `./manage.py migrate`
6. Create a superuser. `./manage.py createsuperuser`

# Working on the project

If you leave the projects virtual environment but you have already setup the project, you can get back into the project by the following steps:

1. Change to the project directory. `cd django-advanced-training-project`
2. Activate the virtual environment. `source env/bin/activate`

# Running the development server

After you have setup the project and are in the projects virtual environment you can run the development server from within the project directory with `./manage.py runserver`. The server will be hosted at [localhost:8000](http://localhost:8000).

# Creating data

With the development server running go in your browser to [localhost:8000/admin/(http://localhost:8000/admin/)], login with the superuser credentials you created at setup, and create a few todo items from the admin screens.

After you have a few todo items, you can go to [localhost:8000/todos/](http://localhost:8000/todos/) to see and interact with the todo items.

# Using the base training project

Once you have the project setup, you can use it to try out features that you learn about from the training as well as attempt to solve the hands-on problems presented for each feature. Use the project as a sandbox to try things out with Django. 

<iframe width="560" height="315" src="https://www.youtube.com/embed/FvJR4H5CSd8" frameborder="0" allowfullscreen></iframe>
