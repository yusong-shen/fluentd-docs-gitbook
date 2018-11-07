record\_transformer Filter Plugin
=================================

The `filter_record_transformer` filter plugin mutates/transforms
incoming event streams in a versatile manner. If there is a need to
add/delete/modify events, this plugin is the first filter to try.


Example Configurations
----------------------

`filter_record_transformer` is included in Fluentd's core. No
installation required.

``` {.CodeRay}
<filter foo.bar>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    tag ${tag}
  </record>
</filter>
```

The above filter adds the new field "hostname" with the server's
hostname as its value (It is taking advantage of Ruby's string
interpolation) and the new field "tag" with tag value. So, an input like

``` {.CodeRay}
{"message":"hello world!"}
```

is transformed into

``` {.CodeRay}
{"message":"hello world!", "hostname":"db001.internal.example.com", "tag":"foo.bar"}
```

Here is another example where the field "total" is divided by the field
"count" to create a new field "avg":

``` {.CodeRay}
<filter foo.bar>
  @type record_transformer
  enable_ruby
  <record>
    avg ${record["total"] / record["count"]}
  </record>
</filter>
```

It transforms an event like

``` {.CodeRay}
{"total":100, "count":10}
```

into

``` {.CodeRay}
{"total":100, "count":10, "avg":"10"}
```

With the `enable_ruby` option, an arbitrary Ruby expression can be used
inside `${...}`. Note that the "avg" field is typed as string in this
example. You may use `auto_typecast true` option to treat the field as a
float.

You can also use this plugin to modify your existing fields as

``` {.CodeRay}
<filter foo.bar>
  @type record_transformer
  <record>
    message yay, ${record["message"]}
  </record>
</filter>
```

An input like

``` {.CodeRay}
{"message":"hello world!"}
```

is transformed into

``` {.CodeRay}
{"message":"yay, hello world!"}
```

Finally, this configuration embeds the value of the second part of the
tag in the field "service\_name". It might come in handy when
aggregating data across many services.

``` {.CodeRay}
<filter web.*>
  @type record_transformer
  <record>
    service_name ${tag_parts[1]}
  </record>
</filter>
```

So, if an event with the tag "web.auth" and record
`{"user_id":1, "status":"ok"}` comes in, it transforms it into
`{"user_id":1, "status":"ok", "service_name":"auth"}`.

Parameters
----------

### \<record\> directive

Parameters inside `<record>` directives are considered to be new
key-value pairs:

``` {.CodeRay}
<record>
  NEW_FIELD NEW_VALUE
</record>
```

For NEW\_FIELD and NEW\_VALUE, a special syntax `${}` allows the user to
generate a new field dynamically. Inside the curly braces, the following
variables are available:

-   The incoming event's existing values can be referred by their field
    names. So, if the record is `{"total":100, "count":10}`, then
    `record["total"]=100` and `record["count"]=10`.
-   `tag` refers to the whole tag.
-   `time` refers to stringanized event time.
-   `hostname` refers to machine's hostname. The actual value is result
    of
    [Socket.gethostname](https://docs.ruby-lang.org/en/trunk/Socket.html#method-c-gethostname).

You can also access to a certain potion of a tag using the following
notations:

-   `tag_parts[N]` refers to the Nth part of the tag.
-   `tag_prefix[N]` refers to the \[0..N\] part of the tag.
-   `tag_suffix[N]` refers to the \[N..\] part of the tag.

All indices are zero-based. For example, if you have an incoming event
tagged `debug.my.app`, then `tag_parts[1]` will represent "my". Also in
this case, `tag_prefix[N]` and `tag_suffix[N]` will work as follows:

``` {.CodeRay}
tag_prefix[0] = debug          tag_suffix[0] = debug.my.app
tag_prefix[1] = debug.my       tag_suffix[1] = my.app
tag_prefix[2] = debug.my.app   tag_suffix[2] = app
```

### enable\_ruby (optional)

When set to true, the full Ruby syntax is enabled in the `${...}`
expression. The default value is false.

With `true`, additional variables could be used inside `${}`.

-   `record` refers to the whole record.
-   `time` refers to event time as Time object, not stringanized event
    time.

Here is the examples:

``` {.CodeRay}
jsonized_record ${record.to_json}
avg ${record["total"] / record["count"]}
formatted_time ${time.strftime('%Y-%m-%dT%H:%M:%S%z')}
escaped_tag ${tag.gsub('.', '-')}
last_tag ${tag_parts.last}
foo_${record["key"]} bar_${record["value"]}
```

### auto\_typecast (optional)

Automatically cast the field types. Default is false.

LIMITATION: This option is effective only for field values comprised of
a single placeholder.

Effective Examples:

``` {.CodeRay}
foo ${record["foo"]}
```

Non-Effective Examples:

``` {.CodeRay}
foo ${record["foo"]}${record["bar"]}
foo ${record["foo"]}bar
foo 1
```

Internally, this keeps the original value type only when a single
placeholder is used.

### renew\_record (optional)

By default, the record transformer filter mutates the incoming data.
However, if this parameter is set to true, it modifies a new empty hash
instead.

### renew\_time\_key (optional, string type)

`renew_time_key foo` overwrites the time of events with a value of the
record field `foo` if exists. The value of `foo` must be a unix time.

### keep\_keys (optional, array type)

A list of keys to keep. Only relevant if `renew_record` is set to true.

### remove\_keys (optional, array type)

A list of keys to delete.

Need more performance?
----------------------

[filter\_record\_modifier](https://github.com/repeatedly/fluent-plugin-record-modifier)
is light-weight and faster version of `filter_record_transformer`.
`filter_record_modifier` doesn't provide several
`filter_record_transformer` features, but it covers popular cases. If
you need better performace for mutating records, consider
`filter_record_modifier` instead.

FAQ
---

### What are the differences between `${record["key"]}` and `${key}`?

`${key}` is short-cut for `${record["key"]}`. This is error prone
because `${tag}` is unclear for event tag or `record["tag"]`. So the
`${key}` syntax is now deprecated for avoiding this problem. Don't use
`${key}` short-cut syntax on the production.

Since v0.14, `${key}` short-cut syntax is removed.

Learn More
----------

-   [Filter Plugin Overview](/articles/filter-plugin-overview.md)




If this article is incorrect or outdated, or omits critical information,
please [let us know](https://github.com/fluent/fluentd-docs/issues?state=open).
[Fluentd](http://www.fluentd.org/) is a open source project under [Cloud
Native Computing Foundation (CNCF)](https://cncf.io/). All components
are available under the Apache 2 License.