===========================================================
Template Preprocessor Readme
===========================================================

Author
  Jonathan Slenders, City Live nv

Description:
    Preprocessor for Django templates which compiles all files in a very
    compact version. This does not only result in less HTTP traffic, but also
    causes the template render engine of Django to have less work at runtime.
    (A lot of template tags can be processed in advance.)


    Why?

    - Templates contain a lot of meaningless information causing more bandwidth
      than required.
    - A lot of information in templates which appears to be dynamic is actually
      static and only needs to be calculated once instead of for every single
      request.

Issues?
    Please report issues if you find one!
    https://github.com/citylive/django-template-preprocessor/issues



How much faster exactly?
------------------------

Some of our test show page loading improvements op to 400% compared to the
default Django loader, and up to 200% compared to the cached loader in Django
1.2. (Although, it may vary between templates.)



Setup
-----

Add this to your settings.py:

::

    # Define languages -> otherwise templates will be compiled in all possible languages
    LANGUAGES = (
         ('en', 'EN'),
    )

    # Define directories
    MEDIA_ROOT = ROOT + 'media/'
    MEDIA_URL = '/media/'
    MEDIA_CACHE_DIR = MEDIA_ROOT + 'cache/'
    MEDIA_CACHE_URL = MEDIA_URL + 'cache/'
    TEMPLATE_CACHE_DIR = ROOT + 'templates/cache/'

    # Wrap template loaders
    if DEBUG:
        TEMPLATE_LOADERS = (
            ('template_preprocessor.template.loaders.ValidatorLoader',
            #('template_preprocessor.template.loaders.RuntimeProcessedLoader',
                TEMPLATE_LOADERS
            ),
        )
    else:
        TEMPLATE_LOADERS = (
            ('template_preprocessor.template.loaders.PreprocessedLoader',
                TEMPLATE_LOADERS
            ),
        )

    # Applications
    INSTALLED_APPS += ( 'template_preprocessor', )


This will recompile every template during every page load at runtime, but it
will use the preprocessed templates during production.


You can finetune the behaviour of the preprocessor, by enabling or disabling
specific options. Add the following to your settings.py

::

    # Enabled modules of the template preprocessor
    TEMPLATE_PREPROCESSOR_OPTIONS = {
            # Default settings
            '*': ('html', 'whitespace-compression', ),

            # Override for specific applications
            ('django.contrib.admin', 'django.contrib.admindocs', 'debug_toolbar'): ('no-html',),
    }

The available options are:

    - whitespace-compression
    - no-whitespace-compression
    - merge-internal-javascript
    - merge-internal-css
    - html
    - no-html
    - html-remove-empty-class-attributes
    - html-check-alt-and-title-attributes
    - pack-external-javascript
    - pack-external-css
    - compile-css
    - compile-javascript
    - parse-all-html-tags
    - validate-html
    - no-validate-html



Configuration at runtime
------------------------

Following template tag is an example of how to alter the preprocessor
behaviour for a specific template.

::

    {% load template_preprocessor %}{% ! no-html %}



Precompile templates
--------------------

Run this command on your shell

::

    ./manage.py compile_templates -v 2


Or if you want to recompile *all* templates. (This is required when base
templates change, because the preprocessor does not yet trace template
dependecies when compiling only the changed templates.)

::

    ./manage.py compile_templates -v 2 --all


Additional recommendations
--------------------------

* Use CDATA for javascript. (will avoid accidently Html tags in script.)

::

    <script type="text/javascript">
        // <![CDATA[
        ...alert('<div>');
        // ]]>
    </script>

* Prefer javascript comments in JS code above Django comments, and use CSS
  comments in CSS code. (cleaner, and will be removed anyway.)

* **Most important**: *always* open and close HTML tags, javascript braces, etc..
  in the same scope. Instead of:

  ::

            {% if test %}
                <a ...
            {% else test %}
                <a ...
            {% endif %}
                ...
            > link </a>

  do:

  ::

            <a
            {% if test %}
                 ...
            {% else test %}
                 ...
            {% endif %}
                ...
            > link </a>

  See? Opening bracket is now in the same text node as the closing bracket.
  This is important for the parser to know that they are a pair, because the
  HTML parser won't or can't be aware of how the Django Template tags are
  rendered. What if the render() method of the ``{% if %}``-node would return
  an empty string, then there's no pair to be found in the first example.



What if some HTML does not compile/validate.
--------------------------------------------

It is possible that some HTML cannot be processed at compile time, but you're
absolutely sure that the output will render valid HTML at runtime. In this case
you can tell the compiler not to try interpreting this part of HTML

Use the following template tags:

::

          {% load template_preprocessor %}
          ...
          {% !raw %}
            ... (tricky html, maybe other template tags, etc...)
          {% !endraw %}
          ...

No HTML optimizations (like compression, removing comments, ...) are done
between ``{% !raw %}`` and ``{% !endraw %}``, while everything outside this
tags is still optimized.


Extending the template preprocessor in your application.
--------------------------------------------------------

Custom template tags can also be preprocessed, if the output does not depend on
context variables. It works as follows:

Make a python module ``preprocessable_template_tags.py`` in your application folder,
and make sure the application appears in ``settings.INSTALLED_APPS``.
In this file, write template tags like:


::

    from template_preprocessor import preprocess_tag
    @preprocess_tag
    def my_custom_tag(*args):
        return 'This is the output of my custom template tag'


Every call of ``{% my_custom_tag %}`` will now be replaced by the output of this
tag.  Also, don't forget to register normal template tags in Django, in case
you don't use the template preprocessor.


Using the Chromium (Google Chrome) extension
--------------------------------------------------------

The template preprocessor has the option of adding debug symbols to the
template which can be used by a web browser extension. It is for instance
possible to view which Django template code was used for rendering any part in
the webpage.

Use the following template loader in your settings.py:

::

    'template_preprocessor.template.loaders.DebugLoader'


The ``src/chromium-extension`` folder in the template_preprocessor repository
contains the unpacked plugin for the Chromium webbrowser.

After loading the plugin, go to a webpage that is rendered through the
DebugLoader, right click anywhere on the webpage, and click "View Django
Source Code". Now you can interactively see the match between the original
template and the rendered output.

To be able to use the *open in editor* functionality, run the following server
from the command line:

::
    ./manage.py open_in_editor_server


More information?
-----------------

Read README_2 for more technical information.
