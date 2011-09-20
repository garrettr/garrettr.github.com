Although I'm sure I'll run into a few little snags that need handling, the code for the new [social media search function] on the Alliance for Appalachia website is pretty much done. One little thing that I just finished up was copying profile images from Facebook and Twitter pages to associate with each feed. Accessing the images themselves was a piece of cake, thanks to the Facebook and Twitter API's, but programatically associating them with a Django ImageField and saving them was surprisingly tricky!

I'm not the only one - there are numerous [Stack Overflow threads](http://stackoverflow.com/questions/1308386/programmatically-saving-image-to-django-imagefield),
[Django snippets](http://djangosnippets.org/snippets/1100/), and [blog posts](http://www.nitinh.com/2009/02/django-example-filefield-and-imagefield/) on this topic. I wanted to document, and explain the rationale behind, a simple version that works with the latest Django 1.3 release.

Ultimately the solution is very simple - it's just tricky to get everything working right all at once.
Here's the code:

{% highlight python linenos %}
# ...
from django.db import models
import urllib2, urlparse
from django.core.files.base import ContentFile
# ...

class YourModel(models.Model):
    # ...
    image = models.ImageField(upload_to="yourmodel/")
    # ...

    def save(self, *args, **kwargs):
        '''
        Retrieve an image from some url and save it to the image field,
        before saving the model to the database
        '''
        image_url = "http://.../image.jpg" # get this url from somewhere
        image_data = urllib2.urlopen(image_url)
        filename = urlparse.urlparse(image_data.geturl()).path.split('/')[-1]
        self.image = filename
        self.image.save(
            filename,
            ContentFile(image_data.read()),
            save=False
        )
        super(YourModel, self).save(*args, **kwargs)

{% endhighlight %}

That's it! The imports at the top will become apparent when you read the code, and the ImageField
definition is the bare minimum. What's important is how you override the model's `save()` method.

Given `image_url`, we use the standard library `urllib2` to open the url on line 18. Rather than just call
`read()` on it and immediately grab the data, I keep the returned object from urllib2 so I can use the
`geturl()` method seen on the next line. This is useful becuase it returns the URL you ended up
retrieving data from, [after redirects](http://docs.python.org/library/urllib2.html#urllib2.urlopen).
Redirects are an especially common occurence when working with RESTful API's like Facebook and Twitter's, and
it's nice to be able to get the original filename.

In the next line, we use urlparse to split the url on forward slashes, and then use array slice
notation with a negative index to get the *last* component of the url. After the redirect, this will
be the filename. Given a redirect to something like `http://facebookcdn.com/awkwardprofilepic.jpg`, this gives us `awkwardprofilepic.jpg`. There's no stopping you from picking any filename you want, on the other hand, so only use this if it makes sense for what you're doing.

Ok, now we set the model's `image` field. As the [Django ImageField docs](https://docs.djangoproject.com/en/dev/ref/models/fields/#imagefield) explain, Django ImageFields are really just strings - paths relative to your project's `MEDIA_ROOT`, to be specific. The ImageField's save() function actually builds this path from a) the filename, and b) the path you defined in the `upload_to=` keyword argument. So the way this works is we

1. Set the ImageField's filename, which it will concatenate with upload_to to create the final path,
   and
2. Use the ImageField's built-in save method to save the image data we retrieved from our URL in an
   image file at this location.

Therefore, the first step is just `self.image = filename`. Now we call the ImageField's save() method,
which is [documented here](https://docs.djangoproject.com/en/1.2/ref/files/file/#additional-methods-on-files-attached-to-objects). This actually saves a Django File object, which is a wrapper over Python's built-in file object.

This was a tricky part - it requires a file-like object, but apparently the file-like object returned by `urllib2.urlopen`. This actually saves a Django File object, which is a wrapper over Python's built-in file object. This was a tricky part - it requires a file-like object, but apparently the file-like object returned by `urllib2.urlopen` isn't good enough for it. However, `urllib2.urlopen.read()` return's the file's contents as a string, and Django provides a simple File wrapper class, [ContentFile](https://docs.djangoproject.com/en/1.2/ref/files/file/#the-contentfile-class), that presents strings as file-like objects. And that is exactly what happens in the first two lines of `self.image.save`. The last parameter, `save=False`, is also very important. If `save=True` (which it defaults to), then `self.image.save` will try to call the model's save() method - which will call itself infinitely and save hundreds of copies of the save image to your disk before you figure out what's wrong. To fix this problem, just set `save=False` and then make sure to call the next line at some point afterwards so the model is saved to the database.

A note: A common attribute of many of the solutions I've seen around the web is the use of django.temp.NamedTemporaryFile. Note that if you *do* want to use NamedTemporaryFile, don't import it from django.temp - [this ticket](https://code.djangoproject.com/ticket/16569) explains that it's not intended to be part of the public-facing API. However, its functionality is easily accessible in [tempfile](http://docs.python.org/library/tempfile.html), part of the Python Standard Library.

[social media search function]: http://appalliance.webfactional.com/feeds/search
