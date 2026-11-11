# Testing Methodologies & TDD

## Introduction

Software testing is fundamental to delivering reliable, maintainable code. Beyond catching bugs, testing provides documentation, enables safe refactoring, and gives developers confidence when making changes.

**Why Testing Matters:**

- **Early Bug Detection**: Find issues before they reach production

- **Documentation**: Tests describe expected behavior better than comments

- **Refactoring Safety**: Change code confidently knowing tests catch regressions

- **Design Feedback**: Hard-to-test code often indicates design problems

- **Collaboration**: Tests help team members understand and modify code safely

------------------------------------------------------------------------

## The Testing Pyramid

The testing pyramid is a strategy for balancing different types of tests:

``` mermaid
        block-beta
    columns 1
    block:e2e:1
        E["E2E Tests — Slow, Expensive (~5-10%)"]
    end
    block:integ:1
        I["Integration Tests — Medium Speed (~20-30%)"]
    end
    block:unit:1
        U["Unit Tests — Fast, Cheap (~60-70%)"]
    end
```

| Level | What It Tests | Speed | Scope |
|----|----|----|----|
| **Unit** | Individual functions/classes in isolation | Fast (ms) | Narrow |
| **Integration** | Components working together | Medium (seconds) | Medium |
| **E2E** | Full user workflows | Slow (minutes) | Wide |

Follow the pyramid: many fast unit tests, fewer integration tests, minimal E2E tests. Inverted pyramids lead to slow, brittle test suites.

------------------------------------------------------------------------

## Unit Testing

Unit tests verify that individual “units” of code work correctly in isolation. A unit is typically a function, method, or class.

### Characteristics of Good Unit Tests (F.I.R.S.T.)

| Principle | Description |
|----|----|
| **Fast** | Run in milliseconds; you should run them constantly |
| **Isolated** | No dependencies on external systems (DB, network, filesystem) |
| **Repeatable** | Same result every time, regardless of environment |
| **Self-validating** | Pass or fail clearly, no manual inspection needed |
| **Timely** | Written close to production code (ideally before with TDD) |

### Python Example with pytest

``` python
# calculator.py
def add(a: int, b: int) -> int:
    return a + b

def divide(a: int, b: int) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

``` python
# test_calculator.py
import pytest
from calculator import add, divide

class TestAdd:
    def test_add_positive_numbers(self):
        assert add(2, 3) == 5

    def test_add_negative_numbers(self):
        assert add(-1, -1) == -2

    def test_add_zero(self):
        assert add(5, 0) == 5

class TestDivide:
    def test_divide_normal(self):
        assert divide(10, 2) == 5.0

    def test_divide_by_zero_raises_error(self):
        with pytest.raises(ValueError, match="Cannot divide by zero"):
            divide(10, 0)
```

``` bash
# Run tests
pytest test_calculator.py -v

# Run with coverage
pytest --cov=calculator --cov-report=term-missing
```

### Test Isolation with Mocking

When code depends on external systems, use **test doubles** to isolate the unit:

| Type | Purpose | Example |
|----|----|----|
| **Mock** | Verify interactions (was method called?) | Check if email was sent |
| **Stub** | Provide canned responses | Return fake API response |
| **Fake** | Working implementation (simplified) | In-memory database |
| **Spy** | Record calls while using real implementation | Track function calls |

``` python
# service.py
class PaymentService:
    def __init__(self, payment_gateway):
        self.gateway = payment_gateway

    def process_payment(self, amount: float) -> bool:
        if amount <= 0:
            raise ValueError("Amount must be positive")
        return self.gateway.charge(amount)
```

``` python
# test_service.py
from unittest.mock import Mock
from service import PaymentService

def test_process_payment_calls_gateway():
    # Arrange: Create a mock gateway
    mock_gateway = Mock()
    mock_gateway.charge.return_value = True
    service = PaymentService(mock_gateway)

    # Act: Process payment
    result = service.process_payment(100.0)

    # Assert: Verify behavior
    assert result is True
    mock_gateway.charge.assert_called_once_with(100.0)

