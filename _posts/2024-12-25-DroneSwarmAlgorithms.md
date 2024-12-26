---
layout: post
title: Drone Swarm Algorithms
---

Inspired by videos of drones used in shows that coordinate to create images in the sky. 

## What I'm trying to do here
I want to write algorithms that guide groups of drones to navigate obstacles, display formations in the sky, and more.
To start off, I'm planning on building a simulation environment first. 

## Selecting the right tools for the job
After a brief exploration of available tools, I settled on [Panda3d](https://github.com/panda3d/panda3d). 
This is because I wanted to use python to quickly experiment with algorithms.
Panda3d seems to be implemented in C++ with Python bindings. 
It also has 3d visualization features.

## Setting up the environment
It took some time to understand how Panda3d works (at a basic level).
There is a hierarchy of objects that are added to the scene.
Each object is loaded from models that can be imported externally or from default files in the library.
Each object can be adjusted in terms of traits such as positioning and color. 
There is the concept of a TaskManager that can continually run tasks.
It also needs to contain the camera positioning apparently to continually render objects added.

Here is what the code looks like to create 5 boxes in a row with this library.
```
from direct.showbase.ShowBase import ShowBase
from direct.task import Task
from panda3d.core import Point3, loadPrcFileData

# Configure the window
loadPrcFileData("", """
    window-title Drone Simulation
    show-frame-rate-meter 1
    win-size 1280 720
""")

class DroneSimulation(ShowBase):
    def __init__(self):
        ShowBase.__init__(self)
        self.setup_drones()
        self.taskMgr.add(self.update, "UpdateScene")

    def update(self, task):
        """Keep the scene rendering with fixed camera position"""
        self.camera.setPos(0, -20, 3)
        self.camera.lookAt(0, 0, 0)
        return Task.cont
    
    def setup_drones(self):
        # Create 5 drones in a line along the X axis
        self.drones = []
        spacing = 3  # Units between each drone
        
        for i in range(5):
            drone = self.loader.loadModel("box")
            drone.setScale(1, 1, 1)
            # Position each drone 3 units apart
            drone.setPos((i - 2) * spacing, 0, 0)  # Centered around 0
            drone.reparentTo(self.render)
            self.drones.append(drone)

# Create and run the simulation
sim = DroneSimulation()
sim.run()
```

## Drone Movement Algorithms Interface
The next step is to add an interface where I can add implementations of algorithms that control the paths of these drones. 

We can create ab abstract class for the drone algorithms:
```
# Abstract base class for movement algorithms
class DroneAlgorithm(ABC):
    @abstractmethod
    def update(self, drones, dt, time):
        """Update drone positions according to algorithm"""
        pass
    
    @abstractmethod
    def get_name(self):
        """Return algorithm name"""
        pass
```

Here is an example implementation that puts the drones into a line formation:
```
class LineFormation(DroneAlgorithm):
    def update(self, drones, dt, time):
        spacing = 3
        for i, drone in enumerate(drones):
            target = Point3((i - 2) * spacing, 0, 0)
            current_pos = drone.getPos()
            direction = target - current_pos
            if direction.length() > 0.1:
                direction.normalize()
                new_pos = current_pos + (direction * dt)
                drone.setPos(new_pos)
    
    def get_name(self):
        return "Line Formation"
```

Here is a slightly more interesting algorithm that causes the drones to move up and down according to an oscillating sine wave:
```
class WaveMotion(DroneAlgorithm):
    def update(self, drones, dt, time):
        import math
        for i, drone in enumerate(drones):
            pos = drone.getPos()
            offset = math.sin(time * 2.0 + i * 0.5) * 0.5
            drone.setPos(pos.x, pos.y, 2 + offset)
    
    def get_name(self):
        return "Wave Motion"
```

## Using the Algorithms in the Simulation
To use this algorithm in the `DroneSimulation` class, we update the `init` method to set up the algorithms and bind keys to each algorithm so that the drones' positions can be updated in real time.

```
def __init__(self):
    ShowBase.__init__(self)
    self.setup_drones()
    self.setup_algorithms()
    self.time = 0
    self.taskMgr.add(self.update, "UpdateScene")
    self.accept("1", self.set_algorithm, [0])  # Bind key "1" to first algorithm
    self.accept("2", self.set_algorithm, [1])  # Bind key "2" to second algorithm
    self.accept("3", self.set_algorithm, [2])  # Bind key "3" to third algorithm
```

In the `update()` function we update the positions of the drones in real time according to the current selected algorithm. 

```
def update(self, task):
    """Update scene and drone positions"""
    self.camera.setPos(0, -20, 3)
    self.camera.lookAt(0, 0, 0)
    
    dt = globalClock.getDt()
    self.time += dt
    
    # Update drones using current algorithm
    self.current_algorithm.update(self.drones, dt, self.time)
    
    return Task.cont
```

## Future Work
- Environment
    - In this environment, I should be able to create and specify obstacles in an environment, specify weather and air density, and the details on the drones (i.e. number of drones, types of drones, speed, battery).
- Improve existing set up
    - Better models (maybe using Blender)
    - Better specific of 2d display
- More complicated formations
- Explore optimizing going from one formation to another to minimize power used
- Organize into 3d display
- Link to actual hardware
- Graphics, Ray Tracing

## Resources
- [Panda3d Tutorials](https://arsthaumaturgis.github.io/Panda3DTutorial.io/tutorial/tut_lesson02.html)
- [Code for this post](https://github.com/andrew128/drone-swarm-algorithms)