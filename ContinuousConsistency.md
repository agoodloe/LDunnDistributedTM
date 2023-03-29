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
  theoretical and practical obstructions to coordination and therefore
  to enforcing safety properties. This self-contained memo surveys
  some of the distributed systems challenges involved in system-wide
  safety and introduces two topics in the literature that may offer
  solutions. Both topics emphasize *continuous* notions of
  consistency. Unlike weak consistency models, continuous consistency
  provides hard upper bounds on the level of inconsistency observable
  by clients.  Unlike strong consistency, it accomodates real-world
  conditions, providing some amount of liveliness even in the presence
  of network partitions. We conclude that continuous consistency
  models are appropriate for analyzing safety-critical systems in
  real-world environments.
---

# Introduction

Civil aviation has traditionally focused primarily on the efficient
and safe transportation of people and goods via the airspace. Owing to
the application of sound engineering practices and conservative
operating procedures, flying is today the safest mode of
transport. The desire not to compromise this safety makes it
challenging to introduce uncrewed vehicles into the airspace. To that
end, the NASA Aeronautics' Airspace Operations and Safety Program
(AOSP) System Wide Safety (SWS) project has been investigating how
crewed and uncrewed aircraft may safely operate in shared
airspace. Wildfire suppression and hurricane relief are being studied
as motivating use cases, in part because the rules for operating in
the US national airspace are typically relaxed during natural
disasters and relief efforts.

Disaster response scenarios present unique challenges for safe
operations. Of fundamental importance is the need for system-wide
coordination in spite of an unpredictable communications
environment. Designing systems that are resilient to these sorts of
environments is a challenge for distributed computing, a subdiscipline
of computer science. This purpose of this memorandum is to enumerate
some of the considerations involved in coordinating air- and
ground-based elements from a distributed computing perspective,
identifying challenges, potential requirements, and frameworks that
suggest possible solutions.

Traditionally, civil aviation has employed simple communication
patterns between airborne and ground-based agents and among
aircraft. For instance, aircraft equipped with Automatic Dependent
Surveillance-Broadcast (ADS-B) monitor their location using GPS and
periodically broadcast this information to air traffic controllers and
nearby aircraft. The use cases under consideration demand more
sophisticated coordination schemes between airborne and ground-based
elements to collectively accomplish goals such as navigating safely in
close proximity, delivering resources to remote locations, and
suppressing fires.

Unfortunately, the operating environment cannot generally be expected
to provide reliable, high-bandwidth internet connections that would
allow any group of system nodes to exchange lots of information
quickly. For instance, obstructions like distance, terrain, smoke, and
weather mean we should expect network packets to be dropped or delayed
in unpredictable ways. We also expect the network characteristics to
vary between deployments and to evolve dynamically in time, with
connections varying in strength as agents move around the
environment. These factors make network performance difficult to
predict and control.

Weak guarantees about network performance make it difficult to
coordinate distributed agents and offer strong safety
guarantees. So-called strong consistency models, the subject of most
of Section \ref{sec:background}, can enforce strong safety guarantees,
but they are unworkably brittle---they can only be provided under
ideal network conditions unless severe performance penalties are
incurred. An examplary result is Brewer's CAP theorem (Theorem
\ref{thm:cap}), which implies that neither *atomic* nor *sequential*
consistency (C) can be guaranteed by an eventually-available (A)
system in the presense of network partitions (P). Partitions, or
transient drops in network connectivity, are virtually guaranteed to
occur in the environments under consideration. The CAP theorem
therefore implies that we cannot use strong consistency to enforce
safety without sacrificing system liveness, i.e. its timeliness in
responding to user requests.

On the other hand, weak consistency models such as *causal* and
*eventual* consistency can be provided by real-world systems. However,
these models are too weak to ensure strong safety guarantees. In
particular, they do not bound the overall divergence between two
replicas of a shared data structure, so they provide few assurances
about the mutual consistency of the data observed by different
clients.

