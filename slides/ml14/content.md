<!-- .slide: class="title" -->

# **Metaprogramming with ML modules** in the MirageOS

Anil Madhavapeddy <small>University of Cambridge</small>
[@avsm](http://twitter.com/avsm)

Thomas Gazagnaire <small>University of Cambridge</small>
[@eriangazag](http://twitter.com/eriangazag)

David Scott <small>Citrix</small>
[@mugofsoup](http://twitter.com/mugofsoup)

Richard Mortier <small>University of Nottingham</small>
[@mort\_\_\_](http://twitter.com/mort___)

[http://openmirage.org/](http://openmirage.org/)<br/>
[http://decks.openmirage.org/ml14/](http://decks.openmirage.org/ml14/#/)

<small>
  Press &lt;esc&gt; to view the slide index, and the &lt;arrow&gt; keys to
  navigate.
</small>


----

## Four Years Ago...

Thomas Gazagnaire and I [talked](http://www.slideshare.net/xen_com_mgr/mirage-extreme-specialization-of-cloud-appliances) at the ML Workshop 2010 about our new OCaml Operating
System.

Since then we have:

- rebuilt the core 3 times
- not given up
- written a [package manager](https://opam.ocaml.org)
- attracted real users
- **functorized the entire stack**


## Project Goal

**Portable code that can be deployed into exotic environments
ranging from standalone Xen unikernels to JavaScript blobs.**

This presentation can be compiled as a:

* Xen unikernel with all data built into image.
* Xen unikernel with data dynamically read from disk.
* Unix binary with data passed through to filesystem.
* Unix binary with OCaml userlevel TCP/IP stack.

> How do we express all these combinations succinctly without OS dependencies creeping
> throughout our codebase?


## Introducing [Mirage OS 2.0](http://openmirage.org/)

These slides were written using Mirage 2.0 (released July 2014) on Mac OSX:

- They are hosted in a **938kB Xen unikernel** written in statically type-safe
  OCaml, including device drivers and network stack.

- Their application logic is just a **couple of source files**, written
  independently of any OS dependencies.

- Running on an **ARM** CubieBoard2, and hosted on the cloud.

- Binaries small enough to track the **entire deployment** in Git!


----

## Modular Architecture
 
From an ML perspective, MirageOS is: 
 
1. A collection of __module types__, describing structural parts of an 
   operating system (including device drivers).
 
2. A collection of independent __libraries that implement the 
   module types.__  Only libraries that application needs
   are linked.
 
3. Use __functors to model dependencies__ between libraries/components. 
   The functor arguments are the module types defined in 1. 


## Key Insight

> Mirage provides an EDSL that __embeds the module and functor
> language into the host OCaml language__, where they can be
> manipulated and applied more easily.

<br />

> Metaprogramming is used to evaluate the result into a set
> of fully applied modules that are executable under Unix or Xen.


## Mirage OS 2.0 Workflow

As easy as 1&mdash;2&mdash;3!

1. Write your OCaml application using the Mirage module types.
   + Express its configuration as OCaml code too!

           $ mirage configure app/config.ml --unix


## Mirage OS 2.0 Workflow

As easy as 1&mdash;2&mdash;3!

1. Write your OCaml application using the Mirage module types.
   + Express its configuration as OCaml code too!

2. Compile it and debug under Unix using the `mirage` tool.

         $ cd app
         $ make depend # install library dependencies
         $ make build  # build the unikernel


## Mirage OS 2.0 Workflow

As easy as 1&mdash;2&mdash;3!

1. Write your OCaml application using the Mirage module types.
   + Express its configuration as OCaml code too!

2. Compile it and debug under Unix using the `mirage` tool.

3. Once debugged, simply retarget it to Xen, and rebuild!

          $ mirage configure app/config.ml --xen
          $ cd app && make depend && make build

   + All the magic happens via the OCaml module system.


## Modular Architecture
 
From an ML perspective, MirageOS is: 
 
1. A collection of __module types__, describing structural parts of an 
   operating system (including device drivers).
 
2. A collection of independent __libraries that implement the 
   module types.__  Only libraries that application needs
   are linked.
 
3. Use __functors to model dependencies__ between libraries/components. 
   The functor arguments are the module types defined in 1. 


## Module Types: Devices

```
module type DEVICE = sig

  type +'a io
  type t
  type id

  val id : t -> id

  val connect: id -> [ `Error of exn | `Ok of t ] io

  val disconnect : t -> unit io

end
```

Generic interface to any device driver...


## Module Types: Flows

```
module type FLOW = sig

  type +'a io
  type buffer
  type flow

  val read : flow -> [`Ok of buffer | `Eof ] io

  val write : flow -> buffer -> [`Ok of unit | `Eof ] io

  val writev : flow -> buffer list -> [`Ok of unit | `Eof ] io 

end

```

...or any I/O flows in the system.


## Module Types: Inclusion

```
module type TCPV4 = sig

  type buffer
  type ipv4
  type ipv4addr
  type flow

  include DEVICE with
  with type id := ipv4

  include FLOW with
  with type 'a io  := 'a io
  and  type buffer := buffer
  and  type flow   := flow
```

...and they can be composed together into other module types, avoiding
the diamond problem.


## Module Types: Entropy

```
module type ENTROPY = sig

  include DEVICE
  type buffer

  type handler = source:int -> buffer -> unit
  (** A [handler] is called whenever the system has extra entropy to announce.
   * No guarantees are made about the entropy itself, other than it being
   * environmentally derived. In particular, the amount of entropy in the buffer
   * can be far lower than the size of the [buffer].
   *
   * [source] is a small integer, describing the provider but with no other
   * meaning.
   **)

  val handler : t -> handler -> unit io
end
```

Complex driver models can be expressed abstractly
(see [V1.ml](https://github.com/mirage/mirage/tree/master/types/)).


## Module Types: Refinement

```
module type FLOW = FLOW
  with type 'a io = 'a Lwt.t
   and type buffer = Cstruct.t

module type NETWORK = NETWORK
  with type 'a io = 'a Lwt.t
   and type page_aligned_buffer = Io_page.t
   and type buffer = Cstruct.t
   and type macaddr = Macaddr.t

module type ETHIF = ETHIF
  with type 'a io = 'a Lwt.t
   and type buffer = Cstruct.t
   and type macaddr = Macaddr.t
   and type ipv4addr = Ipaddr.V4.t
```

The abstract types can be specialised into concrete
library types for common uses
(see [V1_LWT.ml](https://github.com/mirage/mirage/tree/master/types/)).


----

## From Modules to Functors
 
Functors are used everywhere in Mirage to describe OS layers: 
 
- Similar to FoxNet, but for the __whole OS__. We have a full 
  implementation of the network stack (including TLS) in OCaml.
 
- Very __flexible approach__ for customising OS stacks for weird applications
  (HTTP over UPnP over UDP...)

- Lots of __separate implementations__ of the module signatures:
  Unix, Xen microkernels, Javascript, kernel modules, ... 


## Writing a component

A Mirage component usually contains:

- __code parameterised by functors__: very limited dependencies, usually only on
  the Mirage module types.


## Example: Website

```
module Main (C:CONSOLE) (FS:KV_RO) (H:HTTP.Server) = struct

  let start c fs http =
    ...

    let callback conn_id request body =
      C.log "HTTP request received" ...
      >>= fun () ->
      let uri = H.Request.uri request in
      dispatcher (split_path uri)
    in
    let conn_closed (_,conn_id) () = ...  in
    http { S.callback; conn_closed }

end
```


## Modularizing the OS

<p class="stretch center">
  <img src="modules1.png" />
</p>


## Modularizing the OS

<p class="stretch center">
  <img src="modules2.png" />
</p>


## Modularizing the OS

<p class="stretch center">
  <img src="modules3.png" />
</p>


## Writing a component

A Mirage component usually contains:

- code parameterised by functors: very limited dependencies, usually only on
  the Mirage module types.  __No OS dependencies__.

- a collection of libraries where the functors are (fully or partially) applied,
  suitable for interactive use.

> Functors clearly separate the dependencies between OS components
> and break down the monolithic into components.


## The Bad News

Functors are rather heavyweight constructs, and need to be applied in some concrete
combination to make an executable.

The module language is much more limited than the core host language, so we embed
it inside the host language as an eDSL!

> __Metaprogramming__:  manipulate functors as values in the
  host language, and emit resulting module applications as a program stage.


----

## Mirage eDSL: module types

To simplify building applications, we use an eDSL.

```
type 'a typ
(** The type of values representing module types. *)

val (@->): 'a typ -> 'b typ -> ('a -> 'b) typ
(** Construct a functor type from a type and an existing functor
    type. This corresponds to prepending a parameter to the list of
    functor parameters. For example,

    {| kv_ro @-> ip @-> kv_ro |}

    describes a functor type that accepts two arguments -- a kv_ro and
    an ip device -- and returns a kv_ro.
*)
```

This describes all the __module types__ (`NETWORK`, `TCPV4`, etc...)


## Mirage eDSL: module types

The eDSL also describes concrete module implementations for a given
signature (e.g. a socket TCPv4 stack vs direct OCaml one).

```
type 'a impl
(** The type of values representing module implementations. *)

val ($): ('a -> 'b) impl -> 'a impl -> 'b impl
(** [m $ a] applies the functor [a] to the functor [m]. *)

val foreign: string -> 'a typ -> 'a impl
(** [foreign name constr typ] states that the module named
    by [name] has the module type [typ]. *)

val typ: 'a impl -> 'a typ
(** Return the module signature of a given implementation. *)
```


## Example: Website

```
let net =
  try match Sys.getenv "NET" with
    | "direct" -> `Direct
    | "socket" -> `Socket
    | _        -> `Direct
  with Not_found -> `Direct

let dhcp = ...

let stack console =
  match net, dhcp with
  | `Direct, true  -> direct_stackv4_with_dhcp console tap0
  | `Direct, false -> direct_stackv4_with_default_ipv4 console tap0
  | `Socket, _     -> socket_stackv4 console [Ipaddr.V4.any]
```

When given a console, this builds a network stack:

```
val stack : console impl -> stackv4 impl
```


## Example: Website

```
let server =
  let conduit = conduit_direct (stack default_console) in
  http_server (`TCP (`Port 80)) conduit

let main = foreign "Dispatch.Main"
  (console @-> kv_ro @-> http @-> job)

let () =
  register "www" [ main $ default_console $ fs $ server ]
```


## Correspondence

Configuration Code:

```
let server =
  let conduit = conduit_direct (stack default_console) in
  http_server (`TCP (`Port 80)) conduit

let main = foreign "Dispatch.Main"
  (console @-> kv_ro @-> http @-> job)
```

Application Code:

```
module Main (C:CONSOLE) (FS:KV_RO) (S:Cohttp_lwt.Server) = struct

  let start c fs http = ...
```


## Correspondence: Unix

```
module Stackv41 = Tcpip_stack_socket.Make(Console)
module Conduit1 = Conduit_mirage.Make(Stackv41)
module Http1 = HTTP.Make(Conduit1)
module M1 = Dispatch.Main(Console)(Static1)(Http1.Server)
```

Fairly simple application of a kernel socket C binding to a network stack, which is passed to the application.


## Correspondence: Xen

A more complex module assembly for Xen...

```
module Stackv41 = struct
  module E = Ethif.Make(Netif)
  module I = Ipv4.Make(E)
  module U = Udpv4.Make(I)
  module T = Tcpv4.Flow.Make(I)(OS.Time)(Clock)(Random)
  module S = Tcpip_stack_direct.Make(Console)(OS.Time)(Random)(Netif)(E)(I)(U)(T)
  include S
end
module Conduit1 = Conduit_mirage.Make(Stackv41)
module Http1 = HTTP.Make(Conduit1)
module M1 = Dispatch.Main(Console)(Static1)(Http1.Server)

...

let () =
  OS.Main.run (join [t1 ()])
```


## Flexibility

Website can now be assembled via host code:

* Xen unikernel with all data built into image.
* Xen unikernel with data dynamically read from disk.
* Unix binary with data passed through to filesystem.
* Unix binary with OCaml userlevel TCP/IP stack

<br />
> Well-typed rope to hang yourself with, in the grand Unix tradition!


## Module vs value language

- Error messages are significantly simpler in the host language (small type mismatches *vs* dumps of entire module signatures!)

- Optional arguments:

```
val direct_tcpv4:
  ?clock:clock impl ->
  ?random:random impl ->
  ?time:time impl ->
  ipv4 impl -> tcpv4 impl
```

Can select default module implementations for CLOCK, RANDOM and TIME to save manual work.


## Extensibility

Anyone can extend the eDSL with new devices:

- define an abstract type corresponding to the module type signature (e.g. NETWORK or TCPV4 or KV_RO). 
- define a CONFIGURABLE implementation and pack it using first class modules to generate a new impl type.

```
val implementation: 'a -> 'b -> (module CONFIGURABLE with type t = 'b) -> 'a impl
```


## Just like Dynamics

Dynamics is a pair between:

* a *value*
* a value that represents its *type*.

In the Mirage EDSL, an *impl* represents a triple of:

* a *typ*
* a CONFIGURABLE and the constructor function for the module.

GADTs ensure that an *impl* cannot be applied to a *typ* that does not admit it, so the resulting functor applications are sound.


## eDSL: Summary

* Application is a **parameterised module** over dependencies.
  - ML functors separate OS functionality into modular chunks.
  - Mirage provides a library of module types.
* **Metaprogramming** is used to manipulate these modules.
  - Mirage eDSL manipulates module types and implementations in OCaml.
  - Config file is interpreted to generate executable kernel.
  - Flexible way to specify precise device driver policy.


----

## <http://openmirage.org/>

A Linux Foundation Incubator Project lead from the University of Cambridge and Citrix Systems.

Featuring blog posts on new features by:

[Amir Chaudhry](http://amirchaudhry.com/),
[Thomas Gazagnaire](http://gazagnaire.org/),
[David Kaloper](https://github.com/pqwy),
[Thomas Leonard](http://roscidus.com/blog/),
[Jon Ludlam](http://twitter.com/jonludlam),
[Hannes Mehnert](https://github.com/hannesm),
[Mindy Preston](https://github.com/yomimono),
[Dave Scott](http://dave.recoil.org/),
and [Jeremy Yallop](https://github.com/yallop).

<p style="font-size: 48px; font-weight: bold;
          display: float; padding: 2ex 0; text-align: center">
  Thanks for listening! Questions?
  <br />
  Contributions very welcome at [openmirage.org](http://openmirage.org)
</p>
