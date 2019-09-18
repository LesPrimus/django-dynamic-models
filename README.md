# django-dynamic-models


## Overview

Dynamic Django models allow users to define, edit, and populate their own database tables and apply runtime schema changes to the database. `django-dynamic-models` is loosely based on the [runtime dynamic models](https://dynamic-models.readthedocs.io/en/latest/) talk from DjangoCon 2011. The basic concept involves around dynamic class declaration using the built-in `type` function. We use `type` to dynamically declare new Django models at runtime, and it is the goal of this project to provide a simple API to allow developers to get started with runtime dynamic models quickly.

This package provides abstract models to help Django developers quickly implement dynamic runtime models for their specific use case while the runtime schema changes and Django's model registry are handled automatically. The schema changes are applied in pure Django, *without* the migrations framework, so none of your dynamic models will affect your migrations files at all.

## Installation

Install `django-dynamic-model` from PyPi with:

```
pip install django-dynamic-model
```

Then, add `'dynamic_models'` and `django.contrib.contenttypes` to `INSTALLED_APPS`.
> **Note**: 
> 
> Django's Content Types app is currently required, although this dependency may possibly removed in the future.

```python
INSTALLED_APPS = [
    ...
    'dynamic_models',
    'django.contrib.conttenttypes'
]
```

## Usage

To begin, simply subclass `AbstractModelSchema` and `AbstractFieldSchema` from `dynamic_models.models`. The abstract models will still work if no additional fields are provided. The simplest way to get started with dynamic models would be to add subclasses to your app's `models.py`, and run migrations. Then, new models can be created dynamically by creating instances of model schema and field schema models.

```python
from dynamic_models.models import AbstractModelSchema, AbstractFieldSchema

class ModelSchema(AbstractModelSchema):
    pass

class FieldSchema(AbstractFieldSchema):
    pass
```

Now, run the migration commands:
```
$ python manage.py makemigrations 
> ... making migrations ...

$ python manage.py migrate
> ... migrating ...
```

### Creating dynamic models

Creating a dynamic model is as simple as creating a new instance of your concrete model schema class. `AbstractModelSchema` provides a single field. The `name` field will be used to generate the class name and the name of the database table. Once created, the `as_model` method can be used to retreive the dynamic model class.

>**Note**:
>
>The `name` field has `unique=True` by default to help enforce unique table names generated by the `AbstracModelSchema` instance. To use a different table naming scheme, the `db_table` (and probably `model_name`) properties should be overridden. Care should be taken that it is not possible to generate the same `db_table` from different instances of `AbstractModelSchema`. 

The default model_name will be `Car` and the default table_name `myapp_car` where "myapp" is the app label. The table name and model name are derived from the `name` value.

```python
car_model_schema = ModelSchema.objects.create(name='car')
Car = car_model_schema.as_model()
assert issubclass(Car, models.Model)

# The dynamic model can now be used to create Car instances
instance = Car.objects.create()
assert instance.pk is not None
```

### Adding fields

Creating field schema to add to models is quite similar to creating dynamic models. `AbstractFieldSchema` provides two fields, `name` and `data_type`, and they are responsible for naming the database column and choosing which Django field to add to the dynamic model. Fields are not attached to any dynamic model when they are created, so the same field can be applied to any number of dynamic models. The constraints applied to the field, however, are specific for each model-field pair. Currently supported data types are returned by the `get_data_types` method on subclasses of `AbstractModelSchema`. Currently, supported data types and their corresponding Django field classes are listed below:

> **Note**: The value of `data_type` is not editable while data migrations are not supported.

| Data Type | Django Field |
|:---------:|:------------:|
| character | CharField    |
| text      | TextField    |
| integer   | IntegerField |
| float     | FloatField   |
| boolean   | BooleanField |
| date      | DateTimeField|

```python
car_model_schema = ModelSchema.objects.create(name='car')
color_field_schema = FieldSchema.objects.create(name='color', data_type='character')
```
The color field is still completely independent of the Car model, and it has not been added to any database tables. Like normal CharFields, a max_length must be defined for the character data type. Add a field to a dynamic model with the `add_field` method

```python
color = car_model_schema.add_field(
    color_field_schema,
    null=False,
    unique=False,
    max_length=16
)
```
Field constraints can be added when a field schema is attached to a model schema. Now, the new field can be used as you normally would in Django. Be sure to grab
the lastest version of the dynamic model after changing schema or `OutdatedModelError` will be raised on save.

```python
Car = car_model_schema.as_model()
red_car = Car.objects.create(color='red')
assert red_car.color == 'red'

# This will raise an error because the 'color' field is required
another_car = Car.objects.create()
```

Change the schema with 'update_field' to allow null. Null columns cannot currently be changed to not null because a default value will be required to fill the null spaces. This limitation should be removed when default values are implemented.

Existing field schema can be edited or removed with the `update_field` and `remove_field` methods.

```python
car_model_schema.update_field(color_field_schema, null=True)
car_model_schema.remove_field(color_field_schema)
```

## Support
`django-dynamic-models` is tested on Python 3.6+ with Django 2.0+

<hr>

# Reference

## AbstractModelSchema

### Fields

#### `name`

The value of name deterimines the model name and table name of the dynamic model. It has the `unique` constraint applied by default, but can be removed if a custom naming scheme has been implemented to prevent duplicate table and model names.
<hr>

### Methods

#### `as_model()`

Return the dynamic model generated by the model schema instance.
<hr>

#### `get_fields()`

Return all `ModelFieldSchema` instances attached to this model.
<hr>

#### `get_field_for_schema(field_schema)`

Return the `ModelFieldSchema` instance for this model schema and the provided field schema arugument.
<hr>

#### `add_field(field_schema, **options)`

Add a field to this dynamic model by creating a new instance of `ModelFieldSchema`. Takes the field schema instance as the first argument and optionally any constraints applied to this field. Valid constraints are `null`, `unique`, and `max_length` if the field's data type requires.
<hr>

#### `update_field(field_schema, **options)`

Update an existing field with new contraints. Currently, fields cannot be changed from `null=True` to `null=False`.

<hr>

#### `remove_field(field_schema)`

Remove a field from the dynamic model.
<hr>

#### `is_current_model(model)`

Returns `True` if the provided model is current with the latest schema and `False` if the schema has since changed from when the model was generated. This does not work on unsaved schema instances.
<hr>

#### `destroy_model()`

Purges the dynamic model from the app registry and the database without deleting the schema instance. There are few use cases for calling this method manually.
<hr>


## AbstractFieldSchema

### Fields

#### `name`

The name of the field. Used to generate the database column name. Names should not start with an underscore to avoid potentially clashing with private attributes on the generated models.
<hr>

#### `data_type`

Used to determine the Django model field to used on the dynamic model. See above for a list of valid data types.
<hr>

### Methods

#### `get_data_types()`

Return a list of available data type options.
<hr>

#### `update_last_modified()`

Marks all related model schema objects as changed, forcing the dynamic model to be redefined the next time `as_model` is called.
