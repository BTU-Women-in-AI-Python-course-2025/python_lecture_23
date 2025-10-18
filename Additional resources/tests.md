# Testing in Django — Full Documentation

Testing in Django ensures that your app works as expected before pushing it to production. Django is built with testing in mind and extends Python’s `unittest` module.

## What Is Covered

* Why tests matter
* Django testing tools
* Structure and best practices
* Writing test cases (unit & integration)
* Real-world examples
* How to run and organize tests

---

## Why Tests Matter

* Prevent future regressions
* Make refactoring safe
* Speed up development confidence
* Help new contributors understand system expectations
* Automate checks in CI/CD pipelines

### 🧠 Example

```bash
# Example: Run automated tests before deployment
pytest && git push heroku main
```

---

## Django’s Built-in Testing Tools

| Tool                  | Purpose                                           |
| --------------------- | ------------------------------------------------- |
| `TestCase`            | Main test class (uses test DB)                    |
| `Client`              | Simulates requests without running a server       |
| `LiveServerTestCase`  | Runs with live HTTP server (for Selenium)         |
| `TransactionTestCase` | Supports transaction rollback testing             |
| `assert*` methods     | Assertion tools: `assertEqual`, `assertTrue`, etc |

### ⚙️ Example

```python
from django.test import TestCase, Client, LiveServerTestCase

class SimpleTest(TestCase):
    def test_addition(self):
        self.assertEqual(1 + 1, 2)

class ClientTest(TestCase):
    def test_homepage(self):
        client = Client()
        response = client.get('/')
        self.assertEqual(response.status_code, 200)
```

---

## Test Structure & Setup

You can keep all tests in a central file or split them:

```
my_app/
├── models.py
├── views.py
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_views.py
│   └── test_forms.py
```

In `settings.py`, make sure `my_app` is listed in `INSTALLED_APPS`.

### 📁 Example

```python
# my_app/tests/test_urls.py
from django.test import SimpleTestCase
from django.urls import reverse, resolve
from my_app.views import product_list

class TestUrls(SimpleTestCase):
    def test_product_list_url_resolves(self):
        url = reverse('product-list')
        self.assertEqual(resolve(url).func, product_list)
```

---

## Writing Unit Tests

Unit tests target **individual components**, like model methods or utility functions.

### Example: Model Unit Test

```python
# models.py
from django.db import models

class Product(models.Model):
    title = models.CharField(max_length=100)
    price = models.DecimalField(max_digits=6, decimal_places=2)

    def discounted_price(self, percentage):
        return self.price - (self.price * percentage / 100)
```

```python
# tests/test_models.py
from django.test import TestCase
from .models import Product

class ProductModelTest(TestCase):
    def test_discounted_price(self):
        product = Product.objects.create(title='Watch', price=200)
        self.assertEqual(product.discounted_price(10), 180)
        self.assertEqual(product.discounted_price(0), 200)
```

### ✅ Additional Example

```python
class ProductValidationTest(TestCase):
    def test_string_representation(self):
        product = Product(title='Headphones', price=99.99)
        self.assertEqual(str(product.title), 'Headphones')
```

---

## Writing Integration Tests (Views)

Integration tests verify that **multiple parts** work together — like a view querying the DB and returning a response.

### Example: View Integration Test

```python
# views.py
from django.shortcuts import render
from .models import Product

def product_list(request):
    products = Product.objects.all()
    return render(request, 'products/list.html', {'products': products})
```

```python
# urls.py
from django.urls import path
from .views import product_list

urlpatterns = [
    path('products/', product_list, name='product-list'),
]
```

```python
# tests/test_views.py
from django.test import TestCase
from django.urls import reverse
from .models import Product

class ProductViewTest(TestCase):
    def setUp(self):
        Product.objects.create(title="Laptop", price=1500)
        Product.objects.create(title="Mouse", price=50)

    def test_list_products(self):
        response = self.client.get(reverse('product-list'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Laptop")
        self.assertContains(response, "Mouse")
```

### 🌐 Additional Example

```python
class ProductDetailViewTest(TestCase):
    def setUp(self):
        self.product = Product.objects.create(title="Keyboard", price=100)

    def test_product_detail_view(self):
        response = self.client.get(f'/products/{self.product.id}/')
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Keyboard")
```

---

## Running Tests

Run all project tests:

```bash
python manage.py test
```

Run tests from a specific file:

```bash
python manage.py test my_app.tests.test_models
```

Enable verbose mode:

```bash
python manage.py test -v 2
```

### ▶️ Example

```bash
# Run only one test class
python manage.py test my_app.tests.test_models.ProductModelTest
```

---

## Common Assertions

| Assertion                    | Description                      |
| ---------------------------- | -------------------------------- |
| `assertEqual(a, b)`          | a == b                           |
| `assertTrue(x)`              | bool(x) is True                  |
| `assertIn(a, b)`             | a in b                           |
| `assertContains(resp, s)`    | Check HTML contains string `s`   |
| `assertRedirects(resp, url)` | Response was a redirect to `url` |

### 🧩 Example

```python
def test_redirects_to_login(self):
    response = self.client.get('/dashboard/')
    self.assertRedirects(response, '/accounts/login/?next=/dashboard/')
```

---

## Best Practices

* Use `setUp()` for test data creation
* Separate tests by functionality (models, views, forms, serializers)
* Prefer factories or fixtures over raw `create()` in complex cases
* Write tests for both happy and failure paths
* Keep each test focused and isolated

### 💡 Example

```python
from django.test import TestCase
from my_app.models import Product

class ProductFailPathTest(TestCase):
    def test_discount_with_invalid_percentage(self):
        product = Product.objects.create(title='Phone', price=500)
        with self.assertRaises(TypeError):
            product.discounted_price('ten')
```

---

## Summary Table

| Feature            | Description                                      |
| ------------------ | ------------------------------------------------ |
| `TestCase`         | Sets up/tears down a test DB per test case       |
| `Client.get/post`  | Simulate HTTP requests without actual server     |
| `reverse()`        | Resolve view names into actual URLs              |
| `assertContains()` | Checks if HTML response includes a given text    |
| `test_` prefix     | Django runs only functions prefixed with `test_` |

