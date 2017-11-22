# AsyncMessenger
Messenger-->SimplePolicyMessenger-->AsyncMessenger.
AsyncMessenger is represented for mantaining a set of async connection connections, it may own a bind address and the acceptd connections will be managed by AsyncMessenger.

AsyncMessenger Members:

Name  | Type               | Description
------|--------------------|-----------
processors | vector<Processor*> | A vector of Processor instances, they are used to bind socket and handle connection requests.
stack | NetworkStack | Get the real network stack type
conns | unordered_map<entity_addr_t, AsyncConnectionRef> | Hash map of addresses to AsyncConnection

AsyncMessenger Methods:

Name | Return Type | Description
-----|-------------|-------------
AsyncMessenger | construct | Create a network stack -> create workers -> create processors
bind | int | The AsyncMessenger bind a IP address actually is all processors bind a adress (In DPDK, called in Server side)
get_connection | ConnectionRef | Createa a connection in conns is it doesn't exist.
_send_messsage | int | Send a message by connection we are looking for. In the invoke conn->send_message(m)

# Processor
If messenger binds to a specific address, the processor runs and listens for incming connections.:

Processor members and methods:

Name | Type        | Description
-----|-------------|-------------
worker | Worker | its worker thread
listen_handler | EventCallbackRef | The listen handler create when processor construct.
bind | int | Processor bind its own address
accept | void | Processor accept connection from outside, DPDK support local listen table. Add connction into accepting_cons

# Worker
All the detail works are done by workers. They work in a loop, and they keep using their own eventCenter to `process_event`. In process_event function, use event_wait function. Take epoll_wait to get the works need to deal with. EventCenter create with this Worker. We created a NetworkStack and a EventCenter instance when we create a worker in Messenger module boot up.

# NetworkStack
Stack actually is a worker pool. And it also defined two different important sockets 

Name | Type        | Description
-----|-------------|-------------
workers | vector<Worker*> | The worker threads set. Stack put all the created worker into this set
num_workers | unsigned | The owrker numbers
create(CephContext *c, const string &t) | shared_ptr<NetworkStack> | Create a NetworkStack
create_worker(CephContext *c, const string &type, unsigned i) | Worker * | Create a "DPDK/RDMA/POSIX" worker
add_thread(unsigned i) | std::function<void ()> | The function all the workers are running
NetworkStack | construct | Create some workers by create_worker and initialize their event center
start | void | Tell all the workers what function they need to run
get_worker | Worker* | Pick a lowest loaded ref worker to return

## ServerSocket
A listening socket, waiting to accept incoming network connections. All his workers are done by `class ServerSocketImpl`.
Which is implementated in DPDK or other methods.

Name | Type        | Description
-----|-------------|-------------
_ssi | std::unique_ptr<ServerSocketImpl> | ServerSocket Implementation.
accept(ConnectedSocket *sock, const SocketOptions &opt, entity_addr_t *out, Worker *w) | int | Accepts a ConnectedSocket representing the connection the connection, and a entity_addr_t describing the remote endpoint

## ConnectedSocket
A ConnectedSocket represents a full-duplex stream between two endpoints, a local endpoint and a remote endpoint.
All the methods are implemented in the lower level DPDK or Posix.

Name | Type        | Description
-----|-------------|-------------
_csi | unique_ptr<ConnectedSocketImpl> | ConnectedSocket implementation
is_connected() | int | _csi->is_connected() check if the connection is connected
read(char* buf, size_t len) | ssize_t | Copy an object returning data sent from the remote endpoint
zero_copy_read(bufferptr &data) | ssize_t | Gets an object returning data sent from the remote endpoint
send(bufferlist &bl, bool more) | ssize_t | Gets an object that sends data to the remote endpoint

# AsyncConnection
The most important class in async messenger. Connection create delete and read/write, rebuild and messages handle in this class.

AsyncConection Members:

Name | Type        | Description
-----|-------------|-------------
conn_id | uint64_t | connection id
out_q | map<int, list<pair<bufferlist, Message*>>> | Priority queue for outbound msgs.
sent | list<Message*> | All the messages already sent (the first bufferlist need to inject seq)
outcoming_bl | bufferlist | The messages try to send out
read_handler | EventCallbackRef | Handle the read request callback
write_handler | EventCallbackRef | Handle the write request callback
connect_handler | EventCallbackRef | Handle connect request
data_buf | bufferlist | the data bl
data_blp | bufferlist::iterator | data_buf pointer
front,middle,data | bufferlist | Data partition
connect_msg | ceph_msg_connect | Message connection
center | EventCenter | EventCenter object, used to handle events
net | NetHandler | Handle the network connection

