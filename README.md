# Dream Planner

Django REST API for a trip-budget app: accounts, dream destinations, and the expenses logged against them.

Dream Planner was a 2022 [General Assembly](https://generalassemb.ly/) group project. This repository is the API. The interface is a separate React app, [devin-lynch/dreamplanner-client](https://github.com/devin-lynch/dreamplanner-client).

![Destination cards in the React client](img/destinations.png)

*Destination cards from the React client. This API stores each destination's name, budget, photo URL, description, completed flag, and owner.*

## What it does

A person saves places they want to go, attaches a budget, and records what the trip actually cost. Three models back that:

| Model | What it stores |
| --- | --- |
| **User** | Email (the login id), unique username, password, `created_at`, `updated_at`, plus the auth columns from `PermissionsMixin` (`is_superuser`, groups, user permissions) and `last_login`. Custom user on `AbstractBaseUser`. |
| **Destination** | Optional owner, name, budget, photo URL, description, and a `completed` flag. Deleting a user deletes their destinations. |
| **Expense** | Date, destination, optional user, category, merchant, amount, and a short note. Deleting a destination deletes its expenses. |

Expense categories are `transportation`, `lodging`, `food`, `activities`, and `misc`.

A user response embeds that user's destinations, and a destination response embeds its expenses. Creating a record is a flat post to that record's own endpoint. Photo is a URL string. The budget and the expense amount are floats.

JSON Web Tokens are issued at `/api/token/` (email and password in, access and refresh tokens out) and renewed at `/api/token/refresh/`. Access tokens last 5 minutes. Refresh tokens last 1 day. The header the token views expect is `Authorization: Bearer <access>`.

A request with no token still succeeds. These viewsets keep DRF's default permission, allow-any, so list, create, update, and delete are open on users, destinations, and expenses. A request that sends a bad token gets a 401, because JWT is the default authentication class. `POST /api/users/` hashes the password before save. A user payload still includes every column, password hash included.

Django admin is registered for all three models at `/admin/`.

Sample users, destinations, and one expense ship as fixtures under `backend/dreamplanner/fixtures/`.

![Expense summary in the React client](img/dashboard.png)

*Expense summary in the React client. Rows here match the expense model: date, category, merchant, amount, and details, next to the destination budget.*

## API endpoints

Routes are mounted at `/api/` by a DRF router. `{id}` is the integer primary key. Collection URLs accept `GET` and `POST`. Detail URLs accept `GET`, `PUT`, `PATCH`, and `DELETE`.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/` | Lists the registered endpoints |
| `GET`, `POST` | `/api/users/` | List users, or create one |
| `GET`, `PUT`, `PATCH`, `DELETE` | `/api/users/{id}/` | Read or change one user |
| `GET`, `POST` | `/api/destinations/` | List destinations, or create one |
| `GET`, `PUT`, `PATCH`, `DELETE` | `/api/destinations/{id}/` | Read or change one destination |
| `GET`, `POST` | `/api/expenses/` | List expenses, or create one |
| `GET`, `PUT`, `PATCH`, `DELETE` | `/api/expenses/{id}/` | Read or change one expense |
| `POST` | `/api/token/` | Exchange email and password for an access token and a refresh token |
| `POST` | `/api/token/refresh/` | Exchange a refresh token for a new access token |
| | `/admin/` | Django admin for User, Destination, and Expense |

CORS allows one origin, `http://localhost:3000`.

A destination post looks like this:

```json
{
  "name": "Lisbon",
  "budget": 13000,
  "photo": "https://example.com/lisbon.png",
  "description": "Historical and modern, side by side.",
  "completed": false,
  "user": 1
}
```

An expense post looks like this:

```json
{
  "date": "2022-11-23",
  "destination": 1,
  "user": 1,
  "category": "food",
  "merchant": "Market",
  "amount": 98,
  "details": "Groceries for the week"
}
```

## Tech stack

- Python 3.10.8, from `backend/runtime.txt` (the file reads `python -3.10.8`)
- Django 4.1.2
- Django REST Framework 3.14.0
- djangorestframework-simplejwt 5.2.2 and PyJWT 2.6.0
- django-cors-headers 3.13.0
- PostgreSQL via psycopg2 2.9.5
- Gunicorn 20.1.0 and WhiteNoise 6.2.0, with a `web` process in `backend/Procfile`

Install from `backend/requirements.txt`. The `requirements.txt` at the repository root is a shorter list (Django, asgiref, backports.zoneinfo, psycopg2, sqlparse) and leaves out REST framework, JWT, CORS, Gunicorn, and WhiteNoise.

## Run it locally

You need Python 3.10 and PostgreSQL. Settings connect to a database named `dreamplanner` and leave user, password, host, and port unset, so Django uses its defaults: the current OS user, an empty password, a local Unix socket, and port 5432.

`backend/backend/settings.py` has two lines that stop the process before it can serve:

- `ALLOWED_HOSTS = [*]` is a syntax error (`Invalid star expression` on line 28). Change that assignment to a real list, such as `['localhost', '127.0.0.1']`, in your local checkout.
- `STATICFILES_DIRS` includes `backend/static`, and that directory is not in the repo. Create the empty directory, or remove the entry, before `runserver`. Django's staticfiles check fails while the path is missing.

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py loaddata users destinations expenses
python manage.py runserver
```

The API is then at [http://127.0.0.1:8000/api/](http://127.0.0.1:8000/api/). The same URLs render DRF's browsable API in a browser. Start the UI from [dreamplanner-client](https://github.com/devin-lynch/dreamplanner-client).

The user fixture stores plaintext passwords. JWT login checks a Django password hash, so a seed account will not authenticate until its password is saved again through `POST /api/users/` or `set_password`. `UserManager` implements `create_user` and has no `create_superuser`, so `python manage.py createsuperuser` has nothing to call.

## Team

Four of us built this in October 2022 for General Assembly: Valerie, Sean, Prija Cash, and Devin. In this repo I started the Django project, added the fixtures, and added the Gunicorn, WhiteNoise, and Procfile setup. I also merged the pull requests that brought in the models, the JWT views, and extra seed data. The React client is [devin-lynch/dreamplanner-client](https://github.com/devin-lynch/dreamplanner-client).

## What I learned

Working on this API taught me the shape of a Django REST project: a model, a serializer, a viewset, and a router, plus a custom user that logs in with email. Nested serializers are what put a destination's expenses on the destination response. The token URLs, the password hasher, and that custom user have to agree, or login fails. The fixtures made that concrete for me: a password left in JSON as plain text is a value Django's checker rejects. Gunicorn, WhiteNoise, and the Procfile are how this repo runs the API as a `web` process outside `runserver`.

## What I would improve

- **Secrets and settings.** Read the secret key, `DEBUG`, allowed hosts, the database URL, and CORS origins from the environment. The Django secret key is committed in `backend/backend/settings.py`, and `DEBUG` is `True`.
- **Authorization.** Require a JWT on the viewsets and limit destinations and expenses to the signed-in user. Take `password` off the user response.
- **The user manager and seed data.** `create_user` assigns `username = first_name`, and `first_name` is not defined, so that method raises `NameError`. Add `create_superuser`, and store hashed passwords in the fixtures.
- **Tests.** `backend/dreamplanner/tests.py` is still the file Django generated. I would cover user creation, the password hash, and a request rejected when the token is missing.
- **Small correctness fixes.** Give each router entry its own basename (users, destinations, and expenses currently share `dreamplanner`). Return a string from `Expense.__str__` (it returns the float `amount` today). Make the root `requirements.txt` match `backend/requirements.txt`.
- **A savings figure, if I extended the product.** A weekly savings number needs start and end dates. The destination model stores a budget and a completed flag, so that calculation has nowhere to live yet. A hotel search would be a new endpoint too. The client nav has a Hotels link, and this API's routes stop at users, destinations, expenses, tokens, and admin.
