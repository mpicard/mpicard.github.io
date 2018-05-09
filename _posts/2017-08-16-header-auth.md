---
layout: post
title: Header based pre-authentication with Django REST Framework
category: Tutorial
tags:
 - Python
 - Django Rest Framework
 - API
---

As the title suggests, this is a short How-To on using header based pre-authentication with Django REST Framework (DRF). I'm currently working in a system that does authentication through a gateway and forwards requests to corresponding private APIs with a custom HTTP authentication header. This scenario is quite simple to add to DRF's pluggable authentication. You can find the entire project [source code here](https://github.com/mpicard/blog-header-auth-example).

Starting with a basic DRF project, change the DEFAULT_AUTHENTICATION_CLASSES to the path of your class name.

```python
# config/settings.py

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'config.authentication.HeaderAuthentication'
    ]
}
```

In that class we can do the following, using a CustomUser object since my backend isn't interested in using `django.contrib.auth` in this case.

```python
# config/authentication.py

from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed


class CustomUser:

    def __init__(self, uid):
        self.uid = uid

    def __repr__(self):
        return f'<PreAuthUser {self.uid}>'

    def __str__(self):
        return self.__repr__()


class HeaderAuthentication(BaseAuthentication):

    def authenticate(self, request):
        # META dict has custom headers as HTTP_{key} and capitalizes it.
        uid = request.META.get('HTTP_UID')

        if not uid:
            raise AuthenticationFailed('Missing authentication header')

        user = CustomUser(uid)

        return (user, None)
```

If we wanted to use the default Django auth user model, we could do something like so.

```python
# config/authentication.py
from django.contrib.auth.models import User
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed


class HeaderAuthentication(BaseAuthentication):

    def authenticate(self, request):
        # META dict has custom headers as HTTP_{key} and capitalizes it.
        uid = request.META.get('HTTP_UID')

        if not uid:
            raise AuthenticationFailed('Missing authentication header')

        try:
            user = User.objects.get(pk=uid)
        except User.DoesNotExist:
            raise AuthenticationFailed('Invalid user id')

        return (user, None)
```

Even better, if we have a user microservice. Define a custom user class to populate the fields you wish the request.user object to have. Remove `django.contrib.auth` from your `INSTALLED_APPS` if this is the case.

```python
# config/authentication.py
from django.contrib.auth.models import User
from rest_framework.authentication import BaseAuthentication
from rest_framework.exceptions import AuthenticationFailed


class CustomUser:

    def __init__(self, uid, name, email, **kwargs):
        self.id = uid
        self.uid = uid
        self.name = name
        self.email = email

class HeaderAuthentication(BaseAuthentication):

    def authenticate(self, request):
        # META dict has custom headers as HTTP_{key} and capitalizes it.
        uid = request.META.get('HTTP_UID')

        if not uid:
            raise AuthenticationFailed('Missing authentication header')

        r = requests.get(f'http://users.mydomain.com/{uid}')

        try:
            r.raise_for_status()
        except requests.exceptions.RequestException
            raise AuthenticationFailed(
                f'User GET failed ({r.status_code}: {r.reason})')

        user = CustomUser(**r.json())

        return (user, None)
```

Obviously the approaches above a naive so please use them as a template to get started and add more robust error handling based on your requirements. Happy coding.

NOTE: I'm using [PEP 498](https://www.python.org/dev/peps/pep-0498/) f-string interpolation, if you aren't using 3.6+ yet, just replace with `format()` or `% s`
