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

**The problem.** Name it, using the vocabulary from lecture (milestone 2 in the
handout names the three).

**Where in the code.** File and method.

**What it makes expensive.** A concrete future change, or something that already goes
wrong today. What breaks first?

### Problem 2

**The problem.**

**Where in the code.**

**What it makes expensive.**

---

## Milestone 3: Two alternative decompositions

Two different ways to carve up this system. A different split of responsibility, not a
list of local code fixes. Read the handout's appendix before writing this section.

### Alternative A

**The decomposition.** What are the pieces, what does each own, and where do the rules
live?

**One tradeoff.** Something this option actually costs. "No real downside" is not a
tradeoff.

### Alternative B

**The decomposition.**

**One tradeoff.**

### Preference

Which one, and under what conditions? Say what the choice depends on, and what would
make you pick the other one instead.
