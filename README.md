#!/usr/bin/env python3

#Joshua Arbuckle / 100833522
#2023-11-15

# STUDENT version for Project 1.
# TPRG2131 Fall 202x
# Updated Phil J (Fall 202x)
# 
# Louis Bertrand
# Oct 4, 2021 - initial version
# Nov 17, 2022 - Updated for Fall 2022.
# 

# PySimpleGUI recipes used:
#
# Persistent GUI example
# https://pysimplegui.readthedocs.io/en/latest/cookbook/#recipe-pattern-2a-persistent-window-multiple-reads-using-an-event-loop
#
# Asynchronous Window With Periodic Update
# https://pysimplegui.readthedocs.io/en/latest/cookbook/#asynchronous-window-with-periodic-update

import PySimpleGUI as sg
from gpiozero import Button, Servo
from time import sleep
# Hardware interface module
# Button basic recipe: *** define the pin you used
# https://gpiozero.readthedocs.io/en/stable/recipes.html#button
# Button on GPIO channel, BCM numbering, same name as Pi400 IO pin


#Where am I?
hardware_present = False

try:
    key1= Button (5)
    servo = Servo (17)
    hardware_present = True
except ModuleNotFoundError:
    print("Not on a Raspberry Pi or gpiozero not installed.")

# Setting this constant to True enables the logging function
# Set it to False for normal operation
TESTING = True

# Print a debug log string if TESTING is True, ensure use of Docstring, in definition
def log(s):
    
    if TESTING:
        print(s)


# The vending state machine class holds the states and any information
# that "belongs to" the state machine. In this case, the information
# is the products and prices, and the coins inserted and change due.
# For testing purposes, output is to stdout, also ensure use of Docstring, in class
class VendingMachine(object):
    
    PRODUCTS = {
                    "chips 5": ("Chips", 5),
                    "suprise 10": ("SURPRISE", 10),
                    "beer 25": ("Beer", 25),
                    "candy 100": ("Candy", 100),
                    "pop 200": ("Pop", 200),
                }

    # List of coins: each tuple is ("VALUE", value in cents)
    COINS = {
                "nickel": ("Nickel", 5),
                "dime": ("Dime", 10),
                "quarter": ("Quarter", 25),
                "loonie": ("Loonie", 100),
                "toonie": ("Toonie", 200)
            }


    def __init__(self):
        self.state = None  # current state
        self.states = {}  # dictionary of states
        self.event = ""  # no event detected
        self.amount = 0  # amount from coins inserted so far
        self.change_due = 0  # change due after vending
        # Build a list of coins in descending order of value
        values = []
        for k in self.COINS:
            values.append(self.COINS[k][1])
        self.coin_values = sorted(values, reverse=True)
        log(str(self.coin_values))

    def add_state(self, state):
        self.states[state.name] = state

    def go_to_state(self, state_name):
        if self.state:
            log('Exiting %s' % (self.state.name))
            self.state.on_exit(self)
        self.state = self.states[state_name]
        log('Entering %s' % (self.state.name))
        self.state.on_entry(self)

    def update(self):
        if self.state:
            #log('Updating %s' % (self.state.name))
            self.state.update(self)

    def add_coin(self, coin):
        """Look up the value of the coin given by the key and add it in."""
        self.amount += self.COINS[coin][1]

    def button_action(self):
        """Callback function for Raspberry Pi button."""
        self.event = 'RETURN'
        self.update()

    def button_action(self):
        self.event = 'RETURN'
        self.update()
# Parent class for the derived state classes
# It does nothing. The derived classes are where the work is done.
# However this is needed. In formal terms, this is an "abstract" class.
class State(object):
    """Superclass for states. Override the methods as required."""
    _NAME = ""
    def __init__(self):
        pass
    @property
    def name(self):
        return self._NAME
    def on_entry(self, machine):
        pass
    def on_exit(self, machine):
        pass
    def update(self, machine):
        pass

# In the waiting state, the machine waits for the first coin
class WaitingState(State):
    _NAME = "waiting"
    def update(self, machine):
        if machine.event in machine.COINS:
            machine.add_coin(machine.event)
            machine.go_to_state('add_coins')

# Additional coins, until a product button is pressed

                
            
class AddCoinsState(State):
    _NAME = "add_coins"
    def update(self, machine):
        if machine.event == "RETURN":
            machine.change_due = machine.amount  # return entire amount
            machine.amount = 0
            machine.go_to_state('count_change')
        elif machine.event in machine.COINS:
            machine.add_coin(machine.event)
        elif machine.event in machine.PRODUCTS:
            if machine.amount >= machine.PRODUCTS[machine.event][1]:                                                                    
                    machine.go_to_state('deliver_product')
        else:
            pass  # else ignore the event, not enough money for product

