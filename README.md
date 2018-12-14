# real-world-graphql-with-django
Real world GraphQL patterns with Django

1. [Querying](#querying)
2. [Customizing and Optimizing Queries](#customizing-and-optimizing-queries)
3. [Mutations](#mutations)
4. [Better Mutations](#better-mutations)
5. [Custom Field type](#custom-Field-types)


# Querying

All `id`s are serialized as a global id which are base 64 encoded `ModelName:PrimaryKey`

For all Django models:

* Define a `DjangoObjectType`
* Use relay (for better resolve relationships)
* `DjangoListField` for connecting models without pagination
* resolve_FIELD_NAME receives request as `info.context` and optionally defined keyword arguments

When defining the `Query` object:

* `DjangoFilterConnectionField` generates a resolving field with arguments from `filter_fields`
* [graphene-django-extras](https://github.com/eamigo86/graphene-django-extras) provides alternatives & other missing batteries like `DjangoObjectField` for single object lookup


```python
from graphene import relay
from graphene_django import DjangoObjectType
from graphene_django.filter import DjangoFilterConnectionField
from graphene_django.fields import DjangoConnectionField, DjangoListField


class GroupNode(DjangoObjectType):
    class Meta:
        model = Group
        interfaces = (relay.Node, )


class UserNode(DjangoObjectType):
    groups = DjangoListField(GroupNode)

    class Meta:
        model = User
        filter_fields = ['username', 'email']
        exclude_fields = ['password']
        interfaces = (relay.Node, )


class Query(ObjectType):
    auth_self = Field(UserNode)
    user = DjangoObjectField(UserNode)
    users = DjangoFilterConnectionField(UserNode)

    def resolve_auth_self(self, info, **kwargs):
        user = info.context.user
        if user.is_anonymous:
            return None
        return user
```

# Customizing and Optimizing Queries

Define a custom `Field` to be used by our Query

* Replaces `DjangoFilterConnectionField` in our Query object, do not use inside `DjangoObjectType`
* [graphene-django-optimizer](https://github.com/tfoxy/graphene-django-optimizer) to optimize queries across relationships
* `Field.get_resolver` returns a callable which later receives a request as `info.context`
* call custom auth_check inside resolver
* Repeat for `DjangoObjectField` for securing single object lookups


```python
from graphene import Field
import graphene_django_optimizer as gql_optimizer
from graphene_django.filter.utils import (
    get_filtering_args_from_filterset,
    get_filterset_class
)
from functools import partial

#https://github.com/graphql-python/graphene-django/issues/206
class DjangoFilterField(Field):
    '''
    Custom field to use django-filter with graphene object types (without relay).
    '''

    def __init__(self, _type, fields=None, extra_filter_meta=None,
                 filterset_class=None, *args, **kwargs):
        _fields = _type._meta.filter_fields
        _model = _type._meta.model
        self.of_type = _type
        self.fields = fields or _fields
        meta = dict(model=_model, fields=self.fields)
        if extra_filter_meta:
            meta.update(extra_filter_meta)
        self.filterset_class = get_filterset_class(filterset_class, **meta)
        self.filtering_args = get_filtering_args_from_filterset(
            self.filterset_class, _type)
        kwargs.setdefault('args', {})
        kwargs['args'].update(self.filtering_args)
        super().__init__(List(_type), *args, **kwargs)

    @staticmethod
    def list_resolver(manager, filterset_class, filtering_args, root, info, *args, **kwargs):
        auth_check(info.context)
        filter_kwargs = {k: v for k,
                         v in kwargs.items() if k in filtering_args}
        qs = manager.get_queryset()
        qs = filterset_class(data=filter_kwargs, queryset=qs).qs
        return qs

    def get_resolver(self, parent_resolver):
        return partial(self.list_resolver, self.of_type._meta.model._default_manager,
                       self.filterset_class, self.filtering_args)
```


# Mutations

While `id`s from query are global, by default, they are not handled by mutations. https://github.com/graphql-python/graphene-django/issues/460

Three ways to define a mutation:

* Manually with a custom `mutate` method
* Wrapping a Django Rest Framework Serializer
* Wrapping a (model) form with `DjangoModelFormMutation`

IMHO defining mutations based on Django Forms has struck a good balance between being DRY and having too many abstractions.
Generally most of the customizing at the view level can go into two methods: `get_form_kwargs` and `perform_mutate`.

```python
from django.contrib.auth.forms import AuthenticationForm
from django.contrib.auth import login
from graphene_django.forms.mutation import DjangoModelFormMutation


class AuthenticationMutation(DjangoModelFormMutation):
    class Meta:
        form_class = AuthenticationForm

    authUser = Field(UserNode)

    @classmethod
    def get_form_kwargs(cls, root, info, **input):
        kwargs = {"data": input}
        kwargs['request'] = info.context
        return kwargs

    @classmethod
    def perform_mutate(cls, form, info):
        obj = form.get_user()
        if not form.request.COOKIES:
            assert False, 'Cookies are required'
            #TODO return auth token
        else:
            login(form.request, obj)
        return cls(errors=[], authUser=obj)
```

# Better Mutations

We'll extend the `DjangoModelFormMutation` class to do the following:

* generate a model form if none is provided
* support partial updates
* translate global ids
* perform auth check

By default `DjangoModelFormMutation` will create an object if no `id` is provided, we'll want to keep that behavior.

```python
from graphene_django.forms.mutation import DjangoModelFormMutation
from graphql_relay import from_global_id
from django.forms.models import modelform_factory
from django.forms import ModelChoiceField
from stringcase import camelcase
from .schema_tools import auth_check


class DjangoModelMutation(DjangoModelFormMutation):
    '''
    Automatically generates a model form that supports partial updates

    Works just like the regular DjangoModelFormMutation but may also specify the following in Meta:

        only_fields
        exclude_fields
    '''
    class Meta:
        abstract = True

    @classmethod
    def __init_subclass_with_meta__(
        cls,
        **options
    ):
        if 'model' in options and 'form_class' not in options:
            options['form_class'] = form_class = modelform_factory(options['model'],
                fields=options.get('only_fields'),
                exclude=options.get('exclude_fields', [])
            )
            for field in form_class.base_fields.values():
                field.required = False
        super(DjangoModelMutation, cls).__init_subclass_with_meta__(
            **options
        )

    @classmethod
    def get_form(cls, root, info, **input):
        auth_check(info)
        # convert global ids to database ids
        for fname, field in cls._meta.form_class.base_fields.items():
            if isinstance(field, ModelChoiceField) and input.get(fname):
                #TODO assert models match
                input[fname] = from_global_id(input[fname])[1]
        if 'id' in input:
            input['id'] = from_global_id(input['id'])[1]
        form_kwargs = cls.get_form_kwargs(root, info, **input)
        form = cls._meta.form_class(**form_kwargs)
        if 'id' in input:
            for fname, field in list(form.fields.items()):
                #info.variable_values is the raw dictionary of values supplied by the client
                if not field.required and camelcase(fname) not in info.variable_values:
                    del form.fields[fname]
            assert len(form.fields)
        return form
```


# Custom Field types

For each new type, extend an existing type and do the following:

* Define `serialize` & `deserialize` static methods
* The class name will become the type name in GraphQL
* Call `convert_django_field.register` to convert model fields
* Call `convert_form_field.register` to convert form fields

```python
from graphene_django.converter import convert_django_field
from graphene_django.forms.converter import convert_form_field
from graphene.types.generic import GenericScalar
from django.contrib.gis.geos import GEOSGeometry
from django.contrib.gis.db import models
from django.contrib.gis.forms import fields
import json


class GeoJSON(GenericScalar):
    @staticmethod
    def geos_to_json(value):
        return json.loads(GEOSGeometry(value).geojson)

    @staticmethod
    def json_to_geos(value):
        return GEOSGeometry(value)

    serialize = geos_to_json
    deserialize = json_to_geos


@convert_django_field.register(models.PolygonField)
@convert_django_field.register(models.MultiPolygonField)
def convert_geofield_to_string(field, registry=None):
     return GeoJSON(description=field.help_text, required=not field.null)


@convert_form_field.register(fields.PolygonField)
@convert_form_field.register(fields.MultiPolygonField)
def convert_geofield_to_string(field, registry=None):
     return GeoJSON(description=field.help_text, required=field.required)
```
