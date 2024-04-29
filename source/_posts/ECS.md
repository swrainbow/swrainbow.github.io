---
title: How to Build an Entity Component System Game in Javascript
date: 2023-04-17 19:24:01
tags: [设计模式]
categories: [游戏引擎]
abbrlink: 2
swiper_index: 2
cover: https://strainbow.oss-cn-hangzhou.aliyuncs.com/20230718210522.png
---
[搬运翻译](http://vasir.net/blog/game-development/how-to-build-entity-component-system-in-javascript)

创建和操作抽象是编程的本质。在解决问题时，并没有一个“正确”的抽象方法，但有些抽象方法更适合解决特定的问题。基于类的面向对象编程（OOP）是最广泛使用的程序组织范式。当然还有其他范式。基于原型的语言，比如JavaScript，提供了一种不同的思考问题的方式。函数式编程则提供了完全不同的思考和解决问题的方式。编程语言只是一个领域，在这个领域中采用不同的思维方式可以更好地解决问题。

即使在基于类或基于原型的语言中，也存在许多用于结构化代码的方法。我逐渐喜欢上一种更注重数据驱动的代码方法。其中一种技术是实体-组件-系统（ECS）。虽然这是一种通用的架构模式，可应用于许多领域，但它主要用于游戏开发。在本文中，我将介绍ECS的基本概念，并且我们将构建一个基本的HTML5游戏，该游戏的主题是吃方块 - 创意地称为“方块吞噬者”。

# Entity-Component-System
在ECS中，实体只是组件的集合；只是数据的集合。

- 实体：实体只是一个ID。
- 组件：组件只是数据。
- 系统：运行在具有系统组件的每个实体上的逻辑。例如，一个“渲染器”系统将绘制所有具有“外观”组件的实体。

通过这种方法，您避免了创建复杂的继承链。通过这种方法，例如，半兽人不是人类类和兽人类的混合体（可能继承自怪物类）。通过这种方法，半兽人只是一组数据的聚合。

实体就像数据库中的记录一样。组件是实际的数据。以下是高级示例，展示了实体的数据可能是什么样子，按ID和组件显示。这个系统的美妙之处在于，您可以动态构建实体 - 一个实体可以拥有您想要的任何组件（数据）。

|         | component-health  | component-position |  component-appearance |
|---------|-------------------|--------------------|-----------------------|
|entity1  | 100               | {x: 0, y: 0}       | {color: green}        |
|entity2  |                   | {x: 0, y: 0}       |                       |
|entity3  |                   |                    | {color: blue}         |

# Dynamic Data Driven Design
一切都被标记为实体。例如，一颗子弹可能只有一个“物理”和“外观”组件。实体组件系统是数据驱动的。这种方法提供了更大的灵活性和表达能力。其中一个好处是能够在运行时动态添加和删除组件。您可以动态删除外观组件以创建看不见的子弹，或者添加一个“可由玩家控制”的组件，使子弹可以被玩家控制。不需要创建新的类。

这可能会成为一个问题，因为系统必须迭代所有的实体。当然，如果实体太多，优化和结构化代码以使其不必每次迭代都处理所有实体并不是非常困难，但尤其对于基于浏览器的游戏，牢记这一约束是有帮助的。

# 组合（Assemblages）


基于类的方法的一个优点是能够轻松地创建多个相同类型的对象。如果我想要一百个兽人，我只需要创建一百个兽人对象，并知道它们都具有相同的属性。通过ECS，可以通过一个称为组合的抽象来实现这一点，组合只是一种简单地创建具有一些组件组合的实体的方法。例如，一个人类组合可能包含“位置”、“姓名”、“健康”和“外观”组件。一个剑的组合可能只有“外观”和“名称”。

相对于普通的类继承，这种方法的一个优点是能够轻松地向组合添加（或删除）组件。由于它是数据驱动的，您可以根据您想要的任何参数对其进行编程上的操作和更改。也许您想创建大量的人类，但其中一些是看不见的 - 不需要新的类，只需从该实体中删除“外观”组件即可。

# Code
这不是一个构建完善的ECS库的尝试。它旨在作为JavaScript实现的实体组件系统的概述。这并不是最好或最优化的方法，但它可以为对所有内容如何相互关联有一个坚实的理解提供基础。所有代码可以在[GitHub](https://github.com/erikhazzard/RectangleEater/blob/master/scripts/game.js)上找到。

## Entity
抽象化的概念是实体只是一个ID，一个组件的容器。让我们从创建一个可以用来创建实体的函数开始。每个实体只有一个id和components属性。（注意：以下代码期望存在一个名为ECS的全局对象，可以通过var ECS = {};创建。）
```js
ECS.Entity = function Entity(){
    // Generate a pseudo random ID
    this.id = (+new Date()).toString(16) + 
        (Math.random() * 100000000 | 0).toString(16) +
        ECS.Entity.prototype._count;

    // increment counter
    ECS.Entity.prototype._count++;

    // The component data will live in this object
    this.components = {};

    return this;
};
// keep track of entities created
ECS.Entity.prototype._count = 0;

ECS.Entity.prototype.addComponent = function addComponent ( component ){
    // Add component data to the entity
    // NOTE: The component must have a name property (which is defined as 
    // a prototype protoype of a component function)
    this.components[component.name] = component;
    return this;
};
ECS.Entity.prototype.removeComponent = function removeComponent ( componentName ){
    // Remove component data by removing the reference to it.
    // Allows either a component function or a string of a component name to be
    // passed in
    var name = componentName; // assume a string was passed in

    if(typeof componentName === 'function'){ 
        // get the name from the prototype of the passed component function
        name = componentName.prototype.name;
    }

    // Remove component data by removing the reference to it
    delete this.components[name];
    return this;
};

ECS.Entity.prototype.print = function print () {
    // Function to print / log information about the entity
    console.log(JSON.stringify(this, null, 4));
    return this;
};
```

要创建一个新的实体，我们只需要像这样调用：var entity = new ECS.Entity();。

这段代码并没有太多复杂的内容。首先，在函数本身中，我们基于当前时间、Math.random()的调用以及已创建实体的总数生成一个ID。这确保我们为每个实体获得一个唯一的ID。我们递增计数器（原型属性在所有对象实例之间共享，类似于类变量）。然后，我们创建一个空对象来存放组件（即数据）。

我们在原型上公开了addComponent和removeComponent函数（同样，单个函数在所有对象实例之间共享）。每个函数接受一个组件对象，并将传入的组件添加到或从实体中移除。最后，print方法简单地将实体转换为JSON格式，提供所有的数据。我们可以使用这个方法以后将数据导出和重新加载（例如，保存）。

因此，从核心上讲，实体只是一个带有一些数据属性的对象。我们将很快介绍如何创建多个实体以及组合的作用。

# Component

```js
ECS.Components.Health = function ComponentHealth ( value ){
    value = value || 20;
    this.value = value;

    return this;
};
ECS.Components.Health.prototype.name = 'health';
```
要获得一个 Health 组件，您只需要使用 new ECS.Components.Health(VALUE) 来创建它，其中 VALUE 是一个可选的初始值（如果未传入任何值，则默认为 20）。重要的是，在原型上有一个名为 name 的属性，它告诉实体如何称呼这个组件。例如，要创建一个实体并给它一个 health 组件：
```js
var entity = new ECS.Entity();
entity.addComponent( new ECS.Components.Health() );
```
这就是将组件添加到实体所需的全部内容。如果我们现在打印实体（entity.print();），我们将看到类似于以下内容：
```js
{
    "id": "1479f3d15bd4bf98f938300430178",
    "components": {
        "health": {
            "value": 20
        }
    }
}
```
就是这样 - 它只是数据！我们可以直接修改实体的健康值，例如：entity.components.health.value = 40;。我们可以拥有任何我们想要的数据嵌套结构；例如，如果我们创建了一个带有 x 和 y 数据值的位置组件，输出结果将如下所示：
```js
{
    "id": "1479f3d15bd4bf98f938300430178",
    "components": {
        "health": {
            "value": 20
        },
        "position": {
            "x": 426,
            "y": 98
        }
    }
} 
```
由于组件只是数据，它们没有任何逻辑。（根据您的需求，您可以为组件添加一些原型函数，以便在数据计算中使用，但将组件视为纯粹的数据是很有帮助的。）因此，我们现在有了一些数据，但要进行有趣的操作，我们需要在其上运行操作。这就是系统（Systems）的作用所在。

## System
系统运行着你游戏的逻辑。它们接收实体并对具有系统所需的特定组件的实体进行操作。这种思维方式与传统的基于类的编程有所不同。

在基于类的编程中，为了建模一个猫，会存在一个 Cat 类。你会创建猫的对象，然后调用 speak() 方法来让猫发出喵喵声。功能性存在于对象内部。对象不仅仅是数据，还具有功能性。

而在ECS中，为了建模一个猫，你首先会创建一个实体。然后，你会添加一些猫所具有的组件（大小、名字等）。如果你想让实体能够喵喵叫，也许你会给它添加一个 speak 组件，其中的值是 "meow"。不过，这里的区别在于这只是数据 - 也许它看起来像是：
```js
{
    "id": "f279f3d85bd4bf98f938300430178",
    "components": {
        "speak": {
            "sound": "meeeooowww"
        }
    }
}
```

实体本身无法单独执行任何操作。因此，要让一个 "cat" 实体发出声音，你需要使用一个 speak 系统。（注意：组件名称和系统名称不一定要一对一匹配，这只是一个示例。大多数系统使用多个不同的组件。）该系统将查找具有 speak 组件的所有实体，然后运行一些逻辑 - 将实体的数据插入其中。

功能性发生在系统中，而不是在对象本身。你将拥有许多针对你的游戏量身定制的系统。系统是你的主要游戏逻辑所在。对于我们的方块吞噬游戏，我们只需要几个系统：碰撞系统、衰减系统、渲染系统和用户输入系统。

我将系统的结构设计为接收所有实体（这里，实体是一个键值对对象，键是实体ID，值是实体对象）。让我们来看一下渲染系统的一部分代码。请注意，系统只是接收实体的函数。
```js
ECS.systems.render = function systemRender ( entities ) {
    // Here, we've implemented systems as functions which take in an array of
    // entities. An optimization would be to have some layer which only 
    // feeds in relevant entities to the system, but for demo purposes we'll
    // assume all entities are passed in and iterate over them.

    // This happens each tick, so we need to clear out the previous rendered
    // state
    clearCanvas();

    var curEntity, fillStyle; 

    // iterate over all entities
    for( var entityId in entities ){
        curEntity = entities[entityId];

        // Only run logic if entity has relevant components
        //
        // For rendering, we need appearance and position. Your own render 
        // system would use whatever other components specific for your game
        if( curEntity.components.appearance && curEntity.components.position ){

            // Build up the fill style based on the entity's color data
            fillStyle = 'rgba(' + [
                curEntity.components.appearance.colors.r,
                curEntity.components.appearance.colors.g,
                curEntity.components.appearance.colors.b
            ];

            if(!curEntity.components.collision){
                // If the entity does not have a collision component, give it 
                // some transparency
                fillStyle += ',0.1)';
            } else {
                // Has a collision component
                fillStyle += ',1)';
            }

            ECS.context.fillStyle = fillStyle;

            // Color big squares differently
            if(!curEntity.components.playerControlled &&
            curEntity.components.appearance.size > 12){
                ECS.context.fillStyle = 'rgba(0,0,0,0.8)';
            }

            // draw a little black line around every rect
            ECS.context.strokeStyle = 'rgba(0,0,0,1)';

            // draw the rect
            ECS.context.fillRect( 
                curEntity.components.position.x - curEntity.components.appearance.size,
                curEntity.components.position.y - curEntity.components.appearance.size,
                curEntity.components.appearance.size * 2,
                curEntity.components.appearance.size * 2
            );
            // stroke it
            ECS.context.strokeRect(
                curEntity.components.position.x - curEntity.components.appearance.size,
                curEntity.components.position.y - curEntity.components.appearance.size,
                curEntity.components.appearance.size * 2,
                curEntity.components.appearance.size * 2
            );
        }
    }
};
```
这里的逻辑很简单。首先，在进行任何操作之前，我们清除画布。然后，我们遍历所有实体。这个系统负责渲染实体，但我们只关心具有外观（appearance）和位置（position）组件的实体。在我们的游戏中，所有实体都具有这些组件 - 但是，如果我们想要创建玩家可以与之交互的不可见矩形，我们只需要移除外观组件即可。因此，在找到包含系统所需数据的实体之后，我们可以对它们进行操作。

在这个系统中，我们根据外观组件中的颜色属性来渲染实体。另一个好处是我们不必在这里设置所有的外观属性 - 我们可以在碰撞系统、血量系统或衰减系统中设置一些属性；我们对于每个系统要分配哪些角色拥有完全的灵活性。因为系统是由数据驱动的，我们不必将思维局限于“类和对象上的方法”。我们可以根据需要拥有任意数量的系统，无论其复杂性或简单性，以针对任何类型的实体。

Rectangle Eater 游戏系统的概述：

- 碰撞系统（Collision System）：处理实体之间的碰撞逻辑，检测实体之间是否发生碰撞并处理相应的行为。
- 衰减系统（Decay System）：处理实体的衰减逻辑，例如矩形的大小逐渐减小。
- 渲染系统（Render System）：负责渲染实体，根据实体的外观和位置进行绘制。
- 用户输入系统（User Input System）：处理用户输入逻辑，例如键盘事件或鼠标事件，以控制实体的移动或交互等。
这些系统共同协作，构成了 Rectangle Eater 游戏的主要逻辑和功能。每个系统负责不同的任务，并通过操作实体的组件数据来实现相应的行为和效果。

在我们的游戏中的碰撞系统中，我们检查用户控制的实体（由 PlayerControlled 组件指定，通过 userInput 系统处理）与具有碰撞组件的其他实体之间是否发生碰撞。如果发生碰撞，我们会更新实体的健康状态（通过健康组件），立即移除发生碰撞的实体（系统是数据驱动的 - 在每个系统级别动态添加或移除实体是没有问题的），然后随机添加一些新的矩形（其中大部分会随着时间的推移而衰减，在它们变小时，与它们碰撞会获得更多的健康值 - 在这个碰撞系统中我们也会检查这一点）。

和正常的游戏循环一样，调用系统的顺序也很重要。例如，如果你在衰减系统调用之前渲染，你将始终落后一帧。

通过合理安排系统的调用顺序，我们可以确保游戏逻辑按照正确的顺序进行处理，并且各个系统之间的数据和状态得到正确更新和处理。这样可以避免不同系统之间的冲突和不一致，确保游戏的行为和效果符合预期。

# Gluing it all together
最后一步是将所有组件连接在一起。对于我们的游戏，我们只需要完成以下几个步骤：

1. 设置初始实体
2. 设置系统的执行顺序
3. 设置游戏循环，调用每个系统并传入所有实体
4. 设置游戏失败的条件
让我们来看一下设置初始实体和玩家实体的代码：
```js
var self = this;
var entities = {}; // object containing { id: entity  }
var entity;

// Create a bunch of random entities
for(var i=0; i < 20; i++){
    entity = new ECS.Entity();
    entity.addComponent( new ECS.Components.Appearance());
    entity.addComponent( new ECS.Components.Position());

    // % chance for decaying rects
    if(Math.random() < 0.8){
        entity.addComponent( new ECS.Components.Health() );
    }

    // NOTE: If we wanted some rects to not have collision, we could set it
    // here. Could provide other gameplay mechanics perhaps?
    entity.addComponent( new ECS.Components.Collision());

    entities[entity.id] = entity;
}

// PLAYER entity
// Make the last entity the "PC" entity - it must be player controlled,
// have health and collision components
entity = new ECS.Entity();
entity.addComponent( new ECS.Components.Appearance());
entity.addComponent( new ECS.Components.Position());
entity.addComponent( new ECS.Components.Collision() );
entity.addComponent( new ECS.Components.PlayerControlled() );
entity.addComponent( new ECS.Components.Health() );

// we can also edit any component, as it's just data
entity.components.appearance.colors.g = 255;
entities[entity.id] = entity;

// store reference to entities
ECS.entities = entities;
```
请注意我们可以直接修改任何组件数据。所有这些数据都可以随时随地进行操作和修改！使用组合实体（assemblages），甚至可以进一步简化玩家实体的步骤。组合实体实际上是实体模板。例如（使用我们的组合实体）：

```js
entity = new ECS.Assemblages.CollisionRect();
entity.addComponent( new ECS.Components.Health());
entity.addComponent( new ECS.Components.PlayerControlled() );
```
接下来，我们设置系统的执行顺序：
```js
// Setup systems
// Setup the array of systems. The order of the systems is likely critical, 
// so ensure the systems are iterated in the right order
var systems = [
    ECS.systems.userInput,
    ECS.systems.collision,
    ECS.systems.decay, 
    ECS.systems.render
];
```

然后来一个简单的game loop
```js

// Game loop
function gameLoop (){
    // Simple game loop
    for(var i=0,len=systems.length; i < len; i++){
        // Call the system and pass in entities
        // NOTE: One optimal solution would be to only pass in entities
        // that have the relevant components for the system, instead of 
        // forcing the system to iterate over all entities
        systems[i](ECS.entities);
    }

    // Run through the systems. 
    // continue the loop
    if(self._running !== false){
        requestAnimationFrame(gameLoop);
    }
}
// Kick off the game loop
requestAnimationFrame(gameLoop);
```

最后，添加游戏失败条件：
```js
// Lose condition
this._running = true; // is the game going?
this.endGame = function endGame(){ 
    self._running = false;
    document.getElementById('final-score').innerHTML = +(ECS.$score.innerHTML);
    document.getElementById('game-over').className = '';

    // set a small timeout to make sure we set the background
    setTimeout(function(){
        document.getElementById('game-canvas').className = 'game-over';
    }, 100);
};
```