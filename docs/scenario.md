# MASSim Scenario Documentation

## Agents Assemble (2019)

* [Intro](#background-story)
* [Actions](#actions)
* [Percepts](#percepts)
* [Configuration](#configuration)
* [Commands](#commands)

### Background Story

_In the year 2088, after 43 years of depleting Mars' natural resources, humankind now faces a drastically changed ecosystem. Building so many water wells has led to the surface becoming unstable, such that volcanic eruptions have become a regular sight. People have started rebuilding efforts, though they need specialised compounds which are durable enough to survive in the new circumstances. Thus, they once again rely on their trusty All-Terrain-Planetary-Vehicles - now equipped with robotic arms and pneumatic cleanup gear - sending them into particularly hazardous environments, where they can obtain the necessary materials._

### Introduction

The scenario consists of two __teams__ of agents moving on a grid. The goal is to explore the world and acquire blocks to assemble them into complex patterns.

Agents can _attach_ things to each of their 4 sides. When the agents _move_ or _rotate_, the attached things move or rotate with them. Two agents can _connect_ things that are attached to them to create more complex structures.

__Tournament points__ are distributed according to the score of a team at the end of the simulation. Winning a simulation awards 3 points, while a draw results in 1 point for each team.

## Environment

The environment is a rectangular grid. The dimensions are not known to the agents. Agents only perceive positions relative to their own. The x-axis goes from left to right (or eastwards) and the y-axis from top to bottom (southwards).
Each cell of the grid contains up to one thing that may collide with other things (i.e. agents and blocks for now). Only in the beginning of the simulation, an agent shares *the same* cell with one agent of the other team/s (to ensure fairness). Once one of the agents has moved, they cannot overlap again later.

### Entity/Agent

Each agent controls one entity in the simulation (s.t. we can use both terms interchangeably). Agents do not know their absolute positioning in the environment.

### Things

There are number of __things__ that can inhabit a cell of the grid:

* __Entities__: Each agent controls an entity on the grid. Entites can move around and attach themselves to other things.
* __Blocks__: The building blocks of the scenario. Each block has a specific type. Agents can pick up blocks and stick multiple blocks together. Blocks have to be arranged into specific patterns to get score points.
* __Dispenser__: Each dispenser can be used to retrieve a specific kind of block.

### Terrain

Each cell of the grid has a specific terrain type.

* __empty__: If nothing else is specified, a cell is *just a cell*.
* __goal__: Agents have to be on a __goal__ cell in order to be allowed to submit a task.
* __obstacle__: Obstacle cells are not traversable, i.e. they block movement and rotations that would involve the cell.

## Tasks

Tasks have to be completed to get score points. They appear randomly during the course of the simulation.

* __name__: an identifier for the task
* __deadline__: the last step in which the task can be submitted
* __reward__: the number of score points that can be earned by completing the task
* __requirements__: each requirement describes a block that has to be attached to the agent
  * __x/y__: the coordinates of the block (the agent being (0,0))
  * __type__: the required type of the block

## Actions

In each step, an agent may execute _exactly one_ action. The actions are gathered and executed in random order.

Each action has a number of `parameters`. The exact number depends on the type of action. Also, the position of each parameter determines its meaning. Parameters are always string values.

### skip

The agent won't do anything this turn. Always successful.

### move

Moves the agent in the specified direction.

No | Parameter | Meaning
--- | --- | ---
0 | direction | One of {n,s,e,w}, representing the direction the agent wants to move in.

Failure Code | Reason
--- | ---
failed_parameter | Parameter is not a direction.
failed_path | Cannot move to the target location because the agent or one of its attached things are blocked.
failed_forbidden | New agent position would be out of map/grid bounds.

### attach

Attaches a thing to the agent. The agent has to be directly beside the thing.

No | Parameter | Meaning
--- | --- | ---
0 | direction | One of {n,s,e,w}, representing the direction to the thing the agent wants to attach.

Failure Code | Reason
--- | ---
failed_parameter | Parameter is not a direction.
failed_target | There is nothing to attach in the given direction
failed | The thing could not be attached because the agent already has too many things attached OR the thing is already attached to an agent of another team.

### detach

Detaches a thing from the agent. Only the connection between the agent and the thing is released.

No | Parameter | Meaning
--- | --- | ---
0 | direction | One of {n,s,e,w}, representing the direction to the thing the agent wants to detach from.

Failure Code | Reason
--- | ---
failed_parameter | Parameter is not a direction.
failed_target | There was no attachment to detach in the given direction.
failed | There was a thing but not attached to the agent.

### rotate

Rotates the agent (and all attached things) 90 degrees in the given direction. For each attached thing, all _intermediate positions_ for the rotation have to be free as well. For any thing, the intermediate rotation positions are those, which have the same distance to the agent as the thing and are between the thing's current and target positions.

No | Parameter | Meaning
--- | --- | ---
0 | direction | One of {cw, ccw}, representing the rotation direction (clockwise or counterclockwise).

Failure Code | Reason
--- | ---
failed_parameter | Parameter is not a (rotation) direction.
failed | One of the things attached to the agent cannot rotate to its target position OR the agent is currently attached to another agent.

### connect

Two agents can use this action to connect things attached to them. They have to specify their partner and the block they want to connect. Both blocks are connected (i.e. attached to each other) if they are next to each other and the connection would not violate any other conditions.

#### Example

_agent1_ is on (3,3) and _agent2_ is on (3,7). _agent1_ has a block attached on (3,4) and one attached to that block on (3,5). _agent2_ has a block attached on (3,6). Both agents want to connect their attached blocks, namely those on (3,5) (of _agent1_) and (3,6) (attached to _agent2_).
Then, _agent1_ has to perform `connect(agent2,0,2)`, while _agent2_ has to perform `connect(agent1,0,-1)` in the same step. If both actions succeed, the blocks will be connected and still attached to both agents.

No | Parameter | Meaning
--- | --- | ---
0 | agent | The agent to cooperate with.
1/2 | x/y | The local coordinates of the block to connect.

Failure Code | Reason
--- | ---
failed_parameter | First parameter is not an agent of the same team OR x and y cannot be parsed to valid integers.
failed_partner | The partner's action is not `connect` OR failed randomly OR has wrong parameters.
failed_target | At least one of the specified blocks is not at the given position or not attached to the agent or already attached to the other agent.
failed | The given positions are too far apart OR one agent is already attached to the other (or through other blocks), or connecting both blocks would violate the size limit for connected structures.

### disconnect

Disconnects two attachments (probably blocks) of the agent.

No | Parameter | Meaning
--- | --- | ---
0/1 | attachment1 | The x/y coordinates of the first attachment.
2/3 | attachment2 | The x/y coordinates of the second attachment.

Failure Code | Reason
--- | ---
failed_parameter | No valid integer coordinates given.
failed_target | Target locations aren't attachments of the agent or not attached to each other directly.

### request

Requests a new block from a dispenser. The agent has to be in a cell adjacent to the dispenser and specify the direction to it.

E.g. if an agent is on (3,3) and a dispenser is on (3,4), the agent can use `request(s)` to make a block appear on (3,4).

No | Parameter | Meaning
--- | --- | ---
0 | direction | One of {n,s,e,w}, representing the direction to the position of the dispenser to use.

Failure Code | Reason
--- | ---
failed_parameter | Parameter is not a direction.
failed_target | No dispenser was found in the specific position.
failed_blocked | The dispenser's position is currently blocked by another agent or thing.

### submit

Submit the pattern of things that are attached to the agent to complete a task.

No | Parameter | Meaning
--- | --- | ---
0 | task | The name of an active task.

Failure Code | Reason
--- | ---
failed_target | No _active_ task could be associated with first parameter.
failed | One or more of the requested blocks are missing OR the agent is not on a goal terrain.

### all actions

All actions can also have the following failure codes:

Failure Code | Reason
--- | ---
unknown_action | The action is unknown to the server.

## Percepts

Percepts are sent by the server as JSON files and contain information about the current simulation. Initital percepts (sent via `SIM-START` messages) contain static information while other percepts (sent via `REQUEST-ACTION` messages) contain information about the current simulation state.

The complete JSON format is discussed in [protocol.md](protocol.md).

### Initial percept

This percept contains information that does not change during the whole simulation. As mentioned in the protocol description, everything is contained in a `simulation` element.

Complete Example (with bogus values):

```JSON
{
  "type": "sim-start",
  "content": {
    "time": 1556638383203,
    "percept": {
      "name": "agentA1",
      "team": "A",
      "steps": 700,
      "vision": 5
    }
  }
}
```

* __name__: the agent's name
* __team__: the name of the agent's team
* __steps__: the sim's total number of steps

### Step percept

This percept contains information about the simulation state at the beginning of each step.

Agents perceive the state of a cell depending on their vision. E.g. if they have a vision of 5, they can sense all cells that are up to 5 steps away.

Example (complete request-action message):

```json
{
   "type": "request-action",
   "content": {
      "id": 1,
      "time": 1556636930397,
      "deadline": 1556636934400,
      "step" : 1,
      "percept": {
         "score": 0,
         "lastAction": "move",
         "lastActionResult": "success",
         "lastActionParams": ["n"],
         "energy": 300,
         "disabled": false,
         "things": [
            {
               "x": 0,
               "y": 0,
               "details": "",
               "type": "entity"
            },
            {
               "x": 0,
               "y": -5,
               "details": "",
               "type": "entity"
            },
            {
               "x": 2,
               "y": -1,
               "details": "b1",
               "type": "block"
            },
            {
               "x": 2,
               "y": -1,
               "type": "marker",
               "details" : "clear"
            }
         ],
         "terrain": {
            "goal": [[-4,-1],[-4,0],[-5,0]],
            "obstacle": [[4,-1],[4,0],[4,1]]
         },
         "tasks": [
          {
              "name": "task2",
              "deadline": 188,
              "reward" : 44,
              "requirements": [
                  {
                     "x": 1,
                     "y": 1,
                     "details": "",
                     "type": "b0"
                  },
                  {
                     "x": 0,
                     "y": 1,
                     "details": "",
                     "type": "b1"
                  },
                  {
                     "x": 0,
                     "y": 2,
                     "details": "",
                     "type": "b1"
                  }
               ]
            },
         ],
         "attached": [[2,-1]]
      }
   }
}
```

* __score__: the current team score
* __lastAction__: the last action submitted by the agent
* __lastActionResult__: the result of that action
* __lastActionParams__: the parameters of that action
* __energy__: the agent's current energy level
* __disabled__: whether the agent is disabled
* __things__: things in the simulation visible to the agent
  * __x/y__: position of the thing _relative_ to the agent
  * __type__: the type of the thing (entity, block, dispenser, marker,...)
  * __details__: details about the thing
    * for blocks and dispensers: the block type
    * for entities: the team
    * for markers: the type of marker (i.e. clear)
      * _clear_: the cell is about to be cleared
      * _ci_: "clear_immediate" - a clear event will clear the cell in 2 steps or less
      * _cp_: "clear_perimeter" - the cell is in the perimeter of a clear event (i.e. new obstacles may be generated there as part of the event)
* __terrain__: the terrain around the agent (if no value is given for a visible cell, the terrain is just *empty*)
* __task__: a task taht is currently active
  * __name__: the task's identifier
  * __deadline__: the last step during which the task can be completed
  * __reward__: the score points rewarded for completing the job
  * __requirements__: the relative positions in which blocks have to be attached to the agent (the agent being (0,0))
    * __x/y__: the relative position of the required block
    * __type__: the type of the required block
    * __details__: currently not used
* __attached__: an array of position arrays - each position represents a thing that is (directly or indirectly) attached to an entity

## Configuration

Each simulation configuration is one object in the `match` array.

Example:

```JSON
{
  "id": "2019-SampleSimulation",
  "steps": 500,
  "randomSeed": 17,
  "randomFail": 1,

  "attachLimit": 10,
  "clearSteps" : 3,
  "clearEnergyCost" : 30,
  "disableDuration" : 4,
  "maxEnergy" : 300,

  "entities" : [
    {"standard": 10}
  ],

  "blockTypes": [3, 3],
  "dispensers": [5, 10],

  "grid" : {
    "height" : 50,
    "width" : 50,
    "file" : "conf/maps/test40x40.bmp",
    "instructions": [
      ["cave", 0.45, 10, 5, 4],
      ["line-border", 1],
      ["ragged-border", 3]
    ],
    "goals": {
      "number" : 3,
      "size" : [1,2]
    }
  },

  "tasks" : {
    "size" : [2, 4],
    "duration" : [100, 200],
    "probability" : 0.05
  },

  "events" : {
    "chance" : 15,
    "radius" : [3, 5],
    "warning" : 5,
    "create" : [-3, 1],
    "perimeter" : 2
  },

  "setup" : "conf/setup/test.txt"
}
```

For each simulation, the following parameters may be specified:

* __id__: a name for the simulation; e.g. used in replays together with the starting time
* __steps__: the number of steps the simulation will take
* __randomSeed__: the random seed that is used for map generation and action execution
* __randomFail__: the probability for any action to fail (in %)
* __attachLimit__: the maximum number of things that can be attached to each other
* __blockTypes__: upper and lower bounds for the number of block types
* __dispensers__: upper and lower bounds for the number of dispenser per block type
* __entities__: the number of entities (i.e. agents) per type
* __clearSteps__: how often a clear action has to be executed to take effect
* __clearEnergyCost__: how much an effective clear action costs
* __disableDuration__: for how many steps an agent remains disabled
* __maxEnergy__: an agent's initial and maximum energy level
* __grid__:
  * __height/width__: dimensions of the environment
  * __file__: a bitmap file describing the map layout (see examples for more information)
  * __instructions__: an arbitrary number of map generation steps
    * __cave__: generates a cave like structure using a cellular automaton
      * 1st parameter: chance for a cell to start as an obstacle
      * 2nd parameter: number of iterations
      * 3rd parameter: min. number of obstacle neighbours for an empty cell to become an obstacle
      * 4th parameter: min. number of obstacle neighbours for an obstacle to remain an obstacle
    * __line-border__: creates a straight line of obstacles around the map
      * 1st parameter: width of the line
    * __ragged-border__: creates an irregular border around the map
      * 1st parameter: initial (and average) width of the border
  * __goals__:
    * __number__: number of goal areas
    * __size__: bounds for goal area radius
* __tasks__:
  * __size__: bounds for the size of a tasks (i.e. number of blocks)
  * __duration__: bounds for a task's duration (i.e. number of steps)
  * __probability__: probability to create a new task in any step (0-1)
* __events__:
  * __chance__: chance to generate an event in any step (0-100)
  * __radius__: bounds for the event radius
  * __warning__: number of steps the event area is marked before the event occurs
  * __create__: bounds for how many additional obstacles an event can create (besides those that were removed)
  * __perimeter__: an additional radius where new obstacles may be created (added to the event's radius)
* __setup__: a file describing additional steps to be performed before the simulation starts
  * might be useful for testing & debugging
  * see examples for more information

Currently, there is only one standard agent role.