def test_process_payment_rejects_negative_amount():
    mock_gateway = Mock()
    service = PaymentService(mock_gateway)

    with pytest.raises(ValueError):
        service.process_payment(-50.0)

    # Gateway should NOT be called for invalid amounts
    mock_gateway.charge.assert_not_called()
```

------------------------------------------------------------------------

## White Box Testing

``` mermaid
        graph LR
    A[Input] --> B[Software<br>internal structure visible]
    B --> C[Output]
    style B fill:#fff,stroke:#333
```

White box testing examines the **internal structure** of code. The tester has full visibility into the implementation and designs tests to cover specific code paths.

### Key Techniques

**Code Coverage Metrics:**

| Metric                 | Description                   | Target         |
|------------------------|-------------------------------|----------------|
| **Statement Coverage** | % of code statements executed | 80%+           |
| **Branch Coverage**    | % of decision branches taken  | 75%+           |
| **Path Coverage**      | % of possible execution paths | Lower priority |

**Example: Testing All Branches**

``` python
# discount.py
def calculate_discount(price: float, is_member: bool, quantity: int) -> float:
    """Calculate discount based on membership and quantity."""
    discount = 0.0

    if is_member:
        discount += 0.10  # 10% member discount

    if quantity >= 10:
        discount += 0.05  # 5% bulk discount
    elif quantity >= 5:
        discount += 0.02  # 2% small bulk

    return price * (1 - discount)
```

``` python
# test_discount.py - White box approach: cover all branches
import pytest
from discount import calculate_discount

class TestCalculateDiscount:
    # Test member branches
    def test_member_gets_10_percent_discount(self):
        assert calculate_discount(100, is_member=True, quantity=1) == 90.0

    def test_non_member_no_member_discount(self):
        assert calculate_discount(100, is_member=False, quantity=1) == 100.0

    # Test quantity branches
    def test_quantity_10_plus_gets_5_percent(self):
        assert calculate_discount(100, is_member=False, quantity=10) == 95.0

    def test_quantity_5_to_9_gets_2_percent(self):
        assert calculate_discount(100, is_member=False, quantity=5) == 98.0

    def test_quantity_under_5_no_bulk_discount(self):
        assert calculate_discount(100, is_member=False, quantity=4) == 100.0

    # Combined branches
    def test_member_with_large_quantity(self):
        # 10% member + 5% bulk = 15% discount
        assert calculate_discount(100, is_member=True, quantity=10) == 85.0
```

### When to Use White Box Testing

- Security-critical code paths

- Complex algorithms with many branches

- Achieving high code coverage requirements

- Understanding legacy code behavior

------------------------------------------------------------------------

## Black Box Testing

``` mermaid
        graph LR
    A[Input] --> B[Black Box<br>internal structure hidden]
    B --> C[Output]
    style B fill:#000,color:#fff,stroke:#000
```

Black box testing examines behavior from the **outside** without knowledge of internal implementation. Tests are based on requirements and specifications.

### Key Techniques

**Equivalence Partitioning**: Divide inputs into classes that should behave the same way.

``` python
# Testing a registration form age field (valid: 18-120)
# Equivalence classes:
# - Invalid: age < 18 (test with 10)
# - Valid: 18 <= age <= 120 (test with 30)
# - Invalid: age > 120 (test with 130)
```

**Boundary Value Analysis**: Test at the edges of valid ranges.

``` python
# For age field (valid: 18-120)
# Test: 17 (invalid), 18 (valid), 119 (valid), 120 (valid), 121 (invalid)

@pytest.mark.parametrize("age,expected_valid", [
    (17, False),   # Just below minimum
    (18, True),    # At minimum
    (50, True),    # Middle (equivalence class representative)
    (120, True),   # At maximum
    (121, False),  # Just above maximum
])
def test_age_validation(age, expected_valid):
    result = validate_age(age)
    assert result == expected_valid
