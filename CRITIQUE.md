# RoomReserve Critique

Fill in each section. One section per milestone. Keep it short and specific. Point at
files and methods, not adjectives.

---

## Milestone 1: The design as it is

Describe the system as the code actually builds it.

**Data model.** The booking is consisted of room id, date, start time, end time and user who make the booking. There are two HashMap in InMemoryStore holding it: slotsByRoomDate(Map<String, List<long[]>>), keyed by room and date, holding all slots; bookerBySlot(Map<String, String>), keyed by room, date, start time and end time, holding the user who made the booking. 
There are some requirements: 
- start time should be less than end time, checked by the caller
- two maps should be updated together
- overlap time is not allowed for the same room booking

**Operations.** 
The caller interacts with RequestHandler through four public functions as following:
- createBooking(room, date, start, end, user): All the inputs are **String**, output value is also **String** -> This function is for making a new booking
- cancelBooking(room, date, start, end): inputs are String type and output is also String type -> The function is used to cancel the existing booking
- rescheduleBooking(room, date, oldStart, oldEnd, newStart, newEnd): inputs are String type and output is also String type -> The function is for moving the existing booking to another slots
- listBookings(room, date): inputs are String type and output is also String type -> The functions is for listing all booking for the room at the date.

**Structure.** What classes exist, what does each own, and who holds a reference to whom?
- class BookingPolicy: own the business rules and their thresholds
- class InMemoryStore: owns the data model - two Maps
- class RequestHandler: owns implementation of requesting parsing, overlap checking, and format checking
- class ReservationApp: own main function, driving a demo script
References:
- RequestHandler  -> InMemoryStore  (creates one instance, delegates storage to it)
- ReservationApp  -> RequestHandler (creates one instance, calls its methods)

**The no-double-booking invariant.** 
It is enforced in:
- RequestHandler.createBooking line 30-36: checked all slots for overlapping before calling addSlot
- InMemoryStore.addSlot line 24-28: check the completely same slots and not check partially overlapping
- BookingPolicy.validate() line 29-33: defines the rule by using overlaps
For reschedule request:
1. Input information including room, date, old time and new time
2. Check all time format and convert them into long type
3. Check end time is greater then start time
4. Find the user who made the old booking
5. Remove the old slot from two Maps
6. Add the new slot in two Maps and return boolean value
7. Return String with 'OK'


---

## Milestone 2: Two design problems

Two problems. For each one, fill in all three parts.

### Problem 1

**The problem.** There is no class for 'booking' which is represented as different variables - **Representational Gap**

**Where in the code.** 
Primarily **InMemoryStore.java** — there are two fields slotsByRoomDate (Map<String, List<long[]>>) and bookerBySlot (Map<String, String>), and every method that reads/writes them.
It also shows up in **RequestHandler.java** — every method takes a booking's fields as separate String parameters instead of one object.

**What it makes expensive.** 
**Concrete future change:** adding a new field to a booking, like a booking id, a purpose/note, is not a one-place edit. Because there's no Booking class to add a field to, it requires touching every layer that currently reconstructs "a booking"：
- Every RequestHandler method that creates or returns a booking grows another parameter
- Every place that builds the "room|date|start|end" key has to be touched consistently
- InMemoryStore needs a third parallel map (or a restructured value type) keyed by the same string convention, kept in sync with the other two by hand

What **"breaks first"** is consistency between the maps: nothing enforces that all maps get updated together, so a change is easy to apply to slotsByRoomDate and forget in bookerBySlot (or the other new map), producing orphaned entries with no compiler error to catch it.

### Problem 2

**The problem.** The validation responsibility should belong to `BookingPolicy`, but it was implemented by `RequestHandler`. - **Misplaced Responsibility**

**Where in the code.**
**BookingPolicy.java** — the entire class is defined but never referenced by any other class in the codebase. RequestHandler.createBooking, lines 30-36 — reimplements a narrower, inline version of the same overlap check instead of delegating to **BookingPolicy.validate()**.

