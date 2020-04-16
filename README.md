# Visualization of a genetic algorithm applied to the traveling salesman problem

[Live link.](https://abelchiao.github.io/genetic-algorithm-visualization/)

Table of Contents:
- [Background](#background)
  * [What are genetic algorithms?](#what-are-genetic-algorithms-)
  * [The traveling salesman problem](#the-traveling-salesman-problem)
- [Features](#features)
    + [Visualize the algorithm's progression as the solution population converges toward the shortest distance.](#visualize-the-algorithm-s-progression-as-the-solution-population-converges-toward-the-shortest-distance)
    + [Populate the map with a custom set of points and find the shortest route between them.](#populate-the-map-with-a-custom-set-of-points-and-find-the-shortest-route-between-them)
    + [Manipulate algorithm parameters and test the algorithm under new conditions.](#manipulate-algorithm-parameters-and-test-the-algorithm-under-new-conditions)
- [Algorithm implementation](#algorithm-implementation)
  * [Individual and Population](#---individual----and----population---)
  * [Generating the initial population](#generating-the-initial-population)
  * [Fitness](#fitness)
  * [Fitness proportionate selection](#fitness-proportionate-selection)
  * [Mating and crossover events](#mating-and-crossover-events)
  * [Mutation and local/global optima](#mutation-and-local-global-optima)
  * [Elitism](#elitism)
- [Technologies](#technologies)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

<br>

## Background
### What are genetic algorithms?
Genetic algorithms are optimization algorithms inspired by the principle of Darwinian natural selection.
Solutions to a problem are represented as individual chromosomes within a population and are evaluated for individual fitness/performance.
The fittest individuals in the population reproduce, producing offspring with a combination of the parents' traits; the least fit individuals are removed from the population and replaced with the children.
Further population variance/diversity is driven by random events modeled after chromosomal crossover and genetic mutation, resulting in a population that converges toward a more optimal solution with each generation. 

### The traveling salesman problem
The traveling salesman problem (TSP) is a classic algorithmic problem that, given a list of cities on a map, seeks to find the most efficient route that traverses every given city and returns the salesman to the original city. TSP, classified as NP-hard, is highly studied in theoretical computer science and is and has important applications in operations research.

A brute force approach to solving for the shortest route runs in O(n!) time, making this approach unrealistic for sample sizes of greater than a handful of cities. Assuming routes are symmetric (i.e. the distance between two cities is the same if the route is reversed), there exist (n-1)!/2 possible permutations. A problem consisting of 10 cities has 181,440 possible routes; 16 cities yields 653,837,184,000 possibilities. For even small sample sizes, this approach quickly becomes too computationally expensive for practical use.

This project visualizes the use of a genetic algorithm adapted to solve the traveling salesman problem.

<br>

## Features
#### Visualize the algorithm's progression as the solution population converges toward the shortest distance.
![Demo run](images/demo_run.gif)
___

#### Populate the map with a custom set of points and find the shortest route between them.
![Custom run](images/custom_run.gif)
___

#### Manipulate algorithm parameters and test the algorithm under new conditions.
![Algorithm parameters](images/algo_params.gif)

<br>

## Algorithm implementation
### ```Individual``` and ```Population```
In accordance with the theme of natural selection, solutions in genetic algorithms are represented as individuals in a population. 
Each individual carries a chromosome, which in this implementation, represents the sequence in which the cities are visited.
A population is comprised of many such individuals.

#### Implementation: 
Logic is divided between ```Individual``` and ```Population``` classes.
```
class Individual {
  constructor(mutProb, ...coordinates) {
    this.geneCount = coordinates.length;
    this.mutProb = mutProb;
    this.chromosome = coordinates.slice();
    this.calculateFitness();
  };
  ...
};
```
```
class Population {
  constructor(popSize, crossProb, mutProb, elitismRate, ...coordinates) {
    this.coordinates = coordinates
    this.popSize = popSize;
    this.crossProb = crossProb;
    this.mutProb = mutProb;
    this.elitismRate = elitismRate
    this.totalFitness = 0;
    this.currentGen = [];
    this.genNumber = 0;
    this.numPossibleRoutes = factorial(coordinates.length)
    ...
  };
  ...
};
```
___
### Generating the initial population
To start off, an initial population is generated by creating the specified number of individuals, each representing a random ordering of cities visited.
Randomness is achieved by applying the Fisher-Yates algorithm to shuffle the list of cities when creating each individual's chromosome.

#### Implementation:
An adaptation of the Fisher-Yates algorithm is used to construct random individuals in the initial population.
```
Array.prototype.shuffle = function () {
  let currentIdx = this.length;
  let randomIdx;
  while (currentIdx) {
    randomIdx = Math.floor(Math.random() * currentIdx);
    currentIdx -= 1;
    [this[currentIdx], this[randomIdx]] = [this[randomIdx], this[currentIdx]]
  }
  return this;
};
```
___
### Fitness
Because of how quickly the sample space blows up for any decently-sized set of cities, an optimal or near-optimal solution is unlikely to be contained in the initial random population. 
Instead, the population undergoes a process of natural selection, which drives it toward more optimal conditions.

In this case, as in nature, the fittest individuals of the population have a higher probability to reproduce and pass their genetic data on to the next generation.
The relative fitness of each individual is the inverse of the distance required to traverse the cities in the order specified in its chromosome; the shorter the route, the fitter the individual.

#### Implementation:
An ```Individual```'s fitness is inversely proportional to the route it represents.
```
calculateFitness() {
  let sumDist = 0;
  for (let i = 0; i < this.chromosome.length - 1; i++) {
    sumDist += getDistance(this.chromosome[i], this.chromosome[i+1])
  }
  sumDist += getDistance(this.chromosome[0], this.chromosome.slice(-1)[0])
  this.fitness = 1 / sumDist
  this.distance = sumDist;
  return 1 / sumDist;
};
```
___
### Fitness proportionate selection
In this implementation of a genetic algorithm, I've chosen to use a roulette-wheel-selection scheme to apply selection pressure to increase the fitness of the population with each generation.
To select parent individuals for breeding, the fitness scores of the entire population are summed up and a random multiplier between 0 and 1 is applied to create a fitness threshold.
The roulette wheel is "spun" by iterating over a shuffled list of the individuals in the population and summing each individual's fitness score as it is touched upon.
When the summed fitness scores match or exceed the fitness threshold, the current individual is chosen to be a parent in a mating pair.
Individuals with higher fitness scores are more likely to push the tally past the fitness threshold, thereby driving positive selection pressure.

#### Implementation:
Until the next generation is sufficiently populated, mating pairs are selected via roulette-wheel-selection and pass their offspring to the new population.
```
let matingPair = [];
while (nextGen.length < this.popSize) {
  let fitnessThreshold = Math.random() * this.totalFitness;
  let currentFitness = 0;
  let individuals = this.currentGen.shuffle();
  for (let i = 0; i < individuals.length; i++) {
    currentFitness += individuals[i].fitness;
    if (currentFitness >= fitnessThreshold) {
      matingPair.push(individuals[i]);
      if (matingPair.length === 2) {
        let newChildren = matingPair[0].mate(this.crossProb, this.mutProb, matingPair[1]);
        nextGen = nextGen.concat(newChildren);
        matingPair = [];
      }
      break;
    }
  }
}
```
It is important to note that this is a probabilistic process; the fittest individuals are more likely but not guaranteed to be chosen to reproduce. Similarly, the least fit individuals are less likely to reproduce but are not excluded from reproduction.
___
### Mating and crossover events
When two parent individuals reproduce, the children they pass on to the next generation are not necessarily clones (identical copies) of themselves.
Instead, there is some probability (represented here as crossover probability) that the children will be produced by randomly combining portions of each parents' chromosome in a process analogous to genetic crossover in biology. 
As a result, the child inherits traits from both parents.
Because the chromosomes in this case must always contain the complete set of cities, a special type of crossover - ordered crossover - is required here.

Crossovers introduce genetic diversity into the population; without a way of generating new individuals/solutions, the population would not be able to evolve toward more optimal solutions and would instead stagnate.
In this specific scenario, crossover events combine segments of the routes pulled from each parent to form a new route.
Fitter individuals (representing shorted routes) have a higher probability of reproducing and the combination of portions of their chromosomes has the potential to generate a route shorter than any that are currently represented in the population.

#### Implementation:
If a randomly chosen number between 0 and 1 is less than the crossover probability threshold, ordered crossover is used to generate the children; otherwise, clones of the parents are instead passed on to the next generation.
```
mate(crossProb, mutProb, otherInd) {
  if (Math.random() < crossProb) {
    let childChromosomes = [];
    while (childChromosomes.length < 2) {
      let idx1 = Math.floor(Math.random() * this.chromosome.length);
      while (idx1 >= this.chromosome.length - 1) {
        idx1 = Math.floor(Math.random() * this.chromosome.length);
      }
      let idx2 = idx1 + Math.ceil(Math.random() * (this.chromosome.length - idx1));
      let childChromosome = new Array(this.chromosome.length)
      for (let i = idx1; i < idx2; i ++) {
        childChromosome[i] = this.chromosome[i];
      }
      let reorderedSecondParent = [];
      for (let i = 0; i < this.chromosome.length; i++) {
        reorderedSecondParent[i] = otherInd.chromosome[(idx2+i) % this.chromosome.length];
      }
      let childIdx = idx2;
      reorderedSecondParent.forEach(gene => {
        if (!childChromosome.some(ele => JSON.stringify(ele) === JSON.stringify(gene))) {
          childChromosome[childIdx % this.chromosome.length] = gene;
          childIdx += 1;
        } 
      })
      childChromosomes.push(childChromosome)
      childChromosome = [];
    }
    let children = [];
    childChromosomes.forEach(chromosome => {
      let child = new Individual(this.mutProb, ...chromosome);
      child.mutate(this.mutProb);
      children.push(child)
    })
    return children;
  } else {
    let firstParentClone = new Individual(this.mutProb, ...this.chromosome);
    let secondParentClone = new Individual(this.mutProb, ...otherInd.chromosome);
    firstParentClone.mutate(this.mutProb);
    secondParentClone.mutate(this.mutProb);
    return [firstParentClone, secondParentClone];
  }
}
```
___
### Mutation and local/global optima
As in nature, genetic diversity is the critical driver of evolution.
Without sufficient diversity, the population would tend to converge on local rather than global optima.
The evolution of solutions would then be "stuck" and be unable to continue to evolve toward the shortest route.
The above crossover events help prevent this but having an innate mutation probability also allows the population to "break out" of local optima and continue to seek out the global optima.
Too high a mutation rate, however, can slow down or prevent convergence on optima by introducing too much randomness.

Mutation in this algorithm is represented by a chance to randomly swap the position of two cities in a route, thereby ensuring that there is always a source of new routes when building populations.

#### Implementation: 
If a randomly generated number between 0 and 1 is less than the mutation probability threshold, the position of two genes in the ```Individual```'s chromosome are swapped.
```
mutate() {
  if (Math.random() < this.mutProb) {
    let idx1 = Math.floor(Math.random() * this.chromosome.length);
    let idx2 = Math.floor(Math.random() * this.chromosome.length);
    while (idx1 === idx2) idx2 = Math.floor(Math.random() * this.chromosome.length);
    [this.chromosome[idx1], this.chromosome[idx2]] = 
      [this.chromosome[idx2], this.chromosome[idx1]];
  }
  this.calculateFitness();
  return this.chromosome;
};
```
This is a good time to mention that genetic algorithms are __heuristic__ algorithms; unlike deterministic algorithms that always run the same way, heuristic algorithms are based on probability. 
As a result, there is no guarantee that genetic algorithms will find the absolute best answer or that they will reach an acceptable solution in a given amount of time. 
Instead, they are generally allowed to run until an acceptable threshold is reached.
___
### Elitism
Because our selection scheme is based on probability, the fittest individuals representing the most optimal solutions in the current generation are not guaranteed to be propagated to the next.
As a result, the shortest routes are often lost from one generation to the next even if the broader population trends toward greater fitness.
Unlike in nature, here we can cheat a little by manually finding the fittest individuals (elite individuals) and manually pass them into the next generation to prevent any backtracking.
Typically only a very low percentage of the population needs to be carried over in this way.

#### Implementation:
The current generation is sorted with respect to fitness scores and a proportion, defined by the elitism rate, is selected to be directly passed on to the next generation.
```
passElites() {
  let sortedInds = this.currentGen.sort((a, b) => (a.fitness > b.fitness) ? -1 : 1)
  let numElites = Math.floor(this.elitismRate * this.popSize);
  let elites = sortedInds.slice(0, numElites)
  return elites;
};
```

<br>

## Technologies
This project was implemented using only vanilla JavaScript and HTML5 Canvas (and HTML/CSS).