```

### When to Use Black Box Testing

- API testing (testing contract, not implementation)

- User acceptance testing

- When testers don’t have access to source code

- Regression testing from user perspective

------------------------------------------------------------------------

## White Box vs Black Box Comparison

| Aspect | White Box | Black Box |
|----|----|----|
| **Knowledge Required** | Full code visibility | Only specifications |
| **Focus** | Internal code paths | External behavior |
| **Performed By** | Developers | Testers, QA, Users |
| **Test Design** | Based on code structure | Based on requirements |
| **Coverage** | Measures code coverage | Measures feature coverage |
| **Finds** | Logic errors, security flaws | Missing features, spec violations |
| **Best For** | Unit testing, security | Integration, E2E testing |

**Both approaches are complementary!** White box ensures code paths work correctly. Black box ensures the software meets user requirements. Use both for comprehensive coverage.

------------------------------------------------------------------------

## Test-Driven Development (TDD)

TDD is a development methodology where you **write tests before writing production code**. It follows a strict cycle known as **Red-Green-Refactor**.

### The TDD Cycle

``` mermaid
        flowchart TD
    R["RED<br>Write a failing test"] --> G["GREEN<br>Write minimal code to pass"]
    G --> F["REFACTOR<br>Improve design<br>(tests still pass)"]
    F --> R
```

1.  **RED**: Write a test for the next small piece of functionality. Run it—it should fail (you haven’t written the code yet).

2.  **GREEN**: Write the **minimum** amount of code to make the test pass. Don’t over-engineer.

3.  **REFACTOR**: Clean up the code while keeping tests green. Remove duplication, improve names, simplify.

### TDD Example: Building a Password Validator

**Step 1: RED - First failing test**

``` python
# test_password_validator.py
from password_validator import validate_password

def test_password_must_be_at_least_8_characters():
    assert validate_password("short") is False
    assert validate_password("longenough") is True
```

``` bash
$ pytest
# FAILS: ModuleNotFoundError - validate_password doesn't exist yet
```

**Step 2: GREEN - Minimal implementation**

``` python
# password_validator.py
def validate_password(password: str) -> bool:
    return len(password) >= 8
```

``` bash
$ pytest
# PASSES
```

**Step 3: RED - Add next requirement**

``` python
# test_password_validator.py
def test_password_must_contain_uppercase():
    assert validate_password("alllowercase") is False
    assert validate_password("HasUppercase") is True
```

``` bash
$ pytest
# FAILS: "alllowercase" returns True but should be False
```

**Step 4: GREEN - Extend implementation**

``` python
# password_validator.py
def validate_password(password: str) -> bool:
    if len(password) < 8:
        return False
    if not any(c.isupper() for c in password):
        return False
    return True
```

**Step 5: REFACTOR - Improve the code**

``` python
# password_validator.py (refactored)
def validate_password(password: str) -> bool:
    """
    Validate password meets security requirements:
    - At least 8 characters
    - Contains at least one uppercase letter
    """
    checks = [
        len(password) >= 8,
        any(c.isupper() for c in password),
    ]
    return all(checks)
```

**Continue the cycle** for remaining requirements (numbers, special characters, etc.)

### Benefits of TDD

| Benefit | Explanation |
|----|----|
| **Better Design** | Writing tests first forces you to think about interfaces before implementation |
| **High Coverage** | Every feature has tests because tests come first |
| **Documentation** | Tests describe what code should do |
| **Confidence** | Refactor fearlessly with comprehensive tests |
| **Focus** | Work on one small thing at a time |

### When TDD Works Best

- New features with clear requirements

- Complex business logic

- APIs and libraries (designing interfaces)

- When you want high test coverage by default

### When TDD May Not Fit

- Exploratory/prototype code (you’re still figuring out what to build)

- UI code with rapidly changing designs

- Integration with poorly documented external systems

------------------------------------------------------------------------

## Testing in the AI Era (2026 Update)

AI-assisted development generates code faster than ever, but speed without testing discipline produces technical debt at scale. The 2025 DORA Report found that AI-assisted teams saw a 98% increase in PR volume but also a rise in downstream bugs and rework.

### TDD Is More Important with AI Agents

When AI tools generate code, TDD provides the guardrails:

- **Red-Green-Refactor remains the core cycle** — AI accelerates the “Green” step, but you must still write the failing test first

- **Anti-pattern: “Test After” trap** — letting AI write implementation first, then generating tests afterward. These tests often just assert what the code does, not what it *should* do

- **AI-generated tests need human review** — AI can scaffold test cases quickly, but humans must verify edge cases, security scenarios, and business logic correctness

### Mutation Testing

Code coverage alone does not prove test quality. **Mutation testing** verifies that your tests actually catch defects by introducing small changes (mutations) to your code and checking if tests fail:

``` bash
# Python: mutmut
pip install mutmut
mutmut run --paths-to-mutate=src/