At face value, the CAP theorem would seem to imply that either
consistency (hence safety) or availability must be sacrificed by
distributed systems deployed in the field.  A more nuanced view is
that the theorem observes a fundamental *tradeoff* between consistency
and availability; this tradeoff is amplified by suboptimal network
performance. While the CAP theorem rules out highly idealized systems
that maintain perfect consistency and high availability under
realistic conditions, it does not inherently rule out systems that
typically provide moderate amounts of both.

What does it mean to have an "amount" of consistency? This idea is
made precise by continuous consistency models, i.e. formal measures of
consistency as a continuous value rather than a discrete
proposition. This memo describes two continuous consistency models in
the literature, both of which measure consistency as an upper bound on
some measured amount of *inconsistency,* for some flexible and
quantitative definition and of what it means for objects to be
mutually inconsistent. One model, the theory of *conits*, comes from
research into distributed shared memory, e.g. database
replication. The other, *sheaf-theoretic data fusion*, comes from
research in data integration and sensor networks. Both define
consistency as something which, in principle, can vary smoothly. At
one extreme, both models describe a form of "perfect" consistency that
cannot usually be expected in real applications. At the other extreme,
one has no guarantee about the mutual consistency of two objects. In
the middle, the models place upper bounds on the divergence between
related data objects.

It stands to reason that quantitative measurements of consistency
should in turn offer quantiative measurements of safety. One potential
application of having a continuous consistency model is to evaluate
objectively whether the amount of safety provided by a distributed
system is within tolerable limits, despite an inability to maintain
strong consistency. Of course, the CAP theorem implies that network
performance can become so poor that a system cannot provide tolerable
safety levels while maintaining availability. When adequate safety
margins cannot be enforced, authorities can decide to take fewer
risks. What is centrally important is to know *how much* safety one
has, and that is (hopefully) what is provided by the models in this
document.

## Layout of this document

This memo focuses on two (prima facie unrelated) notions of continuous
consistency developed in the literature. This document aims to be
reasonably self-contained and readable to a broad technical
audience. It is laid out as follows.

Section \ref{sec:background} provides a high-level introduction to
distributed systems and consistency models. We define two strong
models, atomic and sequential consistency, both of which provide
highly desirable safety guarantees; we contrast these with the weaker
guarantees implied by the causal consistency model. Then we turn our
attention to the CAP theorem [@2000brewerCAP] [@2002gilbertlynchCAP],
which captures a fundamental consistency/availability tradeoff in the
presense of network partitions. We observe that the theorem
effectively prohibits both forms of strong consistency in our intended
use case. This raises the question of how, if at all, one can
rigorously enforce safety properties without compromising system
performance beyond acceptable levels.

Informed by the previous discussion, Section \ref{sec:des} offers a
list of three desiderata of distributed applications in the contexts
under consideration. These have been selected as especially desirable
and relevant for our use cases, and they provide a basis for assessing
the applicability of frameworks and techniques to overcome the
apparent limitations imposed by the CAP theorem.

Section \ref{sec:contcons} explains how the \emph{conit} framework
[@2002tact] quantifies the C/A tradeoff with respect to three
metrics. Following Yu and Vahdat, we summarize how conit theory can be
used enforce consistency up to some real-valued $\epsilon \geq
0$. More precisely, we discuss three different approaches for
measuring the divergence of data replicas. The semantics of conits are
defined by applications, while the framework we lay out provides
general-purpose mechanisms for enforcing conit consistency. At the
extreme, conits can enforce strong consistency by setting $\epsilon =
0$. For $\epsilon > 0$, the conits framework offers neither
CAP-consistency nor CAP-availability. In return, applications provide
limited amounts of availability, possibly during network partitions,
while strictly bounding levels of inconsistency. One use case for this
is to ``smooth out" intermittent fluctuations in network performance,
a desirable feature for safety-critical systems operating without
strict assumptions about the network.

Section \ref{sec:sheaf} is an introduction to applied sheaf theory,
which provides a highly general framework for measuring the mutual
consistency of "overlapping" observations (i.e. ones which we expect
to be correlated if not equal, such as the data generated by a sensor
network). We discuss an simulated example, due to
[@2017robinsonCanonical], where sheaves are used to integrate
heterogeneous sensor data, thereby improving an estimated location for
a crashed aircraft. Unlike many introductions to the subject, we
emphasize sheaves as "topologically-flavored" presheaves, viewing
presheaves as highly generalized transition systems. Our expectation
is that this approach makes the subject more accessible to computer
scientists and serves to highlight themes common to Sections
\ref{sec:contcons} and \ref{sec:sheaf}. It may even indicate the
possibility of re-grounding conit theory in the principled
mathematical framework of sheaf theory.

