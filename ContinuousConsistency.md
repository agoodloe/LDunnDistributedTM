---
title: Continuous consistency in the coordination of airborne and ground-based agents.
author: Lawrence Dunn
abstract: |
  The System Wide Safety (SWS) program has been investigating how
  crewed and uncrewed aircraft can safely operate in shared airspace,
  taking disaster response scenarios as a motivating use
  case. Enforcing safety requirements for distributed agents requires
  coordination by passing messages over a communication network.
  However, the operating environment will not admit reliable
  high-bandwidth communication between all agents, introducing
  theoretical and practical obstructions to global consistency that make it more difficult
  to maintain safety-related invariants. This self-contained memo discusses
  some of the distributed systems challenges involved in system-wide
  safety, focusing on the practical shortcomings of both strong and weak consistency models for shared memory.
  Then we survey two *continuous* consistency models that come from different parts of the literature.
  Unlike weak consistency models, continuous consistency models
  provides hard upper bounds on the "amount'' of inconsistency observable
  by clients.  Unlike strong consistency, these models are flexible enough to accomodate real-world
  conditions, such as by providing liveness during brief network partitions or tolerating
  disagreements between sensors in a sensor network.
  We conclude that continuous consistency
  models are appropriate for analyzing safety-critical systems that operate
  without strong guarantees about network performance.
---

# Introduction

Civil aviation has traditionally focused primarily on the efficient
and safe transportation of people and goods via the airspace to a
regular destination, typically an airport. Despite the inherent risks,
the application of sound engineering practices and conservative
operating procedures has made flying the safest mode of transport
today. Now the desire not to compromise this safety makes it difficult
to integrate unmanned vehicles into the airspace, accomodate new
applications like package delivery, and keep pace with the rapid
growth in commercial aviation. To that end, the NASA Aeronautics'
Airspace Operations and Safety Program (AOSP) System Wide Safety (SWS)
project has been investigating new technologies and methods by which
crewed and uncrewed aircraft may safely operate in shared airspace.

This memo surveys some of the distributed computing challenges
generally encountered in this setting and methods that may prove
useful in overcoming them. Our primary motivating use cases have been
taken from civil emergency response scenarios, especially wildfire
suppression and hurricane relief. The motivation for this choice is
two-fold. First, the rules for operating in the US national airspace
are typically relaxed during natural disasters and relief
efforts. Second, these settings are an excellent microcosm for the
sorts of general challenges faced by other, non-emergency
applications.

Operations in disaster response scenarios are often encumbered by a
challenging communications environment, the causes of which are
several in number: remote locations, difficult terrain, damaged
infrastructure, harsh weather, and limited battery power, to name a
few. From a networking perspective, these factors lead to heavy packet
loss and significant delays in message-passing. From a systems
perspective, an unreliable network obstructs various kinds of
protocols for coordination among distributed agents. Typically,
coordination requires enforcing some notion of consistency across
replicas of data maintained by multiple agents cooperatively, and
stronger notions of consistency generally require a greater ability to
propagate updates to other agents quickly if system availability is
not to be sacrificed.  Finally, from a civil agency perspective, an
inability to coordinate the actions of distributed agents makes it
difficult to enforce safety conditions, as safe operations typically
require agents to act with reasonably up-to-date information about
other agents in the system.

To give an example of these tradeoffs, consider firefighting
airtankers, the largest examples of which (Very Large Airtankers, or
VLATs) can deposit more than 10,000 gallons of fire retardant at once
with enough force to crush a car. [^carcrush] This manoeauver is known
to pose a danger to ground crews,[^killed] so one potential policy
would be to disallow this action if a pilot does not have up-to-date
information about the location of agents on the ground. However,
sharing this information with the pilot may not be possible if heavy
smoke or a tall mountain ridge prevents radio communications. This
scenario is just one example of an inherent tradeoff between safety
and system availability. To achieve safety, the pilot should only
operate with reliable information. However, if the policy is enforced,
the pilot's operations could be delayed until this information can be
obtained, leaving the aircraft unavailable for useful work in the
meantime. We emphasize that it is not enough that there *aren't any*
ground crews in the way---an agent's actions are restricted whenever
the agent does not *know* that an action is safe, even if it is in
fact safe.

