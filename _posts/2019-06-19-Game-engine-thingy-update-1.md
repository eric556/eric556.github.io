---
layout: post
title:  "Game engine thingy update 1"
date:   2019-06-19 13:29:00 +1200
categories: dev update
---

Hi,

For the past 2 weeks I have been procrastinating studying for finals. To facilitate my procrastination I decided to start working on a small game. I picked up my trusty tools, C++ and [SFML](https://www.sfml-dev.org/), and started working. Before I even got to the fun part, actually making the game, I got bogged down by the amount of setup work there is. This got me thinking, what if I made a codebase that I could reuse for some other projects in the future so I can skip all of the bullshit setup work that I run through every time I try to make something. So I started on my journy to build myself a small game engine. Before jumping in I looked around at other engines to see what I liked and decided on a set of feature that I wanted within my engine. The three most important things I came up with were: ECS structure, plenty of tools to make development faster, and lua scripting integration.

### Premake5
Now before I jump into all of those fancy feature and how I have gone about implementing them in the last 2 weeks. I need to talk a little about how I set this project up with the help of an amazing tool called premake. Premake is a tool that will generate project files for you based on a configuration script written in lua.

{% highlight lua %}
workspace "Game Engine Thingy"
    location "Projects"
    language "C++"
    architecture "x86_64"

    configurations {"Debug", "Release"}

    filter{"configurations:Debug"}
        symbols "On"

    filter{"configurations:Release"}
        optimize "On"

    filter{}

    targetdir("Build/Bin/%{prj.name}/%{cfg.longname}")
    objdir("Build/Obj/%{prj.name}/%{cfg.longname}")

project "ECS"
    location "Projects/ECS"
    kind "StaticLib"

    files "Projects/ECS/*.cpp"
    files "Projects/ECS/*.hpp"
    files "Projects/ECS/*.h"

    files "Projects/ECS/*/*.cpp"
    files "Projects/ECS/*/*.hpp"
    files "Projects/ECS/*/*.h"

project "Application"
    location "Projects/Application"
    kind "ConsoleApp"
    cppdialect "C++17"

    files "Projects/Application/src/*.cpp"
    files "Projects/Application/src/*.h"
    files "Projects/Application/src/*.hpp"

    files "Projects/Application/src/*/*.cpp"
    files "Projects/Application/src/*/*.h"
    files "Projects/Application/src/*/*.hpp"

    files "Projects/Application/src/*/*/*.cpp"
    files "Projects/Application/src/*/*/*.h"
    files "Projects/Application/src/*/*/*.hpp"

    includeSFML()
    linkSFML()

    includeIMGUI()
    linkIMGUI()

    includeLua()
    linkLua()

    includeSOL2()

    useECS()
{% endhighlight %}

This is an example of a premake lua file. Using this file I can describe my projects structure and link libraries into my project without having to deal with make files or visual studios clunky GUI. From this file premake is able to generate visual studio solutions, make files, xcode projects and a whole host of other project file types. Linking libraries was always one of the worst parts of setting up a new project for me. Premake makes it as simple as writing a lua function.

### ECS
ECS stands for Entity Component System which is a design pattern commonly used in games. I won't really go in depth on this because I really don't know what I'm doing. There are a ton of articles out on the internet that explain how it works and how to make it super efficient (seriously this pattern can become crazy efficient). I'll just go over why it is useful and how I went about implementing it.

Say you are making a game and you are using standard OOP to create a class structure for your game objects. You have an item class that is a parent to all items likes weapons and potions both of which are parents to other classes like swords and health potions. You end up with a massive hierarchy of classes which isn't terrible, thats what OOP is for right? Now say you had another class called enemy. That class has a bunch of children that have their own children and you get another hierarchy of classes. Everything is fine until you come up with the idea to make an evil sword that is both an item and an enemy. What do you do now? You can't inherit from both classes. Might as well set the repo on fire and start from scratch. Or at least thats what it feels like when you need to restructure everything for one item.

ECS resolves this problem by completely removing the class hierarchy. Instead ECS relies on three things (I bet you wont be able to guess). Entities, Components, and Systems. Entities are your game objects from the example above. This can be something like a health potion or a sword or a tree. It can even be the game camera or GUI elements. Really an entity is just anything within your game. Entities, instead of being described by a class hierarchy, are described by their components. Components in their simplest form are just sets of data (think of a c struct). So a sword entity might have an item component that is described like so

{% highlight c++ %}
struct ItemComponent{
    char* itemName;
    unsigned int itemId;
    unsigned short rarity; 
}
{% endhighlight %}

So a sword would have this `ItemComponent` attached to it. In addition to that it could have an `EnemyAiComponent` attached to it that could describe how it would work against you. Bam! you have yourself an evil sentient sword without the need to rip apart your class hierarchy. Now you might be wondering how these components will interact if they are just structs holding data. That is were the System part comes in. The systems "do work" on components that are relevant to them. Take these two components for example:

{% highlight c++ %}
// sf namespace is from SFML
struct TransformComponent{
    sf::Vector3f position;
    sf::Vector2f scale;
    float rotation
};

struct KineticBodyComponent{
    sf::Vector3f velocity;
    sf::Vector3f acceleration;
    sf::Vector3f forceAccumulator;
    float mass;
};
{% endhighlight %}

A physics system would find all the entities with a `TransformComponent` and a `KineticBodyComponent` and do some math operations to update the entities position based on the forces accumulated.

That is the basics of ECS. If you are interested in seeing how I implemented it head over to my github repo for this project. [I warn you, the code is messy](https://github.com/eric556/sga).

### Tools
One of the most important feature I noticed when exploring other engines was the plethora of tools that help in development. From scene explorers to being able to edit values of entities while the game was running every engine had a ton of tools. I decided that I wanted some of these tools in my own engine. To create these tools I am using [ImGui](https://github.com/ocornut/imgui) with the config files for SFML created by [Elias Daler](https://github.com/eliasdaler/imgui-sfml) (this guy is awesome and has his own [blog](https://eliasdaler.github.io/) doing what I'm doing but significantly better).

#### Entity Manager
I wanted to be able to see all of the entities in my scene as well as edit their component values while the game is running. 

<iframe src='https://gfycat.com/ifr/assuredremotearcticfox' frameborder='0' scrolling='no' allowfullscreen width='640' height='453'></iframe>


#### Lua Console
One of the big features I wanted was the ability to run lua scripts. In addition to this I thought it would be nice to be able to run lua in the game while it was running. So I made a lua "Console".

<iframe src='https://gfycat.com/ifr/damagedcornycoyote' frameborder='0' scrolling='no' allowfullscreen width='640' height='453'></iframe>

### Lua
One of biggest features I wanted to have was a scripting language that allowed me to make changes to the game without needing to recompile the entire engine. I dedcided to use lua because there are some [really nice libraries](https://github.com/ThePhD/sol2) out there that abstract luas c bindings into more friendly c++ syntax. Figuring out how to structure my engine around this has been a total pain in the ass. This feature is still very much in progress but as of right now I am able to create a script with some basic game logic that gets called from the c++ engine

{% highlight lua %}
require("entities/player")
require("entities/pokemon_player")
require("tilemaps/tilemap")
require("tilemaps/pokemon_tilemap")
require("tilemaps/dungeon_tilemap")
require("entities/house")
require("entities/misc")

function magnitude(vec)
    return math.sqrt(vec.x * vec.x + vec.y * vec.y + vec.z + vec.z)
end

function movePlayer(dt)
    move = Vector3.new(0, 0, 0)

    if(getKeyDown(22)) then
        move = move + Vector3.new(0, -1, 0)
    elseif(getKeyDown(18)) then
        move = move + Vector3.new(0, 1, 0)
    elseif(getKeyDown(0)) then
        move = move + Vector3.new(-1, 0, 0)
    elseif(getKeyDown(3)) then
        move = move + Vector3.new(1, 0, 0)
    end

    if (move.x > 0 and move.y == 0) then
        playerSprite:setCurrentAnimation("right");
    elseif (move.x < 0 and move.y == 0) then
        playerSprite:setCurrentAnimation("left");
    elseif (move.x == 0 and move.y < 0) then
        playerSprite:setCurrentAnimation("up");
    elseif (move.x == 0 and move.y > 0) then
        playerSprite:setCurrentAnimation("down");
    elseif (move.x > 0 and move.y > 0) then
        -- playerSprite:setCurrentAnimation("down_right");
    elseif (move.x > 0 and move.y < 0) then
        -- playerSprite:setCurrentAnimation("up_right");
    elseif (move.x < 0 and move.y > 0) then
        -- playerSprite:setCurrentAnimation("down_left");
    elseif (move.x < 0 and move.y < 0) then
        -- playerSprite:setCurrentAnimation("up_left");
    else
        playerSprite:setCurrentAnimation("idle");
    end

    move = Normalize(move) * playerInput.speed * dt
    playerTransform.position = playerTransform.position + move
end

function load()
    player = PKPlayer.new()
    playerTransform = player:getTransform()
    playerSprite = player:getAnimatedSprite()
    playerInput = player:getInput()
    time = 0
    -- tilemap = makeTilemap()
    tilemap = Tilemap.load(pk_tilemap)
    -- tilemap2 = Tilemap.load(dungeon_tilemap)
    Buildings.Houses.Blue.new(Vector3.new(531, 282, 0))
    Buildings.Houses.Blue.new(Vector3.new(187, 282, 0))
    Misc.Mailbox.new(Vector3.new(633, 509, 0))
    Misc.Mailbox.new(Vector3.new(287, 509, 0))
end

function update(dt)
    movePlayer(dt)
end

function draw(dt)
    drawTilemap()
    drawSprite()
    drawAnimated(dt)
    drawShape()
end
{% endhighlight %}

Now some of you might be versed in lua and might be wondering where things like `entityManager` came from and all of these weird `add<Name of component>` methods are from. The cool thing about lua is that you can expose c++ methods to it. So all of those function calls that look like they are undefined are defined in C++ and exposed by the engine. The engine then runs the load method once and then calls the update and draw methods every frame.

One of the other cool things about having lua scripting is the ability to create Prefabs. Prefabs are just entities with a predefined set of components and component values. Lets take a look at the misc prefab script: 

{% highlight lua %}
Misc = {
    Mailbox = {},
    Lightpost = {}
}

function Misc.Mailbox.new(position)
    local entity = entityManager:createEntity()
    entity.name = "Mailboxs " .. entity.id
    local transform = entity:addTransform()
    transform.scale = Vector2.new(3, 3)
    if position ~= nil then
        transform.position = position
    end
    local sprite = entity:addSprite()
    sprite.texture_id = "pokemon_sheet"
    sprite.useTextureRect = true
    sprite.textureRect = IntRect.new(32, 128, 16, 32)
    
    return entity
end

function Misc.Lightpost.new(position)
    local entity = entityManager:createEntity()
    entity.name = "Lightpost " .. entity.id
    local transform = entity:addTransform()
    transform.scale = Vector2.new(3, 3)
    if position ~= nil then
        transform.position = position
    end
    local sprite = entity:addSprite()
    sprite.texture_id = "pokemon_sheet"
    sprite.useTextureRect = true
    sprite.textureRect = IntRect.new(240, 432, 16, 32)
    
    return entity 
end
{% endhighlight %}

Looking at the code above you can see that the prefabs are defined within their own `new` functions. So if I wanted to create a new lightpost I could run the lua script: 

{% highlight lua %}
Misc.Lightpost.new(Vector3.new(100, 100, 0))
{% endhighlight %}

and I would get a new Lightpost in the scene. Being able to create prefabs like this allows for easy scene definition. A scene could just be defined as a lua table like so

{% highlight lua %}
scene = {
    tilemap = Tilemap.load(pk_tilemap),
    Buildings.Houses.Blue.new(Vector3.new(531, 282, 0)),
    Buildings.Houses.Blue.new(Vector3.new(187, 282, 0)),
    Misc.Mailbox.new(Vector3.new(633, 509, 0)),
    Misc.Mailbox.new(Vector3.new(287, 509, 0))
}
{% endhighlight %}

### The End
If you made it this far you deserve a medal. Thanks for reading my ramblings. Check the repo out on [github](https://github.com/eric556/sga) and check back for updates, if I remember to post any.