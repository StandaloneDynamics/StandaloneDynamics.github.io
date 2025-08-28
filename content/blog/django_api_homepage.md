---
title: "Homepage for your Django API"
date: "2020-11-28"
description: "Creating a very basic homepage to describe your django api"
tags: ["django", "rest-framework", "python"]
---


I created an api for a django application of mine [amalisidi](https://github.com/ebsuku/amalisidi) and I wanted to create a basic homepage for the api. Reading up on the documentation did not really give me a clear picture as to how I would go about it, so I'm documenting it here to save my future self.


First lets add the api's homepage url, in the main ```urls.py``` file

```
from rest_framework.documentation import include_docs_urls
```

then under urlpatterns

```
path("docs", include_docs_urls(title="Project API")),
```


Lets say your views.py file looks as follows:

```
class ArchiveView(APIView):
    """
	Some information about this view
	"""
```

and your api urls look something like

```
urlpatterns = [
	path("v1/archive", views.ArchiveView.as_view(), name="archive"),
]

```
So with the setup above lets look at the view.

```
import coreapi
import coreschema

from rest_framework.schemas import AutoSchema

```
Django rest framework has got an api called [Schema](https://www.django-rest-framework.org/api-guide/schemas) which is responsible for generating documentation about your api.

```Autoschema``` is used if you want to customize documenation for each view

Add ```"DEFAULT_SCHEMA_CLASS": "rest_framework.schemas.coreapi.AutoSchema"``` to your projects settings page.


[CoreApi](https://www.coreapi.org/) is a document specification for describing apis.


Lets say in our view a user can use am optional query parameter (?category=animals) and we'd like to document this.

Under the view doc string add

```
schema = AutoSchema(manual_fields=[
        coreapi.Field(
            'category',
            required=False,
            location='query',
            schema=coreschema.String(
                description='Unique category name'
            )
        ),
    ])
```
So what we are saying is: 

* There is a field called category.
* It is an optional field.
* In the url it is a query paramater.
* It has a particular description.

If our path we wanted to document was ```/api/v1/archive/<category>``` we would have the ```location=url``` and ```required=True```

Once all this has been setup ```localhost:8000/docs``` will show the documenation for the api.