[^carcrush]: [Link](https://www.youtube.com/watch?v=ONdSoiI4zIA)

[^killed]: [Link](https://www.firerescue1.com/flame-retardants/articles/utah-battalion-chiefs-death-may-have-been-linked-to-airplane-retardant-drop-zWKw179u1IXBB8p9/)


[^note]: In the systems literature, a "safe" system avoids bad behaviors that violate some condition, such as performing a dangerous action without knowing the location of other agents. However, delaying operations may itself be a "safety" problem in the everyday sense of the word.

This sort of tradeoff is manifest throughout the design and
implementation of distributed systems, and the tension between the two
ideals becomes especially stark as the reliability of the network
deteriorates---a fact manifest in Brewer's CAP Theorem
(CITE). Designing systems that are resilient to these sorts of
environments is therefore a fundamental challenge for distributed
computing. This purpose of this memorandum is to enumerate some of the
considerations involved in coordinating air- and ground-based elements
from a distributed computing perspective, identifying challenges,
potential requirements, and frameworks that suggest possible
solutions.

## Disaster response networking: present and future

- Talk about some technologies (radio) and difficulties
- Give example of firefighter's heart rate monitor thing
- In the future, talk about sensor data from firefighters and agencies
- Talk about routing information above a drone
- Plan to aggressively attack fires from air and possibly automate this

As background, we discuss some of the technical details of disaster
response communication today and some possible developments for the
future. The reader may be surprised to learn the state of the art in
wildland firefighting communication is often simple, even primitive, a
fact explained partly by contrasting the relative budget of a
volunteer fire department with that of a comparably sized military
unit. Indeed, the need to support commercial off-the-shelf (COTS)
components is one of our general assumptions in this memo.

Communication between firefighters in the field is often facilitated
by handheld radios, which are inherently limited in their battery
life, bandwidth, effective range, and ability to work around
environmental factors like foliage and smoke. As we watched interviews
with experienced wildland firefighters, we found an interview with a
volunteer firefighter who relayed a story of Ironside repeater station
destroyed by fire. (CITE, CITE). This repeater had strategic
importance, between located on a tall ridge, and its loss prevented
communication between operators on different sides of it. This
communication partition continued until agents could ascend the ridge
to deploy a temporary station, a challenge that presumably diverted
operators from other duties. This story demonstrates the potential for
fairly widespread system failure due to the loss of a single system
component when it is shared by multiple users.

Some firefighters have described using the low-technology solution of
simply shouting messages to each other, as this approach can be
cheaper, quicker, and more lightweight than radio-based methods. While
simple, this actually illustrates an exploitable kind of spatial
locality principle that generalizes to other parts of the system: in
general, agents with a higher need to coordinate their actions will
tend to be located closer to each other, which in turn will tend to
increase the likelihood they will be able to communicate. This kind of
principle is a fundamental motivation for the sort of decentralized,
ad-hoc networking protocols considered in Chapter CITE.

Turning now towards the sky, civil aviation has also traditionally
employed simple communication patterns between airborne agents. For
instance, aircraft equipped with Automatic Dependent
Surveillance-Broadcast (ADS-B) monitor their location using GPS and
periodically broadcast this information to air traffic controllers and
nearby aircraft. This sort of scheme has worked well in traditional
applications, where pilots typically only monitor the general
locations of a few nearby aircraft. It is another example of
exploiting spatial locality: aircraft have the highest need to
coordinate when they are physically close, and therefore in range of
each other's ADS-B broadcasts.

As aircraft generally have better line-of-site to ground crews than
ground crews have to each other, firefighters sometimes relaying
messages to air-based units over radio, which in turn is relayed back
down to other ground units. The locality principle comes into play
here, too, this time in the unfortunate reverse direction: this simple
relay scheme allows messages to travel farther, but the extended
reach comes at the cost of introducing delays and possible degradation
of message quality, as in the classic telephone game.

### Disaster response in the future

Strategic decisions are increasingly relying on high-quality data.

At the individual level, firefighters may be equipped with local
sensors---for instance, NIST has found that fatal cardiac events in
firefighters can be accurately predicted,[^cardiac] and future efforts in this
direction may lead to body-worn sensors. Given the ubiquity of
inexpensive GPS devices---this could be as simple as an application
running on a personal smartphone.

[^cardiac]: https://www.nist.gov/news-events/news/2023/07/ai-can-accurately-predict-potentially-fatal-cardiac-events-firefighters

Sensors may be deployed to monitor everything from air quality to
temperature.

At the strategic level, decision makers may rely on computer-generated
forecasts of firefront[^firefront] behavior that take into account
weather, topography, fuel conditions, and observed fire behavior. Such
predictions may be

[^firefront]: The leading edge where a wildfire is actively burning.

In our setting, a large number of aircraft (perhaps as many as 10) may
need to operate in a small area, near complex terrain, and at times
operating at an altitude of less than 1000 ft.---in other words the
demands are many and the margins for error are small. This sort of use
case demands more sophisticated coordination schemes between airborne
and ground-based elements than ADS-B and comparable schemes provide by
themselves.

We envision a system that responds gracefully to a challenging
environment, flexibly coordinates a dynamic ensemble of heterogeneous
mobile agents, and intelligently gathers, fuses, and disseminates the
right information to decision-makers at the right time. When parts of
the system fail---perhaps because some agents are too far away to
contact, or become some sensors are generating conflicting data, or
because a repeater is destroyed by fire---the system should fail no
more than it must.

The system should exploit every opportunity to facilitate
communication between agents. For instance, an overhead drone
surveying a fire may opportunistically act as a mobile base station to
ground crews, allowing them to route messages past a mountain
ridge---a high-tech modernization of the sort of informal relay scheme
operating today over traditional radio channels.


## Layout of this document

This document aims to be reasonably self-contained and readable to a
broad technical audience. We proceed from lower-level details (memory
models and network protocols) to higher-level ones (database
replication and data fusion). A common albeit abstract theme is to
emphasize notions of *continuity*---a preference to make flexible and
balanced tradeoffs rather than committing to extremes, in order to
build systems that offer a favorable balance of desirable properties
while tolerating a range of unfavorable real-world behavior. The
sections are as follows.

is a breif introduction to
distributed systems and memory consistency models. We define two
strong models, atomic and sequential consistency, both of which
provide highly desirable safety guarantees; we contrast these with the
weaker guarantees implied by the causal consistency model. Then we
turn our attention to the CAP theorem [@2000brewerCAP]
[@2002gilbertlynchCAP], which captures a fundamental
consistency/availability tradeoff in the presense of network
partitions. We observe that the theorem effectively prohibits both
forms of strong consistency in our intended use case. This raises the
question of how, if at all, one can rigorously enforce safety
properties without compromising system performance beyond acceptable
levels. This section concludes with a list of three desiderata of
distributed applications in the contexts under consideration.


Section \ref{sec:background} presents general background information
on distributed systems, strict consistency models, and proves Brewer's
fundamental "CAP" theorem (Theorem CITE) that a distributed system
cannot enforce strict consistency without going offline during network
partitions, i.e. without making network performance an upper bound on
overall system performance. Read pessimistically, a mild
generalization (Theorem CITE) proves a realistic distributed system
for emergency response network cannot provide global sequential
consistency, a fact that will later motivate Section CITE.

Section \ref{sec:networks} examines networking considerations. Our
vision of future emergency networks integrates
delay/disruption-tolerant and mobile ad-hoc networking concepts to
provide communications that are robust to their harsh operating
environment. We also feel that software-defined networking (SDN)
offers a plausible path towards deploying sophisticated networking
schemes without requiring all involved parties to purchase
special-purpose, one-off equipment, as SDN-based equipment can be
reprogrammed without replacing the hardware.

Chapter whatever describes a foundational application run over our
hypothetical network: a data-replication service that is attuned for
the particulars of a chaotic network, built on Yu and Vahdat's theory
of *conits*. Unlike some shared memory frameworks, this *continuous
consistency* framework provides neither idealized consistency
(linearizability) nor guaranteed high-availability, but rather a
dynamically-tuned "amount" of both, allowing the system to weigh the
differing importance of various kinds of updates and adjust its
behavior in response to real network conditions. This idea rests on
the observation that many high-level applications can tolerate some
inconsistency among replicas, particularly if one can enforce a hard
limit on how far apart replicas may diverge during periods of network
unavailability.

Finally, nformation may come from sensors in the environment, personal
area networks (PANs) associated with individual first responders
(e.g. a Bluetooth-enabled heartrate monitor). One must avoid "swimming
in data and drowning in noise." Data integration is the subject of
Chapter whatever. This section contains a brief introduction to
applied sheaf theory, which provides a highly general framework for
measuring the mutual consistency of "overlapping"
observations. Intuitively, these are observations which we expect to
be correlated if not equal, such as the data generated by nearby
sensors in a sensor network.

Section \ref{sec:conclusion} concludes.

\newpage
# Distributed systems
\label{sec:background}

A distributed system, broadly construed, is a collection of
independent entities that cooperate to solve a problem that cannot be
individually solved [@kshemkalyani_singhal_2008]. In the context
of computing, [@10.5555/562065] offer the following definition.

> "A collection of computers that do not share common memory or a
> common physical clock, that communicate by message passing over a
> communication network, and where each computer has its own memory
> and runs its own operating system."

A fundamental goal for distributed computing systems is to "[appear]
to the users of the system as a single coherent computer"
[@TanenbaumSteen07]. This can be understood as the requirement that
all nodes present a *mutually-consistent* view of the world, e.g. the
state of a globally-maintained database, to system clients.

In our scenarios, system nodes typically represent various computers,
routers, sensors, and communication devices, while the clients would
typically be firefighters using these devices and other persons
involved in disaster response efforts. The components of the system
collectively accomplish goals such as navigating safely in close
proximity, delivering resources to remote locations, and suppressing
fires. For example, first responders may communicate over a text
messaging application, and global consistency would mean different
users reading their device's local copy of the chat should see the
same messages.

What precisely constitutes consistency? One can choose from multiple
consistency models, and the most appropriate model depends on the
semantics expected by the application and its clients, which must be
weighed against other requirements. All other things being equal, one
wants to have as much consistency as possible. Below, we shall see
that \textrm{na\"ive} notions of system coherence are brittle in the
sense that they generally cannot be guaranteed.

When *strong* notions of consistency are enforced, clients are
presented with the abstraction of a single shared world, i.e. as if
they are all connected to a central computer rather than a complex
system of independent computers. This abstraction shields clients and
application developers from complexity and makes it simpler to reason
about a system's behavior. In our previous example, strong consistency
would require that different users see the same messages arriving at
approximately the same time and always in the same order. Consistency
is a prerequisite to system-wide safety, because bad things can happen
when strong consistency is violated:

- Suppose in a message stream two questions $Q_1$ and $Q_2$ are asked
  and responses $A_1$ and $A_2$ are given, respectively. If the
  answers arrive in the wrong order, i.e. $A_2$ appears as the answer
  to $Q_1$ and vice versa, the potential for confusion can be severe.

- A bank client would be unhappy if deposits that appear in their
  account online are not reflected when they check their balance at an
  ATM, or if they seem to disappear after refreshing the webpage.

- If two air traffic controllers were presented with conflicting or
  information about the trajectory of aircraft, they could potentially
  issue dangerously incorrect instructions to the pilots.

- Resource-tracking systems are not useful if a resource that appears
  to be available cannot actually be used because the information is
  out of date. Alternatively, a resource that is actually available
  may not be used if clients think it is still unavailable.

In general, violating strong consistency means the abstraction of a
single shared universe is broken. In extreme cases, this can
invalidate clients' mental model of the system, make the system's
behavior harder to predict, or cause safety requirements to be
violated. Given this, it seems wise to always build applications that
provide strong consistency. Unfortuntately, there are a variety of
conditions where this condition cannot pragmatically be attained for
theoretical and practical reasons; that is, unless one is willing to
pay with significant performance penalties, including applications
that fail to respond to users under some conditions.

Strong consistency cannot be maintained whenever doing so requires a
level of communication throughput in excess of what the real-world
network can provide. Note that distributed agents can *only*
coordinate by passing messages over the network.[^fn] When messages
cannot be passed quickly enough, agents subject to strong consistency
requirements may not be able to make progress. Therefore, real-world
applications must tolerate weaker notions of consistency. As weaker
consistency imposes fewer constraints on observable system behavior,
applications can be more difficult to reason about, safety-related
invariants more challenging to enforce.

[^fn]: This fact is implied by absence of a common memory, whereas
processes on the same machine have the option to share data by writing
it to a memory location both processes have access to.

A foundational assumption is that the network is almost always less
than perfectly reliable---this fact can be counted on during
emergencies. Imperfection means that message delivery is not
instantaneous may be unpredictable. The network may deliver a packet
zero times (i.e. it may silently delete the packet) or multiple times,
and furthermore different packets may arrive in any order.[^byzantine]
All of these behaviors represent obstructions to consistency and make
it challenging to enforce safety requirements. We shall now make this
discussion more precise.

[^byzantine]: In the case of *Byzantine faults* (CITE) the network may
even act like a malicious adversary, though we will not consider this
scenario in this document.

## System model

A distributed system consists of a set $\mathcal{P} = \{P_i\}_{i\in
I}$ of *processes*, which we think of these as executing on
independent, often geographically dispersed computers that communicate
by message-passing.

Process take requests from clients, such as to read or write a value
in a database. The lifecycle of a typical request is depicted in
Figure \ref{fig:request}. At some physical time (i.e. wall-clock time)
$C.s \in \mathbb{R}$ (client start time), a client sends a message to
a process. At time $E.s$, which we'll call the *start time* of the
event, the message is accepted by the process. The request is
processed until some strictly greater time $E.t > E.s$ when a response
is sent back to the client. The value $E.t - E.s$ is the
\emph{duration} of the event.

\begin{figure}
  \center
  \includegraphics[scale=0.4]{images/request.png}
  \caption{Lifetime of a client request}
  \label{fig:request}
\end{figure}

While handling the request, the process may coordinate with other
processes in the background by sending and receiving more
messages. For example, the process may propagate the client's request
to other processes, retrieve up-to-date values from other processes to
give to the client, or delay handling the client's request in order to
handle other requests.

To discuss consistency models, we shall be less interested in the
details of message-passing and more interested just in the responses
observed by clients. We shall consider the full set of events across a
distributed system, such as shown in Figure
\ref{fig:externalorder}. This is called an *execution*. Consistency
models constrain the set of allowable return values in response to
clients' requests.

As is often the case, we shall assume that requests handled by a
single process do not overlap in time. This can be enforced with local
serialization methods such as two-phase locking (CITE) that can be
used to isolate concurrent transactions from each other, providing the
abstraction of a system that handles requests one at a time. On the
other hand, any two processes may handle two events at the same
physical time, so that there is no obvious total order of events
across the system. Instead, one has a partial order called *external
order*. Intuitively, it is the partial order of events that would be
witnessed by an observer recording the real time at which systems
begin and finish responding to requests.

\begin{definition}
Let $E$ be an execution. Request $E1$ \emph{externally precedes}
request $E2$ if $E1.t < E2.s$. That is, if the first request
terminates before the second request is accepted. This induces an
irreflexive partial order called \emph{external order}.
\end{definition}

Recall that an irreflexive partial order is a binary relation $<$
such that $A \not < A$, $A < B \implies B \not < A$, and $A < B, B < C
\implies A < C$.

Because we assume processes handle events one-at-a-time, the events
handled at any one process are totally ordered by external order---one
event cannot start before another has finished. If $E1$ and $E2$ are
events at *different* processes, they need not be comparable by
external order, i.e. neither $E1.t < E2.s$ nor $E2.t < E1.s$, making
them *physically concurrent*.

\begin{definition}
If two events overlap in physical time
(equivalently, if they are not comparable by external order), we call the events \emph{physically concurrent} and
write $E1 || E2$.
\end{definition}

Physical concurrency is a reflexive and symmetric---but usually not
transitive--- binary relation. Such structures are often called
*compatibility relations.* The general intuition is that anything is
compatabile with itself (reflexivity), and the compatibility of two
objects does not depend on their order (symmetry). But if $A$ and $B$
are compatible with $C$, it need not be the case that $A$ and $B$ are
compatible with each other.

Figure \ref{fig:externalorderexec} shows the external order relation for
an execution. To save space we elide arrows between two events of the
same process and arrows that can be inferred by transitivity. This
corresponds to the directed acyclic graph structure shown in \ref{fig:externalorderdag}.

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/externalorder.png}
         \caption{Depiction of external order between concurrent events across three processes. Intra-process and transitive edges are not depicted.}
		 \label{fig:externalorderexec}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.25]{images/partialorder.png}
         \caption{The directed acyclic graph (DAG) induced by external order.}
		 \label{fig:externalorderdag}
     \end{subfigure}
     \caption{External order}
	 \label{fig:externalorder}
\end{figure}

The reader may wonder if we can consider events to be totally ordered,
say by pairing them with a timestamp that records their physical
start time to resolve ties like $x || y$. This order is not generally
useful for a couple reasons. First, we assume processes have only
loosely synchronized clocks, so timestamps from two different
processes may not be comparable. Additionally, even systems that
enforce linearizable consistency (c.f. Section \ref{sec:atomic}) do not
necessarily handle requests in order of their physical start times.

## Linearizability and sequential consistency
\label{sec:atomic}

A fundamental distributed application is the *shared distributed
memory* abstraction. We shall assume that all processes maintain a
local replica of a globally shared data object, as replication
increases system fault tolerance. For simplicitly, we shall discuss
the data store as a simple key-value store, but it could be something
else like a database, filesystem, persistent object, etc.

We assume clients submit two types of requests to processes. A *read
request* is a request to lookup the current value of a variable. A
request to read the variable $x$ that returns value $a$ is written
$R(x,a)$. A *write request* is a request to set the current value of a
variable. Notation $W(x,a)$ represents writing value $a$ to $x$. We
assume all processes provide access to the same set of shared
variables.

A *memory consistency model* formally constrains the allowable system
responses during executions. *Strong* consistency models are generally
understood as ones provide the illusion that all clients are accessing
just one globally shared replica. As we will see, this still leaves
room for different possible behaviors (i.e. allows non-determinism in
the execution of a distributed application), but the allowable
behavior is tightly constrained.

*Linearizability* [@10.1145/78969.78972] is essentially the strongest
common consistency model. It is known variously as atomic consistency,
strict consistency, and sometimes external consistency. In the context
of database transactions (which come with other guarantees, like
isolation, that are more specific to databases), the analogous
condition is called strict serializability. A linearizable execution
is defined by three features:

- All processes act like they agree on a single, global total order
  defined across all accesses.
- This sequential order is consistent with the actual external order.
- Responses are semantically correct, meaning a read request $R(x, a)$
  returns the value of the most recent write request $W(x, a)$ to $x$.

We can also phrase this in terms of *linearizations.*

\begin{definition}
A \emph{linearization point} $t \in \mathbb{R} \in [E.s, E.t]$ for an
event $E$ is a time between the event's start and termination. An
execution is \emph{linearizable} if and only if there is a choice of
linearization point for each access, which induces a total order called a \emph{linearization},
such that $E$ is equivalent to
the serial execution of events when totally ordered by their
linearization points.
\end{definition}

Intuitively, it should appear to an external observer that each access
instantaneously took effect at some point between its start and end
time. It is assumed no distinct access can have the same linearization
point, so that we get a total order. We say an entire system is
linearizable when all possible executions of the system are
linearizable.

Figure \ref{fig:linear_example11} shows a prototypical example of a
linearizable execution. We assume that all memory locations are
initialized to $0$ at the system start time.  Figure
\ref{fig:linear_example12} shows an execution that is not linearizable
because the read access on $y$ on $P1$ returns stale data instead of
reflecting the write access to $y$ on $P2$.

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
		 \center
		 \includegraphics[scale=0.4]{images/linear1.png}
		 \caption{A linearizable execution. Any choice of linearization works here.}
		 \label{fig:linear_example11}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonlinear0.png}
		 \caption{A non-linearizable execution. The request to read $y$ returns a stale value. }
		 \label{fig:linear_example12}
     \end{subfigure}
  \caption{A linearizable and non-linearizable execution.}
  \label{fig:linear_example1}
