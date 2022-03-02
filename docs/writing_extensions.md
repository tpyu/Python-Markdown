Writing Extensions for Python-Markdown
======================================

Overview
--------

Python-Markdown includes an API for extension writers to plug their own 
custom functionality and/or syntax into the parser. There are preprocessors
which allow you to alter the source before it is passed to the parser, 
inline patterns which allow you to add, remove or override the syntax of
any inline elements, and postprocessors which allow munging of the
output of the parser before it is returned. If you really want to dive in, 
there are also blockprocessors which are part of the core BlockParser.

As the parser builds an [ElementTree][] object which is later rendered 
as Unicode text, there are also some helpers provided to ease manipulation of 
the tree. Each part of the API is discussed in its respective section below. 
Additionally, reading the source of some [Available Extensions][] may be 
helpful. For example, the [Footnotes][] extension uses most of the features 
documented here.

* [Preprocessors][]
* [InlinePatterns][]
* [Treeprocessors][] 
* [Postprocessors][]
* [BlockParser][]
* [Working with the ElementTree][]
* [Integrating your code into Markdown][]
    * [extendMarkdown][]
    * [OrderedDict][]
    * [registerExtension][]
    * [Config Settings][]
    * [makeExtension][]

<h3 id="preprocessors">Preprocessors</h3>

Preprocessors munge the source text before it is passed into the Markdown 
core. This is an excellent place to clean up bad syntax, extract things the 
parser may otherwise choke on and perhaps even store it for later retrieval.

Preprocessors should inherit from ``markdown.preprocessors.Preprocessor`` and 
implement a ``run`` method with one argument ``lines``. The ``run`` method of 
each Preprocessor will be passed the entire source text as a list of Unicode 
strings. Each string will contain one line of text. The ``run`` method should 
return either that list, or an altered list of Unicode strings.

A pseudo example:

    from markdown.preprocessors import Preprocessor

    class MyPreprocessor(Preprocessor):
        def run(self, lines):
            new_lines = []
            for line in lines:
                m = MYREGEX.match(line)
                if m:
                    # do stuff
                else:
                    new_lines.append(line)
            return new_lines

<h3 id="inlinepatterns">Inline Patterns</h3>

