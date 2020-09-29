# Component Lifecycle

* manual
* mount
* component
* integrant


<!--

## Community discussion

hello! i am trying to understand the proper way of using mount. probably my difficulty arises from my object-oriented/spring-y experience that makes component more familiar, but mount’s argument that namespace and functions are the right abstraction kinda makes sense to me.basically what i am trying to wrap my mind around is this: i am working with datomic and understand that the db client is a (def)state. i would declare this state in a namespace that would be responsible for data access stuff, so the state would be internally accessed by functions in this namespace. now suppose that i want to replace the datomic implementation by an in-memory map implementation for tests… then the client as a state doesn’t seem to be the right “stateful” element, since each function uses the client to retrieve a connection an then uses more datomic api to perform its job.i am not sure if my explanation is understandable… can someone help me figure out the way to organize my states with mount? :slightly_smiling_face:
tolitius  14:32
good question. I think the confusion is from the scope of the state you are thinking about. usually db connection would be a state, and this connection then mocked with a connection to memory vs. disk / cloud. your other functions that work with datomic would accept a connection and use it for queries.it is similar how, in spring, you create a datasource. in spring though a datasource is not a stand alone state and is wrapped into the jdbctemplate, which is where the confusion (I think) comes from.
Matheus Moreira  17:18
i can better reason about the oo datasource because: a) i identify the datasource as a resource; b) that is injected into data access objects/classes; and c) these classes are usually behind an interface. in this case i would be able to change not only the datasource implementation, i would be able to change the entire data access layer implementation, e.g. instead of a postgres db it could be an in-memory hashmap.
17:22
maybe it is because i thought about datomic vs hashmap instead of disk vs. memory. e.g. i don’t want to think about datomic now, i just want to have an interface for some persistence mechanism during development and i mock it as a hashmap. does it make sense?
tolitius  17:36
yea, there are two things here:

    your functions working with the datasource consistently regardless whether it is in memory datasource or a file datasource or a network datasource, etc.
    your datasource having a consistent interface (such as make-connection) that can be used by them functions

the state here, however is just this datasource / connection. I tend to only use defstate (i.e. create a stateful component) for low level things: connections, thread pools, listeners, etc.. (edited)
17:40
I went through an exact same transition from  "but  data access objects/classes  is what we use for state, right?"
      to
  "no, we only use the lowest stateful things for state, and use functions for everything else"when I started with functional languages (then Erlang). it is a completely healthy and great stage to make your "design" mind to go through (edited)
sudakatux  22:48
I have a similar confusion as @Matheus Moreira i come from a spring-background do you know of a blog or something explaining mount from a similar background. I would like to know if its possible to mount reitit routes
tolitius  23:07

    I would like to know if its possible to mount reitit routes

sure, it is not exactly mount specific to start a server with reitit routes. mount would be only used to start / stop a server vs. dealing with routes directly. here is a quick example: https://gist.github.com/tolitius/6a6dabfee5df0c6baa97d604560d460d

    do you know of a blog or something explaining mount from a similar background

I don't know whether there is one, but I've worked with spring for many years (before and after it was cool), so I can definitely answer some questions. maybe as a result also convert all the answers to a blog post so it is useful for others
sudakatux  23:08
thanks for that
pmonks  19:03
FWIW the pattern I’ve settled on is to keep my defstates in a separate namespace than my logic fns, and the state (connections or whatever) get passed in as vanilla fn arguments to those logic fns.  It can get a bit unwieldy when you have a logic fn that depends on a lot of different state, but the benefits of having “stateless” logic namespaces seems to me to make up for it.  Testing & experimenting at the REPL are greatly facilitated by this approach.
19:04
Not saying it’s the best or only way to do it, and keen to hear of better alternatives folks have come up with! (edited)
tolitius  19:43

    and the state (connections or whatever) get passed in as vanilla fn arguments to those logic fns

when switching from spring this is the hardest part to unlearn, since, as a spring developer you always have this feeling that something is @Autowired into something else (because it is in spring), and when switching to functional languages (about 12 years ago) it took me awhile to not have functions wrap that state.whether you keep your states separately or with is more of a personal preferences (both work well). things to watch out are circular dependencies and keeping those states as thin and low level as possible. I tend to keep a state with its logical module: i.e. "system-a" namespace will have a connection to "system a" plus some functions to work with "system a". but again, the main thing is to design functions in a way that they do not wrap state: i.e. take all the state they need as args.

    It can get a bit unwieldy when you have a logic fn that depends on a lot of different state

these (potentially) could be good candidates to refactor into several functions and compose them at the edges of the application
pmonks  20:27
100% agree.  And yeah I try to write “pure” namespaces focused on providing fns that do one thing and one thing well, and then compose those together somewhere else, alongside all the state (often a “core” namespace). (edited)
Matheus Moreira  10:21

    I tend to keep a state with its logical module: i.e. “system-a” namespace will have a connection to “system a” plus some functions to work with “system a”.

does it mean that you declare the state and the functions that depend on it in the same namespace (file)? and if this is the case, it wouldn’t be necessary to pass the state as fns arguments, since they can just use the state directly. (edited)
Ilari Tuominen  12:32
I’m a clojure beginner but this is more of a question towards mount so better write it here. I’m having problems with ending up in a DerefableState {:status :pending, :val nil} when loading my state from a file using io/file. Am I trying to access the resource too soon or am I thinking of this wrong? Logging says that my defstate gets started but if I remove the places accessing the state, I can load the namespace in repl and it has the value I’m expecting. How can I get the state properly initialized?
tolitius  16:20

    does it mean that you declare the state and the functions that depend on it in the same namespace (file)? and if this is the case, it wouldn’t be necessary to pass the state as fns arguments, since they can just use the state directly.

the location of state should not really affect the function design. a function would usually take all the state / args it needs. it makes it "self contained" which allows for better testing, refactoring, reuse, .. all the good things. (edited)
16:20
@Ilari Tuominen probably better solved with an example. usually a couple of quick things to look out for:

    call mount/start in case state needs to be created before the function is called

and / or

    to make sure that compiler sees the namespace where the state lives: i.e. (require '[foo.bar]) where state component lives.

roklenarcic  22:04
joined #mount.
roklenarcic  08:25
if I’ve got states A and B, A uses B in its start, when I start state A, state B doesn’t get started… what is the reason for that?
tolitius  16:13
@roklenarcic you can start all with (mount/start) or you can do (mount/start #'foo/a #'foo/b)
when you start a state individually (mount/start #'foo/a) it only focuses on a state specified and does not affect other states
roklenarcic  16:14
so if I want to start the web server, but not scheduled jobs for instance, I have to manually list all the dependencies?
tolitius  16:15
you can start all except  a "scheduler" state: https://github.com/tolitius/mount#composing-states (edited)
16:16
or if you don't need to compose them: https://github.com/tolitius/mount#start-an-application-without-certain-states (edited)
roklenarcic  16:17
right, so I need to list scheduler and dependencies that it uses but web server doesn’t use in except
16:18
I assume same principle applies, putting just scheduler in except will only skip that one, not it’s dependencies
tolitius  16:19
correct. how many states do you need to exclude?
16:19
usually it's from one to just a few, otherwise it might be an indicator to remove number of stateful things
16:20
this depends of course, but if states are kept low level (connections, tread pools, listeners, servers, etc..) there are just not as many of them in a given application (edited)


 -->