\end{figure}

Linearization points are demonstrated in Figure
\ref{fig:linearization}. The figure shows different linearizable
behaviors in response to the same underlying set of accesses. This
demonstrates that linearizability still leaves some room for
non-determinism in the execution of distributed applications. In this
example, the requests must both return 1 or 2. The constraint is that
the values must agree---linearizability forbids the situation in which
one client reads $1$ and another reads $2$ (Figure
\ref{fig:nonlinearizable}).

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linearTemplate.png}
         \caption{An execution with read responses left unspecified.}
         \label{fig:nonlinear}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear3.png}
         \caption{A linearizable execution for which both reads return $1$.}
     \end{subfigure}
     \begin{subfigure}[c]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear2.png}
         \caption{A linearizable execution for which both reads return $2$.}
     \end{subfigure}
  \caption{Two linearizable executions of the same underlying events that return different responses. Possible linearization points are shown in red.}
  \label{fig:linearization}
\end{figure}

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonlinear1.png}
		 \caption{A nonlinearizable execution with the read access returning disagreeing values. We will see later (Figure \ref{fig:sequential}) that this execution is still sequentially consistent. }
		 \label{fig:nonlinear1}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonlinear2.png}
		 \caption{Another nonlinearizable execution with read access values swapped. This execution is not sequentially consistent.}
		 \label{fig:nonlinear2}
     \end{subfigure}
  \caption{Two non-linearizable executions of the same events shown in Figure \ref{fig:linearization}.}
  \label{fig:nonlinearizable}
