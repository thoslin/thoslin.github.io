---
layout: post
title: Dive into Django template engine
published: false
---
### Motives
This is an excerpt from Django documentation [Writing custom template tags](https://docs.djangoproject.com/en/dev/howto/custom-template-tags/#writing-custom-template-tags)
{% blockquote %}
Above, this document explained that the template system works in a two-step process: compiling and rendering. To define a custom template tag, you specify how the compilation works and how the rendering works.

When Django compiles a template, it splits the raw template text into ‘’nodes’‘. Each node is an instance of django.template.Node and has a render() method. A compiled template is, simply, a list of Node objects. When you call render() on a compiled template object, the template calls render() on each Node in its node list, with the given context. The results are all concatenated together to form the output of the template.

Thus, to define a custom template tag, you specify how the raw template tag is converted into a Node (the compilation function), and what the node’s render() method does.
{% endblockquote %}

The above is the explanation from django docs about how django template works and how to write a custom template tag. Honestly the first time I skimmed through this piece I didn't quite get it. and I just skipped it. While django provides decorators like simple_tag, inlucsion_tag which ease the process of writing templatetags. I found no difficulty in using templatetags. So it's like a black box put away in the dungeon which I never bother to open up again.

But a few weeks ago when I was reading some templatetag code from a github repo. I found it hard to get what's going on. I was baffled. And this was not the first time it happened. Also as a self-proclaimed djangonaut : D. I feel this is humiliating. I feel the need to open that box and release the black magic.

### Let the code speak
{% codeblock lang:python %}
class Template(object):
    def __init__(self, template_string, origin=None,
                 name='<Unknown Template>'):
        try:
            template_string = smart_unicode(template_string)
        except UnicodeDecodeError:
            raise TemplateEncodingError("Templates can only be constructed "
                                        "from unicode or UTF-8 strings.")
        if settings.TEMPLATE_DEBUG and origin is None:
            origin = StringOrigin(template_string)
        self.nodelist = compile_string(template_string, origin)
        self.name = name

    def __iter__(self):
        for node in self.nodelist:
            for subnode in node:
                yield subnode

    def _render(self, context):
        return self.nodelist.render(context)

    def render(self, context):
        "Display stage -- can be called many times"
        context.render_context.push()
        try:
            return self._render(context)
        finally:
            context.render_context.pop()

class NodeList(list):
    # Set to True the first time a non-TextNode is inserted by
    # extend_nodelist().
    contains_nontext = False

    def render(self, context):
        bits = []
        for node in self:
            if isinstance(node, Node):
                bit = self.render_node(node, context)
            else:
                bit = node
            bits.append(force_unicode(bit))
        return mark_safe(u''.join(bits))

    def get_nodes_by_type(self, nodetype):
        "Return a list of all nodes of the given type"
        nodes = []
        for node in self:
            nodes.extend(node.get_nodes_by_type(nodetype))
        return nodes

    def render_node(self, node, context):
        return node.render(context)
{% endcodeblock %}

This class quite explains the two-step process: compling and rendering. Basically template is broken down into a list of Node objects after compling and then django will iterate through the node list and call render method on all Node objects and join the results. That's the rendering.

As compared to rendering process, compling is a more complicated process, during which template strings are translated into meaningful Python code. There are also two steps in compiling. First Lexer would tear the template string apart into small pieces(Token objects) based on predefined tokens like
{% raw %}"{{" "{%"
{% endraw %} for further processing.

{% codeblock lang:python %}
{% raw %}
In [1]: from django.template.base import Lexer, Parser, Template

In [2]: lexer = Lexer("Hello {{ username }}, {% block content%}Welcome{% endblock %}", None)

In [3]: tokens = lexer.tokenize()

In [4]: for token in tokens:
   ...:     print token
   ...:     
<Text token: "Hello ...">
<Var token: "username...">
<Text token: ", ...">
<Block token: "block content...">
<Text token: "Welcome...">
<Block token: "endblock...">
{% endraw %}
{% endcodeblock %}

Parser then picks up those tokens and change them into corresponding Node objects.

{% codeblock lang:python %}
{% raw %}
In [5]: parser = Parser(tokens)

In [6]: parser.parse()
Out[6]: 
[<Text Node: 'Hello '>,
 <Variable Node: username>,
 <Text Node: ', '>,
 <Block Node: content. Contents: [<Text Node: 'Welcome'>]>]
{% endraw %}
{% endcodeblock %}

Let's take a look at Parser class:
{% codeblock lang:python %}
class Parser(object):
    def __init__(self, tokens):
        self.tokens = tokens
        self.tags = {}
        self.filters = {}
        for lib in builtins:
            self.add_library(lib)
    ...
{% endcodeblock %}

See that filters and tags. when initializing the parser, both default and custom template filters and template tags are loaded.
When calling the parse method parser iterates through tokens and calls proper handler from self.tags and self.filters depending on token type. Here is where template filter and tags fit themself in.

For example when a <Block token: "block content..."> is met, parser would call self.tags['block']， pass itself and token to the handler and expect a Node object returned.

{% codeblock lang:python %}
@register.tag('block')
def do_block(parser, token):
    """
    Define a block that can be overridden by child templates.
    """
    bits = token.contents.split()
    ...
    return BlockNode(block_name, nodelist)
{% endcodeblock %}

So this is what a template tag should look like. Accept parser and token and return a Node. And what a Node class has to do is to implement a render method.

For more details, check the code lives in django/template/base.py and 
recheck [the docs](https://docs.djangoproject.com/en/dev/howto/custom-template-tags/#writing-custom-template-tags). It should make more sense.

### Not deep enough

Well. I just get this deep so far : ) If you want more there's an [interesting blogpost](http://www.pocoo.org/~blackbird/django-templates-blogpost.html) written by Armin Rocher, the author of [Jinja2](https://github.com/mitsuhiko/jinja2), explaining why django template is slow. Come on. Let's dive in.
