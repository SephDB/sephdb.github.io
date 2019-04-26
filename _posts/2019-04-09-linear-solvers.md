---
layout: post
title:  "Solving VM Allocation Using Linear Programming Solvers"
---

Situation sketch: you have the task of taking in an arbitrary set of virtual machines to place on your fixed set of servers. You start by writing a simple greedy approach that divides them about evenly to not stress any one server too much. Then comes the idea of wanting to use as few servers as possible to get some power savings, so you adapt it. But now that ends up using all available resources on each server before going on to fill the next, which wreaks havoc on performance because now most CPUs are being used hyperthreaded. So you rethink again to only aggressively fill until physical core count and only start using hyperthreading when all servers are in use this way. More soft constraints keep popping up, and most end up requiring a rethinking of the strategy. These range from “we’d like to avoid oversubscribing cores” to “try to keep these pairs of VMs on the same machine”, and it quickly becomes a pain to balance these in a hand-written algorithm.

Enter linear programming solvers. They allow you to encode your problem into a mathematical model of constraints and an objective function to be optimized, and spit out an optimal solution (which, in most cases along the previous paragraph, they do incredibly fast or tell you that the constraints can’t be met). This post will be an introduction to techniques for encoding a resource allocation issue into a linear programming model, and show a glimpse of how powerful and flexible they can be.

## Linear programming?

Linear Programming (LP for short) is built on the simplest equations that exist: linear equations.

A linear equation is one where each term is a variable multiplied by a constant, like 10*x = 20 (where the solution is that x is 2). In contrast, non-linear equations are ones where terms get more complex, like x*y (with x and y both being variables), log(x), etc. Limiting the equations to linear ones makes for a lot of interesting optimizations in the solver (which this post will not dive into, as we’re concerned with how to use them instead of building them ourselves).

In LP these equations are used to constrain what values are valid for which variables. Constraints can also consist of inequalities, as in the following example:

<center><verbatim>
x + y <= 10<br/>
2*x + y >= 14<br/>
</verbatim></center>

Possible solutions include x=5,y=4; x=6,y=4;...

Because a set of constraints like the above often has lots of solutions, not all of which are equally desirable, LP problems also have an objective function, which is a linear equation without a comparison operator. This function gives a value to every solution of the constraints, which LP solvers then try to minimize (or maximize).

Let’s say we want to minimize the value of x-y in the above example. Using that as our objective function gives us one unique solution: x=4,y=6.

## Mapping the domain

To recap, the example problem we’re solving is this: distribute virtual machines over a fixed set of servers with limited capacity, while minimizing the amount of servers used, the amount of hyperthreaded cores, and keeping certain virtual machines together if possible. These constraints are sufficiently non-trivial to show some interesting techniques you can use to map these kinds of problems to LP problems you can throw at a solver.

The first step in this mapping is to figure out what the results you extract will look like, and choosing the variables to represent that result. In our case, the result is which server each VM gets put on. To get this result out, we put a 2D matrix of variables in the problem: one for each combination of server and VM, constrained to be boolean integers (0 or 1). Let’s call this matrix vm_on_server[vms][servers].

### Hard Constraints

Since a VM has to be put on exactly one server to have a valid solution, we create the following constraint for each VM:

<center><verbatim>
For each VM:<br>
vm_on_server[vm][servers[0]] + vm_on_server[vm][servers[1]] + … = 1
</verbatim></center>
 

Since each of these variables can only be 0 or 1, exactly one of them has to be 1 to satisfy the constraint.

Now on to the second and only other hard constraint in the system we’re working on: resource usage constraints. A maximum on resource usage for each server. For this, we multiply each vm_on_server variable by the amount of resources(eg: CPU cores) the VM uses.

  

<center><verbatim>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + cpu(vms[1])*vm_on_server[vms[1]][server] + … <br>
<=<br>
max_cpu(server)
</verbatim></center>

### Objective function

With the current set of constraints, the solver will find one of the (probably very) many possible solutions that will be able to boot and run. The likelihood of it being a solution that we’re happy with is small, because there is nothing telling it to take into account the other things we want it to do, like use as few servers as possible, not using hyperthreaded cores unless necessary, etc.
These get encoded in the objective functions as variables with their own weight.