Section \ref{sec:conclusion} concludes with suggestions for future
work.

# Distributed Systems and Consistency Models
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
state of a globally-maintained database, to system clients. When
strong consistency is enforced, clients cannot tell whether they are
all interacting with a single central computer, or a complex system of
independent computers that cooperate by sending messages over a
network. This abstraction shields clients and application developers
from complexity and makes it simpler to reason about system
behavior. For example, if a client modifies a data structure by
submitting a *write request* to one system node, a strongly consistent
system must ensure that other nodes reflect the update as well. If
different nodes were to reflect inconsistent values, the abstraction
of a shared world is broken and safety requirements may be violated. A
few examples of the perils of inconsistency:

- A bank client would be unhappy if deposits that appear in their
  account online are not reflected when they check their balance at an
  ATM, or if they disappear after refreshing the webpage.

- Air traffic controllers cannot safely coordinate the movement of air
  vehicles if they are presented with inconsistent or out-of-date
  information about their position and velocity.

- Potentially dangerous misunderstandings can arise in a group message
  application if messages appear in different orders to different
  clients.

- Resource-tracking systems cannot be trusted if they do not reflect
  the true information about the availability and location of
  resources.

What exactly constitutes system consistency? There are different
consistency models, and the most appropriate model for an application
depends on the semantics it expects, which must be weighed against
requirements. All other things being equal, one wants to have as much
consistency as possible. As will be elaborated on throughout this
document, \textrm{na\"ive} notions of system coherence are brittle in
the sense that they generally cannot be guaranteed for theoretical and
practical reasons; that is, unless one is willing to pay with
significant performance penalties, potentially including applications
that fail to respond to users at all. Therefore, real-world
applications must tolerate weaker notions of consistency. This makes
applications more difficult to reason about, as their behavior may
depend on uncontrollable factors in the environment. As fewer
behaviors can be ruled out, it becomes more difficult to ensure the
system maintains safety-related constraints.

The difficulty of enforcing strong consistency is that it requires
system nodes to coordinate by exchanging information. Absence of a
common memory implies that inter-process communication takes place
over the network (whereas processes running on the same machine can
write data to a common memory location). A foundational assumption of
distributed systems, and an especially apt one in the context of
disaster response scenarios, is that the network is less than
perfectly reliable. Message delivery is not instantaneous, and
delivery times may be unpredictable. The network may be allowed to
silently drop network packets, reorder them, or deliver them several
times. In some cases the network may even act like a malicious
adversary (though we will not consider this scenario in this
document). Imperfections in communications represent obstructions to
perfect system consistency, which in turn makes it challenging to
enforce safety requirements.

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
processed until some strictly greater time $A.t > A.s$ when a response
is sent back to the client. The value $E.t - E.s$ is the
\emph{duration} of the event.

\begin{figure}[h]
  \center
  \includegraphics[scale=0.4]{images/request.png}
  \caption{Lifetime of a client request}
  \label{fig:request}
\end{figure}

While handling the request, the process may coordinate with with other
processes in the background by sending and receiving more
messages. For example, the process may propagate the client's request
to other processes, retrieve up-to-date values from other processes to
give to the client, or delay handling the client's request in order to
handle other requests.

As is often the case, we shall assume that requests handled by a
single process do not overlap in time. This can be enforced with local
serialization methods such as two-phase locking that can be used to
isolate concurrent transactions from each other, providing the
abstraction of a system that handles requests one at a time. On the
other hand, any two processes are allowed to handle events at the same
physical time, so that there is no obvious total order of events
across the system. Instead, one has a partial order called external
order. External order is the order of events that would be witnessed
by an external observer that can watch the real-time behavior of when
systems begin and finish responding to requests.

\begin{definition}

Let $E$ be an execution. Request $r^1$ \emph{externally precedes}
request $r^2$ if $r^1.t < r^2.s$. That is, if the first request
terminates before the second request is accepted. This induces an
irreflexive partial order called external order.

\end{definition}

Because we assume processes handle events one-at-a-time, the events
handled at any one process are totally ordered by external order---one
cannot start before another has finished. Events across different
processes need not be ordered, which will happen exactly when the
events physically overlap in time. This yields the reflexive and
symmetric (but non-transitive) relation $||$. If $x || y$, we say the
events are *physically concurrent.*

Figure \ref{fig:externalorder} shows the external order relation for
an execution. To save space we only show an arrow between two events
of the same process (which are always totally ordered), nor arrows
that can be inferred by transitivity of external precedence.

\begin{figure}[h]
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/externalorder.png}
         \caption{Depiction of external order between concurrent events across three processes. (Intra-process and transitive edges are not depicted.)}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.25]{images/partialorder.png}
         \caption{A DAG induced by external order}
     \end{subfigure}
     \caption{External order}
	 \label{fig:externalorder}