\end{figure}

#### Enforcing linearizability

Linearizability can be enforced with a total order broadcast mechanism
(CITE). Total order broadcast is a means for processes to come to a
consensus about the order of a set of events. One can imagine that the
total order broadcast API implements a routine that accepts a message
and notifies all other processes of this message in such a way that
all processes see all messages in the same order. To maintain
linearizability, it suffices that each replica applies database
actions in the order they are announced in the total order broadcast.
A subtle point is that a process does not need to handle read requests
originally sent to other clients, so these may be ignored. However,
the originating replica must not handle its own read requests until
*after* they appear in the total order broadcast, rather than at the
time they are first submitted to the total order broadcast
mechanism.


### Sequential consistency

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/sequential1.png}
         \caption{A non-linearizable, sequentially consistent execution.}
         \label{fig:sequential1}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/sequential2.png}
         \caption{An equivalent interleaving of \ref{fig:sequential1}.}
         \label{fig:interleaving1}
     \end{subfigure}
     \caption{A sequentially consistent execution and a possible interleaving.}
	 \label{fig:sequential}
\end{figure}

Enforcing atomic consistency means that an access $E$ at process $P_i$
cannot return to the client until every other process has been
informed about $E$. For many applications this is an unacceptably high
penalty. A weaker model that is still strong enough for most purposes
is *sequential* consistency. This is an appropriate model if a form of
strong consistency is required, but the system is agnostic about the
precise physical time at which events start and finish, provided they
occur in a globally agreed upon order.

