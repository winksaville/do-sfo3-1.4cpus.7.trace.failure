# do-sfo3-1.4cpus.7.trace.failure

At 2022-04-21T22:34:42.211273Z I started relay_v2 and stopped it at 22-04-21T22:38:16.640Z on do-sfo3-01 
in a Basic / 8 GB / 4 vCPUs / 25 GB SSD:
```
wink@do-sfo3-01 22-04-21T22:34:22.603Z:~
$ RUST_LOG=trace relay_v2.Debug-relay_v2-304f6a6c.nightly-1.62.0-879aff385.debug --port 4001 --secret-key-seed 0 &> relay_v2.Debug-relay_v2-304f6a6c.nightly-1.62.0-879aff385.debug.4cpus.7.trace.log
^C
wink@do-sfo3-01 22-04-21T22:38:16.640Z:~
```

On my desktop, 3900x, I ran `libp2p-lookup` twice:
```
$ cat 3900x-libp2p-lookup.cmds.txt
wink@3900x 22-04-21T22:22:03.800Z:~/prgs/downloads/virtualbox
$ libp2p-lookup direct --address /ip4/164.92.118.108/tcp/4001
[2022-04-21T22:35:47Z ERROR libp2p_lookup] Lookup failed: Timeout.
wink@3900x 22-04-21T22:35:47.416Z:~/prgs/downloads/virtualbox
$ libp2p-lookup direct --address /ip4/164.92.118.108/tcp/4001
[2022-04-21T22:37:30Z ERROR libp2p_lookup] Lookup failed: Timeout.
wink@3900x 22-04-21T22:37:30.136Z:~/prgs/downloads/virtualbox
```

The first stopped at 2022-04-21T22:35:47Z and I have observed that when
I see a `Lookup failed: Timeout` the timeout takes 20secs. So if you look
at the logs, `relay_v2.Debug-relay_v2-304f6a6c.nightly-1.62.0-879aff385.debug.4cpus.7.trace.log`,
we actually see at 22:35:27.437 `relay_v2` starting `running` in polling::epoll:
```
[2022-04-21T22:34:42.294524Z TRACE libp2p_swarm] Swarm<TBehavior>::Stream::poll_next:- result.is_ready(): false
[2022-04-21T22:35:27.437652Z TRACE polling::epoll] wait: running epoll_fd=4, events.len=1 events.list:
[2022-04-21T22:35:27.437755Z TRACE polling::epoll] wait: list[0] libc::epoll_event { events: 1 u64: 0 } 
[2022-04-21T22:35:27.437778Z TRACE polling::epoll] modify:+ tid=7 epoll_fd=4, fd=5, ev=Event { key: 18446744073709551615, readable: true, writable: false }
[2022-04-21T22:35:27.437802Z TRACE polling::epoll] ctl:+ tid=7 epoll_fd=4, event_fd=5 libc::epoll_event { events=4000201b u64=18446744073709551615 }
[2022-04-21T22:35:27.437821Z TRACE polling::epoll] ctl:- epoll_fd=4, event_fd=5, ev=Some(Event { key: 18446744073709551615, readable: true, writable: false }) result=Ok(0)
[2022-04-21T22:35:27.437846Z TRACE polling::epoll] modify:- epoll_fd=4, fd=5, ev=Event { key: 18446744073709551615, readable: true, writable: false }
[2022-04-21T22:35:27.437865Z TRACE polling::epoll] wait:- epoll_fd=4, res=1
```

So my intrepation of the above is that.
 * relay_v2 was awoke in epoll because the EPOLLIN (events: 1) event fired and key 0 (u64: 0).
 * epoll then calls modify (which called ctl) with key: 18446744073709551615 (0xFFFFFFFFFFFFFFFF)
   for all of the (4000201b) readablity-events[^1] [readability events](#readability-events)

Test writability-events[^2] link



[^1]: <a id="readability-events">readability events</a> 4000201b libc::EPOLLIN | libc::EPOLLRDHUP | libc::EPOLLHUP | libc::EPOLLERR | libc::EPOLLPRI

[^2]: writabilty-events 0000001c  libc::EPOLLOUT | libc::EPOLLHUP | libc::EPOLLERR

From linux include/uapi/linux/eventpoll.h:
```
#define EPOLLIN		(__force __poll_t)0x00000001
#define EPOLLPRI	(__force __poll_t)0x00000002
#define EPOLLOUT	(__force __poll_t)0x00000004
#define EPOLLERR	(__force __poll_t)0x00000008
#define EPOLLHUP	(__force __poll_t)0x00000010
#define EPOLLNVAL	(__force __poll_t)0x00000020
#define EPOLLRDNORM	(__force __poll_t)0x00000040
#define EPOLLRDBAND	(__force __poll_t)0x00000080
#define EPOLLWRNORM	(__force __poll_t)0x00000100
#define EPOLLWRBAND	(__force __poll_t)0x00000200
#define EPOLLMSG	(__force __poll_t)0x00000400
#define EPOLLRDHUP	(__force __poll_t)0x00002000
```