\end{figure}

A reader may ask if we can consider events to be totally ordered by
pairing them with a timestamp that records their physical start time,
therefore resolving ties like $x || y$. This order is not generally
useful for a couple reasons. First, we assume processes have only
loosely synchronized clocks, so timestamps from two different
processes may not be comparable. Additionally, even systems that
enforce atomic consistency (Section \ref{sec:atomic}) do not
necessarily handle requests in order of their physical start times.

### Physical communications

The details of the physical communication between processes is outside
the scope of this memo. We make just a few high-level observations
about the possibilities, as the details of the network layer are
likely to have an impact on any higher-level distributed applications,
such as the shared memory abstraction we discuss below and in Section
\ref{sec:contcons}. For such applications, it may be important to
optimize for the sorts of usage patterns expected, which are affected
by (among other things) the performance characteristics of the
network.

The *celluar* model (Figure \ref{fig:centralized}) assumes nodes are
within range of a powerful, centralized transmission station that
performs routing functions. Message passing takes place by
transmitting to the base station (labeled $R$), which forwards the
message to the its destination. Such a model could be supported by the
ad-hoc deployment of portable cellphone towers transported into the
field, for instance.

The *ad-hoc* model (Figure \ref{fig:decentralized}) assumes nodes
communicate by passing messages directly to each other. This requires
nodes to maintain information about routing, and indirectly the
position of other nodes in the system, increasing complexity and
introducing a possible source of inconsistency. However, it may be
more workable given (i) the geographic mobility of agents in our
scenarios (ii) the difficult-to-access locations that prohibit setting
up communications towers (iii) something else.

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
        \caption{Network topology models. Edges represent communication links.}
        \label{fig:nettopology}
\end{figure}

Of course, these are fairly \textrm{na\"ive} descriptions. One can
consider hybrid models (Figure REF), such as an ad-hoc arrangement
network of localized cells. In general, one expects more centralized
topologies to be simpler for application developers to reason about,
but requires more physical infrastructure and support. On the other
hand, the ad-hoc model is more fault resistant, but more complicated
to implement and offering fewer assurancess about performance.

## Atomic consistency
\label{sec:atomic}

A fundamental distributed application is the *shared distributed
memory* abstraction. We shall assume that all processes maintain a
local replica of a globally shared data object. (For simplicitly, we
shall discuss the data store as a database, but it could be something
else like a filesystem, persistent data object, etc.).  Replication
increases fault tolerance. Clients submit two types of operations to
processes. A *read request* that reads from variable $x$ and returns
value $a$ is written $R(x,a)$. A *write request*, notation $W(x,a)$
represents writing value $a$ to the data item (variable) represented
by $x$.

A *memory consistency model* formally constrains the allowable system
responses during executions. In this example, a model constrains the
possible values of $a$ and $b$. Strong consistency prevents
divergence.

*Atomic consistency* is essentially the strongest feasible consistency
consistency model. It is also known variously as *strict consistency*,
*linearizability*, and sometimes *external consistency*. In the
context of isolated database transactions, the analogous condition is
called *strict serializability.* It is defined by three features:

- All processes agree on a single, global total order defined across all accesses
- That is consistent with external order
- Where each read request $R(x,a)$ returns the value $a$ of the
  most recent write request $W(x,a)$ to $x$.

Figure \ref{fig:linear_example11} shows a linearizable
execution. Figure \ref{fig:linearexample12} shows an execution that is
not linearizable because the read access on $y$ returns stale data
instead of reflecting the most recent write access to $y$.

\begin{figure}[h]
     \begin{subfigure}[a]{1\textwidth}
		 \center
		 \includegraphics[scale=0.4]{images/linear1.png}
		 \caption{A linearizable execution. Any choice of linearization works here.}
		 \label{fig:linear_example11}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear3.png}
		 \caption{\textbf{Fix image.} A non-linearizable execution. The read request $y$ returns a stale value recent update. }
		 \label{fig:linear_example12}
     \end{subfigure}
  \caption{A linearizable and non-linearizable execution.}
  \label{fig:linear_example1}
