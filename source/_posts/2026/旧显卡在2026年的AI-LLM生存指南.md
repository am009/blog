---
title: 旧显卡在2026年的AI-LLM生存指南
date: 2026/01/20 11:11:12
categories:
- Dev
tags:
- AI
---

旧显卡在2026年的AI-LLM生存指南

<!-- more -->

## 老显卡的问题

我之前买的是NVidia GForce 2080ti显卡 Turing 架构。最直接的当然是Cuda的兼容等级太低。直接的痛点则是FlashAttention完全不支持，不支持BFloat16 BF16。现在各种模型基本都是FlashAttention和BF16了。

但是其实还是有出路的。首先放弃vLLM，各种新模型从来没有在vLLM上跑起来过。

## 出路1 Ollama

能用Ollama跑的就不要用别的跑。旧显卡还是ollama好用。主流模型都支持。

**核心痛点：Chat Template**

如果发现有新模型，只支持vLLM，可以先尝试转到Ollama。使用Ollama的Modelfile，直接下载模型，写一个FROM，然后执行`ollama create`命令。

如果报错：“不支持的架构”，则接下来应该尝试使用llama.cpp的转换脚本，转成gguf文件后再用Ollama的Modelfile的方式导入。此时，你可能发现模型的性能完全不对劲。此时应该检查template的问题了！

