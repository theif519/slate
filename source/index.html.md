#<center>Utilities Package <br>for the <br>C Programming Language</center>

##<center>Development Stages</center>

Stage | Description
:---- | ----------:
Unimplemented | Development has not yet begun, however it will in the future.
Deprecated | The library is to be removed at a later date and should not be used.
Development | The library is currently in development and not ready for public use.
Unstable | The library is in it's late development stages, however is not production-ready; further testing needed.
Stable | The library is mostly finished development and has been tested; further testing needed before production-ready
Finished | The library is finished development and has been extensively tested; it is production ready.

##<center>Artificial Namespace</center>

>With the namespace prefix...

~~~c
struct c_utils_logger *logger;
C_UTILS_LOGGER_AUTO_CREATE(logger, ...);

struct c_utils_thread_pool *pool = c_utils_thread_pool_create(...);
~~~

>Without the namespace prefix...

~~~c
logger_t *logger;
LOGGER_AUTO_CREATE(logger, ...);

thread_pool_t *pool = thread_pool_create(...);
~~~

>How to disable the prefix...

~~~c
#define NO_C_UTILS_PREFIX
#include <logger.h>
#include <thread_pool.h>
~~~

To avoid the issue of namespace collision, as C has only one namespace, all libraries in this package contain the C_UTILS_ prefix for macros, and c_utils_ prefix for functions and structs. Now, this can look rather ugly. For example...

There's no dodging around it, the c_utils_ prefix makes everything more long-winded. However, this is necessary when writing libraries like these. While it may be unwieldy, maybe you think you don't have to worry about collisions for  a logger or a thread_pool because no other library you use has them. This is where the `NO_C_UTILS_NO_PREFIX` define comes in. If you define this before importing the libraries, it will strip the c_utils prefix through macro defines, typedef most (99%) of the library the name, ended with a "_t". For example...

Now it is a LOT less long-winded, and much more elegant looking. This trade off adds the issue of potential collision, so be warned. Because of this conciseness, all below code samples use the `NO_C_UTILS_PREFIX`, and so contain no c_utils_ prefix. 

<aside class="warning"> 
You MUST place it before the inclusion of the package
</aside>

##<center>Configurations</center>

>Creating a map with defaults

~~~c
map_t *map = map_create();
~~~

>Creating a map with little configuration changes

~~~c
map_conf_t conf =
{
    .obj_len = sizeof(struct my_obj),
    .logger = my_logger
};

map_t *map = map_create_conf(&conf);
~~~

>Creating a map with highly-specified functionality

~~~c
map_conf_t conf =
{
    .flags = MAP_CONCURRENT | MAP_RC_INSTANCE | MAP_SHRINK_ON_TRIGGER | MAP_DELETE_ON_DESTROY,
    .num_buckets = 128,
    .callbacks = 
    {
        .destructors = 
        {
            .key = free,
            .value = destroy_value
        },
        .hash_function = my_custom_hash,
        .value_comparator = my_custom_comparator
    },
    .growth = 
    {
        .ratio = 1.5,
        .trigger = .75
    },
    .shrink =
    {
        .ratio = .75,
        .trigger = .1
    },
    .obj_len = sizeof(struct my_obj),
    .logger = my_logger
};

map_t *map = map_create_conf(&conf);
~~~

As this library aims to be completely configurable, adding more and more parameters will no longer do the job. One can see that if an object can have a hundred different uses, having a hundred different parameters, especially when not even needed, can be impractical and cumbersome.

The way the library conquers this is by allowing each object to be created with a configuration object. For example `map_t` has a configuration object called `map_conf_t`. This allows the fine-tunement of any given object when asked for, supplying it's own defaults when needed.

<aside class="notice">
Some defaults are not optimal with all configurations. Sometimes if you specify one configuration, you should also specify another as well to get the behavior you want.
</aside>

##<center>Lifetime Management</center>

>Create reference counted priority queue for producer-consumer relationship.

~~~c
blocking_queue_conf_t conf = { .flags = BLOCKING_QUEUE_RC_INSTANCE };
blocking_queue_t *pq = blocking_queue_create_conf(&conf);
~~~

>Reference count is now 0. Assume that we continue filling up the blocking queue with items as the producer, while the consumer consumes those items. Now, normally this would be tricky, as we have to consider who frees the queue first, and what if we have multiple producers and consumers? We would then have to join and wait until all threads finish, complicating things. Instead, we can do this...

~~~c
REF_INC(queue);
pass_data_to_consumer(queue);
~~~

>We must increment the count BEFORE passing it to the consumer to prevent any race conditions where we decrement our count before they get to increment their own. Now when either is finished...

~~~c
REF_DEC(queue);
// Or...
blocking_queue_destroy(queue);
~~~

Almost all objects returned from this library have some kind of reference counting built in to allow for easier management. This is, of course optional, as it is configurable (see next section). Enabling reference counting allows for other objects of the library to also maintain references to it while. In the end, if the reference counts are managed correctly, it can easily prevent memory leaks and become as easy to manage as a garbage collected language. 

To this end, if reference counting is enabled for an object, you use the helper macro `REF_DEC` instead of it's normal destructor to allow generic destruction of reference counted data, and the other helper macro `REF_INC` to increment the count.

<aside class="warning">
The data passed to REF_DEC and REF_INC MUST have been created with the ref_create function, and it is impossible to reference count an object after it's creation through a normal malloc or calloc call. Note as well, you should NEVER free the data itself, just call REF_DEC when finished.
</aside>

<aside class="success">
If done correctly, it can become a very useful utility for managing shared data between different threads or even different objects in general.
</aside>

#<center>Threading</center>

Library | Version | Status
:------- | :-------: | ------:
Thread Pool | 1.3 | Stable
Scoped Lock | 0.75 | Unstable
Conditional Locks | 1.0 | Stable
Events | 1.2 | Stable

##<center>Thread Pool</center>
>Thread pool task

~~~c
// Any function with the same return type and argument (void *) will work.
static void *task_example(void *args);
~~~

>Creating the thread pool with default arguments

~~~c
thread_pool_t *tp = thread_pool_create();
~~~

>Creating the thread pool with configuration object

~~~c
thread_pool_conf_t conf = 
{
    .pool_size = 4,
    .logger = my_logger
};

