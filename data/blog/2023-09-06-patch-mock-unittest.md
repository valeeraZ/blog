---
title: 'Use patch and mock in async unittest'
date: '2023-09-06'
lastmod: '2023-09-06'
tags: ['Testing', 'Python', 'FastAPI']
draft: false
images: ['/static/images/fastapi_testing.png']
layout: PostLayout
summary: 'Async unittest within Fast API'
---

![fastapi-unittest](/static/images/fastapi_testing.png)

Using a mock object in unittest is a common practice to test a function which depends on other functions. In this post, I will show how did I use `patch` and `mock` in unittest of endpoints of a FastAPI app, especially for async functions and dependency injection.

## Async Route

I try to test an endpoint which depends on another function. The endpoint is defined as:

```python
@router.post("", response_model=ContactReadModel, status_code=status.HTTP_201_CREATED)
async def create_contact(
    contact_create_model: ContactCreateModel,
    contact_command_service: ContactCommand = Depends(get_contact_command_service),
) -> ContactReadModel:
    """
    Create a contact.

    :param contact_create_model: the schema of the contact to create
    :param contact_command_service: the contact command service \
            from the dependency injection
    :return: a contact read model
    """
    return await contact_command_service.create_contact(contact_create_model)
```

For this function, it uses dependency injection by `get_contact_command_service` who provides an object `contact_command_service` which is an instance of `ContactCommand` class. The `create_contact` function is an async fonction which returns a `ContactReadModel` object in coroutine. To test this endpoint, I need to mock the `contact_command_service` who returns a `ContactReadModel` object.  
Since it is an async function, I also need to use `unittest.IsolatedAsyncioTestCase` to test it.

## Mock the parameters and return value

To use assert in unittest, I need to mock the parameters and return value of the function. Create a fake `ContactCreateModel` object as an input parameter in endpoint and a fake `ContactReadModel` object as an output response from the endpoint to use in assert:

```python
contact_dict_a_create = {
    "first_name": "John",
    "last_name": "Doe",
    "job": "developer",
    "address": "1234 Main St"
}

contact_dict_a_create = {
    "id": 1,
    "first_name": "John",
    "last_name": "Doe",
    "job": "developer",
    "address": "1234 Main St"
}

```

## Mock the dependency function

First, create a fake function which returns a `ContactReadModel` object as the real `create_contact` function does:

```python
async def async_create_contact(
    create_model: ContactCreateModel,
) -> ContactReadModel:
    return ContactReadModel(id=1, **create_model.model_dump())
```

The fake function should have the same signature as the real one, as it would be used to replace the real one to mock the return value.

## Create the testing class

Create a testing class which inherits from `unittest.IsolatedAsyncioTestCase`:

```python
from unittest.mock import AsyncMock, patch
from server.web.application import get_app
from httpx import AsyncClient
from server.web.api.contact import (
    ContactCreateModel,
    ContactReadModel,
    get_contact_command_service, # the function who provides the dependency
)

class TestContactsAPI(unittest.IsolatedAsyncioTestCase):
    def setUp(self) -> None:
        self.app = get_app()
```

The `setUp` function is used to initialize the `app` object, which is a FastAPI application object. The `get_app` function just returns a FastAPI application object in my case.

Then create our test function to test the endpoint:

```python
@patch("server.web.api.contact.get_contact_command_service", new_callable=AsyncMock)
async def test_create_contact(
    self,
    mock_get_contact_command_service: AsyncMock,
) -> None:
    mock_get_contact_command_service.create_contact.side_effect = (
        async_create_contact
    )
    self.app.dependency_overrides = {
        get_contact_command_service: lambda: mock_get_contact_command_service,
    }

    async with AsyncClient(app=self.app, base_url="http://test") as client:
        response = await client.post("/api/contacts", json=contact_dict_a_create)

    mock_get_contact_command_service.create_contact.assert_called_once_with(
        ContactCreateModel(**create_model),
    )
    self.assertEqual(response.status_code, 201)
    self.assertEqual(response.json(), contact_dict_a_read)
    self.app.dependency_overrides = {}
```

Let's explain the code line by line:

```python
@patch("server.web.api.contact.get_contact_command_service", new_callable=AsyncMock)
```

This line is used to mock the `get_contact_command_service` function. The `new_callable` parameter is used to specify the type of the mock object. In this case, it is an async function, so I use `AsyncMock` to create the mock object.

```python
async def test_create_contact(
    self,
    mock_get_contact_command_service: AsyncMock,
) -> None
```

The signature of test function. The `mock_get_contact_command_service` parameter is the mock object created by `@patch` decorator. The `self` parameter is the test class itself.

```python
mock_get_contact_command_service.create_contact.side_effect = (
    async_create_contact
)
```

To mock the return value of `create_contact` function. The `side_effect` parameter is used to specify the function who provides return value of the function. In this case, we bind the fake function `async_create_contact`.

```python
self.app.dependency_overrides = {
    get_contact_command_service: lambda: mock_get_contact_command_service,
}
```

To override the dependency function `get_contact_command_service` by the mock object `mock_get_contact_command_service`. In FastAPI, the `dependency_overrides` attribute of the `app` object is used to override the dependency function.

```python
async with AsyncClient(app=self.app, base_url="http://test") as client:
    response = await client.post("/api/contacts", json=contact_dict_a_create)
```

To send a request to the endpoint. The `AsyncClient` object is used to send a request to the endpoint. The `app` parameter is the FastAPI application object. The `base_url` parameter is the base url of the endpoint. The `client.post` function is used to send a POST request to the endpoint. The `json` parameter is used to specify the request body. As we have created an input dict object `contact_dict_a_create`, we can use it directly as the body to send to the endpoint.

```python
mock_get_contact_command_service.create_contact.assert_called_once_with(
    ContactCreateModel(**create_model),
)
```

The `create_contact` function is called once with the input parameter `ContactCreateModel(**create_model)`. The `assert_called_once_with` function is used to assert the function is called once with the input parameter.

```python
self.assertEqual(response.status_code, 201)
```

The status code of the response is 201.

```python
self.assertEqual(response.json(), contact_dict_a_read)
```

The response body should be the same as the `contact_dict_a_read` object.

```python
self.app.dependency_overrides = {}
```

Reset the `dependency_overrides` attribute of the `app` object to empty dict for the other tests.

## Conclusion

The mock/patch technique is very useful to test a function which depends on other functions. In other unit tests of other components, I also use this technique to mock the dependency functions. It is a good practice to test a function in isolation without initializing the whole application with database connection, etc.