A sequentially consistent system ensures that any execution is
equivalent to some global serial execution, even if this is serial
order is not the one suggested by the real-time ordering of
events. When real-time constraints are not important, this provides
essentially the same benefits as linearizability. For example, it
allows programmers to reason about concurrent executions of programs
because the result is always guaranteed to represent some possible
interleaving of instructions, never allowing instructions from one
program to execute out of order.

Processes in a sequentially consistent system are required to agree on
a total order of events, presenting the illusion of a shared database
from an application programmer's point of view. However, this order
need not be given by external order. Instead, the only requirement is
that sequential history must agree with process order, i.e. the events
from each process must occur in the same order as in they do in the
process.

\begin{definition}
\label{def:sequentiallyconsistent}
A \emph{sequentially consistent} execution is
characterized by three features:
\begin{itemize}
\item All processes act like they agree on a single, global total order
  defined across all accesses.
\item This sequential order is consistent with the program order of each process.
\item Responses are semantically correct, meaning reads return the most recent writes (as determined by the global order)
\end{itemize}
\end{definition}

This is nearly the definition of linearizability, except that external
order has been replaced with merely program order. We immediately get
the following lemma.

\begin{lemma}
\label{lem:linearsequential}
    A linearizable execution is sequentially consistent.
\end{lemma}
\begin{proof}
This follows because process order is a subset of external order.
\end{proof}

Visually, sequential consistency allows reordering an execution by
sliding events along each process' time axis like beads along a
string. Two events from the same process cannot pass over each other
as this would violate program order, but events on different processes
may be commuted past each other, violating external order. This
sliding allows defining an arbitrary interleaving of events, a totally
ordered execution with no events overlapping. From this perspective,
while linearizability requires the existence of a linearization,
sequential consistency requires the existence of an equivalent
interleaving.

The converse of Lemma \ref{lem:linearsequential} does not hold. For
example, Figure \ref{fig:sequential1} was previously shown (Figure
\ref{fig:nonlinear1}) as a nonlinearizable execution. However, it is
sequentially consistent, as evidenced by the interleaving in Figure
\ref{fig:interleaving1} that slides the events $W(x,1)$ and $R(x,2)$
past each other.

\begin{figure}
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonsequential1.png}
         \caption{A non-sequentially consistent execution.}
         \label{fig:nonsequential1}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonsequential_x.png}
		 \caption{The sequentially consistent history of $x$.}
		 \label{fig:sequentialx}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/nonsequential_y.png}
		 \caption{The sequentially consistent history of $y$.}
		 \label{fig:sequentialy}
     \end{subfigure}
     \caption{A non-sequentially consistent execution with sequentially-consistent executions at each variable.}
	 \label{fig:nonsequential}
\end{figure}

#### Enforcing sequential consistency

Notably, to enforce sequential consistency for the whole system, it is
not enough to enforce it at the level of individual variables. Figure
\ref{fig:nonsequential1} shows a history that is not sequentially
consistent. However, the histories of accesses to individual variables
(Figures \ref{fig:sequentialx} and \ref{fig:sequentialy}) *are*
sequentially consistent. This is a key difference from linearizability
(CITE).

Like linearizability, sequential consistency can also be enforced with
a total order broadcast mechanism. All write requests are first
broadcast, and replicas only apply updates in the order they appear in
the total order broadcast. The crucial difference from linearizability
is that each process can handle requests immediately, returning its
local replica value, instead of waiting for the broadcast mechanism to
assign a global order to the read request.


## The CAP Theorem

Real-world systems often fall short of behaving as a single perfectly
coherent system. The root of this phenomenon is a deep and
well-understood tradeoff between system coherence and
performance. Enforcing consistency comes at the cost of additional
communications, and communications impose overheads, often
unpredictable ones.

Fox and Brewer [@1999foxbrewer] are crediting with observing a
particular tension between the three competing goals of consistency,
availability, and partition-tolerance. This tradeoff was precisely
stated and proved in 2002 by Gilbert and Lynch
[@2002gilbertlynchCAP]. The theorem is often somewhat misunderstood,
as we discuss, so it is worth clarifying the terms used.

#### Consistency
Gilbert and Lynch define a consistency system as one whose executions
are always linearizable.

#### Availability
A CAP-available system is one that will definitely respond to every
client request at some point.

#### Partition tolerance
A partition-tolerant system continues to function, and ensure whatever
guarantees it is meant to provide, in the face of arbitrary partitions
in the network (i.e., an inability for some nodes to communicate with
others). It is possible that a partition never recovers, say if a
critical communications cable is permanently severed.

A partition-tolerant CAP-available system cannot indefinitely suspend
handling a request to wait for network activity like receiving a
message. In the event of a partition that never recovers, this would
mean the process could wait indefinitely for the partition to heal,
violating availability. On the other hand, a CAP-consistent system is
not allowed to return anything but the most up-to-date value in
response to client requests. Keep in mind that any (other) process may
be the originating replica for an update. Some reflection shows that
the full set of requirements is unattainable---a partition tolerant
system simply cannot enforce both consistency and availability.

### CAP theorem for linearizability

\begin{theorem}[The CAP Theorem]
	\label{thm:cap}
    In the presense of indefinite network partitions, a distributed system
    cannot guarantee both linearizability and eventual-availability.
\end{theorem}
\begin{proof}
Technically, the proof is almost trivial. We give only the informal
sketch here, leaving the interested reader to consult the more formal
analysis by Gilbert and Lynch. The key technical assumption is that a
processes' behavior can only be influenced by the messages it actually
receives---it cannot be affected by messages that are sent to it but
never delivered.

In Figure \ref{fig:linear_example11}, suppose the two processes are on
opposite sides of a network partition, so that no information can be
exchanged between them (even indirectly through a third party). If we
just consider the execution of $P_2$ by itself, without $P_1$,
linearizability would require it to read the value $2$ for $y$. If we
do consider $P_1$, linearizability requires that the read access to
$y$ must return $1$. But if $P_2$ cannot send messages to $P_1$, then
$P_2$'s behavior cannot be influenced by the write access to $y$, so
it would still have to return $2$, violating
consistency. Alternatively, it could delay returning any result until
it is able to exchange messages with $P_1$. But if the partition never
recovers, $P_1$ will wait forever, violating availability.
\end{proof}

While the proof of the CAP theorem is simple, its interpretation is
subtle and has been the subject of much discussion in the years since
[@2012CAP12Years]. It is sometimes assumed that the CAP theorem claims
that a distributed system can only offer two of the properties C, A,
and P. In fact, the theorem constrains, but does not prohibit the
existence of, applications that apply some relaxed amount of all three
features. The CAP theorem only rules out their combination when all
three are interpreted in a highly idealized sense.

