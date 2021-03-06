Creating a Facebook-like URL submission tool with Ext JS
========================================================

[This tutorial was adapted from a `post`_ on Embedly's `blog`_.]

One of our clients is currently using Ext and Embedly to implement a
Facebook-like link submission tool. While writing up some examples we went
overboard and ended up coding the whole thing. It's worth sharing with the
community as a whole, because we have seen this question many times before.
This post ended up being a lot longer than expected, some 4,000+ words
describing the process and design decisions that go into letting users submit
links. Herein lies the bible of using Embedly to create a URL submission tool
with `Ext Core <http://www.sencha.com/products/extcore/>`_.

First off, note that I am not an Ext expert. In fact this is the first I have
ever used the tool. While I disagree with its obsession with ids, I like how
Ext creates HTML and the ``fly`` method. If you see anything that is off or
just looks wrong, blame ignorance.

The final demo is hosted at `embedly.github.com/embedly-tutorial-ext
<http://embedly.github.com/embedly-tutorial-ext/>`_. You can submit a URL and
see how all the interactions work. The demo is optimized for modern browsers
and relies heavily on ``localStorage`` to display the Feed, so older browsers
will fail hard. The source code is available at
`github.com/embedly/embedly-tutorial-ext
<https://github.com/embedly/embedly-tutorial-ext>`_. We encourage you to clone
it and follow along.

Overview
--------

What we are trying to build here is a fairly common design pattern that allows
users to preview the metadata for a link before submitting it to their feed. If
you have used Facebook, Google+, Yammer or any number of services you have seen
this before. Below are a few screenshots from various different services.

Facebook

.. image:: ../images/ext_facebook.png

Yammer (Uses Embedly)

.. image:: ../images/ext_yammer.png

Google+

.. image:: ../images/ext_googleplus.png

The interface for all three are fairly similar. Each has a textarea for the
status and when a user enters a URL, they all recognize it and fetch the
metadata associated with the URL. The user can then select a thumbnail and
edit the title and description. The question is why? Well it turns out that
humans are much better at picking thumbnails, titles and descriptions than
algorithms will ever be. It's worth delving into an example and showing where
all the metadata comes from. We will use the `10k Apart
<http://10k.aneventapart.com/>`_ website as an example and this is a screenshot
of the homepage at the time of writing.

.. image:: ../images/ext_10kapart.png

Let's look at the metadata in the Facebook, Yammer, Google+ comparison from
above. The title is the same for each, but the first image and description
are different. Why is that? Each service uses a different set of algorithms to
pull data out of the page.  The `10k Apart <http://10k.aneventapart.com/>`_
site doesn't specify any ``image_src``, ``og:image``, ``decription`` or
``og:description`` tags in the head of the html so it's everyman for themselves
. Pulling the best image or a great excerpt is hard and when we let code handle
it there is bound to be variance and incorrect data.

When it comes down to it though, it should be up to the user. Any of the three
images above could work, but at the end of the day a user, not code, will
always choose the best image. So the task is simple, create a form that allows
a user to input a URL and select the thumbnail, title and description they wish
to show. There are however a bunch of other considerations that you will
invariable end up coming across that we will look at as well.

Preview Endpoint
----------------

One of Embedly's API endpoints was specifically made for this task. The
`preview endpoint <http://embed.ly/docs/endpoints/1/preview>`_ expands on
oEmbed and passes back an array of images, content, place information, event
details as well as the normal oEmbed html data. It holds a lot of information,
but we are only going to use the following attributes. I'm going to go into
detail here because I think it's import that you know where the data is coming
from and how to use it.

type

    The type of the response that we got back from the server. 90% of the time
    you will get back a ``type`` of ``html``, but another very common one is
    ``image``. There is a whole list of `response types
    <http://embed.ly/docs/endpoints/response#response-types>`_, but generally
    most people code for ``html`` and ``image`` and give up on the other types.
    Later we will go through what that actually means.

title

    Title of the page. Embedly looks for a title in the API response, the Open
    Graph title tag or the ``<title>`` tag in the head of the doc in that order
    . For 10k.aneventapart.com we found the title tag:

    .. code-block:: html

        <title>10K Apart - An Event Apart + Mix Online</title>


description

    A description or brief excerpt from the page. Embedly looks for a title in
    the API response, the Open Graph description, the
    ``<meta name="description">`` tag in the head of the doc in that order. If
    none of those exist, or are too short, Embedly will use an algorithm to try
    to find a good excerpt from the page. It does this by looking for common
    clues like a string of p tags, divs with lots of text at a similar depths
    and a whole slue of other factors.

    Going back to the Facebook, Google+ and Yammer examples above here are the
    descriptions that each give back

    Facebook::

        ''

    Google +::

        10k Apart Responsive Edition. Inspire the Web with Just 10k. Read the
        Rules Submit an Entry. The Gallery. 1-1 of 1. Launch Details. Colorrrs.
        Colorrrs. Dave Rupert. Enter Now. Enter Now. The Rules (FA...

    Yammer (Embedly)::

        Total file size including images, scripts & markup can't be over 10k
        zipped. Details Use the approved list of libraries without it counting
        against your 10K. Details We encourage HTML5, and apps must work
        equally well in IE10 PP2, Firefox & a Webkit browser. Details
        Applications need to be responsive.

    As you can see Facebook gave up and was unable to pull a description.
    Google found the best first line, but quickly degrades to a bunch of
    nonsense. Embedly found the largest body of text and tried to use that.
    It's the longest and respects sentence structure, but the description of
    the rules, not of the contest itself. The fun thing is, if you visit the
    page, none of this text is initially viewable to the user. It's in divs
    that are hidden by css, and therefore we have no way of knowing if they are
    displayed or not.

    A user could easily intervene here and edit the description to something
    that made a little more sense. As a side note, it's very easy to pull text
    out of a page, it's hard to pull a good excerpt and it's even harder to
    know when you should be displaying one at all.

images

    This is a JSON array of images of possible thumbnails for the user to
    select. For 10k.aneventapart.com, the JSON array looks like this:

    .. code-block:: json

        [
          {
            "url": "http://10k.aneventapart.com/Uploads/501/Thumbnail1.jpg",
            "width": 600,
            "height": 400
          },
          {
            "url": "http://10k.aneventapart.com/Content/img/enter_now.jpg",
            "width": 600,
            "height": 400
          },
          {
            "url": "http://10k.aneventapart.com/Content/img/10k_logo.png",
            "width": 235,
            "height": 144
          }
        ]

    As you can see Embedly values the larger images in the middle of the page
    greater than the smaller logo at the top of the page. Within the middle of
    the page we value images that appear higher in the page.

    Images are pulled from a bunch of different sources: API responses, the
    Open Graph image tag, the `image_src` link tag and the page itself. Embedly
    follows all these images to get the correct height, width and verify that
    they exist. Scoring these images is really complicated. Each image is
    scored based on where they lie in the page, what the image type is, if they
    matched a list of commonly used ad servers, did the image redirect and a
    whole slew of other factors that have been added over time.

    Still with all these factors, it's hard to pick the right image every time.

provider_display

    ``provider_display`` is different from oEmbed's ``provider_name``. It is a
    very easy way to get the domain of the provider. For example,
    ``http://www.bbc.co.uk/news/science-environment-14391929`` has a
    ``provider_display`` of ``www.bbc.co.uk``. This allows you to show a user
    what domain they will be visiting.

provider_url

    ``provider_url`` works in conjunction with ``provider_display``. It's the
    URL of the provider. Most of the time you can use it to link to the
    provider like so:

    .. code-block:: html

        <a href="{{provider_url}}">{{provider_display}}</a>

object

    ``object`` is fairly similar to an oEmbed object, but striped down. The
    idea here is that there is an object associated with the url that you
    passed to Embedly. There are three types: ``photo``, ``video`` and ``rich``
    . ``video`` and ``rich`` can be treated the same code wise when displaying
    the embed. The ``html`` element can just be set to the innerHTML of the
    feed item. Here is a simple example in js:

    .. code-block:: javascript

        if (preview.object.type in {'video':'', 'rich':''}){
           Ext.fly('item').dom.innerHtml = preview.object.html;
        }

    The ``photo`` is a little different in that it there is no ``html``
    attribute, but a URL instead. You can very easily use it to build the html
    though:

    .. code-block:: javascript

        if (preview.object.type == 'image'){
           Ext.fly('item').dom.innerHtml = '<img src="'+preview.object.url'"/>';
        }

    Note that we are *not* using the width and height attributes on the ``img``
    tag. The images that are passed back are all different sizes so, it's best
    to use the css style ``max-width`` like so:

    .. code-block:: css

        #item img {
            max-width:400px;
        }

    Don't bother messing with the height. People know how to scroll, so designs
    that are tolerant to different heights of images are the best.

Retrieval
---------
There are two sections to the feed; retrieval and display. Retrieval is the
long section that describes grabbing metadata from Embedly and allowing the
user to edit it before submission. Display is much shorter and just goes into
tips and tricks for displaying the data.

We start off with the following simple form:

    .. code-block:: html

        <form action="." method="post">
            <textarea id="id_status" name="status">
            </textarea>
            <input type="submit" value="Save"/>
        </form>

And the base for our Preview obj that we will use to wire up all the supporting
functions. You can use any of the 20 different object declaration patterns in
JavaScript, ours just happens to look like this:

    .. code-block:: javascript

        var Preview = (function(){
          var Preview = {};
          return Preview;
        })();

A user will come to your site in hopes of posting a status of some sort and
this status may contain a link. There are a few events that we need to listen
to here in order to create the desired effect: ``paste``, ``blur`` and
``keyup``.

paste

    Easily the most common way that users move links around. The event fires
    after anything is pasted into the object you are listening on. In Ext you
    can listen to the event like so:

    .. code-block:: javascript

        Ext.EventManager.on("id_status", 'paste', Preview.fetchMetadata);

    The ``paste`` event is a little inconsistent however and at least in Chrome
    actually fires before the ``textarea`` is filled. Because of that it's
    better to set a short timeout to make sure the pasted value is there:

    .. code-block:: javascript

        Ext.EventManager.on("id_status", 'paste', function(){
            setTimeout(Preview.fetchMetadata, 250);
        });

blur

    When the the status textarea loses focus we need to check if the user added
    anything to it. While ``keyup`` and ``paste`` will catch 95% of the cases
    this one is nice to have:

    .. code-block:: javascript

        Ext.EventManager.on("id_status", 'blur', Preview.fetchMetadata);

keyup

    If a user wants to be so bold that they actually type in the URL, we want
    to fetch it as soon as they hit the spacebar. This one is a little more
    tricky because if they are manually typing a url they may edit it a few
    times causing repeat calls. While we are not going to worry about that here
    it's just something to think about.

    .. code-block:: javascript

        Ext.EventManager.on("id_status", 'blur', Preview.onKeyUp);

    The ``onKeyUp`` function has a different set of rules than just
    ``fetchMetadata`` as we have to listen for just the spacebar after a URL
    has been entered:

    .. code-block:: javascript

        onKeyUp : function(e,t){
          // Ignore Everything but the spacebar Key event.
          if (e.getKey() != 32) return null;

          //See if there is a url in the status textarea
          var url = Preview.getStatusUrl();
          if (url == null) return null;

          // If there is a url, then we need to unbind the event so it doesn't
          // fire again. This is very common for all status updaters as
          // otherwise it would create a ton of unwanted requests.
          Ext.EventManager.un("id_status", 'keyup', Preview.onKeyUp);

          //Fire the fetch metadata function
          Preview.fetchMetadata();
        }, ...

    The ``unbind`` is very important here. A user may go back and edit the URL
    a hundred times here. We assume they got it right the first time, otherwise
    we will update the URL when the textarea loses focus.

Now that all the events are hooked up we need to pull the URL out of the status
textarea. While we won't be handing multiple urls, it's fairly easy to pull out
a single one:

    .. code-block:: javascript

        var status = Ext.fly('id_status').getValue();

        //Simple regex to make sure the url is valid.
        var urlexp = /http(s?):\/\/(\w+:{0,1}\w*)?(\S+)(:[0-9]+)?(\/|\/([\w#!:.?+=&%@!\-\/]))?/;

        //Match the status against the urlexp
        var matches = status.match(urlexp);

        return matches? matches[0] : null

This will catch any url as long as the user has entered the ``http`` or
``https`` scheme. As we know the scheme is becoming less and less prevalent and
most users expect it to work if they leave out the ``http://``. For example it
shouldn't matter if a user enters nyti.ms/qdGs9A or http://nyti.ms/qdGs9A. You
could be clever here and just update the original regex, but I'm not, so I will
create a new one:

    .. code-block:: javascript

        var urlexp = /[-\w]+(\.[a-z]{2,})+(\S+)?(\/|\/[\w#!:.?+=&%@!\-\/])?/g;

        var matches = status.match(urlexp);

        return matches? 'http://'+matches[0] : null

This regex is going to catch a number of false positives here. Users editing
their statuses may type something like "I love it.seriously ..." which will
trigger a request. You could do something clever with `publicsuffix.org
<http://publicsuffix.org>`_ or just be better with regexes. What's interesting
to note is that neither Facebook or Google Plus offer this feature. They both
make you use the 'link' function in order to enter a URL without a scheme. They
must know something or Facebook set the trend and Google+ just copied.

Once you actually have the URL from the status textarea we have to make a JSONP
request to the Embedly Preview endpoint to get the metadata associated with
that URL. I used ``jsonp.js`` that was bundled in ``examples/jsonp`` in the Ext
Core download. Here is the code to get it done, then we will go into all the
available options:

    .. code-block:: javascript

        // Sets up the parameters we are going to use in the request.
        params = {
          url:url,
          key:'key', // replace with your key.
          autoplay:true,
          maxwidth:500,
          wmode : 'opaque',
          words : 30
        }

        // Make the request to Embedly. Note we are using the preview endpoint:
        // http://embed.ly/docs/endpoints/1/preview
        Ext.ux.JSONP.request('http://api.embed.ly/1/preview', {
          callbackKey: 'callback',
          params: params,
          callback: Preview.metadataCallback
        });

When setting up the parameters you have a number of options. We are going to go
into detail on a number of them here so you know just how each will effect your
application.

url

    The ``url`` that you want to retrieve metadata for. Ext takes care of
    encoding the URL, but if you aren't using a library you need to escape the
    URL. Something like this works:

    .. code-block:: javascript

        var url = encodeURIComponent('http://embed.ly')

key

    Your Embedly API key. You can sign up for one at `embed.ly/pricing
    <http://embed.ly/pricing>`_. This tutorial uses the Preview endpoint
    which is only available at the "Starter" plan level and above.

maxwidth

    This is the maximum width of the embed in pixels. ``maxwidth`` is used for
    scaling down embeds so they fit into a certain width. If the container for
    an embed is 500px you should pass ``{ maxwidth: 500 }`` in the parameters.
    For example, if you don’t set a ``maxwidth`` for the a Vimeo video Embedly
    will return the following html:

    .. code-block:: html

        <iframe src="http://player.vimeo.com/video/18150336" width="1280"
        height="720" frameborder="0"></iframe>

    This width may cause the embed to overflow the containing div. If we pass
    ``{ maxwidth: 500 }`` the html will be:

    .. code-block:: html

        <iframe src="http://player.vimeo.com/video/18150336" width="500"
        height="281" frameborder="0"></iframe>

    It is highly recommended that developers pass a ``maxwidth`` to Embedly.

width

    ``width`` will scale embeds type rich and video to the exact ``width`` that
    a developer specifies in pixels. Embeds smaller than this width will be
    scaled up and embeds larger than this width will be scaled down. During the
    scaling process the embed may become distorted, so if you can, it's best to
    use the ``maxwidth`` parameter.

    Width is however really useful if you are working with a small set of
    providers that you know scale really well. It will scale up embeds to give
    a nice constant feel of every embed in your application.

wmode

    ``wmode`` will append the ``wmode`` value to the flash object. Possible
    values include ``window``, ``opaque`` and ``transparent``. Generally you
    always want to have ``{ wmode : 'opaque' }`` in the parameters. This
    prevents embeds from being rendered on top of modals or other html
    positioned on top of them.

autoplay

    Tells ``video`` embeds to start playing as soon as they are loaded.
    Generally this is a reallyannoying feature of some sites, but in our case
    it's a great feature. It allows us to start playing the video as soon as
    the the user clicks on the thumbnail.

words

    The ``words`` parameter has a default value of 50 and works by trying to
    split the description at the closest sentence to that word count. For
    example, the following lorem ipsum description is made up of 33 words
    and 5 sentences:

        Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus
        dapibus auctor aliquam. Donec vitae justo ligula, id luctus ligula.
        Duis eget mauris lacinia sapien aliquet vulputate a et orci. Sed eu
        imperdiet sem.

    Now by default, Embedly will return all 33 words, but say you want only 20
    words. By passing ``{ words : 20}`` Embedly would return:

        Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus
        dapibus auctor aliquam. Donec vitae justo ligula, id luctus ligula.

    This is actually only 19 words, but we split at the closest sentence. Words
    is really useful for controlling how long the descriptive text for each URL
    is. In this case we are going to use 30 words to not overwhelm the page
    with text.

chars

    ``chars`` is like ``words``, but instead of truncating to the nearest
    sentence, Embedly will blindly truncate a description to the number of
    characters you specify adding ... at the end when needed. For the above
    description, if we set ``{ chars : 100 }`` it will return::

        Lorem ipsum dolor sit amet, consectetur adipiscing elit. Vivamus
        dapibus auctor aliquam. Donec ...


Display
-------
Once you make the request we have to deal with the data that we get back from
Embedly. We discussed the different parts of the object that we are going to
use earlier, now it's just putting together the pieces. A lot of this is
actually up to the individual developer, but here are some tips an tricks. We
declare the ``metadataCallback`` from before as

    .. code-block:: javascript

        metadataCallback : function(obj){
          // Deal with the object here.
        }

The first thing you need to do is validate the request. Every obj should have a
``type``. If it's not there this is a clear sign that something is off. This is
a basic check to make sure we should proceed. Generally will never happen, but
it's a nice to have just in case:

    .. code-block:: javascript

        if (!obj.hasOwnProperty('type')){
            console.log('Embedly returned an invalid response');
            return false;
        }

The next thing you need to make sure that there isn't an error. If Embedly
is sent an invalid URL, the URL returns a 404 or some other error Embedly will
return an object of type ``error``. In this general case the default workflow
should occur. Generally you need not alert the user, just proceed as everything
is happening normally:

    .. code-block:: javascript

        if (obj.type == 'error'){
            console.log('URL ('+obj.url+') returned an error: '+ obj.error_message);
            return false;
        }

At this point you have a response that you can work with, but you need to
filter out types that you do not want to handle. In this case we will only be
handling ``html`` and ``image`` type responses. Others link ``pdf`` of
``video`` we can build in another day:

    .. code-block:: javascript

        if (!(obj.type in {'html':'', 'image':''})){
            console.log('URL ('+obj.url+') returned a type ('+obj.type+') not handled');
            return false;
        }

To wire up the form to work on POST we need to set all the attributes to hidden
inputs within the form. When the user is done and hits submit it will send all
this data back to the server for saving. To do this we iterate over a list of
elements we want to save:

    .. code-block:: javascript

        Ext.each(Preview.attrs, function(n){
          Ext.DomHelper.append('preview_form', {
            tag:'input',
            name : n,
            type : 'hidden',
            id : 'id_'+n,
            value : obj.hasOwnProperty(n) && obj[n] ? encodeURIComponent(obj[n]): ''
          });
        });

You can set ``Preview.attrs`` to pretty much anything you want, but in our case
we use:

    .. code-block:: javascript

        attrs: ['type', 'original_url', 'url', 'title', 'description',
                'favicon_url', 'provider_url', 'provider_display', 'safe',
                'html', 'thumbnail_url']

The last part of the ``metadataCallback`` function is handing off the obj to be
rendered by a ``Display`` object. The ``Display`` object lets us change how the
link preview is displayed without worrying about how it effects the ``Preview``
object. It also helped us create multiple versions of the demo:

    .. code-block:: javascript

        Preview.Display.render(obj);

Rendering the link form is actually pretty boring. You show read the `code
<https://github.com/embedly/embedly-tutorial-ext/blob/master/js/preview.js#L109>`_
, but at the end of the day, it's going to be up to you. The only
thing to remember is to to update the hidden inputs with the correct values
after the user has changed any data. For example we run this after a user has
updated the title:

    .. code-block:: javascript

        Ext.fly('id_title').dom.value = encodeURIComponent(elem.dom.value);

Now it's all about saving the data. You can do it as a basic ``post``, but why
make the user wait around for the save to happen? Instead we can write the
status to the feed and save it asynchronously. This way, to the user, it
appears as though the save happened instantaneously. First we need to bind a
callback to the ``submit`` handler of the form:

    .. code-block:: javascript

        Ext.EventManager.on("preview_form", "submit", Preview.Feed.submitFeedItem);

The ``Feed`` object is like ``Display`` in that we can switch in and out
different implementations for various effects. The basic ``Feed`` object holds
the CRUD functions for a set of data:

    .. code-block:: javascript

        var Feed = {
          createFeedItem : function (data){}},
          storeFeedItem: function(data){},
          populateFeed: function(){},
          submitFeedItem: function(e,t){},
          deleteFeedItem: function(e,t){}
        }

The ``submitFeedItem`` call back need to pull out all the data out of the form
like so:

    .. code-block:: javascript

        var data = {};
        // Get the data we need out of the form.
        Ext.select('#preview_form input').each(function(e){
          data[e.dom.name] = decodeURIComponent(e.dom.value)
        });

        //Create the Feed Item and display it in the feed.
        Preview.Feed.createFeedItem(data);

        //Save the Feed Item
        Preview.Feed.storeFeedItem(data);

Note that when we grab the data out of the form we need to decode it via the
``decodeURIComponent`` function. Once that data is out of the form, we can then
use it to create a feed item on the fly and then save it. The basic structure
of a feed item for us looks like so:

    .. code-block:: html

        <div class="item">
          <a class="favicon" href="{{provider_url}}" title="{{provider_display}}">
            <img src="{{favicon_url}}">
          </a>
          <a class="title" href="{{url}}">{{title}}</a>
          <div class="thumbnail">
            <a href="#">
              <img src="{{thumbnail_url}}">
            </a>
          </div>
          <div class="info">
            <a class="provider" href="{{provider_url}}">{{provider_display}}</a>
            <p>{{description}}</p>
            <a class="close" href="#">x</a>
          </div>
        </div>

We build that via a giant JSON object that you can see `here  <https://github.com/embedly/embedly-tutorial-ext/blob/master/js/preview.js#L440>`_
. The important thing here is to also save the item data into div as data
properties. The HTML5 spec describes a method for saving off `custom data
attributes <http://dev.w3.org/html5/spec/Overview.html#custom-data-attribute>`_
that we will use here. To add these attributes the the outer div ``.item`` we
can use something like this when building the JSON object:

    .. code-block:: javascript

        Ext.each(Preview.attrs, function(n){
          elem['data-'+(n == 'html' ? 'embed' : n)] = encodeURIComponent(data[n])
        });

We change the ``data-html`` to ``data-embed`` because it appears to be reserved
by Ext or the browser, but I didn't investigate to deeply. Once this is in
place we can get the title for any item like so:

    .. code-block:: javascript

        elem.dom.dataset.title

To be on the safe side of browser bugs we still use:

    .. code-block:: javascript

        elem.dom.getAttribute('data-title')

Using these data attributes we can create an event to autoplay the video when
a user clicks the thumbnail. In order the accomplish this we need to know that
the url has a video associated with it. In the ``metadataCallback`` from above
we actually change the ``type`` of the embed after we do a number of the checks
to ``video`` or ``rich`` instead of ``html``. We do this by updating the hidden
inputs to have the correct values:

    .. code-block:: javascript

        if (obj.object && obj.object.type in {'video':'', 'rich':''}){
          Ext.fly('id_html').dom.value = obj.object.html;
          Ext.fly('id_type').dom.value = obj.object.type;
        }

If the type is ``video`` or ``rich`` we change the the thumbnail html to look
like so:

    .. code-block:: html

        <a href="#" class="video">
            <img src="{{thumbnail_url}}">
            <span class="player_overlay"></span>
        </a>

This creates an embed that looks like

.. image:: ../images/ext_rdio_item.png

We can then use Ext to bind the click event to the ``Feed.playVideo`` callback:

    .. code-block:: javascript

        Ext.getBody().on('click', Preview.Feed.playVideo, null, {delegate: 'a.video'});

When the event is fired we can then replace the contents of the '.item' div
with the embed html that we saved in custom data attributes:

    .. code-block:: javascript

        playVideo : function(e,t){
          e.preventDefault();
          // Get the parent '.item' div
          var elem = Ext.fly(t).parent('.item');
          // Set the '.items' content to the 'data-embed' value.
          elem.dom.innerHTML = decodeURIComponent(elem.dom.getAttribute('data-embed'));
        }

Once a user clicks the thumbnail, the end result looks like this:

.. image:: ../images/ext_rdio_expanded.png

While we could add a few other features here, we have chosen to keep it simple.
Hopefully you have a good understanding of how all the parts fit together and
can build on additional features. Definitely check out the `demo
<http://embedly.github.com/embedly-tutorial-ext>`_ and the `source code
<https://github.com/embedly/embedly-tutorial-ext>`_ for this project. It's
heavily documented and deals with some of the little things like loading
notifications.

If you have any questions or comments you can send us a note to
support@embed.ly or submit an `issue
<https://github.com/embedly/embedly-tutorial-ext/issues>`_.


.. _post: http://blog.embed.ly/creating-a-facebook-like-url-submission-tool
.. _blog: http://blog.embed.ly/