# Print the product being delivered
class DeliverProductState(State):
    _NAME = "deliver_product"
    def on_entry(self, machine):
        # Deliver the product and change state
        machine.change_due = machine.amount - machine.PRODUCTS[machine.event][1]
        machine.amount = 0
        print("Buzz... Whir... Click...", machine.PRODUCTS[machine.event][0])
        if machine.change_due > 0:
            machine.go_to_state('count_change')
        else:
            machine.go_to_state('waiting')
            servo.value = 1
            sleep (0.5)
            servo.value = -1

# Count out the change in coins 
class CountChangeState(State):
    _NAME = "count_change"
    def on_entry(self, machine):
        # Return the change due and change state
        print("Change due: $%0.2f" % (machine.change_due / 100))
        log("Returning change: " + str(machine.change_due))
    def update(self, machine):
        for coin_index in range(0, 5):
            #print("working with", machine.coin_values[coin_index])
            while machine.change_due >= machine.coin_values[coin_index]:
                print("Returning %d" % machine.coin_values[coin_index])
                machine.change_due -= machine.coin_values[coin_index]
        if machine.change_due == 0:
            machine.go_to_state('waiting') # No more change due, done


# MAIN PROGRAM
if __name__ == "__main__":
    #define the GUI
    sg.theme('BluePurple')    # Keep things interesting for your users

    coin_col = []
    coin_col.append([sg.Text("ENTER COINS", font=("Helvetica", 24))])
    for item in VendingMachine.COINS:
        log(item)
        button = sg.Button(item, font=("Helvetica", 18))
        row = [button]
        coin_col.append(row)

    select_col = []
    select_col.append([sg.Text("SELECT ITEM", font=("Helvetica", 24))])
    for item in VendingMachine.PRODUCTS:
        log(item)
        button = sg.Button(item, font=("Helvetica", 18))
        row = [button]
        select_col.append(row)

    layout = [ [sg.Column(coin_col, vertical_alignment="TOP"),
                     sg.VSeparator(),
                     sg.Column(select_col, vertical_alignment="TOP")
                    ] ]
    layout.append([sg.Button("RETURN", font=("Helvetica", 12))])
    window = sg.Window('Vending Machine', layout)

    # new machine object
    vending = VendingMachine()

    # Add the states
    vending.add_state(WaitingState())
    vending.add_state(AddCoinsState())
    vending.add_state(DeliverProductState())
    vending.add_state(CountChangeState())

    # Reset state is "waiting for coins"
    vending.go_to_state('waiting')

   # Checks if being used on Pi
    if hardware_present:
        key1.when_pressed = vending.button_action

    # The Event Loop: begin continuous processing of events
    # The window.read() function reads events and values from the GUI.
    # The machine.event variable stores the event so that the
    # update function can process it.
    # Now that all the states have been defined this is the
    # main portion of the main program.
    while True:
        event, values = window.read(timeout=10)
        if event != '__TIMEOUT__':
            log((event, values))
        if event in (sg.WIN_CLOSED, 'Exit'):
            break
        vending.event = event
        vending.update()

    window.close()
    print("Normal exit")



# test1
# Joshua Arbuckle / 100833522

"""TPRG2131 Winter 202x RC class starter with simplistic test code."""

class ResistorCapacitor (object):
    """Model a resistor-capacitor network"""
    def __init__(self, resistance, capacitance, initial=0.0):
        return

    #
    # Mutator methods
    #
    def set_voltage(self, voltage):
        """Set the voltage."""
        return


## Test code (place at bottom of the file)
if __name__ == "__main__":
    print("Self testing...")
    rc1 = ResistorCapacitor(1000.0, 1.0e-6)
    rc1.set_voltage(5.0)
    rc2 = ResistorCapacitor(10.0e3, 22.0e-6, 12.0)
    print("rc1:")
    print(rc1.resistance, rc1.capacitance, rc1.initial_voltage)
    for vtime in range(0, 6):
        stime = vtime * 0.5e-3
        print(stime, rc1.voltage(stime))
    print("rc2:")
    print(rc2.resistance, rc2.capacitance, rc2.initial_voltage)
    for vtime in range(0, 6):
        stime = vtime * 150.0e-3
        print(stime, rc2.voltage(stime))
# done
# Resistor class models a resistor that behaves according to Ohm's law.

