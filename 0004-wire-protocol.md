


    ┌─────┐                ┌─────┐
    │Alice│                │ Bob │
    └──┬──┘                └──┬──┘
       │                      │
       │ CONNECT              │
       ├─────────────────────►│
       │ hello                │
       ├╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴►│
       │                      │
       │                hello │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │            inventory │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │                      │
       │ inventory            │
       ├╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴►│
       │                      │
       │                      │
       ┴                      ┴


    ┌─────┐                ┌─────┐                ┌─────┐
    │Alice│                │ Bob │                │ Eve │
    └──┬──┘                └──┬──┘                └──┬──┘
       │                      │                      │
       │                      │           inventory¹ │
       │                     ✕│◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │                      │                      │
       │ subscribe            │                      │
       ├╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴►│                      │
       │           inventory¹ │                      │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤                      │
       │                      │           inventory² │
       │                      │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │           inventory² │                      │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤                      │
       │                      │                      │
       ┴                      ┴                      ┴

    ┌─────┐                ┌─────┐                ┌─────┐
    │Alice│                │ Bob │                │ Eve │
    └──┬──┘                └──┬──┘                └──┬──┘
       │                      │                      │
       │                      │             refs-ann │
       │                      │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │                      │ FETCH                │
       │                      ├─────────────────────►│
       │                      │                      │
       │             refs-ann │                      │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤                      │
       │ FETCH                │                      │
       ├─────────────────────►│                      │
       │                      │                      │
       │                      │                      │
       ┴                      ┴                      ┴


    ┌─────┐                ┌─────┐                ┌─────┐
    │Alice│                │ Bob │                │ Eve │
    └──┬──┘                └──┬──┘                └──┬──┘
       │                      │                      │
       │                      │             node-ann │
       │                      │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤
       │                      │                      │
       │             node-ann │                      │
       │◄╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴┤                      │
       │ CONNECT              │                      │
       ├──────────────────────┼─────────────────────►│
       │                      │                      │
       │                      │                      │
       ┴                      ┴                      ┴


https://freecontent.manning.com/all-about-bloom-filters/
https://www.eecs.harvard.edu/~michaelm/postscripts/tempim3.pdf