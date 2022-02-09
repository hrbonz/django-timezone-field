# django-timezone-field

[![CI](https://github.com/mfogel/django-timezone-field/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/mfogel/django-timezone-field/actions)
[![codecov](https://codecov.io/gh/mfogel/django-timezone-field/branch/main/graph/badge.svg?token=Rwekzmim3l)](https://codecov.io/gh/mfogel/django-timezone-field)
[![pypi downloads](https://img.shields.io/pypi/dm/django-timezone-field.svg)](https://pypi.python.org/pypi/django-timezone-field/)
[![pypi python support](https://img.shields.io/pypi/pyversions/django-timezone-field.svg)](https://pypi.python.org/pypi/django-timezone-field/)
[![pypi django support](https://img.shields.io/pypi/djversions/django-timezone-field.svg)](https://pypi.python.org/pypi/django-timezone-field/)

A Django app providing DB, form, and REST framework fields for
[`zoneinfo`](https://docs.python.org/3/library/zoneinfo.html) and [`pytz`](http://pypi.python.org/pypi/pytz/) timezone
objects.

## The transition from `pytz` to `zoneinfo`

Like Django, this app supports both `pytz` and `zoneinfo` objects while the community transitions away from `pytz` to
`zoneinfo`. All exposed fields and functions that return a timezone object accept an optional boolean kwarg `use_pytz`.

If not explicitly specified, the default value used for `use_pytz` matches Django's behavior:

- Django <= 3.X: `use_pytz` defaults to `True`
- Django == 4.X: `use_pytz` defaults to the value of
  [`django.conf.settings.USE_DEPRECATED_PYTZ`](https://docs.djangoproject.com/en/4.0/ref/settings/#use-deprecated-pytz),
  which itself defaults to `False`
- Django >= 5.X: django plans to
  [drop support for `pytz` altogether](https://docs.djangoproject.com/en/4.0/releases/4.0/#zoneinfo-default-timezone-implementation),
  and this app will likely do the same.

### Differences in recognized timezones between `pytz` and `zoneinfo`

`pytz` and `zoneinfo` search for timezone data differently.

- `pytz` bundles and searches within its own copy of the [IANA timezone DB](https://www.iana.org/time-zones)
- `zoneinfo` first searches the local system's timezone DB for a match. If no match is found, it then searches within
  the [`tzdata`](https://pypi.org/project/tzdata/) package if it is installed. The `tzdata` package contains a copy of
  the IANA timezone DB.

If the local system's timezone DB doesn't cover the entire IANA timezone DB and the `tzdata` package is not installed,
you may run across errors like `ZoneInfoNotFoundError: 'No time zone found with key Pacific/Kanton'` for seemingly valid
timezones when transitioning from `pytz` to `zoneinfo`. The easy fix is to add `tzdata` to your project with
`poetry add tzdata` or `pip install tzdata`.

Assuming you have the `tzdata` package installed if needed, no
[data migration](https://docs.djangoproject.com/en/4.0/topics/migrations/#data-migrations) should be necessary when
switching from `pytz` to `zoneinfo`.

## Examples

### Database Field

```python
import zoneinfo
import pytz
from django.db import models
from timezone_field import TimeZoneField

class MyModel(models.Model):
    tz1 = TimeZoneField(default="Asia/Dubai")               # defaults supported, in ModelForm renders like "Asia/Dubai"
    tz2 = TimeZoneField(choices_display="WITH_GMT_OFFSET")  # in ModelForm renders like "GMT+04:00 Asia/Dubai"
    tz3 = TimeZoneField(use_pytz=True)                      # returns pytz timezone objects
    tz4 = TimeZoneField(use_pytz=False)                     # returns zoneinfo objects

my_model = MyModel(
    tz2="America/Vancouver",                     # assignment of a string
    tz3=pytz.timezone("America/Vancouver"),      # assignment of a pytz timezone
    tz4=zoneinfo.ZoneInfo("America/Vancouver"),  # assignment of a zoneinfo
)
my_model.full_clean() # validates against pytz.common_timezones by default
my_model.save()       # values stored in DB as strings
my_model.tz3          # value returned as pytz timezone: <DstTzInfo 'America/Vancouver' LMT-1 day, 15:48:00 STD>
my_model.tz4          # value returned as zoneinfo: zoneinfo.ZoneInfo(key='America/Vancouver')
```

### Form Field

```python
from django import forms
from timezone_field import TimeZoneFormField

class MyForm(forms.Form):
    tz1 = TimeZoneFormField()                                   # renders like "Asia/Dubai"
    tz2 = TimeZoneFormField(choices_display="WITH_GMT_OFFSET")  # renders like "GMT+04:00 Asia/Dubai"
    tz3 = TimeZoneFormField(use_pytz=True)                      # returns pytz timezone objects
    tz4 = TimeZoneFormField(use_pytz=False)                     # returns zoneinfo objects

my_form = MyForm({"tz3": "Europe/Berlin", "tz4": "Europe/Berlin"})
my_form.full_clean()         # validates against pytz.common_timezones by default
my_form.cleaned_data["tz3"]  # value returned as pytz timezone: <DstTzInfo 'Europe/Berlin' LMT+0:53:00 STD>
my_form.cleaned_data["tz4"]  # value returned as zoneinfo: zoneinfo.ZoneInfo(key='Europe/Berlin')
```

### REST Framework Serializer Field

```python
from rest_framework import serializers
from timezone_field.rest_framework import TimeZoneSerializerField

class MySerializer(serializers.Serializer):
    tz1 = TimeZoneSerializerField(use_pytz=True)
    tz2 = TimeZoneSerializerField(use_pytz=False)

my_serializer = MySerializer(data={
    "tz1": "America/Argentina/Buenos_Aires",
    "tz2": "America/Argentina/Buenos_Aires",
})
my_serializer.is_valid()
my_serializer.validated_data["tz1"]  # <DstTzInfo 'America/Argentina/Buenos_Aires' LMT-1 day, 20:06:00 STD>
my_serializer.validated_data["tz2"]  # zoneinfo.ZoneInfo(key='America/Argentina/Buenos_Aires')
```

## Installation

Releases are hosted on [`pypi`](https://pypi.org/project/django-timezone-field/) and can be installed using various
python packaging tools.

```bash
# with poetry
poetry add django-timezone-field

# with pip
pip install django-timezone-field
```

## Running the tests

From the repository root, with [`poetry`](https://python-poetry.org/):

```bash
poetry install
poetry run pytest
```

## Changelog

#### 5.0 (2022-02-08)

- Add support for `zoneinfo` objects ([#79](https://github.com/mfogel/django-timezone-field/issues/79))
- Add support for django 4.0
- Remove `display_GMT_offset` kwarg (use `choices_display` instead)
- Drop support for django 3.0, 3.1
- Drop support for python 3.5, 3.6

#### 4.2.3 (2022-01-13)

- Fix sdist installs ([#78](https://github.com/mfogel/django-timezone-field/issues/78))
- Officially support python 3.10

#### 4.2.1 (2021-07-07)

- Reinstate `TimeZoneField.default_choices` ([#76](https://github.com/mfogel/django-timezone-field/issues/76))

#### 4.2 (2021-07-07)

- Officially support django 3.2, python 3.9
- Fix bug with field deconstruction ([#74](https://github.com/mfogel/django-timezone-field/issues/74))
- Housekeeping: use poetry, github actions, pytest

#### 4.1.2 (2021-03-17)

- Avoid `NonExistentTimeError` during DST transition ([#70](https://github.com/mfogel/django-timezone-field/issues/70))

#### 4.1.1 (2020-11-28)

- Don't import `rest_framework` from package root ([#67](https://github.com/mfogel/django-timezone-field/issues/67))

#### 4.1 (2020-11-28)

- Add Django REST Framework serializer field
- Add new `choices_display` kwarg with supported values `WITH_GMT_OFFSET` and `STANDARD`
- Deprecate `display_GMT_offset` kwarg

#### 4.0 (2019-12-03)

- Add support for django 3.0, python 3.8
- Drop support for django 1.11, 2.0, 2.1, python 2.7, 3.4

#### 3.1 (2019-10-02)

- Officially support django 2.2 (already worked)
- Add option to display TZ offsets in form field ([#46](https://github.com/mfogel/django-timezone-field/issues/46))

#### 3.0 (2018-09-15)

- Support django 1.11, 2.0, 2.1
- Add support for python 3.7
- Change default human-readable timezone names to exclude underscores
  ([#32](https://github.com/mfogel/django-timezone-field/issues/32) &
  [#37](https://github.com/mfogel/django-timezone-field/issues/37))

#### 2.1 (2018-03-01)

- Add support for django 1.10, 1.11
- Add support for python 3.6
- Add wheel support
- Support bytes in DB fields ([#38](https://github.com/mfogel/django-timezone-field/issues/38) &
  [#39](https://github.com/mfogel/django-timezone-field/issues/39))

#### 2.0 (2016-01-31)

- Drop support for django 1.7, add support for django 1.9
- Drop support for python 3.2, 3.3, add support for python 3.5
- Remove tests from source distribution

#### 1.3 (2015-10-12)

- Drop support for django 1.6, add support for django 1.8
- Various [bug fixes](https://github.com/mfogel/django-timezone-field/issues?q=milestone%3A1.3)

#### 1.2 (2015-02-05)

- For form field, changed default list of accepted timezones from `pytz.all_timezones` to `pytz.common_timezones`, to
  match DB field behavior.

#### 1.1 (2014-10-05)

- Django 1.7 compatibility
- Added support for formatting `choices` kwarg as `[[<str>, <str>], ...]`, in addition to previous format of
  `[[<pytz.timezone>, <str>], ...]`.
- Changed default list of accepted timezones from `pytz.all_timezones` to `pytz.common_timezones`. If you have timezones
  in your DB that are in `pytz.all_timezones` but not in `pytz.common_timezones`, this is a backward-incompatible
  change. Old behavior can be restored by specifying `choices=[(tz, tz) for tz in pytz.all_timezones]` in your model
  definition.

#### 1.0 (2013-08-04)

- Initial release as `timezone_field`.

## Credits

Originally adapted from [Brian Rosner's django-timezones](https://github.com/brosner/django-timezones).

Made possible thanks to the work of the
[contributors](https://github.com/mfogel/django-timezone-field/graphs/contributors).