```
// Tis section are temp variables used by state transition
// Open state
utime_t recv_stamp;
utime_t throttle_stamp;
unsigned msg_left;
uint64_t cur_msg_size;
ceph_msg_header current_header;
bufferlist data_buf;
bufferlist::iterator data_blp;                                               
bufferlist front, middle, data;
ceph_msg_connect connect_msg;

// Connecting state    
bool got_bad_auth;
AuthAuthorizer *authorizer;
bufferlist authorizer_buf;
ceph_msg_connect_reply connect_reply;

// Accepting state
entity_addr_t socket_addr;                                                   
CryptoKey session_key; 
bool replacing; 
bool is_reset_from_peer;
bool once_ready;
  
// used only for local state, it will be overwrite when state transition
char *state_buffer;
// used only by "read_until"
uint64_t state_offset;
Worker *worker;
EventCenter *center;
ceph::shared_ptr<AuthSessionHandler> session_security;

```

AsyncConection Methods:

Name | Type        | Description
-----|-------------|-------------
_try_send | ssize_t | Put outcomming_bl in send buffer by ConnectionSocket. This cs created when AsynCon::accept
prepare_send_message(uint64_t features, Message *m, bufferlist &bl) | void | Encode message into bl
read_until | int | invoke read_bulk until r = 0.
_process_connection() | int | Handle connection state
_connect() | void | Put STATE_CONNECTING into state, and then invoke dispatch_event_external to put read_handler into external_events
handle_ack(uint64_t seq) | void | Handle ack message, delete the message in `sent` queue.
write_message(Message *m, bufferlist& bl, bool more) | ssize_t | Write message into complete_bl, and invoke _try_send to send message.
_reply_accept(char tag, ceph_msg_connect &connect, ceph_msg_connect_reply &reply, bufferlist &authorizer_reply) | ssize_t | Create a reply bufferlist and send it out with try_send
handle_write | void | Take a loop to utilize write_message to put data in bufferlist and send out.

# EventCetner and EventCallback
Asyncmessenger is based on epoll driver. We don't need a pipe to tx/rx data. All the operations are handle by callback.
AsyncMessenger employ event, all the event handler need a data strucut, which is EventCenter. EventCenter defined a FileEvent and a TimeEvent, most events are FileEvent.

EventCenter Methods/Members:

Name | Type        | Description
-----|-------------|-------------
external_events | deque(EventCallbackRef) | Array to store the external events
file_events | vector<FileEvent> | FileEvent instances arrary
time_events | std::multimap<clock_type::time_point, TimeEvent> | Time event container
*driver | EventDriver | EventDriver instance
process_time_events() | int | Handle time events
process_events | int | If the events are EVENT_READABLE or EVENT_WRITEABLE, handle by the callback in event. Otherwise, external_events in cur_process.
create_file_events(int fd, int mask, EventCallbackRef ctxt) | int | Create a file event by fd and mask, add_event into the handler.
create_time_events(uint64_t microseconds, EventCallbackRef ctxt) | uint64_t | Create a time event, add to time_events
dispatch_event_external(EventCallbackRef e) | void | Put an external event into external_events
get_file_events() | FileEvent | Get the file events from file_events
Poller::Poller | construct | Create a poller for this EventCenter
  
# EpollDriver
EventCenter is just an event handle container, it won't deal with the events. The events are handled by the EventDriver, which is an interface implemented by EpollDriver in Linux. `epoll_create` is implemented by a system call. It will create a file endpoint in epoll file system, which is used to store the ready events and open a kernel cache space and build a RB tree. `epoll_ctl` put the listenning socket on a RB tree, then register a callback. If the data for that handler is arrived, put it into the ready list. `epoll_wait` keep tracking if the ready list has data.

EventDriver members and methods:

Name | Type        | Description
-----|-------------|-------------
epfd | int | epoll file description is created by epoll_create in EpollDriver::init
*event | struct epoll_event | An epoll_event object
size | int | The file count when initialize
init(EventCenter *c, int nevent) | int | Create a epoll driver object
add_event(int fd, int cur_mask, int add_mask) | int | Add event by different mask, epoll_ctl add event
del_event(int fd, int cur_mask, int delmask) | int | delete event or modify
resize_events(int newsize) | int | clean the events number
event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tvp) | int | Loop to get a handle events