Inline Patterns implement the inline HTML element syntax for Markdown such as
``*emphasis*`` or ``[links](http://example.com)``. Pattern objects should be 
instances of classes that inherit from ``markdown.inlinepatterns.Pattern`` or 
one of its children. Each pattern object uses a single regular expression and 
must have the following methods:

* **``getCompiledRegExp()``**: 

    Returns a compiled regular expression.

* **``handleMatch(m)``**: 

    Accepts a match object and returns an ElementTree element of a plain 
    Unicode string.

Note that any regular expression returned by ``getCompiledRegExp`` must capture
the whole block. Therefore, they should all start with ``r'^(.*?)'`` and end
with ``r'(.*?)!'``. When using the default ``getCompiledRegExp()`` method 
provided in the ``Pattern`` you can pass in a regular expression without that 
and ``getCompiledRegExp`` will wrap your expression for you and set the 
`re.DOTALL` and `re.UNICODE` flags. This means that the first group of your 
match will be ``m.group(2)`` as ``m.group(1)`` will match everything before the
pattern.

For an example, consider this simplified emphasis pattern:

    from markdown.inlinepatterns import Pattern
    from markdown.util import etree

    class EmphasisPattern(Pattern):
        def handleMatch(self, m):
            el = etree.Element('em')
            el.text = m.group(3)
            return el

As discussed in [Integrating Your Code Into Markdown][], an instance of this
class will need to be provided to Markdown. That instance would be created
like so:

    # an oversimplified regex
    MYPATTERN = r'\*([^*]+)\*'
    # pass in pattern and create instance
    emphasis = EmphasisPattern(MYPATTERN)

Actually it would not be necessary to create that pattern (and not just because
a more sophisticated emphasis pattern already exists in Markdown). The fact is,
that example pattern is not very DRY. A pattern for `**strong**` text would
be almost identical, with the exception that it would create a 'strong' element.
Therefore, Markdown provides a number of generic pattern classes that can 
provide some common functionality. For example, both emphasis and strong are
implemented with separate instances of the ``SimpleTagPettern`` listed below. 
Feel free to use or extend any of the Pattern classes found at `markdown.inlinepatterns`.

**Generic Pattern Classes**

* **``SimpleTextPattern(pattern)``**:

    Returns simple text of ``group(2)`` of a ``pattern``.

* **``SimpleTagPattern(pattern, tag)``**:

    Returns an element of type "`tag`" with a text attribute of ``group(3)``
    of a ``pattern``. ``tag`` should be a string of a HTML element (i.e.: 'em').

* **``SubstituteTagPattern(pattern, tag)``**:

    Returns an element of type "`tag`" with no children or text (i.e.: 'br').

There may be other Pattern classes in the Markdown source that you could extend
or use as well. Read through the source and see if there is anything you can 
use. You might even get a few ideas for different approaches to your specific
situation.

<h3 id="treeprocessors">Treeprocessors</h3>

Treeprocessors manipulate an ElemenTree object after it has passed through the
core BlockParser. This is where additional manipulation of the tree takes
place. Additionally, the InlineProcessor is a Treeprocessor which steps through
the tree and runs the InlinePatterns on the text of each Element in the tree.

A Treeprocessor should inherit from ``markdown.treeprocessors.Treeprocessor``,
over-ride the ``run`` method which takes one argument ``root`` (an Elementree 
object) and returns either that root element or a modified root element.

A pseudo example:

    from markdown.treprocessors import Treeprocessor

    class MyTreeprocessor(Treeprocessor):
        def run(self, root):
            #do stuff
            return my_modified_root

For specifics on manipulating the ElementTree, see 
[Working with the ElementTree][] below.

<h3 id="postprocessors">Postprocessors</h3>

Postprocessors manipulate the document after the ElementTree has been 
serialized into a string. Postprocessors should be used to work with the
text just before output.

A Postprocessor should inherit from ``markdown.postprocessors.Postprocessor`` 
and over-ride the ``run`` method which takes one argument ``text`` and returns 
a Unicode string.

Postprocessors are run after the ElementTree has been serialized back into 
Unicode text.  For example, this may be an appropriate place to add a table of 
contents to a document:

    from markdown.postprocessors import Postprocessor

    class TocPostprocessor(Postprocessor):
        def run(self, text):
            return MYMARKERRE.sub(MyToc, text)

<h3 id="blockparser">BlockParser</h3>

Sometimes, pre/tree/postprocessors and Inline Patterns aren't going to do what 
you need. Perhaps you want a new type of block type that needs to be integrated 
into the core parsing. In such a situation, you can add/change/remove 
functionality of the core ``BlockParser``. The BlockParser is composed of a
number of Blockproccessors. The BlockParser steps through each block of text
(split by blank lines) and passes each block to the appropriate Blockprocessor.
That Blockprocessor parses the block and adds it to the ElementTree. The
[Definition Lists][] extension would be a good example of an extension that
adds/modifies Blockprocessors.

A Blockprocessor should inherit from ``markdown.blockprocessors.BlockProcessor``
and implement both the ``test`` and ``run`` methods.

The ``test`` method is used by BlockParser to identify the type of block.
Therefore the ``test`` method must return a boolean value. If the test returns
``True``, then the BlockParser will call that Blockprocessor's ``run`` method.
If it returns ``False``, the BlockParser will move on to the next 
BlockProcessor.

The **``test``** method takes two arguments:

* **``parent``**: The parent etree Element of the block. This can be useful as
  the block may need to be treated differently if it is inside a list, for
  example.

* **``block``**: A string of the current block of text. The test may be a 
  simple string method (such as ``block.startswith(some_text)``) or a complex 
  regular expression.

The **``run``** method takes two arguments:

* **``parent``**: A pointer to the parent etree Element of the block. The run 
  method will most likely attach additional nodes to this parent. Note that
  nothing is returned by the method. The Elementree object is altered in place.

* **``blocks``**: A list of all remaining blocks of the document. Your run 
  method must remove (pop) the first block from the list (which it altered in
  place - not returned) and parse that block. You may find that a block of text
  legitimately contains multiple block types. Therefore, after processing the 
  first type, your processor can insert the remaining text into the beginning
  of the ``blocks`` list for future parsing.

Please be aware that a single block can span multiple text blocks. For example,
The official Markdown syntax rules state that a blank line does not end a
Code Block. If the next block of text is also indented, then it is part of
the previous block. Therefore, the BlockParser was specifically designed to 
address these types of situations. If you notice the ``CodeBlockProcessor``,
in the core, you will note that it checks the last child of the ``parent``.
If the last child is a code block (``<pre><code>...</code></pre>``), then it
appends that block to the previous code block rather than creating a new 
code block.

Each BlockProcessor has the following utility methods available:

* **``lastChild(parent)``**: 

    Returns the last child of the given etree Element or ``None`` if it had no 
    children.

* **``detab(text)``**: 

    Removes one level of indent (four spaces by default) from the front of each
    line of the given text string.

* **``looseDetab(text, level)``**: 

    Removes "level" levels of indent (defaults to 1) from the front of each line 
    of the given text string. However, this methods allows secondary lines to 
    not be indented as does some parts of the Markdown syntax.

Each BlockProcessor also has a pointer to the containing BlockParser instance at
``self.parser``, which can be used to check or alter the state of the parser.
The BlockParser tracks it's state in a stack at ``parser.state``. The state
stack is an instance of the ``State`` class.

**``State``** is a subclass of ``list`` and has the additional methods:

* **``set(state)``**: 

    Set a new state to string ``state``. The new state is appended to the end 
    of the stack.

* **``reset()``**: 

    Step back one step in the stack. The last state at the end is removed from 
    the stack.

* **``isstate(state)``**: 

    Test that the top (current) level of the stack is of the given string 
    ``state``.

Note that to ensure that the state stack doesn't become corrupted, each time a
state is set for a block, that state *must* be reset when the parser finishes
parsing that block.

An instance of the **``BlockParser``** is found at ``Markdown.parser``.
``BlockParser`` has the following methods:

* **``parseDocument(lines)``**: 

    Given a list of lines, an ElementTree object is returned. This should be 
    passed an entire document and is the only method the ``Markdown`` class 
    calls directly.

* **``parseChunk(parent, text)``**: 

    Parses a chunk of markdown text composed of multiple blocks and attaches 
    those blocks to the ``parent`` Element. The ``parent`` is altered in place 
    and nothing is returned. Extensions would most likely use this method for 
    block parsing.

* **``parseBlocks(parent, blocks)``**: 

    Parses a list of blocks of text and attaches those blocks to the ``parent``
    Element. The ``parent`` is altered in place and nothing is returned. This 
    method will generally only be used internally to recursively parse nested 
    blocks of text.

While is is not recommended, an extension could subclass or completely replace
the ``BlockParser``. The new class would have to provide the same public API.
However, be aware that other extensions may expect the core parser provided
and will not work with such a drastically different parser.

<h3 id="working_with_et">Working with the ElementTree</h3>

As mentioned, the Markdown parser converts a source document to an 
[ElementTree][] object before serializing that back to Unicode text. 
Markdown has provided some helpers to ease that manipulation within the context 
of the Markdown module.

First, to get access to the ElementTree module import ElementTree from 
``markdown`` rather than importing it directly. This will ensure you are using 
the same version of ElementTree as markdown. The module is found at 
``markdown.util.etree`` within Markdown.

    from markdown.util import etree
    
``markdown.util.etree`` tries to import ElementTree from any known location, 
first as a standard library module (from ``xml.etree`` in Python 2.5), then as 
a third party package (``Elementree``). In each instance, ``cElementTree`` is 
tried first, then ``ElementTree`` if the faster C implementation is not 
available on your system.

Sometimes you may want text inserted into an element to be parsed by 
[InlinePatterns][]. In such a situation, simply insert the text as you normally
would and the text will be automatically run through the InlinePatterns. 
However, if you do *not* want some text to be parsed by InlinePatterns,
then insert the text as an ``AtomicString``.

    from markdown.util import AtomicString
    some_element.text = AtomicString(some_text)

Here's a basic example which creates an HTML table (note that the contents of 
the second cell (``td2``) will be run through InlinePatterns latter):

    table = etree.Element("table") 
    table.set("cellpadding", "2")                      # Set cellpadding to 2
    tr = etree.SubElement(table, "tr")                 # Add child tr to table
    td1 = etree.SubElement(tr, "td")                   # Add child td1 to tr
    td1.text = markdown.AtomicString("Cell content")   # Add plain text content
    td2 = etree.SubElement(tr, "td")                   # Add second td to tr
    td2.text = "*text* with **inline** formatting."    # Add markup text
    table.tail = "Text after table"                    # Add text after table