In practice, applications can tolerate much weaker levels of
consistency than linearizability. Furthermore, network partitions are
usually not as dramatic as an indefinite communications blackout. Real
conditions in our context are likely to be chaotic, featuring many
smaller disruptions and delays and sometimes larger
ones. Communications between different clients may be affected
differently, with nearby agents generally likely to have better
communication channels between them than agents that are far
apart. Finally, CAP-availability is a suprisingly weak
condition. Generally one cares about the actual time it takes to
handle user requests, but the CAP theorem exposes difficulties just
ensuring the system handles requests at all. Altogether, the extremes
of C, A, and P in the CAP theorem are not the appropriate conditions
to apply to many, perhaps most, real-world applications.


### CAP theorem for sequential consistency

Next we consider a slightly more relaxed consistency model that admits
a greater range of system behaviors while maintaining the total order
guarantees of atomic consistency.

Sequential consistency is a relaxation of atomic consistency, but not
by much. The model is still too strict to enforce under partition
conditions.

\begin{lemma}
    An eventually-available system cannot provide sequential consistency in the presense of network partitions.
\end{lemma}
\begin{proof}

The proof is an adaptation of Theorem \ref{thm:cap}. Suppose $P_1$ and
$P_2$ form of CAP-available distributed system and consider the
following execution: $P_1$ reads $x$, then assigns $y$ the value
$1$. $P_2$ reads $y$, then assigns $x$ the value $1$. (Note that this
is the sequence of requests shown in Figure \ref{fig:nonsequential1},
but we make no assumptions about the values returned by the read
requests). By availability, we know the requests will be handled (with
responses sent back to clients) after a finite amount of time. Now
suppose $P_1$ and $P_2$ are separated by a partition so they cannot
read each other's writes during this process. For contradiction,
suppose the execution is equivalent to a sequential order.

If $W(y,1)$ precedes $R(y)$ in the sequential order, then $R(y)$ would
be constrained to return to $1$. But $P_2$ cannot pass information to
$P_1$, so this is ruled out. To avoid this situation, suppose the
sequential order places $R(y)$ before $W(y,1)$, in which case $R(y)$
could correctly return the initial value of $0$. However, by
transitivity the $R(x)$ event would occur after $W(x,1)$ event, so it
would have to return $1$. But there is no way to pass this information
from $P_1$ to $P_2$. Thus, any attempt to consistently order the
requests would require commuting $W(y,1)$ with $R(x)$ or $W(x,1)$ with
$R(y)$, which would violate program order.
\end{proof}

As discussed in [@2019wideningcap], this stronger theorem was
essentially proved by Birman and Friedman [@10.5555/866855], before
the CAP theorem.

### Fundamental tradeoffs

Broadening our perspective, the tension between consistency and
availability is a prototypical example of a deeper tension in
computing: that between safey and liveness properties
[@10.1145/5505.5508; @2012perspectivesCAP]. These terms can be
understood as follows.

- **Safety** properties ensure that a system avoids doing something ``bad''
like violating a consistency invariant. Taken to the extreme, one way
to ensure safety is to do nothing. For instance, we could enforce
safety by never responding to read requests in order to avoid offering
information that is inconsistent with that of other nodes.

- **Liveness** properties ensure that a system will eventually do something
``good'', like respond to a client. Taken to the extreme, one very
lively behavior would be to immediately respond to user requests,
without taking any steps to make sure this response is consistent with
that of other nodes.

Note that in our use cases, an unresponsive system could arguably be
"unsafe." The distinction between the terms in this narrow context is
that "safety" constrains a system's allowable responses to clients, if
one is even given, while liveness requires giving responses. The fact
that both of these have implications for human safety is one more
reason to prefer continuous consistency models over CAP-unavailabile
models like linearizability.

Because of the tension between them, building applications that
provide both safety and liveness features is challenging. The
fundamental tradeoff is that if we want to increase how quickly a
system can respond to requests, eventually we must relax our
constraints on what the system is allowed to return.



## Desiderata for emergency response
\label{sec:des}

Having discussed some of the fundamental distributed systems issues
that arise under real-world network conditions, we turn our attention
to three desiderata we will use to frame and analyze the models
discussed in Sections \ref{sec:contcons} and \ref{sec:sheaf}.

The CAP theorem, and others like it, place fundamental limitations on
the consistency of real-world distributed systems. In the absense of a
"perfect" system, engineers are forced to make tradeoffs. Ideally,
these tradeoffs should be tuned for the specific application in
mind---a protocol that works well in a datacenter might not work well
in a heterogeneous geodistributed setting. This section lists three
desirable features of distributed systems and frameworks for reasoning
about or implementing them. We chose this set based on the particular
details of civil aviation and disaster response, where safety is a
high priority and usage/communication patterns may be unpredictable.

### D1: Quantifiable bounds on inconsistency

\emph{A distributed application should quantify the amount of
consistency it delivers. That is, it should (1) provide a mathematical
way of measuring inconsistency between system nodes, and (2) bound
this value while the system is available.}

The CAP theorem implies that an available data replication application
cannot bound inconsistency in all circumstances. When bounded
inconsistency cannot be guaranteed, a system satisfying D1 may become
unavailable. Alternatively, a reasonable behavior would be to continue
providing some form of availability, but alert the user that due to
network and system use conditions the requisite level of consisteny
cannot be guaranteed by the application, leaving the user with the
choice to assess the risk and continue using the system with a weaker
safety guarantees.

### D2: Accommodation of heterogeneous nodes

\emph{An application should not assume that there is a typical system
node. Instead, the system should accomdate a diverse range of
heterogeneous clients presenting different capabilities, tasks, and
risk-factors.}

One can expect a variety of hardware in the field. For example,
wildfires often involve responses from many different fire
departments, and it must be assumed that they are not always using
identical systems. Different participants in the system may be solving
different tasks, with different levels of access to the network, and
they present different risks. With these sorts of factors in mind, one
should hope for frameworks that are as general as possible to
accomodate a wide variety of clients.

### D3: Optimization for a geodistributed wide area network

\emph{An application should be optimized for the sorts of
communication patterns that occur in geodistributed wide area networks
(WANs) under real-world conditions.}

Consider two incidents. Wouldn't want to enforce needless global
consistency, particularly if the agents in one area do not have the
same consistency requirements for another area.

Network throughput has some (perhaps approximately linear)
relationship with throughput. Communications patterns are likely far
from uniform too. In fact, these two things likely coincide---it is
often that nodes which are nearby have a stronger need to coordinate
their actions than nodes which are far away. For example, consider
manoeauvering airplanes to avoid crash.


\newpage
# Networks for civil emergency response

<Introduction>

## Ad-hoc networking

### Physical communications