\end{figure}

A **linearization point** $t \in \mathbb{R} \in [E.s, E.t]$ for an
event $E$ is a time between the event's start and termination. An
execution is linearizable if and only if there is a choice of
linearization point for each access (and no distinct access have the
same linearization point), such that $E$ is semantically equivalent to
the serial execution of events when totally ordered by their
linearization points. In other words, it should appear to an external
order that each access instantaneously took effect at some point
between its start and end time.

Given an execution such as in Figure REF, linearizability can be
visualized as follows: At some point, the linearization point, during
the duration of each access, the access instaneously appears to take
effect. By definition, no two accesses are allowed to the same
linearization point, so the points define a total order, the
linearization order.

\begin{figure}[h]
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear2.png}
         \caption{A linearizable execution in which both reads return $2$.}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear3.png}
         \caption{A linearizable execution in which both reads return $1$.}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/linear3.png}
         \caption{\textbf{Fix image}. A non-linearizable execution---the processes ``disagree'' about which write took effect first.}
         \label{fig:nonlinear}
     \end{subfigure}
  \caption{Two linearizable executions returning different values, with possible linearization points shown in red. In \ref{fig:nonlinear},
  a non-linearizable execution is shown.}
\end{figure}

Figure (REF) shows a linearizable history. Red
dots are drawn to indicate a possible linearization. Figure (REF)
shows a similar execution, except with different return values from
some read accesses. This execution is also linearizable, according to
the linearization shown.

A shared memory system is simply linearizable when all possible
executions of the system are linearizable. For example, a linearizable
system admits the events of Figures REF and REF, but not that of REF.

#### Enforcing linearizability

Linearizability can be enforced with a total order broadcast
mechanism. Total order broadcast is a means for processes to come to a
consensus about the order of a set of events. At an abstract level, a
total order broadcast API implements a routine. The routine accepts a
message and notifies all other processes of this message. Moreover,
the implementation. The routine only returns after the message has
been copied to all other clients and assigned a universally-agreed
upon position in the total order of system events.

All read and write requests are propagated by totally ordered
broadcast. Only after the broadcast mechanism has returned, each
process applies to relevant update to the local replica. For a read
request, the value is read and returned to the client. For a write
request, the write is applied and an acknowledgement is returned to
the client. Executing an accesses only after determining its position
in the total order ensures that all behavior observed by the client
(i.e. the return values) are consistent with the behavior of a
single-replica system.

#### Linearizability vs serializability
What if one write request is still being executed when the next
request is issued? Common local serializability (e.g. two-phase
lockout) mechanisms can be used to ensure the accesses take effect on
the replica in the same total order they are issued. See ??. This
Serializability is about isolating several multi-step transactions
from interfering with each other. Linearizability would be observing
effects in a way that is consistent with the (partial) order an
external viewer would infer by examining start and end times.

### The CAP Theorem

Real-world systems often fall short of behaving as a single perfectly
coherent system. The root of this phenomenon is that there exists a
deep and well-understood tradeoff between system coherence and system
performance. Enforcing consistency comes at the cost of additional
communications, and communications impose overheads, often ones that
can be unpredictable. The network may also fail to deliver messages or
rearrange their order. Such behavior presents obstacles to
consistency, particularly if the system should also exhibit good
performance.

