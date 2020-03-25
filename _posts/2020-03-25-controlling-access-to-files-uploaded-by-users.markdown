---
layout: post
title: "Controlling access to files uploaded by users"
date: 2020-03-25 20:46:00 +0100
categories: web
tags: web http nginx django-rest-framework
permalink: controlling-access-to-files-uploaded-by-users
---

Imagine a situation where you have to check whether or not a user that sent the request can access or download files that were uploaded by another user. Perhaps user A uploaded a file that needs to be shared only with user B or only with authenticated users. If your application is deployed behind a [reverse-proxy](https://en.wikipedia.org/wiki/Reverse_proxy) such as [`nginx`](https://www.nginx.com/resources/wiki/), you can use the best of both worlds: your application for checking the user's permissions and the Web server for serving the files the application tells it to serve.

Before we begin, there are a couple of things I would like to address. First of all, having your Web application serve the media files by loading it into memory and sending it in a response is [grossly inefficient](https://docs.djangoproject.com/en/dev/howto/static-files/#serving-static-files-during-development). You may not know the size of the file, or there could be many requests happening at once. Whatever the case may be, there is a better way.

## `X-Accel`

To quote the [official documentation](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/):

> X-accel allows for internal redirection to a location determined by a header returned from a backend.

> This allows you to handle authentication, logging or whatever else you please in your backend and then have NGINX handle serving the contents from redirected location to the end user, thus freeing up the backend to handle other requests. This feature is commonly known as [`X-Sendfile`](https://www.nginx.com/resources/wiki/start/topics/examples/xsendfile/).

To achieve this, at least two things have to be implemented:

1. The application's response must contain the [`X-Accel-Redirect`](#x-accel-redirect-header) header;
2. The location should be marked as [`internal;`](#internal) to prevent direct access to the URI.

### `X-Accel-Redirect` header

This header tells `nginx` which URI to serve. Although the following example uses [`django-rest-framework`](https://www.django-rest-framework.org/), the same thing can be achieved with any other Web framework.

If we assume all files uploaded by users are located in the `/home/user/repo/media/` directory (also defined in Django's [`MEDIA_ROOT`](https://docs.djangoproject.com/en/dev/ref/settings/#media-root) setting), or more precisely, the `/home/user/repo/media/files/{user.id}/` directory by the `FileField`'s [`upload_to`](https://docs.djangoproject.com/en/dev/ref/models/fields/#django.db.models.FileField.upload_to) function, the view looks something like this:

{%- highlight python -%}
from pathlib import Path

from django.conf import settings
from django.http import HttpResponseRedirect

from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.viewsets import ModelViewSet

from .models import File
from .permissions import CanAccessFile

class FileViewSet(ModelViewSet):
    permission_classes = [CanAccessFile]
    queryset = File.objects.all()

    @action(detail=True, methods=["get"])
    def download(self, request, pk=None):
        obj = self.get_object()
        if settings.DEBUG:
            return HttpResponseRedirect(obj.upload.url)

        file_name = Path(obj.upload.path).name
        headers = {
            "Content-Disposition": f"attachment; filename={file_name}",
            "X-Accel-Redirect": (
                f"/uploads/files/{obj.user_id}/{file_name}"
            ),
        }
        return Response(data=b"", headers=headers)

{%- endhighlight -%}

Note that the `settings.DEBUG` block is here so developers can keep using [Django's `static` mechanism for serving media files during development](https://docs.djangoproject.com/en/dev/howto/static-files/#serving-files-uploaded-by-a-user-during-development).

There are also [other `X-Accel-*` headers](https://www.nginx.com/resources/wiki/start/topics/examples/x-accel/#special-headers) that can be set by the application to further refine the process.

### `internal`

The application's response that contains the `X-Accel-Redirect` header is picked up by the Web server on its way back to the client. In order for `nginx` to locate the file that should be sent to the client, the configuration should look something like this:

{%- highlight nginx -%}
server {
    server_name example.com;

    location /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/user/repo;
    }

    location /uploads/ {
        internal;
        alias /home/user/repo/media/;
    }

    location / {
        include /etc/nginx/proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
{%- endhighlight -%}

With that all set, you're ready to start serving files to select users!
