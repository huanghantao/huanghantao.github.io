---
title: Xdebug与Vscode通信过程
date: 2020-05-15 22:13:17
tags:
- Xdebug
- Vscode
- PHP
---

当`xdebug`初次连接vscode的之后，会调用`xdebug_dbgp_init`函数，构造完`xml`之后，通过`send_message_ex`函数发送数据给`vscode`。

当`vscode`收到了`xdebug`发送的初始化`xml`之后，如果`vscode`设置了断点，那么`vscode`就会把断点信息（那个文件，哪一行）发送给`xdebug`。实际上是发生了`dbgp`的`breakpoint_set`命令。（还会发送`transaction_id`，即事务`id`）

然后，`xdebug`就会调用`xdebug_dbgp_cmdloop`来读取`vscode`发送给`xdebug`的命令。

然后，`xdebug`就会调用`xdebug_dbgp_parse_option`来解析`vscode`发送给`xdebug`的命令。把解析道的命令参数放在`xdebug_dbgp_arg`结构里面。

然后，`xdebug`开始组装`xml`响应，设置`xml`的`command`属性为解析出来的那个`command`。例如，如果`vscode`发来的`command`是设置断点，那么，`xdebug`就会把`xml`的`command`属性设置为`breakpoint_set`；设置`xml`响应`transaction_id`为`vscode`发来的`transaction_id`。

然后检查`xdebug`是否支持`vscode`发来的`command`，通过函数`lookup_cmd`来检查。`xdebug`支持的所有`command`都放在了变量`dbgp_commands`里面（这个变量存了`command`对应的`handler`）。

然后调用`command`对应的`handler`，`DBGP_FUNC(breakpoint_set)`。这些`handler`都在文件`handler_dbgp.c`里面。这个`handler`会判断断点的类型，例如`line`断点、`conditional`断点、`call`断点等等。`xdebug`支持的断点类型在变量`xdebug_breakpoint_types`里面。最后，把断点信息保存在结构`xdebug_brk_info`里面。然后把`xdebug_brk_info`类型转化为`xdebug_llist_element`存放在`context->line_breakpoints`链表里面。

然后，`xdebug`发送设置断点成功的响应给`vscode`。

然后，`vscode`发送`setFunctionBreakpointsRequest`给`xdebug`，实际上就是发送`breakpoint_list` `command`给`xdebug`。

然后`xdebug`触发`breakpoint_list` `command`的`handler`，`DBGP_FUNC(breakpoint_list)`来处理断点。`DBGP_FUNC(breakpoint_list)`会调用`xdebug_hash_apply`来处理所有的断点，对这些断点信息执行回调函数`breakpoint_list_helper`来组装`breakpoint_list` `command`的`xml`响应。

同理，`vscode`发送`setExceptionBreakpointsRequest`给`xdebug`，也是发送`breakpoint_list` `command`给`xdebug`。然后，`xdebug`调用`breakpoint_list_helper`函数来组装`breakpoint_list` `command`的`xml`响应。

最后，`vscode`发送`configurationDoneRequest`给`xdebug`，告诉`xdebug`我已经发送完了所有的断点请求。此时，`vscode`会发送`run` `command`给`xdebug`。

`xdebug`接收到`run`命令之后，`xdebug_dbgp_cmdloop`函数跳出执行。

然后，`xdebug`调用`xdebug_debugger_handle_breakpoints`函数来检查当前执行到的代码是否有断点。

然后，`xdebug`调用`old_execute_ex`来执行代码。此时，会调用`xdebug hook`后的`handelr ZEND_EXT_STMT_SPEC_HANDLER`来执行`opcode`。例如，`xdebug_debugger_statement_call`就是表达式对应的`hook`函数。如果这个函数发现当前`op_array`有断点信息，那么就会调用`XG_DBG(context).handler->break_on_line`函数来进行处理（`XG_DBG(context).handler->break_on_line`只是记录一些信息到日志文件里面）。然后调用`xdebug_handle_hit_value`。这个函数会记录`xdebug`触发了多少次断点。然后调用`XG_DBG(context).handler->remote_breakpoint`，这个函数会发送组建`xml`响应给`vscode`，把触发的断点信息发送给`vscode`。（此时，`vscode`的`ui`并没有停在断点处）

接着，`vscode`发送`stackTraceRequest`，实际上是`stack_get` `command`给`Xdebug`。

然后`xdebug`调用`DBGP_FUNC(stack_get) handler`来处理`stack_get command`请求。`xdebug`会调用`return_stackframe`函数来获取栈帧信息。然后组装成`xml`响应发送给`vscode`。`vscode`收到栈帧响应之后，就会在`UI`上面停住，给我们一种断点触发的视觉。但是，此时，`vscode`的变量板块是没有任何信息的，因为`xdebug`还没有把变量的信息返回给`vscode`。

接着，`vscode`发送`scopesRequest`给`xdebug`，实际上是`context_names command`。然后，`xdebug`调用`DBGP_FUNC(context_names)`来处理命令。这个`context_names`主要是用来告诉`vscode`，`xdebug`支持返回哪些变量类型，例如`Locals`变量，`Superglobals`变量，`User defined constants`常量。

接着，`vscode`发送`variablesRequest`给`xdebug`，实际上是`context_get command`。然后，`xdebug`组装变量当前的值，以`xml`响应给`vscode`。此时，`vscode`的变量板块就有变量的值信息了。

然后，我们点击`vscode`调试器的下一步，就会发送`nextRequest`给`xdebug`，实际上是`step_over command`。

然后，`vscode`和`xdebug`就一直重复上面的过程了。

需要注意的一点就是，如果`vscode`不给`xdebug`发送命令的话，`xdebug`就会在`xdebug_dbgp_cmdloop`函数里面的`recv`函数阻塞住。

如果是调试`Swoole`的`Server`，那么如果`Server`还没有收到数据，是不会回调`PHP`函数的。并且，`PHP`解释器也会因为`Swoole`的事件驱动而停止住，也就意味着，`xdebug`此时收不到数据。也就意味着，即使我们点击了下一步，给`vscode`发送了请求，`xdebug`也无法作出响应。并且，如果我们在`Swoole` `Server`事件没有到来时多次点击，那么，当`Server`事件到来的时候，`xdebug`会发送对应的多个`reponse`给`vscode`。这也算是`xdebug`的一个缺陷吧。