Fox and Brewer [@1999foxbrewer] observed that there is a
fundamental tension between consistency, availability, and
partition-tolerance. This tradeoff was precisely stated and proved in
2002 by Gilbert and Lynch [@2002gilbertlynchCAP]. To summarize
this theorem, we start by defining these terms.

#### Consistency
Consistency in Gilbert and Lynch is defined as atomic consistency
(linearizability). The CAP theorem can be also generalized to
sequential consistency. As discussed in [@2019wideningcap], this
stronger theorem is essentially already proved by Birman and Friedman
[@10.5555/866855].

#### Availability
A CAP-available system is one that will definitely respond to every
client request at some point in the future. In particular, in the
event of a network partition that prevents messages from being
delivered, the system cannot indefinitely suspend processing requests
until the network recovers, as we do not assume partitions eventually
recover. In real applications we also care about the \emph{speed} with
which requests are handled, but the CAP theorem demonstrates there are
already obstacles to ensuring that every request is handled
\emph{eventually}.

#### Partition tolerance
A partition tolerant system continues to function, and ensure whatever
guarantees it is meant to provide, in the face of arbitrary partitions
in the network. Note that partitions may never recover, say if a
critical communications link is permanently severed.

\begin{theorem}[The CAP Theorem]
	\label{thm:cap}
    In the presense of indefinite network partitions, a distributed system
    cannot guarantee both atomic consistency and availability.
\end{theorem}
\begin{proof}
The proof is not complicated. Indeed, it is almost trivial. We give
only a sketch here, leaving the interested reader to consult Gilbert
and Lynch [@2002gilbertlynchCAP]. In the example above, suppose
the two clients have their requests served by two different system
nodes, and suppose these nodes cannot pass messages due to an
indefinite network partition. Linearizability requires that $a$ is $1$
or $2$ and $b = a$.  Clearly $a$ cannot be $2$ if there is no
communication between the two clients. But clearly $b$ cannot be $1$
for the same reason.  To avoid violating the condition $b = a$ we
could suppose the system indefinitely delays responding to the read
requests, but this violates our requirement that system nodes
eventually respond to clients. Therefore, $P$ implies we cannot have
both $C$ and $A$.
\end{proof}

While the proof of the CAP theorem is rather trivial, its
interpretation is subtle and has been the subject of much discussion
in the years since [@2012CAP12Years]. It is sometimes assumed that
the CAP theorem claims that a distributed system can only offer two of
the properties C, A, and P. In fact, the theorem constrains, but does
not prohibit the existence of, applications that apply some relaxed
amount of all three features. The CAP theorem only rules out their
combination when all three are interpreted in a highly idealized
sense.

One way the the CAP theorem is idealized is that it defines
consistency as linearizability, a very strong condition. In practice
one often tolerates weaker levels of consistency. Also, network
partitions are often not as dramatic as an indefinite total
communications blackout. Real-world conditions in our context are
likely more chaotic, featuring many smaller disruptions and delays and
sometimes larger ones. Communications between different clients may be
affected differently, with nearby agents generally likely to have
better communication channels between them than agents that are far
apart. Finally, CAP-availability is a suprisingly weak condition. It
only requires that requests are handled eventually. In a truly highly
available system, we expect requests to be handled quickly almost
always. Altogether, the extremes of C, A, and P in the CAP theorem are
not reflective of most real-world applications.

The tension between consistency and availability is well-understood
[@10.1145/5505.5508]. It is a prototypical example of an even
broader tension in distributed systems: that between safey and
liveness properties [@2012perspectivesCAP]. These terms can be
understood as follows.

#### Safety
Safety properties ensure that a system avoids doing something ``bad''
like violating a consistency invariant. Taken to the extreme, one way
to ensure safety is to do nothing. For instance, we could enforce
safety by never responding to read requests in order to avoid offering
information that is inconsistent with that of other nodes.

#### Liveness
Liveness properties ensure that a system will eventually do something
``good'', like respond to a client. Taken to the extreme, one very
lively behavior would be to immediately respond to user requests,
without taking any steps to make sure this response is consistent with
that of other nodes.

