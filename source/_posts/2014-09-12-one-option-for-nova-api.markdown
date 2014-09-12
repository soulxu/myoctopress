---
layout: post
title: "Some ideas for Nova API in Kilo"
date: 2014-09-12 12:37:17 +0800
comments: true
categories: 
---

Close to Kilo, it is time to think about what next for nova API. Let me
write down some ideas from me at here, although I'm not sure those idea can be
accepted finally, but really hope those can give some help.

Those ideas are for micro-version and policy.

Micro-version
=============
When we propose v2 on v3, the propose already include a series method for
supporting multiple version in API. Those methods can be used in
micro-version. But in finally those methods was considered too complex or too heavy.
(The previous propose https://review.openstack.org/#/c/84695/19/specs/juno/v2-on-v3-api.rst)

Think of use-cases for API change:

* Input changing
* Output changing
* Semantic changing
* Status code changing
* Resource name changing

For the change of status code, and resource name are really simple to implement.

For semantic changes, there isn't choice we must provide two version python code
in api layer. We propose method for distinguish different semantic in v2 on v3,
but it have too much magic, so think about to simplify it.

The most of complex part is input and output change. And in v2 on v3 propose,
this part also implement as most complex. We provide method to translate the input
and json-schema, but that make the json-schema hard to maintenance and read.

The new ideas is want to provide simple implementation for input/output change, and
reduce the magic we used.

Actually there are already done some POC for those ideas:

 * The PoC with mapping input/output into flat dict:
       https://github.com/soulxu/nova-v3-api-doc/commits/new_idea4
 * The PoC with mapping input/output into nova object:
       https://github.com/soulxu/nova-v3-api-doc/tree/new_idea_with_obj2

The frist 3 commits are implement wsgi infrastruct as new ideas.
The last 4 commits are demo the use-case in micro-version.

Let's take a look at some example
------------------------------------

There is fake code for an extension that add new action to server. This fake code
also includes some usual inconsistent and mistake in v2 API that we want to fix in
the future. Anyway most of API code structure looks like as below:

{% codeblock lang:python %}

class SomeActionController(wsgi.Controller):

    def __init__(self, *args, **kwargs):
        self.compute_api = compute.API()

    @extensions.expected_error(())
    @wsgi.response(204)
    @validation.schema(some_action.some_action)
    def some_action(self, req, id, body):
        context = req.environ["nova.context"]
        action_body = body["someAction"]
        param_a = action_body["paramA"]

        instance = common.get_instance(self.compute_api, context, id,
                                       want_objects=True)
        authorize(context, target=instance, action="some_action")
        res = self.compute_api.some_action(context, instance, param_a)
        return {"ActionResult": {"resultA": res}}

{% endcodeblock %}

Also need related json-schema

{% codeblock lang:python %}

some_action = {
    'type': 'object',
    'properties': {
        'someAction': {
            'type': 'object',
            'properties': {
                'paramA': {'type': 'string'}
            },
            'required': ['paramA'],
            'additionalProperties': False,
        },
    },
    'required': ['someAction'],
    'additionalProperties': False,
}

{% endcodeblock %}

Change the API with the method in v2 on v3 propose
--------------------------------------------------

For fix the status code, we just need change the decorator to indict
the specified version status code.

For fix the inconsistent in the input, we need write a translation-schema
file (looks like json-schema).

And for fix the inconsistent in the output, we need change the python code directly.

{% codeblock lang:python %}

base = {
    'request_body': {
        'some_action': {'rename_to': 'someAction'}
        'param_a': {'rename_to': 'paramA'}
    },
    'response_body': {
        'action_result': {'rename_to': 'actionResult'}
        'result_a': {'rename_to': 'resultA'}
    }
}

{% endcodeblock %}

Also need add decorator for api python method.

{% codeblock lang:python %}

    ....

    @extensions.expected_error(())
    @wsgi.response(204, version="2.1")
    @wsgi.response(202, version="3.0")
    @wsgi.v2_translate_body(gap.base)
    @validation.schema(some_action.some_action)
    def some_action(self, req, id, body):
        context = req.environ["nova.context"]
        action_body = body["someAction"]
        param_a = action_body["paramA"]

        ...

        return {"action_result": {"result_a": res}}

{% endcodeblock %}


Before wsgi invoke API python code, the decorator 'v2_translate_body' should translate
input and json-schema as the old one that make API python code can understand the
request. That means API python code always assume the user request is oldest format.

There are two downsides with this method:

* The implementation is complex

  Actually not only need translate the request body, also need translate the json-schema
  that used to validate the input with new format, and also for the output.

  And not only support 'rename_to' instruct, also need support 'move_to' instruct for
  the case like scheduler_hints.

  (The scheduler_hint param is out of the server struct in the request, that is wrong.
   https://github.com/openstack/nova/blob/master/doc/api_samples/OS-SCH-HNT/scheduler-hints-post-req.json)

* The multiple version json-schema is unreadable and hard to maintenance by dev

  When there already doing multiple version changes for the request format, the translation-schema is
  dependence on one by one. Except there is json-schema file for the oldest format, other version
  json-schema only can  get by translation. The dev is no way to get specified version json-schema,
  except they runing the code and print it out.

And except the method input/output change, we also provide some magic for distinguish different semantic.
(https://review.openstack.org/82301). But it also make the code hard to read.

The new method for support multiple version
-------------------------------------------

Thinking of the key reason of make those complex thing is the python code also need
input/output format knowledge. Actually, the json-schema already includes those
information.

The idea of new method for input/output change is just eliminate the format knowledge
from python code. Whatever the format it is, python code just need got a flat dict
without any format.

(And other idea is got a nova objects, and I didn't think out of enough benefit from
using nova object, but this idea is worth to think about. Honestly I'm not familiar
with objects, maybe the POC with nova object is bad)

Then after the input/output format is changed, we just need update the json-schema,
we needn't any modify for API python code.

Note that with this method, we need json-schema for response also, there already
have plan to move response json-schema into nova. So that won't be an extra work.

After make the python code only accept a flat dict instead of whole request body,
the code need some simple change as below:

(PoC with evacuate api as example: https://github.com/soulxu/nova-v3-api-doc/commit/2d8489650ba221a3019b7db548ef8ce7a8a186dd)

{% codeblock lang:python %}

    @wsgi.version(base='2.1')
    @extensions.expected_error(())
    @wsgi.response(204)
    @validation.schema(some_action)
    def some_action(self, req, id, params):
        context = req.environ["nova.context"]
        param_a = params["param_a"]

        instance = common.get_instance(self.compute_api, context, id,
                                       want_objects=True)
        authorize(context, target=instance, action="some_action")
        res = self.compute_api.some_action(context, instance, param_a)

        return {"result_a": res}

{% endcodeblock %}

And change for json-schema:

{% codeblock lang:python %}

some_action_input_2_1 = {
    'type': 'object',
    'properties': {
        'someAction': {
            'type': 'object',
            'properties': {
                'paramA': {'type': 'string'}
            },
            'required': ['paramA'],
            'additionalProperties': False,
        },
    },
    'required': ['someAction'],
    'additionalProperties': False,
    'ext:mapping': {'paramA': 'param_a'}
}

some_action_output_2_1 = {
    'type': 'object',
    'properties': {
        'actionResult': {
            'type': 'object',
            'properties': {
                'resultA': {'type': 'string'}
            },
            'required': ['resultA'],
            'additionalProperties': False,
        },
    },
    'required': ['actionResult'],
    'additionalProperties': False,
    'ext:mapping': {'resultA': 'result_a'}
}

{% endcodeblock %}

Change API with new idea
------------------------

Let's try some use-case to demo this idea. Frist fixed the inconsistent and status code.

(PoC with evacuate api as example: https://github.com/soulxu/nova-v3-api-doc/commit/295a28fdf435f23bfa942fc3d9d46716caee29b8)

We needn't doing anything for python code, except bump the version in the
decorator:

{% codeblock lang:python %}

    @wsgi.version(base='2.1', max='3.0')
    @extensions.expected_error(())
    @wsgi.response(204, version='2.1')
    @wsgi.response(202, version='3.0')
    @validation.schema(some_action)
    def some_action(self, req, obj, params):
        ....

{% endcodeblock %}

Next just need copy the v2.1 json-schema as v3.0, and doing a little change.

{% codeblock lang:python %}

some_action_input_3_0 = {
    'type': 'object',
    'properties': {
        'some_action': {
            'type': 'object',
            'properties': {
                'param_a': {'type': 'string'}
            },
            'required': ['param_a'],
            'additionalProperties': False,
        },
    },
    'required': ['some_action'],
    'additionalProperties': False,
    'ext:mapping': {'param_a': 'param_a'}
}

some_action_output_3_0 = {
    'type': 'object',
    'properties': {
        'action_result': {
            'type': 'object',
            'properties': {
                'result_a': {'type': 'string'}
            },
            'required': ['result_a'],
            'additionalProperties': False,
        },
    },
    'required': ['action_result'],
    'additionalProperties': False,
    'ext:mapping': {'result_a': 'result_a'}
}

{% endcodeblock %}

The code looks more. But actually the most of work is just copy the 2.1 json-schema
as 3.0, and change the inconsistent in the 3.0 json-schema. I chose just put the each
version json-schema in the file, and naming with version. That make the json-schema
easy to maintenance and more readable for dev. And the wsgi infrastruct implementation
is more simple.

There are more things need to be explaned.

* The mapping info:

{% codeblock lang:python %}
    'ext:mapping': {'result_a': 'result_a'}
{% endcodeblock %}

The wsgi code will use this mapping info to map the input/output into/from flat dict
(or nova objects).

Actually this is also kind of translation. But without format knowledge in the
python code, this reduce the complex in our wsgi infrastruct. We just need implement
tranlsation to mapping the input into flat dict (nova objects). We needn't support
different kind of instruct, and needn't to translate the json-schema.

* The decorator wsgi.response

{% codeblock lang:python %}
    @wsgi.version(base='2.1', max='3.0')
{% endcodeblock %}

This decorator means the request version between 2.1 and 3.0 will be routed into
this function. If there isn't support any version between 2.1 and 3.0, there also
no json-schema for that version, the wsgi code will return error for the user.

Make Semantic Change
--------------------

Another use-case is the semantic change.

(PoC with evacuate API as example:
 https://github.com/soulxu/nova-v3-api-doc/commit/704118c9e69891dd32729903b348bf85ab136a72)

If there is semantic change happened, the code will be looks like as below:

{% codeblock lang:python %}

    @wsgi.version(base='2.1', max='3.3')
    @extensions.expected_error(())
    @wsgi.response(204, version='2.1')
    @wsgi.response(202, version='3.0')
    @validation.schema(some_action)
    def some_action(self, req, id, params):
        ....


    @wsgi.version(base='4.0', max='4.5')
    @extensions.expected_error(())
    @wsgi.response(202, version='4.0')
    @validation.schema(some_action)
    def some_action(self, req, id, params):
        ....

{% endcodeblock %}

Those code means verison v2.1~v3.3 will execute frist some_action function,
v4.0~v4.5 will execute second some_action function.

We only write new function for same api when we have semantic change. This is
more readable than we execute different version internal function in same api function.
(compare to https://review.openstack.org/82301)

And also easy to maintenance than we add function for each API version (Add function
for each API version come from micro-version discussion previously).

If there are duplicate code between two version some_action function, we just need
move the duplicate code into common function as we share code in normally.

The complete demo for user-case with real API is in the POC last 4 commits.

* Fix inconsistent for evacuate: https://github.com/soulxu/nova-v3-api-doc/commit/295a28fdf435f23bfa942fc3d9d46716caee29b8
* Add new fake parameter for evacuate: https://github.com/soulxu/nova-v3-api-doc/commit/04896d1109b6ce9d213eaf1f270a8c8c34ca04c6
* Fix status code for evacuate: https://github.com/soulxu/nova-v3-api-doc/commit/3ca3ebc9a3c17f5694918958f8bc8c5009f821d1
* Fake semantic change for evcaute: https://github.com/soulxu/nova-v3-api-doc/commit/704118c9e69891dd32729903b348bf85ab136a72

With those method, hope we got more easy maintenance, more readable, simply to implement
for micro-version.

Policy enforcement
==================

We already have some plan for improve the nova api policy enforcement.
Some of them block by the v2 on v3 discussion, but those will continue in K (but still low priority)

* Enforce policy at Nova REST API layer: https://review.openstack.org/92005
* Provide separated policy rule for each API: https://review.openstack.org/92326
* Support policy configuration directoy: https://review.openstack.org/105362 (Already merged in oslo-incubator)

After we provide separated policy rule for all the API. There is one more thing
we can do. It is about doing the policy enforcemance by wsgi infrastructure.
Currently the each policy enforcement need write one line code in the api layer:

{% codeblock lang:python %}
authorize(context, target=instance, action="some_action")
{% endcodeblock %}

We can eliminate this duplicated code.

So hope the wsgi infrasturcture can doing that. For policy enforcement, policy rule
need check with 'target'. So there need way let wsgi infrastruct to know how to
generate the target for policy checks.

So the API code will looks like as below:

{% codeblock lang:python %}

class SomeActionController(wsgi.Controller):
    resource_obj_cls = objects.Instance

    def __init__(self, *args, **kwargs):
        self.compute_api = compute.API()

    @wsgi.version(base='2.1')
    @extensions.expected_error(())
    @wsgi.response(204)
    @validation.schema(some_action)
    def some_action(self, req, obj, params):
        context = req.environ["nova.context"]
        param_a = params["param_a"]
        res = self.compute_api.some_action(context, instance, param_a)
        return {"result_a": res}

{% endcodeblock %}

The api code will accept nova object instead of resource id. Wsgi code will know
to instance the target by resource_obj_cls. Then the policy enforcement can be
done by wsgi code.

I didn't write some PoC code for this yet. I will write some code to prove this
idea works later.