thread_pool_t *tp = thread_pool_create_conf(&conf);
~~~

>Adding a task with an asynchronous result, then retreiving said result

~~~c
int flags = 0;
void *args = NULL;
result_t *result = thread_pool_add(tp, task_example, args, flags);

// -1 = no timeout, wait until task finishes.
long timeout = -1;
void *retval = result_get(result, timeout);
result_destroy(result);
~~~

>Adding task with no result, with a different priority

~~~c
int flags = NO_RESULT | HIGH_PRIORITY;
void *args = NULL;
thread_pool_add(tp, task_example, args, flags);
~~~

>Pause the thread pool

~~~c
// 5 Seconds.
long timeout = 5000;
thread_pool_pause(tp, timeout);
~~~

>Wait for it to finish and then destroy it

~~~c
long timeout = -1;
thread_pool_wait(tp, timeout);

thread_pool_destroy(tp);
~~~

`thread_pool_t` is a thread pool which makes use of a prioritized `blocking_queue_t` for tasks. As implied by the use of a priority queue, tasks may be submitted via 6 different priorities, Lowest, Low, Medium, High and Highest. High Priority tasks would jump ahead of tasks of Low priority, intuitively. 

The static thread pool maintains a steady amount of threads, never growing or shrinking in size, however unused threads will block, hence it will not waste resources waiting for a new task to be submitted. 

Each task can return an asynchronous result, `result_t`, which, using `event_t`, you may wait (or poll) for when the task finishes. So, to reiterate, a task, by default, returns a result which can be waited on.

When submitting tasks, it comes with it's own default priority and will return a result_t result to wait on, but by passing certain flags, like `HIGH_PRIORITY | NO_RESULT` you may flag tasks specifically.

Finally you can pause the thread pool, meaning, that currently running tasks finish up, but it will not run any more until after either a timeout elapses or the call to resume is made.

Another note to mention is that the thread pool showcases the use of `event_t`, as waiting on a result is an event, so is to pause and resume.

##<center>Scoped Locks</center>

>Scoped Spinlock

~~~c
scoped_lock_conf_t conf = 
{
  .logger = my_logger,
  .flags = SCOPED_LOCK_RC_INSTANCE
};

int spinlock_flags = 0;
scoped_lock_t *s_lock = scoped_lock_spinlock_conf(spinlock_flags, &conf);
~~~

>Scoped Reader-Writer Lock

~~~c
pthread_rwlockattr_t *attr = NULL;
scoped_lock_t *s_lock = scoped_lock_rwlock_conf(attr, &conf);
~~~

>Scoped Lock Generic Creation

~~~c
// Initialized lock
pthread_mutex_t *lock;
// Uses mutex lock.
scoped_lock_t *s_lock = SCOPED_LOCK_FROM_CONF(lock, &conf);
~~~

>Don't need a lock at times? We have you covered!

~~~c
scoped_lock_t *s_lock = scoped_lock_no_op();
~~~

>Automatic acquire and release of lock.

~~~c
SCOPED_LOCK(s_lock) {
  do_something();
  do_something_else();
  if (is_something) {
      // Note, we return without needing to unlock.
      return;
  }
  finally_do_something();
}
~~~

>Even for one-liners

~~~c
SCOPED_LOCK(s_lock)
  do_something_cool();
~~~

>Specifically for Reader-Writer locks

~~~c
SCOPED_WRLOCK(s_lock);

SCOPED_RDLOCK(s_lock);
~~~

>For that nagging compiler warning

~~~c
C_UTILS_UNACCESSIBLE;
~~~

`scoped_lock_t` is an implementation of a C++-like scope_lock. The premise is that locks should be managed on it's own, and is finally made possible using GCC and Clang's compiler attributes, `__cleanup__`. The locks supported so far are `pthread_mutex_t`, `pthread_spinlock_t`, `pthread_rwlock_t`, and soon sem_t. It will lock when entering the scope, and unlock when leaving (or in the case of sem_t, it will increment the count, and then decrement). This abstracts and relaxes the acquire/release semantics for the lock, as well as generifying the type of lock used as well, as the allocation is done using C11 generics. Hence, the type of underlying lock is type-agnostic.

Lastly, another key feature to it being type-agnostic is that you can effortlessly change the underlying lock from a mutex, to a spinlock, to a semaphore and keep the code (and critical sections) the same. Of course, you can also disable locking if you specifically want to remove synchronization as well.

<aside class='warning'>
The underlying lock must support secondary locking, I.E Reader-Writer lock, to use SCOPED_RDLOCK. Hence, if you attempt to invoke it with a mutex, it will throw an assertion and abort.
</aside>

<aside class='notice'>
Since it is possible to always return inside of a for loop, and the for loop will ALWAYS execute the block, the compiler does not know this. Therefore, it will complain about not returning after a scoped block. The official way to do this is to use the C_UTILS_UNACCESSIBLE macro.
</aside>

<aside class='success'>
If used correctly, it can be an invaluable tool for writing newer multi-threaded code. The type agnosticism easily allows you to switch out locks even at runtime with no effort, or even disable locks altogether.
</aside>

##<center>Conditional Locks</center>

>Supports Mutexes

~~~c
COND_MUTEX_LOCK(lock, logger);
COND_MUTEX_UNLOCK(lock, logger);
~~~

> And Reader-Writer Locks

~~~c
// Writer Lock
COND_RWLOCK_WRLOCK(lock, logger);

// Reader Lock
COND_RWLOCK_RDLOCK(lock, logger);

// Unlock
COND_RWLOCK_UNLOCK(lock, logger);
~~~

Library provides helper macros for...

1. Automatically log any errors returned by the lock
2. Conditionally lock or unlock depending on whether the passed lock was NULL, providing a safe wrapper for disabling locks in the future.

To give an example of it's usefulness, you have to imagine a scenario where you do not want to lock due to some changes at runtime, for example, a list may utilize a lock, but, if there is only one thread, it is wasting time by acquiring and releasing the lock. This is the way to do so in the case that you do not want to use the scoped_lock_t objects and want to have manual control over when you lock and unlock.

It also extremely useful for debugging `EDEADLK` and where they occur.

##<center>Events</center>

>Creation with configuration object (optional)