The details of the physical communication between processes is outside
the scope of this memo. We make just a few high-level observations
about the possibilities, as the details of the network layer are
likely to have an impact on distributed applications, such as the
shared memory abstraction we discuss below and in Section
\ref{sec:contcons}. For such applications, it may be important to
optimize for the sorts of usage patterns encountered in real
scenarios, which are affected by (among other things) the low-level
details of the network.

The *celluar* model (Figure \ref{fig:centralized}) assumes nodes are
within range of a powerful, centralized transmission station that
performs routing functions. Message passing takes place by
transmitting to the base station (labeled $R$), which routes the
message to its destination. Such a model could be supported by the
ad-hoc deployment of portable cellphone towers transported into the
field, for instance.

The *ad-hoc* model (Figure \ref{fig:decentralized}) assumes nodes
communicate by passing messages directly to each other. This requires
nodes to maintain information about things like routing and the
approximate location of other nodes in the system, increasing
complexity and introducing a possible source of
inconsistency. However, it may be more workable given (i) the
geographic mobility of agents in our scenarios (ii)
difficult-to-access locations that prohibit setting up communication
towers (iii) the inherent need for system flexibility during disaster
scenarios.

\begin{figure}[h]
     \centering
     \begin{subfigure}[b]{0.48\textwidth}
         \centering
         \includegraphics[width=\textwidth]{images/Centralized.png}
         \caption{Cellular network topology}
         \label{fig:centralized}
     \end{subfigure}
     \hfill
     \begin{subfigure}[b]{0.48\textwidth}
         \centering
         \includegraphics[width=\textwidth]{images/Decentralized.png}
         \caption{Ad-hoc network topology}
         \label{fig:decentralized}
     \end{subfigure}
        \caption{Network topology models for geodistributed agents. Edges represent communication links (bidirectional for simplicity).}
        \label{fig:nettopology}
\end{figure}

One can also imagine hybrid models, such as an ad-hoc arrangement of
localized cells. In general, one expects more centralized topologies
to be simpler for application developers to reason about, but to
require more physical infrastructure and support. On the other hand,
the ad-hoc model is more fault resistant, but more complicated to
implement and potentially offering fewer assurancess about
performance. In either case, higher-level applications such as shared
memory abstractions should be tuned for the networking environment. It
would be even better if this tuning can take place dynamically, with
applications reconfiguring manually or automatically to the
particulars of the operating environment. This requires examining the
relationship between the application and networking layers, rather
than treating them as separate blackboxes.


## Delay-tolerant networking

## Ad-hoc DTNs

An interesting possibility is for the *network* to automatically
configure itself to the quality-of-service needs of the
application. For example, a client that receives a lot of requests may
be marked as a preferred client and given higher-priority access to
the network. If UAV vehicles can be used to route messages by acting
as mobile transmission base stations, one can imagine selecting a
flight pattern based on networking needs. For example, if the
communication between two firefighting teams is obstructed by a
geographical feature, a UAV could be dispatched to provide overhead
communication support. Such an arrangement could greatly blur the line
between the networking and application layers.

## Software-defined networking

## Verification of networking protocols


\newpage
# Continuous consistency for shared memory
\label{sec:contcons}

Strong consistency is a discrete proposition: an application provides
strong consistency or it does not. For many real-world applications,
it evidently makes sense to work with data that is consistent up to
some $\epsilon \in \mathbb{R}^{\geq 0}$. Thus, we shift from thinking
about consistency as an all-or-nothing condition, towards consistency
as a bound on inconsistency.

The definition of $\epsilon$ evidently requires a more or less
application-specific notion of divergence between replicas of a shared
data object. Take, say, an application for disseminating the most
up-to-date visualization of the location of a fire front. It may be
acceptable if this information appears 5 minutes out of date to a
client, but unacceptable if it is 30 minutes out of date. That is, we
could measure consistency with respect to *time*. One should expect
the exact tolerance for $\epsilon$ will be depend very much on the
client, among other things. For example, firefighters who are very
close to a fire have a lower tolerance for stale information than a
central client keeping only a birds-eye view of several fire fronts
simultaneously.

Now suppose many disaster-response agencies coordinate with to update
and propagate information about the availability of resources. A
client may want to lookup the number of vehicles of a certain type
that are available to be dispatched within a certain geographic range.
We may stipulate that the value read by a client should always be $4$
of the actual number, i.e. we could measure inconsistency with respect
to some numerical value.

In the last example, the reader may wonder we should tolerate a client
to read a value that is incorrect by 4, when clearly it is better to
be incorrect by 0. Intuitively, the practical benefit of tolerating
weaker values is to tolerate a greater level of imperfection in
network communications. For example, suppose Alice and Bob are
individually authorized to dispatch vehicles from a shared pool. In
the event that they cannot share a message.

Or, would could ask that the the value is a conservative estimate,
possibly lower but not higher than the actual amount. In these
examples, we measure inconsistency in terms of a numerical value.

As a third example,

By varying $\epsilon$, one can imagine consistency as a continuous
spectrum. In light of the CAP theorem, we should likewise expect that
applications with weaker consistency requirements (high $\epsilon$)
should provide higher availability, all other things being equal.

Yu and Vahdat explored the CAP tradeoff from this perspective in a
series of papers [@2000tact; @2000tactalgorithms;
@10.5555/1251229.1251250; @DBLP:conf/icdcs/YuV01; @2002tact]. They
propose a theory of \emph{conits}, a logical unit of data subject to
their three metrics for measuring consistency. By controlling the
threshold of acceptable inconsistency of each conit as a continuous
quantity, applications can exercise precise control the tradeoff
between consistency and performance, trading one for the other in a
gradual fashion.

They built a prototype toolkit called TACT, which allows applications
to specify precisely their desired levels of consistency for each
conit. An interesting aspect of this work is that consistency can be
tuned \emph{dynamically}. This is desirable because one does not know
a priori how much consistency or availability is acceptable.

The biggest question one must answer is the competing goals of
generality and practicality. Generality means providing a general
notion of measuring $\epsilon$, while practicality means enforcing
consistency in a way that can exploit weakened consistency
requirements to offer better overall performance.


- The tradeoff of CAP is a continuous spectrum between linearizability
  and high-availability. More importantly, it can be tuned in real
  time.

- TACT captures neither CAP-consistency (i.e. neither atomic nor
  sequential consistency) nor CAP-availability (read and write
  requests may be delayed indefinitely if the system is unable to
  enforce consistency requirements because of network issues).

## Causal consistency

Causal consistency is that each clients is consistent with a total
order that contains the happened-before relation. It does not put a
bound on divergence between replicas. Violations of causal consistency
can present clients with deeply counterintuitive behavior.

- In a group messaing application, Alice posts a message and Bob
  replies. On Charlie's device, Bob's reply appears before Alice's
  original message.
