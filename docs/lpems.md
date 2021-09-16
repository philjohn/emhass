# An EMS based on Linear Programming

In this section we present the basics of the Linear Programming (LP) approach for a household Energy Management System (EMS).

## Motivation

Imagine that we have installed some solar panels in our house. Imagine that we have Home Assistant and that we can control (on/off) some crucial power consumptions in our home. For example the water heater, the pool pump, a dispatchable dishwasher, and so on. We can also imagine that we have installed a battery like a PowerWall, in order to maximize the PV self-consumption. With Home Assistant we also have sensors that can measure the power produced by our PV plant, the global power consumption of the house and hopefully the power consumed by the controllable loads. Home Assistant has released the Energy Dashboard where we can viusalize all these variables in somme really good looking graphics. See: [https://www.home-assistant.io/blog/2021/08/04/home-energy-management/](https://www.home-assistant.io/blog/2021/08/04/home-energy-management/)

Now, how can we be certain of the good and optimal management of these devices? If we define a fixed schedule for our deferrable loads, is this the best solution? When we can indicate or force a charge or discharge on the battery? This is a well known academic problem for an Energy Management System.

The first and most basic approach could be to define some basic rules or heuristics, this is the so called rule-based approach. The rules could be some fixed schedules for the deferrable loads, or some threshold based triggering of the battery charge/discharge, and so on. The rule-based approach has the advantage of being simple to implement and robust. However, the main disadvantage is that optimality is not guaranteed. 

The goal of this work is to provide an easy to implement framework where anyone using Home Assistant can apply the best and optimal set of instructions to control the energy flow in a household. There are many ways and techniques that can be found in the literature to implement optimized EMS. In this package we are using just one of those techniques, the Linear Programming approach, that will be presented below.

When I was designing and testing this package in my own house I estimated a daily gain between 5% and 8% when using the optimized approach versus a rule-based one. In my house I have a 5 kWp PV installation with a contractual grid supply of 9 kVA. I have a grid contract with two tariffs for power consumption for the grid (peak and non-peak hours) and one tariff for the excess PV energy injected to the grid. I have no battery installed, but I suppose that the margin of gain would be even bigger with a battery, adding flexibility to the energy management. Of course the disadvantage is the initial capital cost of the battery stack. In my case the gain comes from the fact that the EMS is helping me to decide when to turn on my water heater and the pool pump. If we have a good clear sky day the results of the optimization will normally be to turn them on during the day where solar production is present. But if the day is going to be really clouded, then is possible that the best solution will be to turn them on during the non-peak tariff hours, for my case this is during the night from 9pm to 2am. All these decisions are made automatically by the EMS using forecasts of both the PV production and the house power consumption.

Some other good packages and projects offer similar approaches to EMHASS. I can cite for example the good work done by my friends at the G2ELab in Grenoble, France. They have implemented the OMEGAlpes package that can also be used as an optimized EMS using LP and MILP (see: [https://gricad-gitlab.univ-grenoble-alpes.fr/omegalpes/omegalpes](https://gricad-gitlab.univ-grenoble-alpes.fr/omegalpes/omegalpes)). But here in EMHASS the first goal was to keep it simple to implement using configuration files and the second goal was that it should be easy to integrate to Home Assistant. I am sure that there will be a lot of room for optimize the code and the package implementation as this solution will be used and tested in the future.

I have included a list of scientific references at the bottom if you want to deep into the technical aspects of this subject.

Ok, let's start by a resumed presentation of the LP approach.

## Linear programming

Linear programming is an optimization method that can be used to obtain the best solution from a given cost function using a linear modeling of a problem. Typically we can also also add linear constraints to the optimization problem.

This can be mathematically written as:

$$
& \underset{x}{\text{Maximize  }}   && \mathbf{c}^\mathrm{T} \mathbf{x}\\
& \text{subject to  } && A \mathbf{x} \leq \mathbf{b} \\
& \text{and  } && \mathbf{x} \ge \mathbf{0}
$$

with $\mathbf{x}$  the variable vector that we want to find, $\mathbf{c}$ and $\mathbf{b}$ are vectors with known coefficients and $\mathbf{A}$ is a matrix with known values. Here the cost function is defined by $\mathbf{c}^\mathrm{T} \mathbf{x}$. The inequalities $A \mathbf{x} \leq \mathbf{b}$ and $\mathbf{x} \ge \mathbf{0}$ represent the convex region of feasible solutions. 

We could find a mix of real and integer variables in $\mathbf{x}$, in this case the problem is referred as Mixed Integer Linear Programming (MILP). Typically this kind of problem use the branch and boud type of solvers or similars.

The LP has of course its set of advantages and disadvantages. The main advantage is the that if the problem is well posed and the region of feasible possible solutions is convex, then a solution is guaranteed and solving times are usually fast when compared to other optimization techniques (as dynamic programming for example). However we can easily fall into memory issues, larger solving times and convergence problems if the size of the problem is too high (too many equations).

## Household EMS with LP

The LP problem for the household EMS is posed to minimize the following obtective function:

$$
\sum_{i=1}^{\Delta_{opt}/\Delta_t} 0.001*\Delta_t(unit_{LoadCost_i}*(P_{load_i}+P_{defSum_i})+prod_{SellPrice}*P_{gridNeg_i})
$$

where $\Delta_{opt}$ is the total period of optimization in hours, $\Delta_t$ is the optimization time step in hours, $unit_{LoadCost_i}$ is the cost of the energy from the utility in EUR/kWh, $P_{load}$ is the electricity load consumption, $P_{defSum}$ is the sum of the deferrable loads defined, $prod_{SellPrice}$ is the price of the energy sold to the utility, $P_{gridNeg}$ is the negative component of the grid power, this is the power exported to the grid. All these power are expressed in Watts.

The goal with this cost function is to maximize the self-consumption from PV power while minimizing the energy exported to the grid.

The problem constraints are written as follows.

### The main constraint: power balance 

$$
P_{PV_i}-P_{defSum_i}-P_{load_i}+P_{gridNeg_i}+P_{gridPos_i}+P_{stoPos_i}+P_{stoNeg_i}=0
$$

with $P_{PV}$ the PV power production, $P_{gridPos}$ the positive component of the grid power (from grid to household), $P_{stoPos}$ and $P_{stoNeg}$ are the positive (discharge) and negative components of the battery power (charge).

Normally the PV power production and the electricity load consumption are considered known. In the case of a day-ahead optimization these should be forecasted values. When the optimization problem is solved the others power defining the power flow are found as a result: the deferrable load power, the grid power and the battery power.

### Other constraints

Some other special linear constraints are defined. A constraint is introduced to avoid injecting and consuming from grid at the same time, which is physically impossible. Other constraints are used to control the total time that a deferrable load will stay on and the number of start-ups. 

Constraints are also used to define semi-continuous variables. Semi-continuous variables are variables that must take a value between their minimum and maximum or zero.

A final set of constraints is used to define the behavior of the battery. Notably:
- Ensure that maximum charge and discharge powers are not exceeded.
- Minimum and maximum state of charge values are not exceeded.
- Force the final state of charge value to be equal to the initial state of charge.

### Perfect forecast optimization

This is the first of two type of optimization task that are proposed with this package. In this case the main inputs, the PV power production and the house power consumption, are fixed using historical values from the past. This mean that in some way we are optimizing a system with a perfect knowledge of the future. This optimization is of course non-practical in real life. However this can be give us the best possible solution of the optimization problem that can be later used as a reference for comparison purposes.

### Day-ahead optimization

In this second type of optimization task the PV power production and the house power consumption are forecasted values. This is the action that should be performed in a real case scenario and is the case that should be launched from Home Assistant to obtain an optimized energy management of future actions.

As the optimization is bounded to forecasted values, it will also be bounded to uncertainty. The quality and accuracy of the optimization results will be inevitably linked to the quality of the forecast used for these values. The better the forecast error, the better accuracy of the optimization result.

We are now ready to configure our system using the proposed configuration file and link our package to Home Assistant!

## References

- Camille Pajot, Lou Morriet, Sacha Hodencq, Vincent Reinbold, Benoit Delinchant, Frédéric Wurtz, Yves Maréchal, Omegalpes: An Optimization Modeler as an EfficientTool for Design and Operation for City Energy Stakeholders and Decision Makers, BS'15, Building Simulation Conference, Roma in September 24, 2019.

- Gabriele Comodi, Andrea Giantomassi, Marco Severini, Stefano Squartini, Francesco Ferracuti, Alessandro Fonti, Davide Nardi Cesarini, Matteo Morodo,
and Fabio Polonara. Multi-apartment residential microgrid with electrical and thermal storage devices: Experimental analysis and simulation of energy management strategies. Applied Energy, 137:854–866, January 2015.

- Pedro P. Vergara, Juan Camilo López, Luiz C.P. da Silva, and Marcos J. Rider. Security-constrained optimal energy management system for threephase
residential microgrids. Electric Power Systems Research, 146:371–382, May 2017.

- R. Bourbon, S.U. Ngueveu, X. Roboam, B. Sareni, C. Turpin, and D. Hernandez-Torres. Energy management optimization of a smart wind power plant comparing heuristic and linear programming methods. Mathematics and Computers in Simulation, 158:418–431, April 2019.