~~~c
event_conf_t conf = 
{
  .logger = my_logger,
  .name = "My Event",
  .flags = EVENT_SUCCESS_ON_TIMEOUT | EVENT_RC_INSTANCE
};

event_t *evt = event_create_conf(&conf);
~~~

>Signal that an event has occured

~~~c
// Wakes one random thread.
event_signal(event);
// Wakes all threads.
event_broadcast(event);
~~~

>Wait for an event to occur

~~~c
// Milliseconds...
int timeout = 5000;
while(!event_wait(event, timeout))
  poll_then_sleep();
~~~

>Destroy the event. If it is reference counted, this decrements the count instead. When it is being destroyed, it will wake up all threads waiting on the event.

~~~c
// How long you're willing to wait until all threads finish. -1 = infinite
int max_timeout = -1;
event_destroy(event, max_timeout);
~~~

`event_t` is an events implementation built on top of a condition variable and a mutex. Provides an abstraction for using both, and some utilities for managing threads waiting on that event, and also configurations to modify the actions taken while operating under the event. This event is similar to Win32's events, in that it supports flags to allow similar functionality.

`event_t` allows you to wait on events, and supports flags which allow you to set the default state, whether or not to signal the event after a timeout, and whether or not to auto-reset the event after a thread exits the event, or after the last waiting thread leaves. 

`event_t` also will wait for other threads to finish before destruction (although it is better used with a reference count if that becomes a problem).

<aside class="warning">
Extra special care must be taken if reference counting is not being used. Although the event will not be destroyed until ALL threads inside of the event exit, any threads attempting to access it afterwards will invoke undefined behavior. Hence, you need some external way to notify threads that the event is dead.
</aside>

#<center>Memory Management</center>

This library features useful tools and abstractions which will not only help with memory management, but also improve overall efficiency.

Library | Version | Status
:------- | :-------: | ------:
Hazard Pointers | .5 | Unstable
Reference Counting | 0.75 | Unstable
Object Pool | N/A | Unimplemented

##<center>Hazard Pointers</center>

>Creating the hazard pointer

~~~c
hazard_conf_t conf = 
{
    .logger = logger,
    .hazards_per_thread = 1,
    .max_threads = 4,
    .callbacks.destructor = shared_data_destroy
};

hazard_t *h = hazard_create_conf(&conf);
~~~

>Acquiring data (and checking if data is still valid)

~~~c
int index = 0;
struct shared_data *dat;

hazard_acquire(h, dat);

hazard_acquire_at(h, dat, index);
~~~

>Releasing and retiring the data

~~~c
hazard_release(h, dat);

// Retiring means the data is set to be deleted.
hazard_retire_all(h);
~~~

>Example - Lock-Free Stack's Pop procedure

~~~c
Node_t *head, *next;
while (true) {
    head = stack->head;
    if (!head) 
        return NULL;
    
    hazard_acquire(h, head);
    if (head != stack->head) {
        pthread_yield();
        continue;
    }

    next = head->next;
    if (__sync_bool_compare_and_swap(&stack->head, head, next)) 
        break;

    pthread_yield();
}
void *data = head->item;
hazard_retire(h, head);

~~~

