# Python Test Rules

Apply these rules when editing or generating `*_test.py`, `test_*.py`, or `conftest.py` files (Kit-A2, H3).

## File Naming
- Test files: `test_<feature>.py` (e.g., `test_login.py`, `test_checkout.py`)
- Conftest: `conftest.py` per directory for fixtures
- Page Objects (H3): `<page_name>_page.py` (e.g., `login_page.py`)

## Imports
- Always use: `import pytest`
- Requests (API): `import requests` — do NOT use `httpx` unless specified
- Selenium (H3): `from selenium import webdriver`, `from selenium.webdriver.common.by import By`

## Fixtures
- All shared setup/teardown goes in `conftest.py` as `@pytest.fixture`
- Scope appropriately: `scope="session"` for expensive setup, `scope="function"` for isolated state
- API base URL fixture:
  ```python
  @pytest.fixture(scope="session")
  def base_url():
      return os.getenv("BASE_URL", "https://api.example.com")
  ```
- Session fixture:
  ```python
  @pytest.fixture(scope="session")
  def api_session(base_url):
      session = requests.Session()
      session.base_url = base_url
      yield session
      session.close()
  ```

## Test Structure
- Test functions named: `test_<tc_id>_<description>` (e.g., `test_tc001_login_valid_credentials`)
- Use docstrings for test documentation: `"""TC-001: Verify login with valid credentials"""`
- Group related tests in classes: `class TestLogin:` — class name starts with `Test`

## API Assertions (Kit-A2)
- Always assert status code first: `assert response.status_code == 200`
- Use `response.json()` for JSON body assertions
- Validate response schema using `jsonschema.validate()` for critical endpoints
- Never assert on full response body strings — assert on specific fields

## Selenium Rules (Kit-H3)
- Use explicit waits: `WebDriverWait(driver, 10).until(EC.visibility_of_element_located(...))`
- Never `time.sleep()` — use `WebDriverWait` only
- Locator priority: `By.ID` > `By.NAME` > `By.CSS_SELECTOR` > `By.XPATH`

## BDD with pytest-bdd (optional)
- Feature files in `features/` directory
- Step files: `step_defs/test_<feature>_steps.py`
- Use `@given`, `@when`, `@then` decorators from `pytest_bdd`
- Fixtures are shared between step files via `conftest.py`

## Reporting
- Allure: use `@allure.title()`, `@allure.description()`, `@allure.step()` decorators
- `pytest-html`: configured via `pytest.ini` or `pyproject.toml` `addopts = --html=report.html`