- Alice sees a deposit for $100 made to her bank account and, because
  of this, decides to withdraw $50. When she refreshes the page, the
  deposit is gone and her account is overdrawn by $50$. A little while
  later, she refreshes the page and the deposit reappears, but a
  penalty has been assessed for overdrawing her account.

In these scenarios, one agent takes an action *in response to* an
event, but other processes observe these causally-related events
taking place in the opposite order. In the first example, Charlie is
able to observe a response to a message he does not see, which does
not make sense to him. In the second example, Alice's observation at
one instance causes her to take an action, but at a later point the
cause for her actions appears to have occurred after her response to
it. Both of these scenarios already violate atomic and sequential
consistency because those models enforce a system-wide total order of
events. Happily, they are also ruled out by causally consistent
systems. The advantage of the causal consistency model is that it
rules out this behavior without sacrificing system availability, as
shown below.

Causal consistency enforces a global total order on events that are
\emph{causally related}. Here, causal relationships are estimated very
conservatively: two events are potentially causally if there is some
way that the outcome of one could have influenced another.

\begin{figure}
  \center
  \includegraphics[scale=0.4]{images/causal1.png}
  \caption{A causally consistent, non-sequentially-consistent execution}
\end{figure}

\begin{lemma}
	Sequential consistency implies causal consistency.
\end{lemma}
\begin{proof}
This is immediate from the definitions. Sequential consistency
requires all processes to observe the same total order of events,
where this total order must respect program order. Causal consistency
only requires processes to agree on events that are potentially
causally related. Program order is a subset of causal order, so any
sequential executions also respects causal order.
\end{proof}

However, causal consistency is not nearly as strong as sequential
consistency, as processes do not need to agree on the order of events
with no causal relation between them. This weakness is evident in the
fact that the CAP theorem does not rule out highly available systems
that maintain causal consistency even during network partitions.

\begin{lemma}
	A causally consistent system need not be unavailabile during partitions.
\end{lemma}
\begin{proof}

Suppose $P_1$ and $P_2$ maintain replicas of a key-value store, as
before, and suppose they are separated by a partition. The strategy is
simple: each process immediately handles read requests by reading from
its local replica, and handles write requests by applying the update
to its local replica. It is easy to see this leads to causally
consistent histories. Intuitively, the fact that no information flows
between the processes also means the events of each process are not
related by causality, so causality is not violated.  \end{proof}

Note that in this scenario, a client's requests are always routed to
the same processor. If a client's requests can be routed to any node,
causal consistency cannot be maintained without losing
availability. One sometimes says that causal consistency is "sticky
available" because clients must stick to the same processor during
partitions.

The fact that causal consistency can be maintained during partitions
suggests it is too weak. Indeed, there are no guarantees about the
difference in values for $x$ and $y$ across the two replicas.

## TACT system model

As in Section \ref{sec:background}, we assume a distributed set of
processes collaborate to maintain local replicas of a shared data
object such as a database. Processes accept read and write requests
from clients to update items, and they communicate with each other to
ensure to ensure that all replicas remain consistent.

However, access to the data store is mediated by a middleware library,
which sits between the local copy of the replica and the client. At a
high level, TACT will allow an operation to take place if it does not
violate user-specific consistency bounds. If allowing an operation to
proceed would violate consistency constraints, the operation blocks
until TACT synchronizes with one or more other remote replicas. The
operation remains blocked until TACT ensures that executing it would
not violate consistency requirements.

$$\textrm{Consistency} = \langle \textrm{Numerical error, \textrm{Order error}, \textrm{Staleness}} \rangle.$$

Processes forward accesses to TACT, which handles commiting them to
the store. TACT may not immediately process the request---instead it
may need to coordinate with other processes to enforce
consistency. When write requests are processed (i.e. when a response
is sent to the originating client), they are only commited in a
\emph{tenative} state. Tentative writes eventually become fully
committed at some point in the future, but when they are commited,
they may be reordered. After fullying committing, writes are in a
total order known to all processes.


\begin{figure}[h]
  \center
  \includegraphics[scale=0.4]{images/TACT Logs.png}
  \caption{Snapshot of two local replicas using TACT}
  \label{fig:tact_logs}
\end{figure}


A write access $W$ can separately quantify its *numerical weight* and
*order weight* on conit $F$. Application programmers have multiple
forms of control:

Consistency is enforced by the application by setting bounds on the
consistency of read accesses. The TACT framework then enforces these
consistency levels.

## Measuring consistency on conits

#### Numerical consistency


#### Order consistency

When the number of tentative (uncommitted) writes is high, TACT
executes a write commitment algorithm. This is a *pull-based* approach
which pulls information from other processes in order to advance
$P_i$'s vector clock, raising the watermark and hence allowing $P_i$
to commit some of its writes.

#### Real time consistency

## Enforcing inconsistency bounds

#### Numerical consistency

We describe split-weight AE.
Yu and Vahdat also describe two other schemes for bounding numerical
error. One, compound AE, bounds absolute error trading space for
communication overhead. In their simulations, they found minimal
benefits to this tradeoff in general. It is possible that for specific
applications the savings are worth it. They also consider a scheme,
Relative NE, which bounds the relative error.

#### Order consistency


#### Real time consistency

## Future work

# Data fusion
\label{sec:sheaf}

Strong consistency models provide the abstraction of an idealized
global truth. In the case of conits, the numerical, commit-order, and
real-time errors are measured with respect to an idealized global
state of the database. This state may not exist on any one replica,
but it is the state each replica would converge to if it were to see
all remaining unseen updates.

We consider distributed applications that receive data from many
different sources, such as from a sensor network (broadly defined). It
will often be the case that some sources of data should be expected to
agree with each other, but they may not. A typical scenario, we want
to integrate these data into a larger model of some kind. Essentially
take a poll, and attempt to synthesize a global picture that agrees as
much as possible with the data reported from the sensor network.

Here, we need a consistency model to measure how successful our
attempts are to synthesize a global image. And to tell us how much our
sensors agree. Ideally, we could use this system to diagnose
disagreements between sensors, identifying sensors that appear to be
malfunctioning, or to detect abberations that necessitate a response.

## Fusion centers

To be written.

## Sheaf theory

### Introduction to presheaves

\begin{definition}
A \emph{partially order-indexed family of sets} is a family of sets indexed by a partially-ordered set,
such that orders between the indices correspond to functions between the sets.
\end{definition}

We can also set $(P, \leq)$ *acts on* the set $\{S_i\}_{i \in I}$.

\begin{definition}
A \emph{semiautomaton} is a monoid paired with a set.
\end{definition}

This is also called a *monoid action* on the set.

\begin{definition}
A copresheaf is a *category acting on a family of sets*.
\end{definition}

\begin{definition}
A presheaf is a *category acting covariantly on a family of sets*.
\end{definition}

### Introduction to sheaves

To be written.

### The consistency radius

To be written.

# Conclusion
\label{sec:conclusion}

To be written

# Bibliography