# View results
mutmut results

# JavaScript: Stryker
npx stryker run
```

| Metric | What It Tells You |
|----|----|
| **Code coverage** | Which lines were executed during tests |
| **Mutation score** | Which lines are actually tested (mutations caught / total mutations) |

A codebase can have 95% code coverage but a 40% mutation score — meaning most tests are shallow. Aim for 70%+ mutation score on critical business logic.

### Testing Trophy (Alternative to Testing Pyramid)

Kent C. Dodds proposed the **Testing Trophy** model, which emphasizes integration tests over unit tests for web applications:

```
         ___
        / E2E \          ← Few, focused on critical user journeys
       /________\
      /Integration\      ← MOST tests here (components working together)
     /______________\
    /   Unit Tests   \   ← Focused on pure business logic utilities
   /__________________\
  /   Static Analysis   \ ← TypeScript, ESLint, Ruff (free, always on)
 /________________________\
```

**When to use which model:**

- **Testing Pyramid**: Backend services, libraries, APIs with clear unit boundaries

- **Testing Trophy**: Frontend applications, full-stack apps where integration points matter most

- **Testing Diamond**: Microservices where service-to-service integration is the main risk

### Key Metrics for 2026

| Metric             | Target                 | Source               |
|--------------------|------------------------|----------------------|
| Code coverage      | 80%+                   | Industry standard    |
| Mutation score     | 70%+ on critical paths | QASkills 2026        |
| Test-to-code ratio | 1:1 to 1.5:1           | DORA 2025            |
| Test suite runtime | \< 10 minutes          | CI/CD best practices |

------------------------------------------------------------------------

## Modern Testing Frameworks

| Language | Unit Testing | Mocking | Coverage | Mutation Testing |
|----|----|----|----|----|
| **Python** | pytest, unittest | unittest.mock, pytest-mock | pytest-cov, coverage.py | mutmut |
| **JavaScript** | Jest, Vitest, Mocha | Jest mocks, Sinon | Istanbul, c8 | Stryker |
| **Java** | JUnit 5, TestNG | Mockito, MockK | JaCoCo | PITest |
| **Go** | testing (built-in) | gomock, testify | go test -cover | go-mutesting |

------------------------------------------------------------------------

## Best Practices

### Test Organization

```
project/
├── src/
│   ├── user_service.py
│   └── payment_service.py
└── tests/
    ├── unit/
    │   ├── test_user_service.py
    │   └── test_payment_service.py
    ├── integration/
    │   └── test_api.py
    └── conftest.py  # Shared fixtures
```

### Naming Conventions

``` python
# Pattern: test_<thing>_<expected_behavior>_<condition>

def test_user_registration_succeeds_with_valid_email():
    ...

def test_user_registration_fails_when_email_already_exists():
    ...

def test_payment_is_rejected_when_amount_is_negative():
    ...
```

### The Arrange-Act-Assert Pattern

``` python
def test_user_can_update_profile():
    # Arrange: Set up test data and dependencies
    user = User(name="Alice", email="[email protected]")
    new_email = "[email protected]"

    # Act: Perform the action being tested
    user.update_email(new_email)

    # Assert: Verify the expected outcome
    assert user.email == new_email