对话模版的作用是插入一些特殊的token。比如下面的是[混元翻译模型的模版](https://huggingface.co/tencent/HY-MT1.5-7B/blob/main/chat_template.jinja) (注：经过了手动格式化)

```
{% set ns = namespace(has_head=true) %}
{% set loop_messages = messages %}
{% for message in loop_messages %}
    {% set content = message['content'] %}
    {% if loop.index0 == 0 %}
        {% if content == '' %}
            {% set ns.has_head = false %}
            {% elif message['role'] == 'system' %}
            {% set content = '<|startoftext|>' + content + '<|extra_4|>' %}
        {% endif %}
    {% endif %}
    {% if message['role'] == 'user' %}
        {% if loop.index0 == 1 and ns.has_head %}
            {% set content = content + '<|extra_0|>' %}
        {% else %}
            {% set content = '<|startoftext|>' + content + '<|extra_0|>' %}
        {% endif %}
    {% elif message['role'] == 'assistant' %}
        {% set content = content + '<|eos|>' %}
    {% endif %}
    {{ content }}
{% endfor %}
```

它会在system message插入特殊token `<|extra_4|>`，这种特殊的东西我们看不懂，但是模型之前学过，看得懂，而且可能对性能有影响。其他特殊token可以阅读[这个文章](https://medium.com/@khalandar.mokula/understanding-eos-token-and-pad-token-in-model-generate-config-huggingface-transformers-fef6845ed92a)学习eos_token和pad_token。

各种框架通常采用的是基于jinja的对话模版，转换为GGUF格式，里面存的也是jinja模板。但是**在导入ollama的时候，模版就丢了！！**因为其他框架一般都是jinja的模版，ollama用的是golang模板！

[官网对模版的介绍](https://docs.ollama.com/modelfile#message)不全，先看[这里](https://github.com/ollama/ollama/blob/31085d5e53811bedc53a9cb2afdea45554749d01/docs/template.mdx)，现在一般用Messages模板，

简单看了下代码，总之有下面的元素可用。必须用到"Messages"。其他的可能是legacy旧版本的语法

```
    return t.Template.Execute(w, map[string]any{
        "System":     system,
        "Messages":   convertMessagesForTemplate(messages),
        "Tools":      convertToolsForTemplate(v.Tools),
        "Response":   "",
        "Think":      v.Think,
        "ThinkLevel": v.ThinkLevel,
        "IsThinkSet": v.IsThinkSet,
    })
```

## 出路2 MemoryEfficient Attention

就算ollama不能跑，我们也可以直接用huggingface 的transformers包跑模型。大部分情况和ollama一样，是可以直接跑的，少部分情况会发现**内存随着输入长度的增加而暴涨**。此时可能是因为没有启用Memory Efficient Attention。

通常新款显卡都用的是FlashAttention，它也很节约内存，但是我们老款显卡一般都用不了这个。因此往往会用MemoryEfficient Attention。因为往往显存大小是瓶颈。普通的直接算Attention，输入长一点可能直接需要的显存比模型还大了。

这一点在支持图片的多模态模型上尤其明显。Qwen-VL系列，一方面使用了Grouped Query Attention GQA，导致可能底层pytorch不会调用MemoryEfficient Attention后端。因为Query的张量大小和Key，Value不相等。

```
UserWarning: For dense input, both fused kernels require query, key and value to have the same num_heads. Query.sizes(): [1, 32, 11861, 128], Key sizes(): [1, 8, 11861, 128], Value sizes(): [1, 8, 11861, 128] instead.
```

Grouped Query Attention的原理是多个Query头共享一个Key和Value，但是MemoryEfficient Attention后端不支持这种情况，导致调用了其他底层后端，消耗过多显存。然而，只需要**把Key和Value直接多复制几份**，使其大小和Query一致，就可以继续用MemoryEfficient Attention后端了！不会出现内存暴涨的问题。

解决方案：[**更新transformers库里的实现**](https://gist.github.com/am009/be749ce4b6133a2208366205e9d9728b) 可以解决这一问题。基本上只有 MemoryEfficient Attention能用了，其他的占显存太多。最终跑下来速度也不会特别慢，除非输入特别长（或者前面有一张特别清晰的图片）。

另外注意，Qwen系列模型，输入图片的时候最好**长和宽都是28像素的倍数**，不然可能坐标有问题。因为分词器是按照28*28字节划分token的。

## 让基于Python脚本的LLM API也可以超时自动卸载模型，节约显存

之前搜了半天怎么写逻辑，在不用的时候卸载模型节约显存，结果发现怎么都卸载不干净。

总结：直接区分进程，在要用的时候才启动相关进程。

使用这个[**通用的脚本**](https://gist.github.com/am009/78989aa6597245c9444b9253e920ff64)，它多监听一个端口，将流量转发到另外一个端口。但是如果长时间没有连接，就会启动停止进程的命令，停止占显存的API Server脚本。如果又进来了新的连接，会自动启动那个API Server，然后等待启动后转发流量过去。实现了类似Ollama的效果。

- 推荐挂到systemd中成为系统服务，实现开机自动启动，毕竟基本不占什么资源。
- 如果是Docker程序，则启动和停止命令只需要设置为docker start docker stop
- 如果是本地程序，推荐挂载为systemd服务，启动停止命令设置为systemctl start stop。这个就不用设置开机启动了。

## 总结

- 老显卡还可以再继续用。
- 话说，BFloat说不定真的只应该是存储类型？具体运算的时候每一层拿出来再转float32算是不是也完全可以？

## 附录1 Jinja模板转Ollama的Golang模板的提示词

````
你需要为某模型的chat_template（使用的jinja模板）生成对应的，使用golang模板语法的模版，下面是golang模板的教程

## Adding templates to your model

By default, models imported into Ollama have a default template of `{{ .Prompt }}`, i.e. user inputs are sent verbatim to the LLM. This is appropriate for text or code completion models but lacks essential markers for chat or instruction models.

Omitting a template in these models puts the responsibility of correctly templating input onto the user. Adding a template allows users to easily get the best results from the model.

To add templates in your model, you'll need to add a `TEMPLATE` command to the Modelfile. Here's an example using Meta's Llama 3.

```dockerfile
FROM llama3.2

TEMPLATE """{{- if .System }}<|start_header_id|>system<|end_header_id|>

{{ .System }}<|eot_id|>
{{- end }}
{{- range .Messages }}<|start_header_id|>{{ .Role }}<|end_header_id|>

{{ .Content }}<|eot_id|>
{{- end }}<|start_header_id|>assistant<|end_header_id|>

"""
```

## Variables

`System` (string): system prompt

`Prompt` (string): user prompt

`Response` (string): assistant response

`Suffix` (string): text inserted after the assistant's response

`Messages` (list): list of messages

`Messages[].Role` (string): role which can be one of `system`, `user`, `assistant`, or `tool`

`Messages[].Content` (string): message content

`Messages[].ToolCalls` (list): list of tools the model wants to call

`Messages[].ToolCalls[].Function` (object): function to call

`Messages[].ToolCalls[].Function.Name` (string): function name

`Messages[].ToolCalls[].Function.Arguments` (map): mapping of argument name to argument value

`Tools` (list): list of tools the model can access

`Tools[].Type` (string): schema type. `type` is always `function`

`Tools[].Function` (object): function definition

`Tools[].Function.Name` (string): function name

`Tools[].Function.Description` (string): function description

`Tools[].Function.Parameters` (object): function parameters

`Tools[].Function.Parameters.Type` (string): schema type. `type` is always `object`

`Tools[].Function.Parameters.Required` (list): list of required properties

`Tools[].Function.Parameters.Properties` (map): mapping of property name to property definition

`Tools[].Function.Parameters.Properties[].Type` (string): property type

`Tools[].Function.Parameters.Properties[].Description` (string): property description

`Tools[].Function.Parameters.Properties[].Enum` (list): list of valid values

## Tips and Best Practices

Keep the following tips and best practices in mind when working with Go templates:

- **Be mindful of dot**: Control flow structures like `range` and `with` changes the value `.`
- **Out-of-scope variables**: Use `$.` to reference variables not currently in scope, starting from the root
- **Whitespace control**: Use `-` to trim leading (`{{-`) and trailing (`-}}`) whitespace

样例：原始模版：
```
{% if messages[0]['role'] == 'system' %}
  {% set loop_messages = messages[1:] %}
  {% set system_message = messages[0]['content'] %}
  <｜hy_begin▁of▁sentence｜>{{ system_message }}<｜hy_place▁holder▁no▁3｜>
{% else %}
  {% set loop_messages = messages %}
  <｜hy_begin▁of▁sentence｜>
{% endif %}
{% for message in loop_messages %}
  {% if message['role'] == 'user' %}
    <｜hy_User｜>{{ message['content'] }}
  {% elif message['role'] == 'assistant' %}
    <｜hy_Assistant｜>{{ message['content'] }}<｜hy_place▁holder▁no▁2｜>
  {% endif %}
{% endfor %}
{% if add_generation_prompt %}
  <｜hy_Assistant｜>
{% else %}
  <｜hy_place▁holder▁no▁8｜>
{% endif %}
```
上面的样例jinja模板，正确的转换结果是：
```
<｜hy_begin▁of▁sentence｜>
{{- if .System }}{{ .System }}<｜hy_place▁holder▁no▁3｜>{{ end }}

{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 -}}

{{- if eq .Role "user" -}}
<｜hy_User｜>{{ .Content }}
{{- end }}

{{- if eq .Role "assistant" -}}
<｜hy_Assistant｜>{{ .Content }}<｜hy_place▁holder▁no▁2｜>
{{- end }}

{{- if and $last (ne .Role "assistant") -}}
<｜hy_Assistant｜>
{{- else -}}
<｜hy_place▁holder▁no▁8｜>
{{- end }}

{{- end -}}
```

对于下面的模板内容，

```
{% set ns = namespace(has_head=true) %}
{% set loop_messages = messages %}
{% for message in loop_messages %}
    {% set content = message['content'] %}
    {% if loop.index0 == 0 %}
        {% if content == '' %}
            {% set ns.has_head = false %}
            {% elif message['role'] == 'system' %}
            {% set content = '<|startoftext|>' + content + '<|extra_4|>' %}
        {% endif %}
    {% endif %}
    {% if message['role'] == 'user' %}
        {% if loop.index0 == 1 and ns.has_head %}
            {% set content = content + '<|extra_0|>' %}
        {% else %}
            {% set content = '<|startoftext|>' + content + '<|extra_0|>' %}
        {% endif %}
    {% elif message['role'] == 'assistant' %}
        {% set content = content + '<|eos|>' %}
    {% endif %}
    {{ content }}
{% endfor %}
```

转换后的Golang模板结果是什么？认真思考
````