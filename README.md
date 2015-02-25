# ngx_tasks

The management support features:

* preemption single worker tigger task in all nginx workers.
* per-worker/per-request drive mode in worker processes.
* re-write reponse of request in task.
* custom keys/key generator in dictionary.

## Installation
download the ngx_tasks.lua file and put into lua search path or current directory.

load it as a file module from lua(in nginx only).

the module need shared_dict support in ngxin.conf.

## Base usage
```lua
tasks = require('ngx_tasks')
tasks.dict = 'your_shared_dict_config_in_<nginx.conf>'
tasks:push(rule)
...
tasks:run()
```

## Rules import
```lua
tasks = setmetatable(require('rules'), {
    __index = require('ngx_tasks')
})
tasks.dict = 'your_shared_dict_config_in_<nginx.conf>'
tasks:run()
```

## Rule/rules
a rules is array(table) of rule. a rule is a record(table) with callback() method:
```lua
rule = {
    name = 'test ding',
    identifier = 'testDing',
    interval = 60,      -- second
    typ = 'normal',     -- normal/preemption
    callback = function(self)
		ngx.log(ngx.ALERT, 'DING')
    end
}

rules = {
    rule,
    ...
}

-- push rule into tasks
tasks = require('ngx_tasks')
for _, rule in ipairs(rules) do
    tasks.push(rule)
end
...
```

## Task types: 'normal'/'preemption'
if typ is 'normal', tigger task by interval time for each worker. else 'preemption', tigger task for first active worker only(on interval timeout).

for sample case, if you want DING per-worker at interval 10s, and DONG once at interval 1m. next is task rules:
```lua
function dingdong(self)
    ngx.log(ngx.ALERT, self.identifier)
end

rules = {
    {
        identifier = 'TRYDING',
        interval = 10,
        typ = 'normal',
        callback = dingdong
    },
    {
        identifier = 'TRYDONG',
        interval = 60,
        typ = 'preemption',
        callback = dingdong
    }
}
```
## How to drive these tasks?
the task manager need a drive cycle/behavior. you need update nginx.conf to implement it.

1) case 1, create per-worker reoccurring timers
```conf
http {
	lua_shared_dict PreemptionTasks 10M;
	## see: http://wiki.nginx.org/HttpLuaModule#init_worker_by_lua
    init_worker_by_lua '
        tasks = require('ngx_tasks');
        ngx.timer.at(60, tasks.run, tasks);
    ';
```

2) cast 2, (or, ) create per-request tasks scheduling
```conf
http {
	lua_shared_dict PreemptionTasks 10M;
    init_worker_by_lua '
        tasks = require('ngx_tasks');
    ';
	# trigger on request's response time
	rewrite_by_lua '
	    tasks:run();
    ';
```

## choice per-request or per-worker?
for per-request task trigger, you can rewrite reponse body of request, but do not in per-worker mode. because  per-worker mode drive by ngx.timer.at:
> timer callbacks run in the background and their running time will not add to any client request's response time, ... 
see: http://wiki.nginx.org/HttpLuaModule#ngx.timer.at

## what keys/key generator and how do
ngx_tasks will write shared dictionary with the key/keys. the management include a default generator. but you can do better. there are a example:
```lua
tasks = require('ngx_tasks')
-- tasks.dict = 'PreemptionTasks'  -- default
tasks.keys = function(self, task, now)
    return math.floor((now - self.base_time)/task.interval), task.identifier .. 'Preemption'
end
tasks.base_time = global_start_time
tasks:run()
```
the generator is faster. but you need update nginx.conf (insert a lua code line into init_by_lua content):
```conf
http {
	lua_shared_dict PreemptionTasks 10M;
    init_by_lua '
        global_start_time = os.time()
    ';
```