```

------------------------------------------------------------------------

## Summary

**Key Takeaways:**

1.  **Testing Pyramid**: Many fast unit tests, fewer integration tests, minimal E2E tests

2.  **Unit Testing**: Test isolated units following F.I.R.S.T. principles. Use mocks to isolate dependencies.

3.  **White Box vs Black Box**:

    - White box: Test internal code paths with code visibility

    - Black box: Test external behavior from specifications

    - Use both for comprehensive coverage

4.  **TDD Cycle**: Red (failing test) → Green (minimal code) → Refactor (clean up)

5.  **Benefits of TDD**: Better design, high coverage, documentation, confidence

------------------------------------------------------------------------

## References

1.  [Test-Driven Development by Example - Kent Beck](https://www.oreilly.com/library/view/test-driven-development/0321146530/)

2.  [pytest Documentation](https://docs.pytest.org/)

3.  [Martin Fowler - Test Pyramid](https://martinfowler.com/bliki/TestPyramid.html)

4.  [The Art of Unit Testing - Roy Osherove](https://www.artofunittesting.com/)

5.  [Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com/)

6.  [Python unittest.mock Documentation](https://docs.python.org/3/library/unittest.mock.html)

7.  [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)

## Practice

“Unit testing is an important component of the software development process, as it helps ensure your code’s correctness and reliability. In Python, one of the most popular testing frameworks is pytest, due to its simplicity, ease of use, and features.

In this guide, we will go through the basics of unit testing using pytest, with examples to help you get started with writing and running tests for your projects.

### What is Unit Testing?

Unit testing is a software testing technique that focuses on testing individual units of code, usually functions or methods, in isolation.

By testing these individual units of code across your applications, you can ensure that each part of your code works as expected, making it easier to identify and fix bugs, refactor code, and add new features without introducing regressions.

### Why use pytest?

![Pytest](https://ai-roadmap.ops-ai.dev/_images/pytest.png)

Pytest offers several advantages over other testing frameworks in Python, such as unittest and nose:

1.  **Simplified syntax**: The syntax for tests is concise and easy-to-understand, making it easier and faster to write them.

2.  **Automatic test discovery**: Test functions are automatically discovered without the need for explicit registration.

3.  **Rich plugin ecosystem**: A large number of plugins can extend pytest functionalities and connect it with other tools and services, such as Coverage.py, Django or Elasticsearch.

### Getting started with pytest

#### Installing pytest

During this guide we will use Python 3.11, but pytest supports all Python versions above 3.7. You can install pytest using pip:

``` bash
pip install pytest
```

#### Project structure

We will create a new Python project to go through the different examples. Create a new directory and add the following subdirectories and files:

```
├── main_project_folder
│   ├── src
│   │   └── __init__.py
│   ├── tests
│   │   └── __init__.py
│   └── main.py
```

The `main.py` file will serve as the entry point for your application. In the `src` directory, we’ll store the source code for your project, and in `tests` your test code. Also, we include `__init__.py` files in these directories, which are necessary for pytest to discover the modules, and can remain empty for this tutorial.

### Writing your first tests

#### Testing a Python function

Let’s write a simple Python function that we will test. Create a file called `math_utils.py` in the `src` folder and add the following code:

``` python
def divide(a: int, b: int) -> float:
    """Return the result of a division operation between two numbers.

    Args:
        a (int): The numerator.
        b (int): The denominator.

    Returns:
        float: The result of the division operation.

    Raises:
        ZeroDivisionError: If b is zero.
    """
    if b == 0:
        raise ZeroDivisionError("You can't divide by zero!")
    return a / b
```

Now, we can write some tests for this function. Create a new file called `test_math_utils.py` in the `tests` folder and add the following code:

``` python
from src.math_utils import divide


def test_divide_positive_numbers() -> None:
    """Test that divide returns the correct result when given two numbers."""
    assert divide(1, 2) == 0.5