**What it makes expensive.**
Something already goes wrong today: BookingPolicy encodes two rules that createBooking never checks at all — business hours (08:00–20:00) and max booking length (4 hours). Since nothing calls BookingPolicy, a call like handler.createBooking(room, date, "22:00", "23:00", "user") succeeds and returns "OK", even though it violates both rules the code claims to enforce.

It also creates a maintenance trap going forward: if a developer wants to change a business rule (e.g. extend the max length to 6 hours), the obvious place to edit is BookingPolicy — but that edit has zero effect on actual behavior. The system ends up with two competing definitions of "is this booking valid" — one real but incomplete (inline in RequestHandler), one complete but dead (BookingPolicy) — and every future change risks widening the gap between them instead of closing it.

---

## Milestone 3: Two alternative decompositions

### Alternative A

**The decomposition.** 

Split RequestHandler's four operations into four owners instead of one class doing everything:

- **BookingCreation** — owns creating a booking end-to-end: parses the request, calls the shared BookingPolicy (business hours, max length, overlap) before writing, commits to storage.
- **BookingCancellation** — owns cancelling: parses the request, removes from storage.
- **BookingReschedule** — owns rescheduling: parses the request, looks up the old booking, calls the same BookingPolicy against the new time, then removes-and-adds booking.
- **BookingQuery** — owns listBookings: reads and formats, no mutation.

All four share one InMemoryStore (storage only) and one BookingPolicy (rules) as collaborators. And rules live in BookingPolicy, injected into whichever feature needs to check them.

Each feature module is a distinct, explicit place responsible for deciding when to call it.

**One tradeoff.** 

This split doesn't fix the **missing Booking abstraction** — it spreads it further. Today, only RequestHandler has to assemble the five loose fields (room, date, start, end, user) to call InMemoryStore. After splitting into four feature classes, each of them independently receives and threads through the same five loose parameters to reach the same InMemoryStore. If a Booking type is introduced later, the change now touches four classes instead of one.

### Alternative B

**The decomposition.**

**RoomDaySchedule** — one instance per (room, date), owns the actual list of bookings for that slot and enforces the no-double-booking invariant locally: create, cancel, and reschedule are all just method calls on this one object, so none of them can bypass the check.
**ScheduleDirectory** — owns looking up or creating the right RoomDaySchedule for a given (room, date) pair — replaces InMemoryStore's string-concatenated keys with a real lookup on structured values.
**RequestHandler** — shrinks to parsing/formatting only: turn strings into room/date/start/end/user, hand off to ScheduleDirectory to find the right schedule, call its method, format the result.

Rules live inside RoomDaySchedule itself — it is structurally impossible to mutate one into a double-booked state, because create/cancel/reschedule all fall through the same guarded door.

**One tradeoff.**

Because everything is indexed by (room, date), any query that cuts across dates or across a single user's bookings requires scanning every RoomDaySchedule instance instead of asking one place. The decomposition optimizes for "is this room free right now" at the cost of "what has this user booked."

### Preference

I'd pick Alternative B. The choice depends on two things: what queries the system actually needs today, and how badly it needs the no-double-booking invariant to hold by construction rather than by convention.

On the first point: RequestHandler.listBookings is the only read operation this system has, and it's already (room, date) shaped — there is no "list a user's bookings" or "list bookings across a date range" anywhere in the codebase. So Alternative B's cost — cross-date and cross-user queries being expensive — isn't a cost RoomReserve is actually paying today. I'd be choosing B's weakness in a dimension the system doesn't currently use.

On the second point: the real bugs we found earlier (rescheduleBooking silently dropping addSlot's failure) exist precisely because rule-checking is something **each RequestHandler method has to remember to do**. Alternative A keeps that same shape — BookingReschedule "calls BookingPolicy" only because its description says it should, which is exactly the kind of convention that already failed once in this codebase. Alternative B closes that gap structurally: there's only one door into RoomDaySchedule, so a future BookingReschedule-style class literally **cannot skip the check** the way the current one does.