# MODIFIED TUE FEB 6: fix the PyLint warnings in the unit test code.

# Accessors return value of electrical properties and state variables.

# Mutators set_voltage(), set_current() keep electrical state variables
# voltage, current and power in a self consistent state.

# Series-parallel:
 # Resistor.series(other) returns new Resistor sum of resistances
 # Resistor.parallel(other) returns new Resistor resistances in parallel

#Model a resistor that behaves according to Ohm's law.
class Resistor (object):
    #Constructor for Resistor.
    #
    # res = resistance in Ohms (default 1000 ohms)
    # tol = tolerance as a percentage (default 5%)
    # pwr = power rating in Watts (default 1/4W)
    # Must also initialize the electrical state variables to zero.
    def __init__(self, res=1000.0, tol=5.0, pwr=0.25):
            # Initialize the properties from parameters
            self.resistance = float(res)
            self.tolerance = float(tol)
            self.rating = float(pwr)

            # Initialize the state variables as zero
            self._voltage = 0.0
            self._current = 0.0
            self._power = 0.0
            return

    #
    # Mutator methods
    #
    def set_voltage(self, v):
        """Set the voltage, update current and power accordingly."""
        self._voltage = v
        self._current = v / self.resistance
        self._power = abs(self._voltage * self._current)
        return

    def set_current(self, amps):
        """Set the current, update voltage and power accordingly."""
        self._current = amps
        self._voltage = amps * self.resistance
        self._power = abs(self._voltage * self._current)
        return

    #
    # Accessor methods
    #
    def get_resistance(self):
        return self.resistance
    def get_tolerance(self):
        return self.tolerance
    def get_rating(self):
        return self.rating

    def get_voltage(self):
        return self._voltage
    def get_current(self):
        return self._current
    def get_power(self):
        return self._power

    #
    # Series and parallel combinations
    #

    # Returns a Resistor with resistance equal to self + other.
    #
    # parameter other is an instance of Resistor.
    # The combined tolerances and ratings are the more pessimistic:
    # tolerance is the wider of the two e.g. 5% and 10% --> 10%;
    # ratings is the lesser of the two, e.g. 1/4 W and 1/2 W --> 1/4 W

    def series(self, other):
        rs = self.resistance + other.resistance
        # new tolerance is the larger of the two
        if self.tolerance >= other.tolerance :
            newtol = self.tolerance
        else :
            newtol = other.tolerance
        # new power rating is the smaller of the two
        if self.rating <= other.rating :
            newrating = self.rating
        else :
            newrating = other.rating
        return Resistor( rs, tol=newtol, pwr=newrating)

    def __add__( self, other) :
        """Overload the addition + operator for series resistances."""
        return self.series(other)

    def parallel( self, other) :
        """Returns a Resistor with resistance equal to self // other.
        
        parameter other is an instance of Resistor.
        The combined tolerances and ratings are the more pessimistic:
        tolerance is the wider of the two e.g. 5% and 10% --> 10%;
        ratings is the lesser of the two, e.g. 1/4 W and 1/2 W --> 1/4 W
        """
        rp = 1.0 / (1.0/self.resistance + 1.0/other.resistance)
        # new tolerance is the larger of the two
        if self.tolerance >= other.tolerance :
            newtol = self.tolerance
        else :
            newtol = other.tolerance
        # new power rating is the smaller of the two
        if self.rating <= other.rating :
            newrating = self.rating
        else :
            newrating = other.rating
        return Resistor( rp, tol=newtol, pwr=newrating)

    def __floordiv__( self, other) :
        """Redefine floor division // operator for parallel resistances."""
        return self.parallel(other)

    def __str__(self) :
        """Return a string describing this instance.
        
        Uses Unicode numbers: Omega \x03A9 and Plus-Minus \x00B1.
        """
        return str(self.resistance) + "\u03a9 " \
            + "\u00b1" + str(self.tolerance) + "%" \
            + " (" + str(self.rating) + "W)"