def test_divide_negative_numbers() -> None:
    """
    Test that divide returns the correct result when given a positive and
    a negative number.
    """
    assert divide(5, -2) == -2.5
    assert divide(-2, 5) == -0.4
```

In this example, we’ve written two test functions called `test_divide_positive_numbers` and `test_divide_negative_numbers` that tests the `divide` function from our `math_utils` module.

Their results are tested through the `assert` statement, which checks that the result of the `divide` function is equal to the expected result. If an assertion fails, pytest will stop and report the failure.

You can add as many assertions as you want in a test function, but it’s generally recommended to keep it small. Also, notice how the test module and function names start with the `test_` prefix, this allows pytest to automatically discover them.

Your project structure should now look like this:

```
├── main_project_folder
│   ├── src
│   │   ├── __init__.py
│   │   └── math_utils.py
│   ├── tests
│   │   ├── __init__.py
│   │   └── test_math_utils.py
│   └── main.py
```

To run the tests, navigate to the project root directory and execute the following command in your terminal:

``` bash
pytest
```

If all tests pass, you can be confident that our `divide` function works as expected. 🙂

#### Handling failed tests

Now, let’s see what happens if a test fails. Change the `test_divide_positive_numbers` test function to the following:

``` python
def test_divide_positive_numbers() -> None:
    """Test that divide returns the correct result when given two integers."""
    assert divide(1, 2) == 0.6
```

Notice that we’ve altered the expected result from 0.5 to 0.6. If we run pytest again, it will report an error. Pytest reports that the test failed, indicating that the expected result was 0.6 and not 0.5. The failed assert statement is also highlighted. By changing the expected result back to 0.5, the test will pass again.

#### Testing for exceptions

In our `divide` function, we raise a `ZeroDivisionError` if the denominator is zero. Let’s write a test to ensure that this exception is raised correctly. Add the following test function to your `test_math_utils.py` file:

``` python
import pytest

def test_divide_by_zero() -> None:
    """Test that divide raises a ZeroDivisionError when the denominator is zero."""
    with pytest.raises(ZeroDivisionError, match="You can't divide by zero!"):
        divide(1, 0)
```

The `pytest.raises` function checks that the code inside the `with` block raises the specified exception. You can also provide an optional `match` argument to verify that the exception message matches a specific pattern.

### Writing More Complex Tests

#### Use fixtures to reuse test data

Let’s now look at a more complex example. To avoid the need for instantiating objects in each test you write, you can create **fixtures** that can be passed as parameters to the test functions. Create a new file called `user.py` in the `src` directory and add the following code:

``` python
class User:
    """
    This class represents a user with attributes ID, name, and age.

    Attributes:
        id (Integer): The ID of the user, serves as the primary key.
        name (String): The name of the user.
        age (Integer): The age of the user.

    Methods:
        greet(): Returns a greeting message that includes the user's name and age.
    """
    id: int
    name: str
    age: int

    def __init__(self, id: int, name: str, age: int) -> None:
        """
        Initialize a User instance.

        Args:
            id (int): The ID of the user.
            name (str): The name of the user.
            age (int): The age of the user.
        """
        self.id = id
        self.name = name
        self.age = age

    def greet(self) -> str:
        """
        Create a greeting message that includes the user's name and age.

        Returns:
            str: A greeting message.
        """
        return f"Hello, my name is {self.name} and I am {self.age} years old."
```

To test this User object, create a new file called `test_user.py`:

``` python
import pytest
from src.user import User

@pytest.fixture
def user() -> User:
    """Pytest fixture to create a User instance for testing."""
    return User(1, "John Doe", 30)

def test_user_creation(user: User) -> None:
    """Test the creation of a User instance."""
    assert user.id == 1
    assert user.name == "John Doe"
    assert user.age == 30

def test_greet(user: User) -> None:
    """Test the greet method of a User instance."""
    greeting: str = user.greet()
    assert greeting == "Hello, my name is John Doe and I am 30 years old."
