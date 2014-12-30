
# StateMan API Reference

## Quirk Links

### API
- [new StateMan(option) or StateMan(option)](#instance)
- [__stateman.state__](#state)
- [__stateman.start__](#start)
- [__stateman.nav__](#nav)
- [__stateman.go__](#nav)
- [stateman.is](#nav)
- [stateman.encode](#encode)
- [stateman.on](#on)
- [stateman.off](#off)
- [stateman.emit](#emit)
- [stateman.stop](#stop)
- [stateman.decode](#decode)

### [Event](#event)

- __`begin`__: emit when an navigating is begin
- __`end`__:  emit when an navigating is over
- `notfound`:  path is changed but no state is founded
- `start`: emit when calling stateman.start
- `stop`: emit when calling stateman.stop
- `history:change`: if path  is changed . emitted by [stateman.history](). 

### Properties

- [stateman.current](#current):  target state;
- [stateman.previous](#previous):  previous state;
- [stateman.active](#active): valuable at navigating. represent the active state.
- [stateman.param](#param):  current param.

### Deep Guide

* [nested state](#nested)
* [__params in routing__](#param)


### before taking document into detail, suppose we already have a state config like below

```js

var config = {
  enter: function(option){ console.log("enter: " + this.name + "; param: " + JSON.stringify(option.param)) },
  leave: function(option){ console.log("leave: " + this.name + "; param: " + JSON.stringify(option.param)) },
  update: function(option){ console.log("update: " + this.name + "; param: " + JSON.stringify(option.param)) },
}

function cfg(o){
  o.enter = config.enter  
  o.leave = config.leave  
  o.update = config.update  
  return o;
}

var stateman = new StateMan();

stateman.state({

  "app": config,
  "app.contact":  config,
  "app.contact.detail": cfg({url: ":id(\\d+)"}),
  "app.contact.detail.setting": config, 
  "app.contact.message": config,
  "app.user": config,
  "app.user.list": cfg({url: ""})

}).on("notfound", function(){
  this.go('app') // if not found
}).start();

```

config is used to help us log the statechange process, you don't need to understand the code right now, the document will explain later.

You can find __the demo [here](http://leeluolee.github.io/stateman/api.html)__. type something in console can help you to understand api more clearly.



## Class: StateMan

### 1. new StateMan() or StateMan()

__return__:  the StateMan instance


```javascript
var StateMan = require("stateman");

var stateman = new StateMan();  
// or...
var stateman = StateMan();
```


## Instance: stateman

## 1. stateman.state(stateName[, config])
stateman.state is the most important api in stateman, all routing config is through it.

 __Arguments__

- __stateName__ < String  |Object >: the state's name , like `contact.detail`, if a `Object` is passed in, there will be a multiple operation.
- __config__ < Function | Object | String>: the configure of the state that stateName point to.
	if config is not passed, target [state](#State)   will be return. if a `Function` is passed, it will be considered as the [enter](#enter) property. If `String` is passed, it will be the [`config.url`](#url) property

you can simply replace the `.` with `/` to get the url that state represent. for example , the url of 'app.contact' is '/app/contact'. but you can adjust the url by passing the [url](#url) in config.

 __Return__ : this

__Example__

```js

stateman
	.state("app", {
		enter: function(){
			console.log("enter the state: app")
		},
		leave: function(){
			console.log("leave the state: app")
		}
	})
	.state("app.contact", function(){
			console.log("enter the app.contact state")
	})

// pass in a Object for multiple operation
stateman.state({
	"demo.detail": function(){
			console.log("enter the demo.detail state")
	},
	"demo.list": {
		enter: function(){}
		leave: function(){}
	}
})


```

As you see, we haven't created the `demo` state before creating `demo.detail`, beacuse stateman have created it for you. 


if config is not passing, `state.state(stateName)` will return the target state.

```js

var state = stateman.state('demo.list'); // return the demo.list state

```


### __The detail of the  `config`__


* config.url:  default url is the lastName: like `detail` in `contact.detail`

	```js
    	 //=> /contact/:id
	stateman.state('contact.detail', {url: ':id'}) 
			
	```

	The whole url of a state is the combination of the whole path to this state. like `app.contact.detail` is the combination of `app`,`app.contact` and `app.contact.detail`. For Example: 


	```js
	state.state("app", {})
		.state("app.contact", "users")
		.state("app.contact.detail", "/:id")

	```
	the caputred url of `app.contact.detail` equals to `/app/users/:id`. YES, obviously you can define the param captured in the url. see [param in routing](#param) for more infomation.

	missing `/` or redundancy of `/` is all valid.

	__absolute url__: if you dont want the url that defined in ancestry, you can use a prefix `^` . for example

	__empty url__: if you pass `url:""`, state's captured_url will be same as its parent. but the child state have the higher Priority than the parent state. for Example, the state `app.user.list` and `app.user` that we defined before, have the same captured url `/app/user`, but we will match the state`app.user.list`.


	```sh
	state.state("app.contact.detail", "^/detail/:id");
	```

	the captured url will be `/detail/:id`.

* config.enter(option) : a function that will be called when the state be entered into.
* config.leave(option) : a function that will be called when the state be leaved out.
* config.update(option): a function called by states that included by current state, but not be entered into or leave out. 


__example__: the current state is `app.contact.detail.setting`, when navigating to `app.contact.message`. the complete process is

1. leave: app.contact.detail.setting
2. leave: app.contact.detail
3. update: app.contact
3. update: app
4. enter: app.contact.message

you can test it in [api.html](http://leeluolee.github.io/stateman/api.html);



`enter`, `leave` and `update` they all accepet an param named `option`. option contains a special property `option.param` represent the param from url [see param for detail](#param)

other property in option passed into [__stateman.go__](#go) or [__stateman.nav__](#nav), will also passed in `enter`, `leave`, `update`



<a name='param'></a>

### __Important: param captured in routing__

you can use [__stateman.decode__](#decode) to find the matched state for specified path.


####1. named param without pattern, the most usually usage.

```shell
/contact/:id
```

the router will be matched by path `/contact/1`, and find the param `{id:1}`

in fact, all named param have a default pattern `[-\$\w]+`. but you can change it use self-defined pattern.

####2. named param with pattern

named param follow with `(regexp)` can restrict the pattern for current param (don't use sub capture it regexp). for example.

```sh
/contact/:id(\\d+)
```

now, only the number is valid the "id" param;


####3. unnamed param with pattern
you can also define a plain pattern for route matching. imagine that you have a pattern like below

```sh
/contact/(friend|mate)/:id([0-9])/(message|option)
```

it will match the path `/contact/friend/4/message` and get the param `{0: "friend", id: "4", 1: "message"}`

unnamed param will be put one by one in `param` use autoincrement index. 

#### 4. param in search

you can also passing query search in the url. take `/contact/:id` for example.

it will matched the url `/contact/1?name=heloo&age=1`, and get the param `{id:'1', name:'heloo', age: '1'}`





<a name="start"></a>
## 2. stateman.__start__ (option)

start the state manager.

__option__
	- **option.html5**: whether to open the html5 history support.
	- **option.root**: the root of the url , __only need when html5 is actived__. defualt is `/`
	- **option.prefix**: for the hash prefix , default is '' (you can pass `!` to make the hash like `#!/contact/100`), works in hash mode.
	- **option.autolink**: whether to delegate all link(a[href])'s navigating, only need when __html5 is actived__, default is `true`.
	
<a name="nav"></a>
## 3. stateman.__nav__(url[, option][, callback]);

nav to a specified url, you can pass a option that will also be passed into function `enter`, `leave`, `update`.

__Argument__

-	url < String >: url to navigate
- option < Object > : [Optional] navigate option, option will merge the [param from url](#param) as its `param` property. and will be passed in `enter`, 'leave' and `update`, there are some special properties can control the navigating:
	* option.silent: if silent is true, only the location is change in browser, but will not trigger the stateman's navigating process
	* option.replace: if replace is true. the previous path in history will be replace by current( you can't use back or go in browser to restore it)
- callback < Function >: [Optional] once navigating is done, callback will be called.

All other property in option will passed to `enter`, `leave` , `update`.

```js
stateman.nav("/app/contact/1?name=leeluolee", {data: 1});

the final option passed to `enter`, `leave` and `update` is `{param: {id: "1", name:"leeluolee"}, data: 1}`.

```



## 4. stateman.__go__(stateName [, option][, callback]);

nav to specified state, very similar with [stateman.nav](#nav). but stateman use stateName to navigating. 

- option.encode: default is true. if encode is false, url will not change at  location, only state is change (means will trigger the stateman's navigating process). stateman use the [__encode__](#encode) method to compute the real url.
- option.param: the big different between __nav__ and __go__ is __go__ method  may need a param to compute the real url, and place it in location.
you can use stateman.encode to test how stateman compute url from a state with specifed param

All other property in option will passed to `enter`, `leave` , `update`. just like nav.

__we always recommend that use go instead of nav in large project to control the state more clearly __.


__example__

```
stateman.go('app.contact.detail', {param: {id:1, name: 'leeluolee'}});
```

location.hash will change to `#/app/contact/1?name=leeluolee` , you can find that uncaputed param (name) will be append to url as the querystring.


### 5. stateman.is( stateName[, param] [, isStrict] )

determine if the active state is equal to or is the child of the state. If any params are passed then they will be tested for a match as well. not all the parameters need to be passed, just the ones you'd like to test for equality.


__Arguments__

- stateName: test state's name
- param: the param need to be tested
- isStrict: if the target state need strict equals to active state.


__example__

```js
stateman.nav("#/app/contact/1?name=leeluolee");
stateman.is("app.contact.detail") // return true
stateman.is("app.contact", {}, true) // return false, 
stateman.is("app.contact.detail", {name: "leeluolee"}, true) // return true
stateman.is("app.contact.detail", {id: "1"}) // return true
stateman.is("app.contact.detail", {id: "2"}) // return false
stateman.is("app.contact.detail", {id: "2", name: "leeluolee"}) // return false
```

<a name="encode"></a>
### 6. stateman.encode( stateName[, param] )

get a url from state and specified param. [__go__](#go) is based on this method.

__Arguments__

```js
stateman.encode("app.contact.detail", {id: "1", name: "leeluolee"}) === "/app/contact/1?name=leeluolee"

```

### 7. stateman.decode( url )

find the state that match the url, [__nav__](#nav)is based on this method

__Example__

```js
var state = stateman.decode("/app/contact/1?name=leeluolee")

state.name === 'app.contact.detail'
state.param // =>{id: "1", name: "leeluolee"}

```


### 8. stateman.stop()

stop the stateman.

### 9. Other Useful property in stateman

1. __stateman.current__: 
	the current state, if a navigating is still in process, the current represent the destination state. 

2. __stateman.previous__: 
	the previous state.
	
3. __stateman.active__: 
	the active state, point to the state that still in active.

imagine that you navigating form state named 'app.contact.detail' to the state named 'app.user', the current is point to contact.detail and the previous will point to 'user.detail'. and the active state is changed from `app.contact.detail` to `app.user`

`current`, `active`, `previous` is __all living__.


## Emitter

StateMan have simple EventEmitter implementation for event driven development, The Following Class have the Emitter Mixin .



1. __StateMan__: The State Manager.
2. __StateMan.State__: Every state in state mannager is StateMan.State's instance
2. StateMan.Histery : for cross-platform 's location manipulation, generally speaking, you will nerver to use it.

all Class that list above have same API below


### 1. emitter.on(event, handle)
bind handle to specified event.

### 2. emitter.off(event, handle)
unbind handle 

### 3. emitter.emit(event, param)

trigger a specified event with specified param


## StateMan's builtin Event

1. start: when stateman is start
2. stop: when stateman is stop
2. begin: when a navigating is perform
3. end: when a navigating is over
4. history:change: when a change envet is emitted by history
5. notfound: when no state is founded during a navigating.