## Test code (place at bottom of the file)
if __name__ == "__main__":
    # Note: for the purpose of the PyLint exercise, the invalid name
    # warnings in the unit test code are disabled. The warnings refer
    # to attribute names r1, r2, rs and rp. They are too short.
    # (eschew foolish consistency...)
    # pylint: disable=invalid-name

    # Import the testing framework (Python standard library section 26.4)
    import unittest

    class TestResistorMethods(unittest.TestCase):
        """This class runs several tests with Resistor objects."""
        def setUp(self):
            """initialize test objects"""
            self.r1 = Resistor(1000.0, tol=10.0, pwr=0.25)
            self.r2 = Resistor(2000.0, tol=5.0, pwr=0.50)
            self.r_default = Resistor()
            self.rs = None
            self.rp = None

        def test_get_resistance(self):
            """test get_resistance"""
            self.assertEqual(
                self.r1.get_resistance(), 1000.0,
                "Incorrectly set resistance in constructor, R1 != 1000")
            self.assertEqual(
                self.r2.get_resistance(), 2000.0,
                "Incorrectly set resistance in constructor, R2 != 2000")
            self.assertEqual(
                self.r_default.get_resistance(), 1000.0,
                "Incorrect default resistance in constructor, R != 1000")

        def test_get_tolerance(self):
            """test get_tolerance"""
            self.assertEqual(
                self.r1.get_tolerance(), 10.0,
                "Incorrectly set tolerance in constructor, tolerance != 10%.")
            self.assertEqual(
                self.r2.get_tolerance(), 5.0,
                "Incorrectly set tolerance in constructor, tolerance != 5%.")
            self.assertEqual(
                self.r_default.get_tolerance(), 5.0,
                "Incorrect default tolerance, tolerance != 5%.")

        def test_get_rating(self):
            """test get_rating"""
            self.assertEqual(
                self.r1.get_rating(), 0.25,
                "Incorrectly set power rating in constructor != 1/4 W.")
            self.assertEqual(
                self.r2.get_rating(), 0.5,
                "Incorrectly set power rating in constructor != 1/2 W.")
            self.assertEqual(
                self.r_default.get_rating(), 0.25,
                "Incorrect default power rating != 1/4 W.")

        def test_initial_state_variables(self):
            """test initial value of state variables"""
            self.assertEqual(
                self.r1.get_voltage(), 0.0,
                "Incorrect initial voltage != 0.")
            self.assertEqual(
                self.r1.get_current(), 0.0,
                "Incorrect initial current != 0.")
            self.assertEqual(
                self.r1.get_power(), 0.0,
                "Incorrect initial power != 0.")

        def test_set_current(self):
            """test set_current"""
            self.r1.set_current(1.0e-3)  # 1mA --> 1V
            self.assertEqual(
                self.r1.get_voltage(), 1.0,
                "Incorrect voltage: I = 1mA, V != 1.0")
            self.assertEqual(
                self.r1.get_power(), 1.0e-3,
                "Incorrect power: I = 1mA, P != 1mW")
            self.r1.set_current(-1.0e-3)  # -1mA --> -1V
            self.assertEqual(
                self.r1.get_voltage(), -1.0,
                "Incorrect voltage: I = -1mA, V != -1.0")
            self.assertEqual(
                self.r1.get_power(), 1.0e-3,
                "Incorrect power: I = -1mA, P != 1mW")

        def test_string(self):
            """test __str__ method"""
            self.assertEqual(
                str(self.r1), "1000.0\u03a9 \u00b110.0% (0.25W)",
                "Incorrect __str__ method.")

        def test_series(self):
            """test series"""
            self.rs = self.r1.series(self.r2)
            self.assertEqual(
                self.rs.get_resistance(), 3000.0,
                "Incorrect series resistance: 1k + 2k != 3k")
            self.assertEqual(
                self.rs.get_tolerance(), 10.0,
                "Incorrect series tolerance: 5% + 10% != 10%")
            self.assertEqual(
                self.rs.get_rating(), 0.25,
                "Incorrect series power rating: 1/4 W + 1/2W != 1/4 W")
            self.assertEqual(
                (self.rs.get_voltage(), self.rs.get_current(), self.rs.get_power()),
                (0.0, 0.0, 0.0),
                "Incorrect series initial state; should be 0,0,0 from init.")

        def test_parallel(self):
            """test parallel"""
            self.rp = self.r1.parallel(self.r2)
            self.assertEqual(
                round(self.rp.get_resistance(), 2), 666.67,
                "Incorrect parallel resistance: 1k // 2k != 666.67")
            self.assertEqual(
                self.rp.get_tolerance(), 10.0,
                "Incorrect series tolerance: 5% // 10% != 10%")
            self.assertEqual(
                self.rp.get_rating(), 0.25,
                "Incorrect series power rating: 1/4 W // 1/2W != 1/4 W")
            self.assertEqual(
                (self.rp.get_voltage(), self.rp.get_current(), self.rp.get_power()),
                (0.0, 0.0, 0.0),
                "Incorrect parallel initial state; should be 0,0,0 from init.")


    # Run the tests that were just defined.
    print("Running unit testing on class Resistor:")
    unittest.main()

# done