Note that in our use cases, one can imagine that an unresponsive
system could indeed be considered ``unsafe.'' The distinction between
the two here is that safety constrains a system's allowable responses
to clients, if one is even given, while liveness requires giving
responses.

Because of the tension between them, building applications that
provide both of these features is challenging. The fundamental
principle is that if we want to increase how quickly a system can
respond to requests, eventually we must relax our constraints on what
the system is allowed to return. In light of this deep tradeoff and
the safety-critical nature of our use cases, the next section
enumerates three features of distributed applications we consider
particularly desirable in the contexts under consideration.

## Sequential consistency

\begin{figure}[h]
     \begin{subfigure}[a]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/sequential1.png}
         \caption{A non-linearizable, sequentially consistent execution.}
         \label{fig:sequential1}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/sequential2.png}
         \caption{An equivalent serial execution of \ref{fig:sequential1}}
     \end{subfigure}
     \begin{subfigure}[b]{1\textwidth}
         \center
         \includegraphics[scale=0.4]{images/sequential2.png}
         \caption{\textbf{Fix image}. An non-sequentially consistent execution.}
     \end{subfigure}
     \caption{Sequentially consistent execution.}
\end{figure}

At a minimum, enforcing atomic consistency means that an access $E$ at
process $P_i$ cannot return to the client until every other process
has been informed about $E$. For many applications this is an
unacceptably high penalty. A weaker model that is still strong enough
for all most purposes is *sequential* consistency. Processes in a
sequentially consistent system are required to agree on a global total
order of events. However, this order need not be given by external
order. Instead, the only requirement is that events must be given in
process order.

Clearly linearizability implies sequential consistency, seeing as
program order is a subset of external order. The reverse direction is
not true. Figure (REF) shows a sequentially consistent execution and
its equivalent serial execution. This example was given earlier as an
execution that is not linearizable.

FIGURE

Visually, sequential consistency allows "sliding" events along process
time axes like beads along a string. Two events on the same process
cannot cross over each other, but events on different processes may
freely be commuted.

A sequentially consistent system ensures that any execution is
equivalent to some global serial execution, even if this is serial
order is not the one suggested by the actual real-time ordering of
events. When real-time constraints are not important, this provides
essentially the same benefits as linearizability. For example, it
allows programmers to reason about concurrent executions of programs
because the result is always guaranteed to represent some possible
interleaving of instructions, never allowing instructions from one
program to execute out of order.

### Enforcing sequential consistency

Requires a total order broadcast mechanism. All requests broadcast in
such a way that allows all clients to see all write requests in the
same order, but allow read requests to return immediately.

### CAP unavailability

\begin{lemma}
    An eventually-available system cannot provide sequential consistency in the presense of network partitions.
\end{lemma}.
\begin{proof}
\end{proof}

## Causal consistency

\begin{figure}
  \center
  \includegraphics[scale=0.4]{images/causal1.png}
  \caption{A causally consistent, non-sequentially-consistent execution}
\end{figure}

Weak notions of consistency blah foo bar.

Causal consistency is that each clients is consistent with a total
order that contains the happened-before relation. It does not put a
bound on divergence between replicas.

Violations of causal consistency can present clients with
counterintuitive, potentially unsafe, system behavior.

- Alice replies to Bob's message. Charlie sees Alice's reply before
  Bob's original message is delivered.
- Alice sees a deposit for $100 made to her bank account and decides
  to withdraw $50. When she refreshes the page, the deposit is gone
  and her account is overdrawn. A little while later, she refreshes
  the page and the deposit has reappeared.

In these scenarios, a system seems to have a split mind about whether
an event occurred and in what order. Both of these scenarios already
violate atomic and sequential consistency, as both of those models
enforce a system-wide total order of events. They are also ruled out
by causally consistent systems. Causal consistency only enforces a
total order on events that are \emph{causally related}. Here, causal
relationships are estimated very conservatively: two events are
potentially causally if there is some way that the outcome of one
could have influenced another.

\begin{lemma}
	A causally consistent system need not be unavailabile during partitions.
