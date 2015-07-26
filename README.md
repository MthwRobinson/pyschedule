# pyschedule - fast forward resource-constrained scheduling in python

![](https://github.com/timnon/pyschedule/blob/master/pics/gantt.png)

pyschedule is the easiest way to match tasks with resources, period. It covers problems such as flow- and job-shop scheduling, travelling salesman, vehicle routing with time windows, and many more combinations of theses. Install it with pip:

```python
pip install pyschedule
```

This is how it works:

```python
# load pyschedule and create a scenario
import pyschedule
S = pyschedule.Scenario('hello pyschedule')

# create two resources
Alice, Bob = S.Resource('Alice'), S.Resource('Bob')

# create three tasks with lengths 1,2 and 3
cook, wash, clean = S.Task('cook',1), S.Task('wash',2), S.Task('clean',3)

# assign tasks to resources, either Alice or Bob. The %-operator connects tasks and resource
S += cook % Alice | Bob
S += wash % Alice | Bob
S += clean % Alice | Bob

# solve and print solution
S.use_makespan_objective()
pyschedule.solvers.pulp.solve(S)
print(S.solution())
```

The printout should look as follows (without the additional makespan task):

```python
[(clean, Bob, 0, 3), (wash, Alice, 0, 2), (cook, Alice, 2, 3)]
```

Here we use a makespan objective which means that we want to minimize the completion time of the last task. Hence, Alice should do the washing from 0 to 2 and then do the cooking from 2 to 3, whereas Bob will only do the cleaning from 0 to 3. This will ensure that both are done after three hours. This table representation is a little hard to read, if you want a visualization, first install matplotlib:

```python
pip install matplotlib
```

and then you can run:

```python
pyschedule.plotters.matplotlib.plot(S)
```

This should show you the following gantt chart:

![](https://github.com/timnon/pyschedule/blob/master/pics/hello-pyschedule.png)

pyschedule supports different solvers, classical MIP-based ones as well as CP. All solvers and their capabilities are listed <a href="https://github.com/timnon/pyschedule/wiki/Overview">here</a>. The default solver used above uses a standard IP-model in combination with CBC, which is part of package <a href="https://pypi.python.org/pypi/PuLP">pulp</a>. If you have CPLEX installed (command "cplex" must be running), you can switch to CPLEX using:

```python
pyschedule.solvers.pulp.solve(S,kind='CPLEX')
```

You can also use GLPK (command "glpsol" must be running). For more details go to the examples folder above or have a look at the following example:

#Alice and Bob optimize their bike paint shop with pyschedule

Alice and Bob are running a nice paint shop for bikes where they pimp bikes with fresh colors. Today they have to paint a green and a red bike. For starters they create a new scenario using pyschedule:
```python
import pyschedule
S = pyschedule.Scenario('bike paint shop')
```

Then they add themselves as resources to the scenario:

```python
Alice = S.Resource('Alice')
Bob = S.Resource('Bob')
```

Painting a bike takes two hours. Moreover, before and after a bike can be painted, it needs to get pre-processed (e.g. tires removed) and post-processed (e.g. tires attached), respectively. Both tasks take one hour each:

```python
green_pre = S.Task('green pre',length=1)
green_paint = S.Task('green paint',length=2)
green_post = S.Task('green post',length=1)

red_pre = S.Task('red pre',length=1)
red_paint = S.Task('red paint',length=2)
red_post = S.Task('red post',length=1)
```

Clearly, one can only paint a bike after the pre-processing and before the post-processing:

```python
S += green_pre < green_paint, green_paint < green_post
S += red_pre < red_paint, red_paint < red_post
```

Since they are equally qualified, each task can be done by either Alice or Bob:

```python
S += green_pre % Alice | Bob
S += green_paint % Alice | Bob
S += green_post % Alice | Bob

S += red_pre % Alice | Bob
S += red_paint % Alice | Bob
S += red_post % Alice | Bob
```

Now we have the first version of our scenario, lets solve and plot it. 

```python
S.use_makespan_objective()
pyschedule.solvers.pulp.solve(S)
pyschedule.plotters.matplotlib.plot(S,color_prec_groups=True)  
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-first.png)

This schedule completes after four hours and suggests to paint both bikes at the same time. However, Alice and Bob have only a single paint shop which they need to share:

```python
Paint_Shop = S.Resource('Paint Shop')
S += red_paint % Paint_Shop
S += green_paint % Paint_Shop
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-paint-shop.png)

Great, everybody can go home after six hours and have a late lunch! Unfortunately, Alice receives a call that the green bike will only arrive after two hours:
```python
S += green_pre > 2
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-later.png)

Still everybody can go home after six hours. However, we encounter another problem, it is actually quite hard to switch the paint shop from red to green because the red color is quite sticky, this takes two hours of external cleaning. We model this with the following conditional precedence constraint, which says that there needs to be a break of two hours if the green painting follows the red one:

```python
S += red_paint + 2 << green_paint
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-changeover-cost.png)

To avoid the cleaning, the red painting is now scheduled after the green painting. Bob will now have to stay in the paint shop for eight hours, so they want to plan for a joint lunch between the third and the fifth working hour:

```python
Lunch = S.Task('Lunch')
S += Lunch > 3, Lunch < 5
S += Lunch % [Alice, Bob]
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-lunch.png)

The lunch will start after the fourth hour, but it requires to switch the order of the paintings again. Our goal so far is to minimize the makespan, that is the final completion time of any task. If no other objective is defined, then this is the default one. We can also make this more explicit:

```python
MakeSpan = S.Task('MakeSpan')
S += MakeSpan > {green_post,red_post}
S += MakeSpan*1 # add the makespan to scenario for use as objective
```

Finally Bob gets a call from the owner of the green bike which offers to give them a huge tip if they manage to finish his bike as early as possible. After talking with Alice, they give a six times higher priority to the green one, which translates into the following flow-time objective (six times the completion time of the green bike to one time of the red one):

```python
S += green_post*6 + red_post
```

![](https://github.com/timnon/pyschedule/blob/master/pics/bike-shop-flow-time.png)

All this sounds quite trivial, but think about the same problem with many bikes and many persons!



























