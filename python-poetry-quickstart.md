# 🧪 How to Test Python Scripts Without Installing Libraries Globally

This guide helps you set up and test Python apps in an isolated environment using [Poetry](https://python-poetry.org/), without polluting your system Python or globally installing packages.

---

## ✅ Step 1: Install Poetry Using Pipx

It's recommended to use [pipx](https://pypa.github.io/pipx/) for installing CLI tools like Poetry.

```shell
# Install pipx (if not already installed)
python -m pip install --user pipx
pipx ensurepath

# Install Poetry with pipx
pipx install poetry
```

---

## 🗂️ Step 2: Create Your Project

```shell
poetry new scraper_app --src
cd scraper_app
```

This sets up a project structure where your source code lives in `src/` and tests go under `tests/`.

---

## 📦 Step 3: Add Dependencies

```shell
poetry add requests beautifulsoup4
poetry add --dev pytest
```

---

## 🛠️ Step 4: Write Your Code

- Write your main script inside:  
  `src/scraper_app/scraper.py`

- Create your test file:  
  `tests/test_scraper.py`

---

## 🧪 Step 5: Run the Script or Tests

### Option A: Run the Script

```shell
poetry run python src/scraper_app/scraper.py
poetry run python src/scraper_app/scraper.py https://example.com
```

### Option B: Run Tests with `pytest`

Since your code lives inside `src/`, you’ll need to tell `pytest` where to find it. Create a `pytest.ini` file at the project root:

```ini
#scraper_app/pytest.ini

[pytest]
pythonpath = src
```

Now run your tests:

```shell
poetry run pytest
```

---

## ⚙️ Optional: Add a CLI Entry Point

To make your script runnable as a CLI command:

Edit `pyproject.toml`:

```toml
#python/scraper_app/pyproject.toml

[tool.poetry.scripts]
scrape = "scraper_app.scraper:cli_scrape"
```

Now you can run:

```shell
poetry run scrape https://example.com
```