You can also manipulate an existing tree. Consider the following example which 
adds a ``class`` attribute to ``<a>`` elements:

	def set_link_class(self, element):
		for child in element: 
		    if child.tag == "a":
                child.set("class", "myclass") #set the class attribute
            set_link_class(child) # run recursively on children

For more information about working with ElementTree see the ElementTree
[Documentation](http://effbot.org/zone/element-index.htm) 
([Python Docs](http://docs.python.org/lib/module-xml.etree.ElementTree.html)).

<h3 id="integrating_into_markdown">Integrating Your Code Into Markdown</h3>

Once you have the various pieces of your extension built, you need to tell 
Markdown about them and ensure that they are run in the proper sequence. 
Markdown accepts a ``Extension`` instance for each extension. Therefore, you
will need to define a class that extends ``markdown.extensions.Extension`` and 
over-rides the ``extendMarkdown`` method. Within this class you will manage 
configuration options for your extension and attach the various processors and 
patterns to the Markdown instance. 

It is important to note that the order of the various processors and patterns 
matters. For example, if we replace ``http://...`` links with ``<a>`` elements, 
and *then* try to deal with  inline html, we will end up with a mess. 
Therefore, the various types of processors and patterns are stored within an 
instance of the Markdown class in [OrderedDict][]s. Your ``Extension`` class 
will need to manipulate those OrderedDicts appropriately. You may insert 
instances of your processors and patterns into the appropriate location in an 
OrderedDict, remove a built-in instance, or replace a built-in instance with 
your own.

<h4 id="extendmarkdown">extendMarkdown</h4>

The ``extendMarkdown`` method of a ``markdown.extensions.Extension`` class 
accepts two arguments:

* **``md``**:

    A pointer to the instance of the Markdown class. You should use this to 
    access the [OrderedDict][]s of processors and patterns. They are found 
    under the following attributes:

    * ``md.preprocessors``
    * ``md.inlinePatterns``
    * ``md.parser.blockprocessors``
    * ``md.treepreprocessors``
    * ``md.postprocessors``

    Some other things you may want to access in the markdown instance are:

    * ``md.htmlStash``
    * ``md.output_formats``
    * ``md.set_output_format()``
    * ``md.registerExtension()``
    * ``md.html_replacement_text``
    * ``md.tab_length``
    * ``md.enable_attributes``
    * ``md.smart_emphasis``

* **``md_globals``**:

    Contains all the various global variables within the markdown module.

Of course, with access to those items, theoretically you have the option to 
changing anything through various [monkey_patching][] techniques. However, you 
should be aware that the various undocumented or private parts of markdown 
may change without notice and your monkey_patches may break with a new release.
Therefore, what you really should be doing is inserting processors and patterns
into the markdown pipeline. Consider yourself warned.

[monkey_patching]: http://en.wikipedia.org/wiki/Monkey_patch

A simple example:

    from markdown.extensions import Extension

    class MyExtension(Extension):
        def extendMarkdown(self, md, md_globals):
            # Insert instance of 'mypattern' before 'references' pattern
            md.inlinePatterns.add('mypattern', MyPattern(md), '<references')

<h4 id="ordereddict">OrderedDict</h4>

An OrderedDict is a dictionary like object that retains the order of it's
items. The items are ordered in the order in which they were appended to
the OrderedDict. However, an item can also be inserted into the OrderedDict
in a specific location in relation to the existing items.

Think of OrderedDict as a combination of a list and a dictionary as it has 
methods common to both. For example, you can get and set items using the 
``od[key] = value`` syntax and the methods ``keys()``, ``values()``, and 
``items()`` work as expected with the keys, values and items returned in the 
proper order. At the same time, you can use ``insert()``, ``append()``, and 
``index()`` as you would with a list.

Generally speaking, within Markdown extensions you will be using the special 
helper method ``add()`` to add additional items to an existing OrderedDict. 

The ``add()`` method accepts three arguments:

* **``key``**: A string. The key is used for later reference to the item.

* **``value``**: The object instance stored in this item.

* **``location``**: Optional. The items location in relation to other items. 

    Note that the location can consist of a few different values:

    * The special strings ``"_begin"`` and ``"_end"`` insert that item at the 
      beginning or end of the OrderedDict respectively. 
    
    * A less-than sign (``<``) followed by an existing key (i.e.: 
      ``"<somekey"``) inserts that item before the existing key.
    
    * A greater-than sign (``>``) followed by an existing key (i.e.: 
      ``">somekey"``) inserts that item after the existing key. 

Consider the following example:

    >>> from markdown.odict import OrderedDict
    >>> od = OrderedDict()
    >>> od['one'] =  1           # The same as: od.add('one', 1, '_begin')
    >>> od['three'] = 3          # The same as: od.add('three', 3, '>one')
    >>> od['four'] = 4           # The same as: od.add('four', 4, '_end')
    >>> od.items()
    [("one", 1), ("three", 3), ("four", 4)]

Note that when building an OrderedDict in order, the extra features of the
``add`` method offer no real value and are not necessary. However, when 
manipulating an existing OrderedDict, ``add`` can be very helpful. So let's 
insert another item into the OrderedDict.

    >>> od.add('two', 2, '>one')         # Insert after 'one'
    >>> od.values()
    [1, 2, 3, 4]

Now let's insert another item.

    >>> od.add('twohalf', 2.5, '<three') # Insert before 'three'
    >>> od.keys()
    ["one", "two", "twohalf", "three", "four"]

Note that we also could have set the location of "twohalf" to be 'after two'
(i.e.: ``'>two'``). However, it's unlikely that you will have control over the 
order in which extensions will be loaded, and this could affect the final 
sorted order of an OrderedDict. For example, suppose an extension adding 
'twohalf' in the above examples was loaded before a separate  extension which 
adds 'two'. You may need to take this into consideration when adding your 
extension components to the various markdown OrderedDicts.

Once an OrderedDict is created, the items are available via key:

    MyNode = od['somekey']

Therefore, to delete an existing item:

    del od['somekey']

To change the value of an existing item (leaving location unchanged):

    od['somekey'] = MyNewObject()

To change the location of an existing item:

    t.link('somekey', '<otherkey')

<h4 id="registerextension">registerExtension</h4>

Some extensions may need to have their state reset between multiple runs of the
Markdown class. For example, consider the following use of the [Footnotes][] 
extension:

    md = markdown.Markdown(extensions=['footnotes'])
    html1 = md.convert(text_with_footnote)
    md.reset()
    html2 = md.convert(text_without_footnote)

Without calling ``reset``, the footnote definitions from the first document will
be inserted into the second document as they are still stored within the class
instance. Therefore the ``Extension`` class needs to define a ``reset`` method
that will reset the state of the extension (i.e.: ``self.footnotes = {}``).
However, as many extensions do not have a need for ``reset``, ``reset`` is only
called on extensions that are registered.

To register an extension, call ``md.registerExtension`` from within your 
``extendMarkdown`` method:


    def extendMarkdown(self, md, md_globals):
        md.registerExtension(self)
        # insert processors and patterns here

Then, each time ``reset`` is called on the Markdown instance, the ``reset`` 
method of each registered extension will be called as well. You should also
note that ``reset`` will be called on each registered extension after it is
initialized the first time. Keep that in mind when over-riding the extension's
``reset`` method.

<h4 id="configsettings">Config Settings</h4>

If an extension uses any parameters that the user may want to change,
those parameters should be stored in ``self.config`` of your 
``markdown.Extension`` class in the following format:

    self.config = {parameter_1_name : [value1, description1],
                   parameter_2_name : [value2, description2] }

When stored this way the config parameters can be over-ridden from the
command line or at the time Markdown is initiated:

    markdown.py -x myextension(SOME_PARAM=2) inputfile.txt > output.txt

Note that parameters should always be assumed to be set to string
values, and should be converted at run time. For example:

    i = int(self.getConfig("SOME_PARAM"))

<h4 id="makeextension">makeExtension</h4>

Each extension should ideally be placed in its own module starting
with the  ``mdx_`` prefix (e.g. ``mdx_footnotes.py``).  The module must
provide a module-level function called ``makeExtension`` that takes
an optional parameter consisting of a dictionary of configuration over-rides 
and returns an instance of the extension.  An example from the footnote 
extension:

    def makeExtension(configs=None) :
        return FootnoteExtension(configs=configs)

By following the above example, when Markdown is passed the name of your 
extension as a string (i.e.: ``'footnotes'``), it will automatically import
the module and call the ``makeExtension`` function initiating your extension.

You may have noted that the extensions packaged with Python-Markdown do not
use the ``mdx_`` prefix in their module names. This is because they are all
part of the ``markdown.extensions`` package. Markdown will first try to import
from ``markdown.extensions.extname`` and upon failure, ``mdx_extname``. If both
fail, Markdown will continue without the extension.

However, Markdown will also accept an already existing instance of an extension.
For example:

    import markdown
    import myextension
    configs = {...}
    myext = myextension.MyExtension(configs=configs)
    md = markdown.Markdown(extensions=[myext])

This is useful if you need to implement a large number of extensions with more
than one residing in a module.

[Preprocessors]: #preprocessors
[InlinePatterns]: #inlinepatterns
[Treeprocessors]: #treeprocessors
[Postprocessors]: #postprocessors
[BlockParser]: #blockparser
[Working with the ElementTree]: #working_with_et
[Integrating your code into Markdown]: #integrating_into_markdown
[extendMarkdown]: #extendmarkdown
[OrderedDict]: #ordereddict
[registerExtension]: #registerextension
[Config Settings]: #configsettings
[makeExtension]: #makeextension
[ElementTree]: http://effbot.org/zone/element-index.htm
[Available Extensions]: extensions/
[Footnotes]: extensions/footnotes.html
[Definition Lists]: extensions/definition_lists.html

Introduction
In addition to providing a number of built-in extensions, Python-Markdown provides an application programming interface (API) which allows anyone to write their own extensions to alter the existing behavior and/or add new behavior. As the API Documentation can be a little overwhelming when starting out, the following tutorial will step you through the process of getting a simple Inline Processor extension working, then adding more features to it. Various steps will be repeated in different ways to demonstrate various parts of the API.

First, we need to establish the syntax we will be implementing. Rather than re-implement any existing Markdown syntax, lets create some different syntax that is not typical of Markdown. In fact, we'll implement a subset of the inline syntax used by the txt2tags markup language. The syntax looks like this:

Two hyphens for strike: --del-- => <del>del</del> => del
Two underscores for underline: __ins__ => <ins>ins</ins> => ins
Two asterisks for bold: **strong** => <strong>strong</strong> => strong.
Two slashes for italic: //emphasis// => <em>emphasis</em> => emphasis.
Boilerplate Code
The first step is to create the boilerplate code that will be required by any Python-Markdown Extension.

Warning: This tutorial is very generic and makes no assumptions about your development environment. Some of the commands below may generate errors on some (but not all) systems unless they are run by a user who has the correct permissions. To avoid these types of issues, it is suggested that virtualenv be used for development in an environment isolated from your primary system; although doing so is certainly not required. As setting up an appropriate development environment applies to any Python development (developing Markdown extensions adds no additional requirements), it is beyond the scope of this tutorial. A basic understanding of Python development is expected.

First create a new directory to save your extension files to. From the commandline do the following:

mkdir myextension
cd myextension
Be sure to save all files within the "myextension" directory you just created. Note that we are naming the extension "myextension". You may use a different name, but be sure to use whatever name you chose consistently throughout.

Create the first Python file, name it myextension.py, and add the following boilerplate code to it:

from markdown.extensions import Extension

class MyExtension(Extension):
   def extendMarkdown(self, md):
       # Insert code here to change markdown's behavior.
       pass
After saving that file, create a second Python file, name it setup.py, and add the following code to it:

from setuptools import setup
setup(
    name='myextension',
    version='1.0',
    py_modules=['myextension'],
    install_requires = ['markdown>=3.0'],
)
Finally, from the commandline run the following command to tell Python about your new extension:

python setup.py develop
Note that the develop subcommand was run rather than the install subcommand. As the plugin isn't finished yet, this special development mode sets up the path to run the plugin from the source file rather than Python's site-packages directory. That way, any changes made to the file will immediately take effect with no need to re-install the extension.

Also note that the setup script expects that setuptools is installed. While setuptools is not necessary (just do from distutils.core import setup instead), we only get the develop subcommand if we use setuptools. Any system which has pip and/or virtualenv installed (both recommended) will also have setuptools installed.

To ensure that everything is working correctly, try passing the extension to Markdown. Open a python interpreter and try the following:

>>> import markdown
>>> from myextension import MyExtension
>>> markdown.markdown('foo bar', extensions=[MyExtension()])
'<p>foo bar</p>'
Obviously, the extension doesn't do anything useful, but now that we have it in place with no errors, we can actually start implementing our new syntax.

Using Generic Patterns
To start, let's implement the one part of that syntax that doesn't overlap with Markdown's standard syntax; the --del-- syntax, which will wrap the text in <del> tags.

The first step is to write a regular expression to match the del syntax.

DEL_RE = r'(--)(.*?)--'
Note that the first set of hyphens ((--)) are grouped in parentheses, making our content the second group. This is because we will be using a generic pattern class provided by Python-Markdown. Specifically, the SimpleTextPattern which will modify the pattern to prepend another group, and then expects the text content to be found in group(3) of the new regular expression. We add the extra group to force the content we want intogroup(3).

Also note that the content is matched using a non-greedy match (.*?). Otherwise, everything between the first occurrence and the last would all be placed inside one <del> tag, which we do not want.

So, let's incorporate our regular expression into Markdown:

from markdown.extensions import Extension
from markdown.inlinepatterns import SimpleTagPattern

DEL_RE = r'(--)(.*?)--'
    
class MyExtension(Extension):
    def extendMarkdown(self, md):
        # Create the del pattern
        del_tag = SimpleTagPattern(DEL_RE, 'del')
        # Insert del pattern into markdown parser
        md.inlinePatterns.add('del', del_tag, '>not_strong')
If you noticed, we added two lines. The first line creates an instance of a markdown.inlinePatterns.SimpleTagPattern. This generic pattern class takes two arguments; the regular expression to match against (in this case DEL_RE), and the name of the tag to insert the text of group(3) into ('del').

The second line adds our new pattern to the Markdown parser. In the event that it is not obvious, the extendMarkdown method of any markdown.Extension class is passed "md", the instance of the Markdown class we want to modify. In this case, we are inserting a new inline pattern named 'del', using our pattern instance del_tag after the pattern named "not_strong" (thus the '>not_strong').

This time, we used the add method, even though it is deprecated. In a future version, you will first need to determine the actual priority number, looking in inline_patterns.py's build_inlinepatterns(), choosing 75 as a bit before "not_strong", and then using md.inlinePatterns.register(del_tag, 'del', 75).

Now let's test our new extension. Open a python interpreter and try the following:

>>> import markdown
>>> from myextension import MyExtension
>>> markdown.markdown('foo --deleted-- bar', extensions=[MyExtension()])
'<p>foo <del>deleted</del> bar</p>'
Notice that we imported the MyExtension class from the 'myextension' module. We then passed an instance of that class to the extensions keyword of markdown.markdown. We can also see the HTML returned, which would display in the browser as:

foo deleted bar

Let's add our syntax for __ins__, which will use the <ins> tag.

DEL_RE = r'(--)(.*?)--'
INS_RE = r'(__)(.*?)__'
    
class MyExtension(Extension):
    def extendMarkdown(self, md):
        del_tag = SimpleTagPattern(DEL_RE, 'del')
        md.inlinePatterns.add('del', del_tag, '>not_strong')
        ins_tag = SimpleTagPattern(INS_RE, 'ins')
        md.inlinePatterns.add('ins', ins_tag, '>del')
That should be self explanatory. We simply created a new pattern which matches our 'ins' syntax and added it after the 'del' pattern.

We could be done with the 'ins' syntax, except that we now have two possible results defined for text surrounded by double underscores. Recall that Markdown's existing bold syntax (__bold__) is still defined in the parser. However, as our new insert syntax was inserted in the inlinePatterns before the bold pattern, the insert pattern runs first and consumes the double underscore markup before the bold pattern ever has a chance to find it. Even so, the existing bold pattern is still being run against the text and slowing down the parser unnecessarily. Therefore, it is always good practice to remove any parts that are no longer needed.

However, as we will be defining our own new bold syntax, we can actually override or replace the old pattern with our new one. The same applies to our emphasis pattern.

First, we need to define our new regular expressions. We can use the same expressions from last time with a few modifications.

STRONG_RE = r'(\*\*)(.*?)\*\*'
EMPH_RE = r'(\/\/)(.*?)\/\/'
Now we need to insert these into the markdown parser. However, unlike with insert and delete, we need to override the existing inline patterns. Markdown's strong and emphasis syntax is currently implemented with two inline patterns; 'em_strong' (for asterixes) and 'em_strong2' (for underscores).

Let's override 'em_strong' first.

class MyExtension(Extension):
    def extendMarkdown(self, md):
        ...
        # Create new strong pattern
        strong_tag = SimpleTagPattern(STRONG_RE, 'strong')
        # Override existing strong pattern
        md.inlinePatterns['em_strong'] = strong_tag
Notice that rather than "add"ing a new pattern before or after an existing pattern, we simple reassigned the value of a pattern named 'em_strong'. This is because the old pattern named 'strong' already existed and we don't need to change its location in the parser. So we simply assign a new pattern instance to it. Like, add(), this method is deprecated, so you may need to md.inlinePatterns.register(strong_tag, 'em_strong', 60) later.

We can set 'emphasis' by assigning it as well. It will get a default priority of very low:

class MyExtension(Extension):
    def extendMarkdown(self, md):
        ...
        emph_tag = SimpleTagPattern(EMPH_RE, 'em')
        md.inlinePatterns['emphasis'] = emph_tag
Now we have one old pattern left, 'em_strong2'. The 'em_strong2' pattern just handled underscores, including the special case that under_scored_words are not emphasis, but as our new syntax requires double underscores, it's not needed any more. Therefore, we can delete it. With the Markdown syntax, due to both strong and emphasis using the same characters, special cases were needed to match the two nested together (i.e.: ___like this___ or ___like_this__). Again this isn't needed for our new syntax. We can delete it by deregistering it:

class MyExtension(markdown.Extension):
    def extendMarkdown(self, md):
        ...
        md.inlinePatterns.deregister('em_strong2')
That implements all of our new syntax. For completeness, the entire extension should look like this:

from markdown.extensions import Extension
from markdown.inlinepatterns import SimpleTagPattern

DEL_RE = r'(--)(.*?)--'
INS_RE = r'(__)(.*?)__'
STRONG_RE = r'(\*\*)(.*?)\*\*'
EMPH_RE = r'(\/\/)(.*?)\/\/'

class MyExtension(Extension):
    def extendMarkdown(self, md):
        del_tag = SimpleTagPattern(DEL_RE, 'del')
        md.inlinePatterns.add('del', del_tag, '>not_strong')
        ins_tag = SimpleTagPattern(INS_RE, 'ins')
        md.inlinePatterns.add('ins', ins_tag, '>del')
        strong_tag = SimpleTagPattern(STRONG_RE, 'strong')
        md.inlinePatterns['em_strong'] = strong_tag
        emph_tag = SimpleTagPattern(EMPH_RE, 'em')
        md.inlinePatterns['emphasis'] = emph_tag
        md.inlinePatterns.deregister('em_strong2')
And to make sure it is working properly, run the following from the Python interpreter:

>>> import markdown
>>> from myextension import MyExtension
>>> txt = """
... Some __underline__
... Some --strike--
... Some **bold**
... Some //italics//
... """
... 
>>> markdown.markdown(txt, extensions=[MyExtension()])
"<p>Some <ins>underline</ins>\nSome <del>strike</del>\nSome <strong>bold</strong>\nSome <em>italics</em>"
Creating your own Pattern Class
However, you may notice that there is a lot of repetition in that code. In fact, all four of our new regular expressions could easily be condensed into one regular expression. And having only one pattern to run would be more performant that four.

Let's refactor our four regular expressions into one new expression:

MULTI_RE = r'([*/_-]{2})(.*?)\2'
Note the regular expression will be modified to capture one group first, so this can be read as 'get two matching punctuation marks as group 2, the tagged text as group 3, and then another copy of the punctuation marks'.

As no generic pattern class exists that will be able to use that regular expression, we will need to define our own. All pattern classes should inherit from the markdown.inlinepatterns.Pattern base class. At the very least, our subclass should define a handleMatch method which accepts a regex MatchObject and returns an ElementTree Element.

from markdown.inlinepatterns import Pattern
from markdown.extensions import Extension
import xml.etree.ElementTree as etree

class MultiPattern(Pattern):
    def handleMatch(self, m):
        if m.group(2) == '**':
            # Bold
            tag = 'strong'
        elif m.group(2) == '//':
            # Italics
            tag = 'em'
        elif m.group(2) == '__':
            # Underline
            tag = 'ins'
        else:   # must be m.group(2) == '--':
            # Strike
            tag = 'del'
        # Create the Element
        el = etree.Element(tag)
        el.text = m.group(3)
        return el
Now we need to tell Markdown about our new pattern and delete the now unnecessary existing patterns:

class MultiExtension(Extension):
    def extendMarkdown(self, md):
        # Delete the old patterns
        md.inlinePatterns.deregister('em_strong')
        md.inlinePatterns.deregister('em_strong2')
        md.inlinePatterns.deregister('not_strong')

        # Add our new MultiPattern
	multi = MultiPattern(MULTI_RE)
        md.inlinePatterns['multi'] = multi
For completeness, the newly added code should look like this:

from markdown.inlinepatterns import Pattern
from markdown.extensions import Extension
import xml.etree.ElementTree as etree

MULTI_RE = r'([*/_-]{2})(.*?)\2'

class MultiPattern(Pattern):
    def handleMatch(self, m):
        if m.group(2) == '**':
            # Bold
            tag = 'strong'
        elif m.group(2) == '//':
            # Italics
            tag = 'em'
        elif m.group(2) == '__':
            # Underline
            tag = 'ins'        
        else:   # must be m.group(2) == '--':
            # Strike
            tag = 'del'
        # Create the Element
        el = etree.Element(tag)
        el.text = m.group(3)
        return el

class MultiExtension(Extension):
    def extendMarkdown(self, md):
        # Delete the old patterns
        md.inlinePatterns.deregister('em_strong')
        md.inlinePatterns.deregister('em_strong2')
        md.inlinePatterns.deregister('not_strong')

        # Add our new MultiPattern
        multi = MultiPattern(MULTI_RE)
        md.inlinePatterns['multi'] = multi
After adding that code to the myextension.py file, open the Python interpreter:

>>> import markdown
>>> from myextension import MultiExtension
>>> txt = """
... Some __underline__
... Some --strike--
... Some **bold**
... Some //italics//
... """
... 
>>> markdown.markdown(txt, extensions=[MultiExtension()])
"<p>Some <ins>underline</ins>\nSome <del>strike</del>\nSome <strong>bold</strong>\nSome <em>italics</em>"
Adding Config Options
Now suppose that we want to offer some configuration options to our extension. Perhaps we want to only offer the insert and delete syntax as an option which the user can turn on and off.

To start, let's break our regular expression into two:

STRONG_EM_RE = r'([*/]{2})(.*?)\2'
INS_DEL_RE = r'([_-]{2})(.*?)\2'
Then, we need to define our config options on our newly renamed Extension subclass:

class ConfigExtension(Extension):
    def __init__(self, **kwargs):
        # Define config options and defaults
        self.config = {
            'ins_del': [False, 'Enable Insert and Delete syntax.']
        }
        # Call the parent class's __init__ method to configure options
        super().__init__(**kwargs)
We defined our config options as the dict, self.config with keys being the names of the options. Each value is a two item list, the default value of the option and its description. We use a list instead of a tuple because the Extension class requires config to be mutable.

Finally, refactor the extendMarkdown method to account for the config option:

    def extendMarkdown(self, md):
        ...
        # Add STRONG_EM pattern
        strong_em = MultiPattern(STRONG_EM_RE)
        md.inlinePatterns['strong_em'] = strong_em
        # Add INS_DEL pattern if active
        if self.getConfig('ins_del'):
            ins_del = MultiPattern(INS_DEL_RE)
            md.inlinePatterns['ins_del'] = ins_del
We simply created one instance of our MultiPattern class for strong and emphasis, and if the 'ins_del' config option is True, we create a second instance of the MultiPattern class.

For completeness, all of the newly added code should look like this:

STRONG_EM_RE = r'([*/]{2})(.*?)\2'
INS_DEL_RE = r'([_-]{2})(.*?)\2'

class ConfigExtension(Extension):
    def __init__(self, **kwargs):
        # Define config options and defaults
        self.config = {
            'ins_del': [False, 'Enable Insert and Delete syntax.']
        }
        # Call the parent class's __init__ method to configure options
        super().__init__(**kwargs)
        
    def extendMarkdown(self, md):
        # Delete the old patterns
        md.inlinePatterns.deregister('em_strong')
        md.inlinePatterns.deregister('em_strong2')
        md.inlinePatterns.deregister('not_strong')

        # Add STRONG_EM pattern
        strong_em = MultiPattern(STRONG_EM_RE)
        md.inlinePatterns['strong_em'] = strong_em
        # Add INS_DEL pattern if active
        if self.getConfig('ins_del'):
            ins_del = MultiPattern(INS_DEL_RE)
            md.inlinePatterns['ins_del'] = ins_del
After saving your changes, open the Python interpreter:

>>> import markdown
>>> from myextension import ConfigExtension
>>> txt = """
... Some __underline__
... Some --strike--
... Some **bold**
... Some //italics//
... """
... 
>>> # First try it with ins_del set to True
>>> markdown.markdown(txt, extensions=[ConfigExtension(ins_del=True)])
"<p>Some <ins>underline</ins>\nSome <del>strike</del>\nSome <strong>bold</strong>\nSome <em>italics</em>"
>>> # Now try it with ins_del defaulting to False
>>> markdown.markdown(txt, extensions=[ConfigExtension()])
"<p>Some __underline__\nSome --strike--\nSome <strong>bold</strong>\nSome <em>italics</em>"
Supporting Extension Names (as strings)
You may have noted that each time we tested our extension, we had to import the extension and pass in an instance of the Extension subclass. While this is the prefered way to call extensions, at times a user may need to call Markdown from the command line or a templating system, and may only be able to pass in strings.

This feature is built-in for free. However, your users will need to know and use the import path (Python dot notation) of the Extension class you defined. For example, each of the three classes we defined above would be called like this:

>>> markdown.markdown(txt, extensions=['myextension:MyExtension'])
>>> markdown.markdown(txt, extensions=['myextension:MultiExtension'])
>>> markdown.markdown(txt, extensions=['myextension:ConfigExtension'])
Note that a colon (:) must be used between the path and the Class. Whereas a dot (.) must be used for the rest of the path. Think of it as replacing the import part of the "from" import statement with the colon. For example, if you had an extension class, FooExtension, defined in the file somepackage/extensions/foo.py, then the import statement would be from somepackage.extensions.foo import FooExtension and the string based name would be 'somepackage.extensions.foo:FooExtension'.

In fact, if you created a new class in each of the steps above rather than refactoring the previous one, all three extensions could live within the same module and still all be called separately. This works great when you have built a number of extensions as part of a larger project (perhaps a CMS, a static blog generator, etc) that will only be used internally.

However, if you intend to distribute your extension as a standalone module for others to incorporate into their projects, you may want to enable support for a shorter name. No doubt, 'myextension' is easier for your users to type (and you to document) than 'myextension:MyExtension'. And as all of the built-in extensions that ship with Python-Markdown work this way, users will likely expect the same. To enable this feature, add the following to the bottom of your extension:

def makeExtension(**kwargs):
    return ConfigExtension(**kwargs)
Note that this module level function simply returns an instance of your Extension subclass. When Markdown is provided with a string, it expects that string to use Python's dot notation pointing to the importable path of the module. Then if no colon is found in the string, it calls the makeExtension function found in that module.

Let's test our extension by opening the python interpreter again:

>>> import markdown
>>> txt = """
... Some __underline__
... Some --strike--
... Some **bold**
... Some //italics//
... """
... 
>>> markdown.markdown(txt, extensions=['myextension'])
"<p>Some __underline__\nSome --strike--\nSome <strong>bold</strong>\nSome <em>italics</em>"
As we used the ConfigExtension above, let's pass some config options to the extension:

>>> markdown.markdown(
... 	txt, 
... 	extensions=['myextension'],
... 	extension_configs = {
... 		'myextension': {'ins_del': True}
... 	}
...	)
"<p>Some <ins>underline</ins>\nSome <del>strike</del>\nSome <strong>bold</strong>\nSome <em>italics</em>"
Notice that we got support for the extension_configs keyword with no extra work. See the documentation for a full explanation of the extension_configs keyword.

Preparing for Distribution
As a setup.py script has already been created, the most important part of preparing an extension for distribution is completed. However, the setup script was pretty basic. It is recommended that a little more metadata be included, in particular the developer's name, email address and a URL for the project (see the section Writing the Setup Script of the Python documentation for an example). It is also suggested that at a minimum README and LICENCE files be included in the directory.

At this point, you could commit your code to a version control system (such as Git, Mercurial, Subversion or Bazaar) and upload it to a host which supports your system of choice. Then your users can easily use a pip command to download and install your extension.

Or, for an even simpler command, you could upload your project to the Python Package Index. Alternatively, you could use some of the subcommands (such as sdist) available on the setup.py script to create a file (such as a zip or tar file) to provide for your users to download. While the specifics are beyond the scope of this tutorial, the Python documentation on Distributing Python Modules and the Setuptools Documentation on Building and Distributing Packages both offer explanations of the options available.

Conclusion
While this tutorial only demonstrated use of Inline Processors, the extension API also includes support for Preprocessors, Block Processors, Tree Processors and Postprocessors. Even though each type of processor serves a different purpose---running at a different stage in the parsing process---the same basic principles apply to each type of processor. In fact, a single extension can alter multiple different types of processors.

Reviewing the API Documentation and the source code of the various built-in extensions should provide you with enough information to build you own great extensions. Of course, if you would like assistance, feel free to ask for help on the mailing list. And please, don't forget to list your extensions on the wiki so other people can find them.