Provides a flexible and easy to use implementation of hazard pointers, described in the research paper by Maged M. Michael, [here](http://www.research.ibm.com/people/m/michael/ieeetpds-2004.pdf).

The hazard pointer is useful for when you have multiple threads contesting atomically over shared data, as you cannot readily free or delete the data while it is in use, as this would result in undefined behavior. In particular, this is used in lock free data structures to ensure rapid deletion and avoiding the ABA problem.

<aside class="warning">
The configured maximum number of threads using the data structure MUST be accurate. If there are too many, you end up wasting memory, and if you have too little, then there aren't enough hazard pointers to assign to each thread, which fails an assertion.
</aside>

When data is finished with, it will be destroyed via it's destructor, which cannot be changed after initial configuration.

##<center>Reference Counting</center>

>Allocating reference counted data

~~~c
struct ref_count_conf_t conf = 
{
    .logger = logger,
    .initial_ref_count = 3,
    .callbacks.desturctor = custom_destructor
};

struct A *a = ref_count_create_conf(sizeof(*a), &conf);
~~~

>Increment Reference Count

~~~c
REF_INC(a);
~~~

>Decrement Reference Count

~~~c
REF_DEC(a);
~~~

>Remove all Reference Counts

~~~c
REF_CLEAR(a);
~~~

Reference counting in this library is a bit different from most. For example, the data itself does not need to keep its own reference count, or rather, the structure itself does not need to be modified. Instead, the library allocates and manages creating meta data for you, which can easily be retrieved later to manipulate and manage the object itself. 

<aside class="warning">
You must NEVER attempt to manipulate the reference count of an object not initially created through this library. I.E, you cannot reference count malloc'd memory, nor can you free memory returned by this.
</aside>

The reference count is atomically updated on increment and decrement, but extreme care must be taken to ensuredly use this library. If you attempt to decrement the count below -1, or access after it has reached -1, then you will invoke undefined behavior so...

<aside class="warning">
Do NOT attempt to manipulate the data AFTER there are no longer any references to said data, or else this may invoke undefined behavior.
</aside>

On a side note, there are some checks in place to help ensure the appropriate use. For example, if you attempt to `REF_INC`, `REF_DEC`, or `REF_CLEAR` the data after it has been destroyed OR data never allocated by this library, chances are it will fail a check and you will be able to see why it failed.

<aside class="notice">
Almost every library in this package support reference counting in some way, and make it easier to manage them. If they support reference counting, they can optionally be destroyed by calling REF_DEC instead of their custom destructors.
</aside>

##<center>Object Pool</center>

>Create Object pool.

~~~c
object_pool_conf_t conf = 
{
    .flags = OBJECT_POOL_RC_INSTANCE | OBJECT_POOL_CONCURRENT,
    .callbacks =
    {
        .destructor = destroy_data,
        .constructor = create_data,
        .prepare = prepare_data,
        .finished = finish_data
    },
    .destroy_timer.seconds = 360,
    .growth_factor = 1.5,
    .initial_size = 10,
    .logger = logger
};

object_pool_t *pool = object_pool_create_conf(sizeof(struct my_data), &conf);
~~~

>Acquire from object pool

~~~c
struct my_data *dat = object_pool_acquire(pool);
~~~

>Release object back to pool

~~~c
object_pool_release(pool, dat);
~~~

>Destroy object

~~~c
object_pool_destroy(pool);
~~~

`object_pool_t` acts as a memory pool, which memory submitted can safely be recycled for next use. By assigning callbacks, such as destructor and constructor, one can easily have the memory pool destroy data as they are no longer needed, or create new ones when they are. Callbacks such as prepare and finished, prepare allows you to prepare data to be recycled and used, and finished allow you to configure its state so that it may be recycled. Hence, data prepared may be configured for immediate use, while finished may relinquish resources without actually destroying the data.

`destroy_timer` allows the pool to be asynchronously managed by a global instance of an `event_loop_t`. This feature is optional, but can help manage shrinkage of the pool.

<aside class="notice">
The object_pool can be managed asynchronously if the appropriate flags are used. The asynchronousity is used by a global event_loop_t instance. Hence, if you want the pool to shrink after a certain time has passed since the object has last been used, it will do so.
</aside>

#<center>String</center>

Useful abstractions for C-Strings.

Library | Version | Status
:------- | :-------: | ------:
String Manipulations | 1.3 | Stable
String Buffer | 0.75 | Unstable
Regular Expressions | N/A | Unimplemented

##<center>String Manipulations</center>

>Typedef abstraction for cstrings

~~~c
string str = "Hello World";
~~~

>Can be used with non-null terminated strings

~~~c
/*
  If length passed is 0, it will use strlen, otherwise it will do so up to the passed length. This is useful for allowing manipulations of substrings within a string itself, or for general non-null terminated strings.
*/
string_reverse(str + 6, 0);
~~~

>Concatenate multiple strings (no sentinel needed)

~~~c
// Result stored in storage.
string storage;
STRING_CONCAT_ALL(&storage, ",", str, "How are you today", "Good I hope", "Good day!");
~~~

>Ease memory management of temporary strings

~~~c
// Gets cleaned up once it leaves it's scope automatically.
string TEMP str = strdup("Hello World");
~~~

This library provides basic but much needed string abstractions for working with raw cstrings, even providing a typedef to abstract the usage of `char *`'s. It also can work on non-null terminated strings by passing the length yourself, or leaving it to be determined in advance with `strlen`.

<aside class="warning">
The string MUST be null terminated if you leave the length argument as 0. Otherwise it will result in undefined behavior.
</aside>

The `TEMP` macro-keyword, allows you to easily manage the lifetime of temporary strings without having to free them yourself, by having them cleaned up for you once they leave scope.

<aside class="warning">
Note that the variable must point to the string itself. Also note that this should NOT be used with string literals, as this will result in a segmentation fault/undefined behavior.
</aside>

##<center>String Buffer</center>

>Creating a string buffer

~~~c
string_buffer_conf_t conf =
{
  .logger = logger,
  .synchronized = true,
  .to_string = obj_to_str
};

string_buffer_t *buf = string_buffer_create_conf("Hello World", &conf);
~~~

>Append a normal cstring

~~~c
string_buffer_append(str_buf, ", I am ");
~~~

>Or, ANY type...

~~~c
// integers
STRING_BUFFER_APPEND(str_buf, 22);

// doubles
STRING_BUFFER_APPEND(str_buf, 2.3);

// Custom objects
struct obj *o;
STRING_BUFFER_APPEND(str_buf, o);
~~~

>Clear the string buffer

~~~c
string_buffer_clear(str_buf);
~~~

>Append with printf-like format

~~~c
STRING_BUFFER_APPEND_FORMAT(str_buf, "Hello World, I am %d years old!", 22);
~~~

>Delete from the beginning...

~~~c
// Now lets delete Hello World
string_buffer_delete(str_buf, 0, 12);
~~~

>Or from the end...

~~~c
// And remove the "old!" part
string_buffer_delete(str_buf, STRING_BUFFER_END - 3, STRING_BUFFER_END);
~~~

>Get underlying string

~~~c
char *str = string_buffer_get(str_buf);
puts(str);
~~~

`string_buffer_t` abstracts away the need to manually allocate strings and do tedious string manipulations. The string_buffer automatically manages resizing itself and shrinking when needed. It features a generic macro (requires C11 `_Generic` keyword) to automatically append, prepend or insert any of the standard types. It is also optionally thread-safe.

`string_buffer_t` supports an option to enable synchronizaiton, which is done through a spinlock. Majority of cases do not require synchronization, however if ever you have a case where you require one, for say a producer-consumer relationship, it can be enabled easily.

##<center>Regular Expressions</center>

###<center>#Planned</center>

* Easy abstracions for regular expressions
    - No need for cleaning up anything
* Special regex printf function
    - Use printf with regex to determine what to select from a passed string

#<center>I/O</center>

Library | Version | Status
:------- | :-------: | ------:
Logger | 1.5 | Stable
Event Loop | 0.75 | Unstable
File Buffering | N/A | Unimplemented
Streams | N/A | Unimplemented

Brings useful abstractions when dealing with streams through file descriptors. Buffering (I.E line-by-line), to asynchronous reading/writing without needing to worry if it is a FILE or socket file descriptor. Also features a configurable logging utility.

##<center>Logger</center>

>Create a logger

~~~c
char *name= "Test_File.txt";
char *mode = "w";
log_level_e lvl = LOG_LEVEL_ALL;
logger_t *logger = logger_create(name, mode, lvl);
~~~

>Automatically create and destroy

~~~c
LOGGER_AUTO_CREATE(logger, name, mode, lvl);
~~~

>Log different types of messages

~~~c
LOG_INFO(logger, "Hello %s", "World");
DEBUG("Hello World");
ASSERT(1 == 0, logger, "1 != 0!");
~~~

`logger_t` is a minimal logging utility with support for level-based logging, and even creating your own custom log level. As well, the user may define their own custom format. This allows the user to determine what information they want to see, and what they do not. All libraries inside of the library support logging, and will log to the passed logger, allowing the user to easily inject their own loggers to debug and trace information.

`logger_t` can be created through `LOGGER_AUTO_CREATE`, which works by using the clang and gcc compiler attributes, `__constructor__` and `__destructor__`, to automatically handle creation and destruction based on linkage.

Below is an example of a custom format. This is the default logging format used when no custom format has been provided.

`"%tsm \[%lvl\](%fle:%lno) %fnc(): \n\"%msg\"\n"`

Would produce the following:

9:39:32 PM \[INFO\](test_file:63) main():
"Hello World!"

The currently implemented log format tokens are...

Tokens | Format
:----- | -----:
%tsm | Timestamp (HH/MM/SS AM/PM)
%lvl | Log Level
%fle | File
%lno | Line Number
%fnc | Function
%msg | Message
%cond | Condition (Used for assertions)

##<center>Event Loop</center>

>Below is a rather extensive example of how a dispatch for the event loop would look for a server that reads data from a client, formulates it into an HTTP request, and formulates their own HTTP response. Quite a few things are left out, but it should get the point across.

~~~c
event_flags_e dispatch(void *usr_data, int fd, int flags) {
  char buf[BUFSIZ];
  string_buffer_t *buf = usr_data;

  if(flags & EVENT_FLAGS_READ) {
    int bytes = read(fd, buf, BUFSIZ);
    if(!bytes)
      return EVENT_FLAGS_READ_DONE;

    STRING_BUFFER_APPEND_FORMAT(".*s", buf, bytes);

    // We re-enable polling for write here if it has been disabled elsewhere.
    return EVENT_FLAGS_WRITE;
  }

  if(flags & EVENT_FLAGS_WRITE) {
    char *str = string_buffer_take(buf);
    /*
      We have to wait until we can READ first, so we opt out of receiving events for write unless we have some data in our buffer. More efficient this way.
    */
    if(!str)
      return EVENT_FLAGS_WRITE_DONE;

    /*
      Treat what we read as an HTTP request, and formulate a response here.
    */
    response_t *res = create_response_from_request_str(str);
    char *res_str = response_to_string(res);
    int bytes = write(fd, res_str, strlen(res_str));
    if(bytes != strlen(res_str))
      handle_excess_data(res_str + bytes);

    return EVENT_FLAGS_NONE; // or return 0;
  }
}
~~~

>Create event sources, with the extensive configuration objects

~~~c
event_source_conf_t conf = 
{
  .user_data = str_buf,
  .logger = my_logger,
  .callbacks.finalizer = string_buffer_destroy,
  .flags = EVENT_SOURCE_RC_INSTANCE
};

event_source_t *source = event_source_create_conf(int fd, dispatch, &conf);
~~~

>Create a local event using pipes; the event loop polls on the reader file descriptor, and the user writes to the writer file descriptor

~~~c
int write_fd;
event_source_t *source;
EVENT_SOURCE_LOCAL_CONF(source, write_fd, local_dispatcher, &conf);
~~~

>Create an event for asynchronously reading a FILE.

~~~c
FILE *fp;
event_source_t *source;
EVENT_SOURCE_FILE_CONF(source, fp, async_file_reader, &conf);
~~~

>Create a timer event

~~~c
// Milliseconds.
int timeout = 5000;
event_source_t *source = event_source_create_timed_conf(timeout, timed_dispatch, &conf);
~~~

>Finally, create the loop add these sources!

~~~c
event_loop_t *loop = event_loop_create();
event_loop_add(source);
~~~

`event_loop_t` is a callback-based event loop which dispatches events once the file descriptors associated with them are ready. `event_source_t` objects can be created to configure their specific behavior, such as the functions used to dispatch once ready, the user_data to pass to each dispatch function, and how to finalize the data once finished, if applicable.

The `event_source_t` objects can also be created through their useful helper constructors and macros to allow for easier setting up of events. Dispatcher functions can return flags which help to notify the `event_loop_t` what to poll for when using that file descriptor, hence allowing dynamic and responsive events when done correctly.

<aside class="notice">
Dispatch functions SHOULD be short and should NOT ever block. That is, one should NOT poll for data from one file descriptor and write to another as they could potentially block. Instead, local event_source_t objects can help act as a medium between reading, writing to a buffer, then writing the contents of that buffer to another file descriptor once ready. That, or reading all into a buffer, then waiting until they have finished, and THEN submit the buffer as a new event to be written.
</aside>

The `event_source_t` objects, when reference counted, are extremely useful, as they can then be have their reference stolen by the `event_loop_t` and have it handle destruction of the event once it finishes, as well as finalizing the data.

The main benefit of using an `event_loop_t` over a thread pool is that, one, it uses less resources and is more efficient when you need to involve multiple threads, and as well there is no need to worry about synchornization as things occur sequentially  if everything is handled by the `event_loop_t`. 

##<center>File Buffering</center>

###<center>Planned</center>

* Stream over a file by buffering a certain amount of it at a time
    - Will support buffering by line.
        + Effortless next_line and prev_line abstractions

##<center>Asyncronous Streams</center>

###<center>Planned</center>

* Stream over an abitrary collection of items
    - strings 
    - lists
    - maps
    - arrays
    - etc.
* Streams can be updated concurrently as they are being taken from
    - Easy Producer-Consumer

#<center>Networking</center>

Provides basic, but extremely powerful networking abstractions to help make using sockets less of a headache.

Library | Version | Status
:------- | :-------: | ------:
Socket Utilities | N/A | Unimplemented
HTTP | 0.5 | Unstable
Connections | 1.0 | Deprecated
Server | 1.0 | Deprecated
Client | 1.0 | Deprecated

##<center>Socket Utilities</center>

>Connect to an endpoint... Synchronously

~~~c
char *ip_addr = "192.168.1.2";
unsigned int port = 8000;
// Milliseconds
long long timeout = 5000;
int sfd = socket_connect(ip_addr, port, timeout);
~~~

>Or Asynchronously

~~~c
socket_connect_async(ip_addr, port, timeout, on_connect, on_failure);
~~~

>Become an endpoint

~~~c
unsigned int backlog = 10;
int bound_fd = socket_host(port, backlog);
~~~

>Accept new connections... Synchronously

~~~c
// -1 == infinite timeout.
long long timeout = -1;
int conn_fd = socket_accept(bound_fd, timeout);
~~~

>Or asynchronously

~~~c
socket_accept_async(bound_fd, timeout, on_accept, on_failure);
~~~

>Timedout operations... Synchronous

~~~c
char buf[BUFSIZ];
int read = socket_read(conn_fd, buf, BUFSIZ, timeout);

int written = socekt_read(conn_fd, buf, read, timeout);
~~~

>Asynchronous

~~~c
char *buf = malloc(BUFSIZ);
socket_read_async(conn_fd, buf, BUFSIZ, on_read, on_failure);

socket_write(conn_fd, buf, buf_size, on_write, on_failure);
~~~

Provides a minimalistic and easy to use API abstraction that allows for easy and effortless management and creation of connections. Both synchronous and asynchronous operations are supported, however the asynchronous operations utilize the global `event_loop_t`. 

The synchronous differ over normal read and write by offering timeout operations for anything that could block. The asynchronous versions allow the user to easily manage multiple asynchronous sockets without having to poll on it themselves. 

<aside class="warning">
You MUST NOT block inside of the asynchronous handlers, as they WILL stall the event_loop_t instance, causing all operations to slow down. If you NEED to block, use a thread_pool_t instead and handle asynchronicity yourself.
</aside>

<aside class="success">
When asynchronocity is used correctly, one can abstract the need to use event_loop_t directly yourself.
</aside>

##<center>HTTP</center>

>Create an HTTP generic header

~~~c
header_t *header = header_create();
~~~

>Or generate from an HTTP header (request or response)

~~~c
int fd;
char header_str[BUFSIZ];
int header_size = read(fd, header_str, BUFSIZ);
// Returns offset after the actual header.
header_t *header = header_from(header_str, &header_size);
~~~

>Obtain mapped field-value

~~~c
// Or "Content-Length"
char *len = header_get(header, HTTP_CONTENT_LENGTH);
// Or "User-Agent"
char *ua = header_get(header, HTTP_USER_AGENT);
~~~

>Set header field values.

~~~c
size_t len;
char *len_str;
asprintf(&len_str, "%zu", len);

header_set(header, HTTP_CONTENT_LENGTH, len_str);
~~~

>Set mass header fields

~~~c
HEADER_WRITE(header, 
    { 
        { HTTP_CONTENT_LENGTH, len_str },
        { HTTP_CONTENT_TYPE, content_type },
        { HTTP_VERSION, HTTP_VERSION_1_0 },
        { HTTP_STATUS, HTTP_STATUS_400 }
    }
);
~~~

>Obtain type of header

~~~c
header_type_e type = header_type(header);
if(type != HTTP_TYPE_REQUEST)
    handle_bad_header(header);
~~~

>Generate header

~~~c
size_t len;
char *res = header_generate(header, &len);
~~~

`header_t` contains some very minimal and basic HTTP mapping, where it can determine what type of HTTP header has been sent (Response vs Request) and can be used to generate an HTTP header as well. It can be constructed either from a pre-existing HTTP string header, or from scratch. 

`header_t` can be used when you require a basic HTTP header handling/parsing tool.

<aside class='notice'>
To generate a header from a string, keep in mind that the pointer to the length must maintain the size, and it will return the offset after the header (I.E, where the message body begins).
</aside>

#<center>Data Structures</center>

An assortment of data structures, all of which are highly configurable, thread-safe, and support reference counting. Some provide iterator implementations that are thread-safe and highly concurrent when used right.

##<center>Iterator</center>

>Create an iterator from a list

~~~c
iterator_t *it = list_iterator(list);
~~~

>Create a scoped-based iterator

~~~c
AUTO_ITERATOR *it = list_iterator(list);
~~~

>For-Each iterator

~~~c
void *item;
// General Iterator For-Each
ITERATOR_FOR_EACH(item, it)
    do_something_with(item);

// Helper Macro for List
LIST_FOR_EACH(item, list) {
    do_something();
    do_something_with(item);
    print_item(item);
    if(bad_item(item))
        continue;

    iterator_remove(_this_iterator);
}
~~~

>Iterate manually

~~~c
void *item;
while(item = iterator_next(it))
    do_something_with(item);

while(item = iterator_prev(it))
    do_something_else_with(item);
~~~

`iterator_t` is a callback-based (ergo implementation-specific) iterator, which in this library, makes concurrent iterations easy across multiple threads. `iterator_t` will also maintain a reference count to the underlying data structure if the data structure itself is reference counted (implementation specific). The iterator may maintain a reference to the items it currently is iterating over to ensure that the current user can always safely use this iterator, if and only if the data structure it was created from was configured to do so.

<aside class='warning'>
While the reference counting can prevent any such memory leaks or undefined behavior (freeing while in use) from occuring through concurrent access, improper management of the iterator, I.E keeping it around long than it should, will leak not only the current item it is on, but also the data structure as well. If a scoped-based iterator is needed, use the AUTO_ITERATOR macro.
</aside>

`iterator_t` in the scope of this utilities package is a highly concurrent and easy-to-use iterator for a data structure. Memory management is made easier through reference counted, and concurrent access is generally enforced through reader-writer locks. Concurrent writes generally will not invalidate the iterator unless the actual node is removed, however it does feature a node-corrections algorithm where it will attempt to recover if at all possible.

<aside class='notice'>
Although the iterator can iterate through a thread-safe data structure, it should NOT be used concurrently itself. Two threads manipulating the iterator can and will invoke undefined behavior.
</aside>

`iterator_t` is more useful in the cases where you have multiple concurrent readers iterating over the data structure at once, and fewer writers, although the `iterator_t` may be used in any cases. The data structures which implement the iterator will generally create their own specific helper macros.

Not all callbacks need to be implemented by the underlying data structure. If they are not, they automatically return a failing value (false or NULL).

##<center>Linked List</center>

>Create a list

~~~c
list_t *list = list_create();
~~~

>Create a concurrent, logged, sorted list

~~~c
list_conf_t conf =
{
    .logger = logger,
    .flags = LIST_CONCURRENT,
    .callbacks =
    {
        .comparator = my_comparator,
        .destructor = my_destructor
    }
};

list_t *list = list_create_conf(&conf);
~~~

>Add an element to the list

~~~c
void *item;
list_add(list, item);
~~~

>Obtain an element in the list

~~~c
int index = 4;
list_get(list, index);
~~~

>Remove or Delete an element

~~~c
// Remove will just remove from the list, transfering ownsership to caller
list_remove(list, item);
list_remove_at(list, index);
list_remove_all(list);

// Delete will either call destructor or decrement count over item.
list_delete(list, item);
list_delete_at(list, index);
list_delete_all(list);
~~~

>For-Each macro & function

~~~c
LIST_FOR_EACH(item, list)
    do_something_with(item);

list_for_each(list, do_something_with);
~~~

A highly-concurrent, configurable, doubly linked list implementation. Concurrency is optional, however when enabled, all calls will be done through reader-writer lock, allowing for parallelized access so long as they do not mutate the map. This is ideal for `iterator_t` objects, as it allows multiple readers. 

If a comparator is specified, it will act as an ordered linked list, however this feature cannot be changed after creation. The list can be reference counted if specified, and also allow the items themselves to be as well.

<aside class='warning'>
The items MUST have been created with ref_count_create call. If they have not, this may invoke undefined behavior.
</aside>

<aside class='success'>
If used correctly, reference counting can make managing the list and accesses between threads easy. You could have multiple threads remove items from the list, add more, iterate, etc., and the items themselves will remain valid for as long as the iterator maintains a reference to it.
</aside>

The list, by default, is a normal non-synchronized, non-reference counted linked list, and is suitable for use in all applications.

##<center>Blocking Queue</center>

>Create a default blocking queue

~~~c
blocking_queue_t *bq = blocking_queue_create();
~~~

>Create a reference counted, prioritized, logging, bounded blocking queue

~~~c
blocking_queue_conf_t conf =
{
  .logger = logger,
  .flags = BLOCKING_QUEUE_RC_INSTANCE,
  .callbacks.comparator = my_comparator
};

blocking_queue_t *bq = blocking_queue_create_conf(&conf);
~~~

>Enqueue (Producer)

~~~c
// Only blocks if bounded
long timeout = -1;
void *item;
blocking_queue_enqueue(bq, item, timeout);
~~~

>Dequeue (Consumer)

~~~c
// Blocks if empty, in milliseconds
long timeout = 5000;
void *item = blocking_queue_dequeue(bq, timeout);
~~~

>Shutdown - Wakes up any blocked threads.

~~~c
blocking_queue_shutdown(bq);
~~~

`blocking_queue_t` is an ideal producer-consumer queue, which can optionally be bounded, and even turn into a priority blocking queue if a comparator is passed. It can be reference counted, and supplies a method to wake up all blocked threads.

<aside class='notice'>
Destroying the queue intrinsically calls blocking_queue_shutdown, so any blocked threads will wake up and exit the queue.
</aside>

##<center>Vector</center>

>Create a default vector

~~~c
vector_t *vec = vector_create();
~~~

>Create a custom vector

~~~c
vector_conf_t conf =
{
  .logger = logger,
  .flags = VECTOR_RC_INSTANCE,
  .callbacks =
  {
    .comparator = my_comparator,
    .destructor = my_destructor
  }
};

vector_t *vec = vector_create_conf(&conf);
~~~

>Add an element to the vector

~~~c
void *item;
vector_add(vec, item);
~~~

>Get an element from vector

~~~c
int index = 4;
void *item = vector_get_at(vec, index);
~~~

>Remove or Delete elements from the vector

~~~c
// Remove will transfer ownership to caller, removing from vector.
int index = 4;
void *item = vector_remove_at(vec, index);
vector_remove(vec, item);
vector_remove_all(vec);

// Delete will invoke destructor or decrement count
vector_delete_at(index);
vector_delete(vec, item);
vector_delete_all(vec);
~~~

`vector_t`, unlike `list_t`, is backed by a dynamically allocated array. This means that it is specialized for random access, however insertions and deletions should be done more sparingly. Also, unlike `list_t` or `map_t`, it isn't really built for concurrent access, so it is synchronized with a `pthread_mutex_t` when concurrent access is flagged. Even so, the iterator is still available for iteration, although it should be noted, it is not parallel like `list_t`, hence this should be used for more random-access oriented data.

Like `list_t` it can become a sorted vector if a comparator is passed on creation, but this cannot be changed at runtime.

##<center>Stack</center>

>Create a simple stack - NOTE: stack_t is reserved by POSIX and so not typedef'd

~~~c
struct c_utils_stack *stack = stack_create();
~~~

>Create a lock-free stack

~~~c
struct c_utils_stack_conf_t conf =
{
  .flags = STACK_CONCURRENT,
  .logger = logger
};

struct c_utils_stack *stack = stack_create_conf(&conf);
~~~

>Push

~~~c
void *item;
stack_push(stack, item);
~~~

>Pop

~~~c
void *item = stack_pop(stack);
~~~

A simple, yet thread-safe stack implementation. When flagged to be concurrent, all operations use atomics (compare-and-swap) and hazard pointers to ensure optimal thread safety. Overall it is relatively simple.

##<center>Queue</center>
>Create a simple queue

~~~c
queue_t *queue = queue_create();
~~~

>Create a lock-free queue

~~~c
queue_conf_t conf =
{
  .flags = QUEUE_CONCURRENT,
  .logger = logger
};

queue_t *queue = queue_create_conf(&conf);
~~~

>Enqueue

~~~c
void *item;
queue_enqueue(queue, item);
~~~

>Dequeue

~~~c
void *item = queue_dequeue(queue);
~~~

A simple, yet thread-safe queue implementation. When flagged to be concurrent, all operations use atomics (compare-and-swap) and hazard pointers to ensure optimal thread safety. Overall it is relatively simple.

##<center>Ring Buffer</center>

>Create a Ring Buffer

~~~c
buffer_t *buf = buffer_create();
~~~

>Create a more specific ring buffer

~~~c
buffer_conf_t conf = 
{
  .logger = logger,
  .flags = BUFFER_CONCURRENT,
  .callbacks =
  {
    .allocator = my_alloc,
    .destructor = my_free
  },
  .initial_size = 1024 * 1024
};

buffer_t *buf = buffer_create_conf(&conf);
~~~

>Write

~~~c
char *buf;
buffer_write(buf);
~~~

>Read

~~~c
// How much is read returned in len.
size_t len = BUFFER_READ_ALL;
char *data = buffer_read(buf, &len);

char tmp_buf[BUFSIZ];
size_t read = buffer_read_into(buf, tmp_buf, BUFSIZ);
~~~

`buffer_t` is a basic ring buffer implementation allowing to overwrite older portions with newer data.

##<center>Hash Map</center>

>Create a hash map for normal `char *`, `void *` pairs

~~~c
map_t *map = map_create();
~~~

>Create a reference counted hash map for `void *`, `void *` pairs

~~~c
map_conf_t conf = 
{
  .logger = logger,
  .flags = MAP_RC_INSTANCE | MAP_RC_KEY | MAP_RC_VALUE | MAP_CONCURRENT,
  .key_len = sizeof(struct my_obj),
}
~~~

>Add a new key-value pair

~~~c
char *key = "Hello";
char *value = "World";

map_add(map, key, value);
~~~

>Get a key-value pair

~~~c
char *key = "Hello";
char *value = map_get(map, key);
~~~

>Remove or Delete a key-value pair

~~~c
char *key = "Hello";
char *value = map_remove(map, key);

map_delete(map, key);
~~~

>Set value to a key (if it exists)

~~~c
char *key = "Hello";
char *value = "Good Bye"
map_set(map, key, value);
~~~

>Iterate over a map

~~~c
char *key, *value;
MAP_FOR_EACH_KEY(key, map)
  do_something_with(key);

MAP_FOR_EACH_VALUE(value, map)
  do_something_with(value);

MAP_FOR_EACH_PAIR(key, value, map)
  do_something_with_pair(key, value);
~~~

A highly concurrent and configurable hash map implementation. The hash map allows and is optimized for concurrent readers through it's reader-writer lock, and hence is ideal for the `iterator_t` instances. 

Like `list_t`, the `iterator_t` will keep a reference count to the underlying data structure if it has been configured, and the key-value pairs can be reference counted as well.

<aside class='warning'>
The reference counted data MUST have been created with ref_count_create or else you invoke undefined behavior!
</aside>

`map_t` by default is set up to allow `char *` string keys with any type of value, however if the `key_len` is specified, it's default hash can be used, meaning the user doesn't need to implement their own. Doing so will allow any type of data to be used as the hash, as it will just hash each byte up to the passed `key_len`.

<aside class='warning'>
If key_len or hash_function is not specified, it will invoke strlen on it, as it will treat it like a cstring. Hence it is CRUCIAL that if you use the default map, you are using a string (or at least have a 0 byte set within the struct).
</aside>

`map_t` can also have it's growth rate and trigger configured, as well as enable shrinking once it approaches the configured trigger by the configured rate. Defaults are general enough, but if specific features are needed, they can be done so through `map_conf_t` object.

`iterator_t` instances returned from `map_iterator` will be iterating over a copy of the data at the given time it's first used after it is reset. Hence, it will keep a reference to each key-value pair as well, so it is urged that you do not leak this, as you risk leaking your data as well as the data structure as well, which means the other data it holds too.

It should further be noted that due to this, while a iterator is in use, the data at the time it began MUST be valid if it is not being reference counted. If you do not like this arrangement, `map_for_each` can handle non-reference counted data easily as it is done under one pass of the reader-lock.


#<center>Misc.</center>

Miscallaneous utilities which help abstract and manage tedious operations.

##<center>Flags [<b>Stable</b>] Version: 1.0</center>

>Check if a bit-flag is set in mask

~~~c
if(FLAG_GET(mask, flag))
~~~

>Set a bit-flag in mask

~~~c
FLAG_SET(mask, flag);
~~~

>Clear flag from mask

~~~c
FLAG_CLEAR(mask, flag);
~~~

>Toggle bit-flag in mask

~~~c
FLAG_TOGGLE(mask, flag);
~~~

Convenience macros for bitwise operations for when you do not want to bother remembering all of them.

Macro | Operation
:---- | --------:
`FLAG_GET` | `mask & flag`
`FLAG_SET` | `mask |= flag`
`FLAG_CLEAR` | `mask &= ~(flag)`
`FLAG_TOGGLE` | `mask ^= flag`

##<center>Argument Checking</center>

>Example of checking argument of a function's parameters

~~~c
struct {
    bool is_valid;
} A;

bool test_func(char *msg, int val, struct A *a) {
    ARG_CHECK(logger, false, msg, val > 0 && val < 100, a, a && a->is_valid);
}
~~~

Features a very simple and easy to use macro that can check up to 8 arguments, logging the conditionals as strings and whether or not they are true or false. It should be noted that due to the limitations of macros, it does not feature short-circuit evaluations, hence if you are going to be checking struct members for validity you must check each time to see if the struct exists.

If any of the conditions fail, it will output the following. For this example, assume test's is_valid member is false.

`Invalid Arguments=> { msg: TRUE; val > 0 && val < 100: TRUE; test: TRUE; test && test->is_valid: FALSE }`

##<center>Allocation Checker</center>

>Malloc and Check

~~~c
char *str;
ON_BAD_MALLOC(str, logger, 6)
    goto err;
~~~

>Calloc and Check

~~~c
pthread_mutex_t *lock;
ON_BAD_CALLOC(lock, logger, sizeof(*lock))
    goto err_lock;
~~~

>Realloc and Check

~~~c
int *arr;
ON_BAD_REALLOC(&arr, logger, sizeof(*arr) * 5)
    goto err_arr;
~~~

Simple macros which check for a bad allocation, and if so will execute the block of code after it, using the macro for loop trick. So, if you wanted to use malloc, but return NULL or free up other resources on error, you would define the on-error block which will ONLY be called if things go wrong.

##<center>Portable TEMP_FAILURE_RETRY</center>

>Restart fread on signal interrupt Example

~~~c
FILE *file = fopen(...);
/// Assume this contains a valid file.
char buf[BUFSIZ];
size_t bytes_read;
C_UTILS_TEMP_FAILURE_RETRY(bytes_read, fread(buf, 1, BUFSIZ, file));
/// Etc.
~~~

As GCC's TEMP_FAILURE_RETRY macro allows you to restart functions which return -1 and set errno to EINTR, which allow for consistent programming regardless of signals. GCC's implementation uses statement expressions, which unfortunately Clang does not support, hence it has been ported into the utilities package.