\end{lemma}
\begin{proof}
Note that in this scenario, a client's requests are always routed to
the same processor. If a client's requests can be routed to any node,
causal consistency cannot be maintained without losing
availability. One sometimes says that causal consistency is "sticky
available" because clients must stick to the same processor during partitions.
\end{proof}

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

In general, processes do not need to agree on the order of events with
no possible causal connection between them, so causal consistency is a
weaker condition. The fact that causal consistency can be maintained
during partitions suggests it is too weak. Indeed, there are no
guarantees about the difference in values for $x$ and $y$ across the
two replicas.

# Desiderata
\label{sec:des}

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
inconsistency bound.

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

# Continuous Continuous and the Conit Framework
\label{sec:contcons}

Strong consistency is a discrete proposition: an application provides
strong consistency or it does not. For many real-world applications,
it evidently makes sense to work with data that is consistent up to
some $\epsilon \in \mathbb{R}^{\geq 0}$, but this requires having some
more or less application-specific, quantitative notion of divergence
between replicas. For a weather broadcast, it may be acceptable if
information about the current weather is 2 minutes out of date, but
unacceptable if it is 2 hours out of date. For information about the
current number of available resources, it may be acceptable if the
user observes a value within $10$ of the actual number, or if the
value is a conservative estimate (possibly lower but not higher than
the actual amount).

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

## TACT system model

- As in Section \ref{subsec:system_model}, we assume a distributed set
  of processes collaborate to maintain local replicas of a shared data
  object such as a database. Processes accept read and write requests
  from clients to update items, and they communicate with each other
  to ensure to ensure that all replicas remain consistent. Formally,
  consistency is defined by some *consistency semantics.*

- However, access to the data store is mediated by a middleware
  library, which sits between the local copy of the replica and the
  client. At a high level, TACT will allow an operation to take place
  if it does not violate user-specific consistency bounds. If allowing
  an operation to proceed would violate consistency constraints, the
  operation blocks until TACT synchronizes with one or more other
  remote replicas. The operation remains blocked until TACT ensures
  that executing it would not violate consistency requirements.


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


- TACT mediates and provides bounds on numerical/order/time
  consistency. For each read access $R$, for each conit $F$, the
  consistency of the return value is measured along a
  three-dimensional vector:

- A write access $W$ can separately quantify its *numerical weight*
  and *order weight* on conit $F$. Application programmers have
  multiple forms of control:

- By defining conits

- By defining the weight various updates have on each conit

- Consistency is enforced by the application by setting bounds on the
  consistency of read accesses. The TACT framework then enforces these
  consistency levels.

- The *self-determination* property states that TACT correctly
  enforces the consistency requirements of each read access, i.e. each
  individual read request can determine its acceptable levels of
  consistency. In general, all other things being equal, requests with
  higher consistency constraints will take longer to return.

- TACT captures neither CAP-consistency (i.e. neither atomic nor
  sequential consistency) nor CAP-availability (read and write
  requests may be delayed indefinitely if the system is unable to
  enforce consistency requirements because of network issues).

- The tradeoff of CAP is a continuous spectrum between linearizability
  and high-availability. More importantly, it can be tuned in real
  time.


## Enforcing consistency

### Numerical consistency

We describe split-weight AE.

Yu and Vahdat also describe two other schemes for bounding numerical
error. One, compound AE, bounds absolute error trading space for
communication overhead. In their simulations, they found minimal
benefits to this tradeoff in general. It is possible that for specific
applications the savings are worth it. They also consider a scheme,
Relative NE, which bounds the relative error.

### Order consistency

When the number of tentative (uncommitted) writes is high, TACT
executes a write commitment algorithm. This is a *pull-based* approach
which pulls information from other processes in order to advance
$P_i$'s vector clock, raising the watermark and hence allowing $P_i$
to commit some of its writes.

### Real time consistency

## Future work


# Sheaf theoretic data fusion
\label{sec:sheaf}

To be written.

## Data fusion

To be written.

## Presheaves

To be written.

## Sheaves and assignments

To be written.

## The consistency radius

To be written.

# Conclusion
\label{sec:conclusion}

To be written

# Bibliography