TM_SPAWN_MULTI README

The goal of this project is to implement a parallel version of tm_spawn using tpp_mcast
functions. 

Currently the tm_spawn_multi interface, in file tm.c, looks like this:

int
tm_spawn_multi(int argc, char **argv, char **envp,
	tm_node_id where[], int list_size,  tm_task_id *tid[], tm_event_t *event)

* argc - argument count
* argv - list of arguments, argv[0] is the name of the executable to run
* where - a list of nodes to spawn the task on
* list_size - the size of where and tid array
* tid - a list of tids
* event - the event number that will be used to track this request

Below is a description of all the steps of a tm_spawn_multi, with the code location in 
parenthesis. 

Step 1: client calls tm_spawn_multi interface

When a client of tm interface such as pbsdsh, which I use to test my implementation, calls 
tm_spawn_multi, the tm_spawn_multi function (in tm.c) sends the request header to the mom 
(the startcom() function), with a new command that I created call TM_SPAWN_MULTI
(definition in tm_.h), and then sends the node count, the list of nodes, the argument count,
the list of arguments, and environment variables. The client, pbsdsh, then waits on tm_poll(). 

Step 2: mother superior receives tm_spawn_multi request 

When MS gets the TM_SPAWN_MULTI request (in mom_comm.c, tm_request() function, under case
TM_SPAWN_MULTI of a very large switch statement, line 6432) it reads all the information
about the spawn, and iterates through the list of nodes, if MS itself is in the list, MOM
spawns a tasks on itself, and if there are other nodes in the list, MS composes a 
IM_SPAWN_MULTI request (definition in job.h) and sends it to the concerning nodes using 
tpp_mcast. During this process it also allocates events to track these requests. 

Step 3: sisters receive im_spawn_multi request

Once a sister mom receives a IM_SPAWN_MULTI request, (in mom_comm.c, im_request() function,
line 3975) it creates a new task, and sends back an IM_ALL_OKAY message, along with the new 
task id, back to Mother Supiror. If any error occurs, the sister sends IM_ERROR or IM_ERROR2
messages instead. 

Step 4: mother superior receive IM_ALL_OKAY or IM_ERROR

Once mother superior receieves a reply from a sister, it looks up an event created previously
that matches the event number in the reply, and the command type, and deletes that event 
(in mom_comm.c line 3507). So once all the events created for the IM_SPAWN_MULTI command is
gone we know all sisters have replied, and the mother superior can reply back to the client,
pbsdsh, using tm_reply(this part is in mom_comm.c starting at line 4928), and also sends a
list of task ids. 


Step 5: the client receive a reply from mother superior 

The tm_poll() function, running on the client, receives the reply and task ids from mother
superior and creates internal data structures for the tasks (in tm.c), and the client
stops waiting. 


In the current state of this branch, step 1, 2, 3 and 4 are implemented and is working.
Step 3 has a bug that sometimes mother superior's message ends prematurely, and one of the 
sisters will not receive the IM_SPAWN_MULTI request. I have added many log statements
inside the code to help debugging. You will see them in the mom log. They all begin with
#MLIU or @MLIU or $MLIU. 

Step 4 is not finished yet as the mother superior is not storing all the task ids received 
from sisters, and sending them back to the client yet.  

Step 5 is not implemented yet, if you look at tm_poll() of tm.c, under case TM_SPAWN_MULTI
you will see that it is doing the same thing as TM_SPAWN.