To give each server used a cost, we can add a variable for each server that says whether it is used or not (again, a boolean variable that is either 0 or 1), along with a (positive) weight w_1, which then gets added into the objective function:

<center><verbatim>
w_1*server_used[servers[0]] + w_1*server_used[servers[1]] + …
</verbatim></center>


As it stands, the solver will simply set all of them to 0 because there are no constraints saying they need to be set to 1 if there is a VM on a server. To do that, we can change right-hand side of the resource usage constraint to say the following:

<center><verbatim>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + cpu(vms[1])*vm_on_server[vms[1]][server] + … <br>
<=<br>
max_cpu(server)<b>*server_used[server]</b>
</verbatim></center>


If server_used[server] is 0, there can be no VMs on that server(because no VM uses fewer than 0 CPU cores), but when it is 1 the constraint works the same way it did before. Because the solver tries to minimize the objective function, and all server_used variables are associated with a positive weight in it, the solver will set all the ones it can to 0.

Making the LP solver avoid using hyperthreaded cores if possible means associating a cost with every core being hyperthreaded. Ideally, we’d have a variable called hyperthreaded_cores[server] that would indicate this that we can use as a cost. It is possible to get this variable, and doing so uses a (in my opinion really cool) technique called elastic constraints, which gets its own section.

### Elastic constraints

To get constraints that have an upper(or lower) limit that the solver is allowed to cross with an associated cost for each unit it crosses it by, we take the hard constraint that we want to soften, add in a variable to represent the amount the constraint is “violated” by, and use that variable as a cost in the objective function. Eg:

<center><verbatim>
x + y <= 10
</verbatim></center>


Gets transformed into

<center><verbatim>
x + y - excess <= 10<br>
excess >= 0
</verbatim></center>


And excess is added to the objective function with a weight specifying how badly the LP solver needs to avoid violating the constraint.

In the case of a >= constraint, the excess is added to the constraint instead of subtracted, with the same addition to the objective function to penalize violating the constraint.

<center><verbatim>
x + y >= 10<br>
x + y + excess >= 10<br>
excess >= 0
</verbatim></center>


For an equality constraint, violation can go in either direction, and thus two excess variables are needed to make the penalty always positive:

<center><verbatim>
x + y = 10<br>
x + y + excess_neg - excess_pos = 10<br>
excess_neg >= 0<br>
excess_pos >= 0<br>
</verbatim></center>


But what if you want your constraint to have some give before penalisation? If, for example, you only want to penalize when x + y goes under five, or over 12? You could split it up into two constraints(same can be done for the above equality constraint), but a faster way is adding one extra variable that doesn’t appear in the objective function that is allowed to be in the range [-2,5]:

<center><verbatim>
x + y = 10<br>
x + y + excess_neg - excess_pos + slack = 10<br>
excess_neg >= 0<br>
excess_pos >= 0<br>
-2 <= slack <= 5<br>
</verbatim></center>


If x + y is 5, slack will be set to 5 causing no additional cost to the objective function. If it is 4, slack will be set to 5 and excess_neg to 1, etc.

### Hyperthreaded cores

Now we know how elastic constraints work, it’s a lot more clear how to insert a cost for each hyperthreaded core used. We can put an elastic constraint on numbers of cores used per server with an upper limit of the physical core count:

<center><verbatim>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + … - excess <= physical_cores(server)<br>
excess >= 0<br>
</verbatim></center>

And we can put excess into the objective function as a cost. This will cause the solver to spread out the VMs in order to not use hyperthreaded cores if it can help it (but, in combination with the cost per server in use, will try to pack them).

This makes all hyperthreaded cores have the same cost, so the solver won’t care whether it fills up one server completely and uses only physical cores on another, or spreading them out more evenly. To make the solver spread hyperthreaded use more evenly, we can introduce extra costs for extra cores used:

<center><verbatim>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + … - excess <= physical_cores(server)<b>+1</b><br>
excess >= 0<br><br>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + … - excess <= physical_cores(server)<b>+2</b><br>
excess >= 0<br>
…<br>
For each server:<br>
cpu(vms[0])*vm_on_server[vms[0]][server] + … - excess <= physical_cores(server)<b>*2-1</b><br>
excess >= 0
</verbatim></center>

Given that this might introduce a lot of extra constraints and variables, it is useful to limit this by having only a couple of these extra constraints at every N hyperthreaded cores used(with N being for example physical_cores(server)/4) instead of for every single one. Note that the weights associated with each of the excess variables will keep counting up, so giving each of them a weight of 1 will, when e.g. 4 cores are hyperthreaded, result in a cost of 10.

### Keeping certain VMs together

Another interesting use of elastic constraints is to try and keep VMs colocated on the same server(let’s say you want to keep vm1 and vm2 together).

If you want to force them to be together, you can do so in a single hard constraint:

<center><verbatim>
1*vm_on_server[vm1][servers[0]] + 2*vm_on_server[vm1][servers[1]] +...<br/>
=<br/>
1*vm_on_server[vm2][servers[0]] + 2*vm_on_server[vm2][servers[1]] +...
</verbatim></center>

Making this constraint elastic introduces an unintended side effect that if the solver has to pull them apart, it will still try to put them on servers close in number to each other. This happens because when vm1 gets put on server 3 and vm2 on server 5, the difference will be 2(and thus the penalty will be 2 times whatever weight you assigned to the excess variables), while if vm2 would be put on server 4 the excess would only be 1.
To get rid of this side effect, we need a different set of constraints:

<center><verbatim>
For each server: <br>
vm_on_server[vm1][server] - vm_on_server[vm2][server] = 0
</verbatim></center>

Making these elastic makes the solver try to keep them together without caring about how far apart they are if they’re split up, since the only thing that influences the constraints is whether or not they’re on the same server):

<center><verbatim>
For each server:<br>
vm_on_server[vm1][server] - vm_on_server[vm2][server] + excess_neg - excess_pos = 0
excess_neg >= 0<br>
excess_pos >= 0
</verbatim></center>

The excess variables in this case will be either 0 or 1. Note that when there’s a penalty, one excess_pos and one excess_neg variable(the ones for vm1’s server and vm2’s server, respectively) will be 1, so the weight for these will be counted twice, or we can treat one of them as a slack variable instead of an excess variable and not include it in the objective function.

### Putting it all together

Now that we have constraints for all things we think are important, the objective function is a sum of the server_used and all the excess variables with their weight. With the weights, we can decide how important each soft constraint is. Making the weight higher will make the solver try harder to satisfy that particular constraint. This is where tuning and experimentation comes in. Want to power up as few servers as possible? Increase the weight of the server_used variables. Ok with hyperthreaded core usage if it means two VMs can stay together? Ramp up the weight for those excess variables. Etc.

## Generalizing to flexible set of servers

If we have a flexible set of servers to our disposal, we can still use an LP solver, but not without some extra work up front. The problem with a flexible set of servers is that the LP problem needs constraints per server that’s potentially used, so we need a fixed set of servers. To get a fixed set, we can run a greedy algorithm first that fills servers up to their physical core count, and thus get an upper bound on the amount of servers we need. With that number, we can then use an LP model like described in this post to solve while minimizing the amount of those servers used.

## Conclusion

In this post, we looked at an example problem of resource allocation and the various techniques in LP we can use to solve it. It hopefully showed how flexible and powerful LP solvers are to use, and maybe gave you some ideas how to apply it in a situation of your own, or simply another tool you know of if a problem that fits it comes up.

## Further reading

PuLP, a Python library with a nice syntax for describing(and solving) LP problems: [https://pythonhosted.org/PuLP/](https://pythonhosted.org/PuLP/)

A paper on elastic constraints: [https://link.springer.com/chapter/10.1007/978-3-540-36510-5_14](https://link.springer.com/chapter/10.1007/978-3-540-36510-5_14)

Another paper doing this at a cloud computing scale, dealing with communication costs between VMs: [https://www.mdpi.com/2073-8994/10/12/756/pdf](https://www.mdpi.com/2073-8994/10/12/756/pdf)