```

In this example, we’ve created a fixture called `user` that returns a new User instance. We then passed this fixture as an argument to our test functions, allowing us to reuse the User instance across multiple tests.

To run these tests, simply execute the `pytest` command again.

#### Mock to isolate tests

Your application may need to interact with external services. Testing these interactions can be hard because it involves setting up test databases or APIs. To make testing easier, you can use Python’s `unittest.mock` library. It allows you to mimic (“mock”) methods, replacing the real connection with a fake one.

Example scenario using `sqlalchemy`:

``` python
# src/user_repository.py (Simplified)
# ... imports ...
# class UserRepository:
#     def get_users(self) -> list[User]:
#         users = self.session.query(User).all()
#         return users
```

To test this function without connecting to a database, you can utilize `unittest.mock.create_autospec`:

``` python
import pytest
from unittest.mock import create_autospec
from sqlalchemy.orm import Session

from src.user_repository import UserRepository
from src.user import User

@pytest.fixture
def mock_session() -> Session:
    """Pytest fixture to create a mock Session instance."""
    session = create_autospec(Session)
    return session

def test_get_users(mock_session: Session) -> None:
    """Test the get_users method of the UserRepository class."""
    # Create a fake user
    fake_user = User(id=1, name="Alice", age=28)

    # Mock the Session.query() method to return our fake user
    mock_session.query.return_value.all.return_value = [fake_user]

    # Create a UserRepository instance with the mocked session
    user_repository = UserRepository("postgresql://test:test@test/test")
    user_repository.session = mock_session

    # Call the get_users method
    users = user_repository.get_users()

    # Ensure that Session.query() was called with the correct argument
    mock_session.query.assert_called_with(User)

    # Assert that the method returned our fake user
    assert users == [fake_user]
```

### Key Takeaways

1.  **Keep tests simple and focused**: Write tests that check one specific aspect of your code.

2.  **Use descriptive test function names**: E.g., `test_divide_positive_numbers`.

3.  **Organize your tests**: Group related tests in the same module.

4.  **Test for edge cases**: Cover extreme scenarios and invalid inputs.

5.  **Use pytest fixtures**: Avoid duplicating setup code.

6.  **Mock external dependencies**: Isolate your code from external dependencies.

### Conclusion

Unit testing is crucial to check if your code works right and is reliable. Using best practices and pytest’s tools, you can build tests to protect your code from errors and regressions.

### References

- [Pytest Documentation](https://docs.pytest.org/en/stable/)

- [Pytest GitHub](https://github.com/pytest-dev/pytest)

- [Pytest Example](https://github.com/adocquin/medium_unit_testing/tree/main/tests)

## Review Questions

### Theory: White Box vs Black Box Testing

1.  **What is the fundamental difference between White Box and Black Box testing?**

    - *Hint*: Think about what the tester can see regarding the internal code.

2.  **List the three basic steps to perform White Box testing.**

    - *Hint*: It starts with understanding something specific.

3.  **What are two advantages of Black Box testing?**

    - *Hint*: Consider the tester’s perspective and the implementation details.

4.  **Compare White Box and Black Box testing in terms of “Module Communication”.**

    - *Hint*: Which one facilitates verifying how modules talk to each other?

5.  **Why is Unit Testing considered a “White Box” technique?**

    - *Hint*: Who writes unit tests and what do they inspect?

### Practice: Pytest

1.  **How does `pytest` automatically discover test files and functions?**

    - *Hint*: What naming convention must be followed?

2.  **What is the purpose of a `fixture` in pytest? Give an example of when you would use one.**

    - *Hint*: Think about setup code and reusability.

3.  **How do you assert that a specific exception is raised in a test function?**

    - *Hint*: Look for a specific context manager provided by pytest.

4.  **What is “Mocking” and why is it useful when testing database interactions?**

    - *Hint*: Consider what happens if the database is down or slow.

5.  **In the directory structure, why are `__init__.py` files included in `src` and `tests` directories?**

    - *Hint*: How does Python handle modules?
