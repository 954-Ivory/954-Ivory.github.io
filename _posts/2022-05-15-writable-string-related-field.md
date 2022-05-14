---
layout: post
title: Use `WritableStringRelatedField` in `Django REST Framework`
category: Django REST Framework
tags: Django Django-REST-Framework DRF
---

## Cause:

There has a question on
StackOverflow: [How can I show the StringRelatedField instead of the Primary Key while still being able to write-to that field using Django Rest Framework?](https://stackoverflow.com/q/65248178/10219851)

I also encountered this problem during development. So I started trying to solve this problem myself.

I want to see the success solution directly. [Jump to Success Solution](#create-a-custom-field)

---

## Error direction - use `@property`:

First, I tried to add a `@property` function in the `model`, and use `SlugRelatedField` to realize.

It always useful in `SlugRelatedField(read_only=True,)`.

```python
# Models:

class RosterInstance(models.Model):
  # ...

  @property
  def representation(self):
    return self.date.strftime("%d %b, %Y")

  @representation.setter
  def representation(self, value):
    self.date = value

  def __str__(self):
    return self.representation
```

```python
# Serializers:

class RosterInstanceSerializer(serializers.ModelSerializer):
  deckhand_watchkeeper = serializers.SlugRelatedField(
    queryset=CrewMember.objects.all(),
    slug_field='representation',
    label='Deckhand Watchkeeper',
  )
  night_watchkeeper = serializers.SlugRelatedField(
    queryset=CrewMember.objects.all(),
    slug_field='representation',
    label='Night Watchkeeper',
  )

  class Meta:
    model = RosterInstance
    fields = "__all__"
```

But I got an error in:
`to_internal_value` [/rest_framework/relations.py - line 462](https://github.com/encode/django-rest-framework/blob/cdc956a96caafddcf4ecaf6218e340ebb3ce6d72/rest_framework/relations.py#L462)

```python
# /rest_framework/relations.py

def to_internal_value(self, data):
  queryset = self.get_queryset()
  try:
    return queryset.get(**{self.slug_field: data})
  except ObjectDoesNotExist:
    self.fail('does_not_exist', slug_name=self.slug_field, value=smart_str(data))
  except (TypeError, ValueError):
    self.fail('invalid')
```

Because there use `querset.get(**{self.slug_field: data})` to get the object. It needs a real `Field`,
but `self.slug_field` is a custom `property`.

The `@representation.setter` is foolish and superfluous, it's impossible be called by the function stack at all.

We got a conclusion:

**`SlugRelatedField` is exactly a writeable field, but it doesn't support read and write a not `Field` property at same
time.**

---

## What does `SlugRelatedField` do?

So, I think: Why didn't rewrite `SlugsRelatedField`?

Let's see
the `SlugsRelatedField` [rest_framework/relations.py - line 444](https://github.com/encode/django-rest-framework/blob/cdc956a96caafddcf4ecaf6218e340ebb3ce6d72/rest_framework/relations.py#L444):

- It extends from `RelatedField` and rewrite the `to_internal_value` and `to_representation` function.
- `to_internal_value`: It try to get a queryset to build relation.
- `to_representation`: It return a value what you want to display in views.

```python
# /rest_framework/relations.py

class SlugRelatedField(RelatedField):
  """
  A read-write field that represents the target of the relationship
  by a unique 'slug' attribute.
  """
  default_error_messages = {
    'does_not_exist': _('Object with {slug_name}={value} does not exist.'),
    'invalid': _('Invalid value.'),
  }

  def __init__(self, slug_field=None, **kwargs):
    assert slug_field is not None, 'The `slug_field` argument is required.'
    self.slug_field = slug_field
    super().__init__(**kwargs)

  def to_internal_value(self, data):
    queryset = self.get_queryset()
    try:
      return queryset.get(**{self.slug_field: data})
    except ObjectDoesNotExist:
      self.fail('does_not_exist', slug_name=self.slug_field, value=smart_str(data))
    except (TypeError, ValueError):
      self.fail('invalid')

  def to_representation(self, obj):
    return getattr(obj, self.slug_field)
```

---

## Create a custom `Field`

We can create a custom field to replace the `SlugsRelatedField`, I named it `WritableStringRelatedField`.

Because it's sames like a `StringRelatedField`, although it extends from `SlugRelatedField`.

Maybe you want name it `MagicSlugsRelatedField`, whatever.

Let's see code:

- It extends from `WritableStringRelatedField` and rewrite the `__init__`, `to_representation`, `slug_representation`
  , `get_choices` function.
- `__init__`:
  - I defined a `display_field` to set what property you want to display, if `None`, it will use `__str__`.
  - If you don't set the `label`, it will auto get `verbose_name_raw` by `queryset`.
  - So you can just set the `queryset`.
- `to_representation`: It combines `StringRelatedField.to_representation` and `SlugRelatedField.to_representation`, as
  judged by `display_field`.
- `slug_representation`: Sames as `SlugRelatedField.to_representation`.
- `get_choices`: Same as `SlugRelatedField.get_choices`, only the `self.slug_representation(item)` of the `return`
  changed.
- In source is `self.to_representation(item)`
  [rest_framework/relations.py - line 204](https://github.com/encode/django-rest-framework/blob/cdc956a96caafddcf4ecaf6218e340ebb3ce6d72/rest_framework/relations.py#L204)

```python
# fields.py

from collections import OrderedDict
from rest_framework import serializers


class WritableStringRelatedField(serializers.SlugRelatedField):
  def __init__(self, slug_field="pk", display_field=None, *args, **kwargs):
    self.display_field = display_field
    # Set what attribute to be represented.
    # If `None`, use `Model.__str__()` .
    if not kwargs.get("label", None):
      kwargs["label"] = kwargs["queryset"].model._meta.verbose_name_raw
    super(WritableStringRelatedField, self).__init__(slug_field, **kwargs)

  def to_representation(self, obj):
    # This function controls how to representation field.
    if self.display_field:
      return getattr(obj, self.display_field)
    return str(obj)

  def slug_representation(self, obj):
    # It will be called by `get_choices()`.
    return getattr(obj, self.slug_field)

  def get_choices(self, cutoff=None):
    queryset = self.get_queryset()
    if queryset is None:
      # Ensure that field.choices returns something sensible
      # even when accessed with a read-only field.
      return {}

    if cutoff is not None:
      queryset = queryset[:cutoff]

    return OrderedDict([
      (
        self.slug_representation(item),
        # Only this line has been overridden,
        # the others are the same as `super().get_choices()`.
        self.display_value(item)
      )
      for item in queryset
    ])
```
You can use it easily.
```python
# Serializers.py

class RosterInstanceSerializer(serializers.ModelSerializer):
  deckhand_watchkeeper = WritableStringRelatedField(
    queryset=CrewMember.objects.all(),
    slug_field='id',
    label='Deckhand Watchkeeper',
  )
  # Or only set the queryset.
  night_watchkeeper = WritableStringRelatedField(
    queryset=CrewMember.objects.all(),
  )

  class Meta:
    model = RosterInstance
    fields = "__all__"
```
