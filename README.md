# klimaschutzmonitor

## Dev notes

Quick overview:

- We use poetry (so pyproject.toml lists the requirements, and poetry.lock the exact versions used to fulfill those requirements).
- To make sure that exactly those are installed we use a python virtual environment
- black is used to format the code
- a pre-commit hook is used to keep commits clean
- We use pyright to get a little bit of type checking. Currently this is just in basic mode and without any special handling for the django ORM. I expect both things to potentially need tweaking (for example: <https://github.com/sbdchd/django-types> looks potentially useful).

### How to install the dev environment

- Windows only: Install WSL (Windows Subsystem for Linux) with `wsl --install` (See <https://learn.microsoft.com/en-us/windows/wsl/setup/environment>)
  - Note: The Microsoft setup description says that Build 20262 is required. That seems to be a prerelease build number from 2020 or 2021. Windows 10 22H2 with build number 19045 works.
  - After the installation: Update the Linux distribution (in the WSL shell) with `sudo apt-get update` 
- Windows only: Use the WSL shell for all commands from here on. (For vscode see below.)
- Install python version 3.10 or greater, e.g. with `sudo apt install python3`.
- Install venv, e.g. with `sudo apt install python3-venv` (the version needs to match the python version).
- Install poetry with `curl -sSL https://install.python-poetry.org | python3 -` (See <https://python-poetry.org/docs/>. The version in Ubuntu is too old. A new login shell might be needed after that.)
- Install `sudo apt install python-is-python3` so that poetry can run python3 with the python command.

The above steps are needed only once per machine.

Then run the following commands:

```shell
git clone https://github.com/GermanZero-de/klimaschutzmonitor.git
cd klimaschutzmonitor # Or `code klimaschutzmonitor` to open it with vscode.
python3 -m venv .venv
poetry shell
poetry install --sync
pre-commit install
```

This will

- clone this repository,
- create a python virtual environment in the sub-directory `.venv` of the cloned repository,
- install all the dependencies of the project as specified in `pyproject.toml` and `poetry.lock` into this virtual environment, and
- install a pre-commit hook in the cloned repository. (Will be run before each commit and check the code.)

Whenever you work with the project, call "`poetry shell`" first. Windows only: Do this within the WSL shell.
In this shell python points to the python virtual environment in `.venv`.

Whenever dependencies changed, call "`poetry install --sync`" first.

Otherwise, these commands need only be repeated when a new clone is created.

### Tips for vscode

Windows only: After installing WSL,

- install the WSL extension in vscode,
- click the bottom left corner (green) and select "Reopen folder in WSL", and
- select Terminal -> New Terminal for a WSL shell.

After cloning the repository, open the new directory with vscode. (Windows only: Make sure you are in WSL, possibly "Reopen folder in WSL".)

Recommended extensions should be offered. If not, got to the "Extensions" side-bar and enter `@recommended`. Then select "Install Workspace Recommended Extensions".

### How to run the development server

We roughly follow some of the project layout / config layout ideas of the Two Scoops of Django book (<https://www.feldroy.com/books/two-scoops-of-django-3-x>).

So the configuration lives in `config/settings/<what>.py`. Most importantly that means when testing
locally you want to do this:

```shell
# (inside your poetry shell)

#prepare database
python manage.py makemigrations --settings=config.settings.local
python manage.py migrate --settings=config.settings.local

#generate user for admin UI
python manage.py createsuperuser --settings=config.settings.local

#start the server
python manage.py runserver --settings=config.settings.local
```

The main app is called `cpmonitor` short for `climate protection monitor`. As that needs to be a python
package / module name it follows python style conventions of being short and all in one lowercase word.

### Testing
Test are written in pytest (https://docs.pytest.org/en/7.2.x/index.html).
End-to-end tests are written with the playwright plugin of pytest (https://playwright.dev/python/docs/intro).

Execute tests with
```shell
# run all tests
pytest

# run a single test
pytest <path-to-test>

# run e2e test in headed mode
pytest --headed <path-to-e2e-test>
```

New test files have to be named according to the convention: `*_test.py`. <br>
Test names should follow the convention: `test_should_do_x_when_given_y`.

### Containerization and Deployment

The application is deployed to the server as a pair of Docker containers:
- container 1 runs the gunicorn webserver to host the django app itself,
- container 2 runs nginx, a proxy that hosts the static files while providing stability and security.

Only the port of nginx is exposed, which will forward requests to the django app or provide any requested static files directly.

#### Building the Django app Docker image and running the container

The Dockerfile for the django app is based on the following resources:

- [DigitalOcean blog post](https://www.digitalocean.com/community/tutorials/how-to-build-a-django-and-gunicorn-application-with-docker)
- [Stackoverflow answer](https://stackoverflow.com/a/57886655)
- [Sample Repo](https://github.com/mgnisia/Boilerplate-Docker-Django-Gunicorn-Nginx)

It uses a multi-stage build to prevent shipping unnecessary files which would increase image size and attack surface.

To build the image, run the following command in the repository root directory (containing the Dockerfile):

```shell
docker build . -t cpmonitor
```

To run the container, run (replacing directory of `db.sqlite3` and SECRET_KEY placeholders):

```shell
docker run --rm -it -p 8000:8000 -v <absolute path of directory containing db.sqlite3>:/db -e SECRET_KEY=<...> cpmonitor
```

By default, the container will run the app using the production configuration.
To use the local configuration, run:

```shell
docker run --rm -it -p 8000:8000 -e DJANGO_SETTINGS_MODULE=config.settings.local-container -v <absolute path of directory containing db.sqlite3>:/db cpmonitor
```

Instead of passing the absolute path to this repo, you may instead use

- in a Linux shell:

    ```shell
    -v "$PWD":/db
    ```

- in Powershell:

    ```powershell
    -v ${pwd}:db
    ```

- in a WSL Linux shell, if you decided to store your projects outside the WSL file system (otherwise see above):

    ```shell
    -v "${PWD/\/mnt/}":/db
    ```

#### Deployment including nginx

To run both containers together, build the app container as described above and then run the following command in the repository root directory:
```shell
# using production config
docker compose --env-file .env.production up
```
```shell
# using local config
docker compose --env-file .env.local up
```
This will start both the Django app and the nginx containers. The website can then be reached at http://localhost:80.
