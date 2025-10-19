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
from django.test import TestCase, Client

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

You can keep all tests in a central file or split them by function:

```
blog/
├── models.py
├── views.py
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_views.py
│   └── test_urls.py
```

In `settings.py`, make sure `blog` is listed in `INSTALLED_APPS`.

### 📁 Example

```python
# blog/tests/test_urls.py
from django.test import SimpleTestCase
from django.urls import reverse, resolve
from blog.views import blog_post_list

class TestUrls(SimpleTestCase):
    def test_blog_post_list_url_resolves(self):
        url = reverse('blog-post-list')
        self.assertEqual(resolve(url).func, blog_post_list)
```

---

## Writing Unit Tests

Unit tests target **individual components**, such as model methods or computed properties.

### Example: Model Unit Test

```python
# blog/models.py
from datetime import date
from django.db import models

class Author(models.Model):
    first_name = models.CharField(max_length=100)
    last_name = models.CharField(max_length=100)
    birth_date = models.DateField(null=True)

    @property
    def age(self) -> int:
        today = date.today()
        return today.year - self.birth_date.year - (
            (today.month, today.day) < (self.birth_date.month, self.birth_date.day)
        )

    def __str__(self):
        return self.first_name + " " + self.last_name
```

```python
# blog/tests/test_models.py
from datetime import date
from django.test import TestCase
from blog.models import Author, BlogPost

class AuthorModelTest(TestCase):
    def test_author_age_property(self):
        author = Author.objects.create(
            first_name='Mariam',
            last_name='Kipshidze',
            birth_date=date(2000, 10, 10)
        )
        self.assertIsInstance(author.age, int)
        self.assertGreater(author.age, 0)

    def test_string_representation(self):
        author = Author(first_name='Ana', last_name='Smith')
        self.assertEqual(str(author), 'Ana Smith')


class BlogPostModelTest(TestCase):
    def test_blogpost_str_method(self):
        post = BlogPost(title='My First Post', text='Hello world!')
        self.assertEqual(str(post), 'My First Post')
```

---

## Writing Integration Tests (Views)

Integration tests verify that **multiple parts** (models, views, templates) work together —
for example, a view retrieving authors or blog posts.

### Example: View Integration Test

```python
# blog/views.py
from django.shortcuts import render
from blog.models import BlogPost

def blog_post_list(request):
    posts = BlogPost.objects.filter(active=True, published=True)
    return render(request, 'blog/list.html', {'posts': posts})
```

```python
# blog/urls.py
from django.urls import path
from blog.views import blog_post_list

urlpatterns = [
    path('blog/', blog_post_list, name='blog-post-list'),
]
```

```python
# blog/tests/test_views.py
from django.test import TestCase
from django.urls import reverse
from blog.models import BlogPost

class BlogPostViewTest(TestCase):
    def setUp(self):
        BlogPost.objects.create(title="Django Testing", text="Intro to testing", active=True, published=True)
        BlogPost.objects.create(title="Unpublished Post", text="Hidden", active=True, published=False)

    def test_list_published_posts(self):
        response = self.client.get(reverse('blog-post-list'))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Django Testing")
        self.assertNotContains(response, "Unpublished Post")
```

### 🌐 Additional Example

```python
class BlogPostDetailViewTest(TestCase):
    def setUp(self):
        self.post = BlogPost.objects.create(title="Detail Test", text="Post content", active=True, published=True)

    def test_blog_post_detail_view(self):
        response = self.client.get(f'/blog/{self.post.id}/')
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Detail Test")
```

---

## Running Tests

Run all tests in your Django project:

```bash
python manage.py test
```

Run tests from a specific file:

```bash
python manage.py test blog.tests.test_models
```

Enable verbose mode for more details:

```bash
python manage.py test -v 2
```

### ▶️ Example

```bash
# Run only one test class
python manage.py test blog.tests.test_models.AuthorModelTest
```

---

## Common Assertions

| Assertion                    | Description                       |
| ---------------------------- | --------------------------------- |
| `assertEqual(a, b)`          | a == b                            |
| `assertTrue(x)`              | bool(x) is True                   |
| `assertIn(a, b)`             | a in b                            |
| `assertContains(resp, s)`    | Check if HTML contains string `s` |
| `assertRedirects(resp, url)` | Response was a redirect to `url`  |

### 🧩 Example

```python
def test_redirects_to_login(self):
    response = self.client.get('/admin/blog/')
    self.assertRedirects(response, '/accounts/login/?next=/admin/blog/')
```

## 🌐 LiveServerTestCase Example

`LiveServerTestCase` runs a **live Django server** for browser-based or HTTP testing. Useful for Selenium tests or any test that needs a real HTTP request.

```python
# blog/tests/test_live_server.py
from django.test import LiveServerTestCase
from selenium import webdriver
from blog.models import BlogPost

class BlogLiveServerTest(LiveServerTestCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.browser = webdriver.Chrome()  # Requires ChromeDriver installed

    @classmethod
    def tearDownClass(cls):
        cls.browser.quit()
        super().tearDownClass()

    def setUp(self):
        self.post = BlogPost.objects.create(
            title="Live Server Test",
            text="Testing with live server",
            active=True,
            published=True
        )

    def test_blog_post_list_visible_in_browser(self):
        self.browser.get(self.live_server_url + '/blog/')
        body_text = self.browser.find_element("tag name", "body").text
        self.assertIn("Live Server Test", body_text)
```

✅ **What this does:**

* Starts a temporary live server
* Opens the blog list page in a real browser
* Checks that published posts are visible

---

## ⚡ TransactionTestCase Example

`TransactionTestCase` allows testing **manual DB commits and rollbacks**. Useful if your code uses `transaction.atomic()` or other transactional behavior.

```python
# blog/tests/test_transaction.py
from django.test import TransactionTestCase
from django.db import transaction
from blog.models import BlogPost

class BlogTransactionTest(TransactionTestCase):
    def test_manual_transaction_rollback(self):
        try:
            with transaction.atomic():
                BlogPost.objects.create(
                    title="Temp Post",
                    text="This should rollback",
                    active=True,
                    published=True
                )
                raise ValueError("Force rollback")
        except ValueError:
            pass

        # The post should not exist because the transaction was rolled back
        self.assertEqual(BlogPost.objects.filter(title="Temp Post").count(), 0)
```

✅ **What this does:**

* Tests that a manual transaction is rolled back correctly
* Cannot be done with regular `TestCase` because it wraps tests in automatic transactions

## Best Practices

* Use `setUp()` for creating test data
* Separate tests by functionality (models, views, urls)
* Prefer **factories or fixtures** over raw `.create()` for large setups
* Test both valid and invalid scenarios
* Keep each test focused and independent

### 💡 Example

```python
from django.test import TestCase
from blog.models import BlogPost

class BlogPostEdgeCaseTest(TestCase):
    def test_create_post_without_title_raises_error(self):
        with self.assertRaises(ValueError):
            BlogPost.objects.create(title=None, text='No title here')
```

---

## Summary Table

| Feature            | Description                                     |
| ------------------ | ----------------------------------------------- |
| `TestCase`         | Sets up and tears down a test DB per test class |
| `Client.get/post`  | Simulates HTTP requests without a real server   |
| `reverse()`        | Resolves view names into URLs                   |
| `assertContains()` | Checks if response includes given text          |
| `test_` prefix     | Django runs only methods starting with `test_`  |

---

✅ **In summary:**
This documentation now fully demonstrates Django testing using your **Blog models** — `Author`, `BlogPost`, and related objects — with realistic unit and integration examples you can use in your own app.
