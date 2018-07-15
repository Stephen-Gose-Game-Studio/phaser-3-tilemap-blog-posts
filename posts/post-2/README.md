# Modular Game Worlds in Phaser 3 (Tilemaps #2)

**CREDITS!**

Author: [Mike Hadley](https://www.mikewesthad.com/)

Reading this on GitHub? Check out the Medium Post (_coming soon..._)

This is a series of blog posts about creating modular worlds with tilemaps in the [Phaser 3](http://phaser.io/) game engine. If you haven't, check out the first [post](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-1-958fc7e6bbd6) where we used static tilemaps to create a Pokémon-style game world. In this post, we'll dive into dynamic tilemaps and create a puzzle-y platformer where you can draw platforms to help get around obstacles:

![](./images/paint-platformer-demo-optimized.gif)

_↳ Final example that we'll create_

In the next two posts in the series, we'll create a procedural dungeon with dynamic tilemaps and integrate [Matter.js](http://brm.io/matter-js/) to create a wall-jumping platformer.

Before we dive in, all the code and assets that go along with this post can be found in [this repository](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/tree/master/examples/post-2).

## Background Knowledge

As with the last post, this post will make the most sense if you have some experience with JavaScript, Phaser and the [Tiled](https://www.mapeditor.org/) map editor. If you don't, you might want to start at the beginning of the [series](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-1-958fc7e6bbd6), or follow along and check keep the [Phaser tutorial](https://phaser.io/tutorials/making-your-first-phaser-3-game) & Google handy.

## The Tilemap API

We covered the following parts of the Phaser Tilemap API last time:

- [`Tilemap`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tilemap.html)
- [`Tileset`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tileset.html)
- [`StaticTilemapLayer`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.StaticTilemapLayer.html)

In this post, we'll dive into two new pieces of the API:

- [`DynamicTilemapLayer`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html)
- [`Tile`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tile.html)

When you use [`this.make.tilemap`](https://photonstorm.github.io/phaser3-docs/Phaser.GameObjects.GameObjectCreator.html#tilemap) (or [`this.add.tilemap`](https://photonstorm.github.io/phaser3-docs/Phaser.GameObjects.GameObjectFactory.html#tilemap)) in a scene, you get a `Tilemap`. This isn't a display object. It holds data about the map and allows you to add tilesets & tilemap layers.

A map can have one or more layers, which are the display objects that actually render tiles from a `Tileset`. They come in two flavors: `StaticTilemapLayer` & `DynamicTilemapLayer`. Each has a 2D array of `Tile` objects that it renders. A `StaticTilemapLayer` is super fast, but the tiles in that layer can't be modified. A `DynamicTilemapLayer` trades some speed for the flexibility and power of manipulating individual tiles.

Static and dynamic tilemaps share much of the same API. They both have methods for checking whether a tile exists (`hasTileAt(...)`, etc.). They both have methods for getting access to tiles in the map (`getTileAt(...)`,`findTile(...)`, `forEachTile(...)`, etc.). The key difference is that dynamic layers can be manipulated after being created, whereas static layers can't. Dynamic layers have a set of addition methods for adding, removing, randomizing, etc. tiles within the layer (e.g. `putTileAt(...)`, `removeTileAt(...)`, `randomize(...)`, etc.).

You can mix static and dynamic layers together in the same map. You can load up the non-changing background of your world as a static layer while loading up a layer with chests, coins, keys and the like in a dynamic layer.

You can also convert a dynamic layer into a static layer, allowing you to generate a level on the fly and then optimize it.

## Painting Tiles

For the first example, we'll dive into making the humble beginnings of a level editor. We will load up a level made with Tiled and then paint & erase tiles from a layer dynamically:

![](./images/paint-tiles-demo.gif)

You set up dynamic layers in the same way as static layers, except using [`map.createDynamicLayer`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tilemap.html#createDynamicLayer__anchor) instead of [`map.createStaticLayer`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tilemap.html#createStaticLayer__anchor):

```js
let groundLayer;

function preload() {
  this.load.image("tiles", "../assets/tilesets/0x72-industrial-tileset-32px-extruded.png");
  this.load.tilemapTiledJSON("map", "../assets/tilemaps/platformer.json");
}

function create() {
  const map = this.make.tilemap({ key: "map" });
  const tiles = map.addTilesetImage("0x72-industrial-tileset-32px-extruded", "tiles");

  map.createDynamicLayer("Background", tiles);
  groundLayer = map.createDynamicLayer("Ground", tiles);
  map.createDynamicLayer("Foreground", tiles);
}
```

Once you've got a dynamic layer loaded up, you can start manipulating tiles using the [DynamicTilemapLayer API](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html):

```js
// Put tile index 1 at tile grid location (20, 10) within layer
groundLayer.putTileAt(1, 20, 10);

// Put tile index 2 at world pixel location (200, 50) within layer
// (This uses the main camera's coordinate system by default)
groundLayer.putTileAtWorldXY(2, 200, 50);
```

The tilemap layer (and tilemap) methods that get or manipulate tiles often come in pairs. One method - like [`putTileAt`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html#putTileAt__anchor) - will operate on tile grid units, e.g. (0, 2) would correspond to the first column and third row. Another method - like [`putTileAtWorldXY`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html#putTileAtWorldXY__anchor) - will operate in world pixel units, making it easier to do things like find which tile is under the mouse. There are also methods for converting from tile grid units to world pixel coordinates and vice versa: [`worldToTileXY`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html#worldToTileXY__anchor), [`tileToWorldXY`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html#tileToWorldXY__anchor).

Putting these methods together with Phaser input, we can draw tiles in a layer with the mouse:

```js
function update() {
  // Convert the mouse position to world position within the camera
  const worldPoint = this.input.activePointer.positionToCamera(this.cameras.main);

  // Draw or erase tiles (only within the groundLayer)
  if (this.input.manager.activePointer.isDown) {
    groundLayer.putTileAtWorldXY(353, worldPoint.x, worldPoint.y);
  }
}
```

The following example puts all of this together and allows you to paint tiles with by clicking/tapping and erase tiles clicking while holding shift. `worldToTileXY` & `tileToWorldXY` are used to create a simple graphic overlay to visualize which tile the mouse is currently over.

Note: you'll want to click on the "Edit on CodeSandbox" button and check out the code in full screen where you can see all the files easily.

[![Edit Phaser Tilemap Post 2: 01-drawing-tiles](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/31xpvv85om?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1)

<!-- Embed link for medium: https://codesandbox.io/s/31xpvv85om?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1 -->

_↳ Check out the [CodeSandbox](https://codesandbox.io/s/31xpvv85om?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1), [live example](https://www.mikewesthad.com/phaser-3-tilemap-blog-posts/post-2/01-drawing-tiles) or the source code [here](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/blob/master/examples/post-2/01-drawing-tiles)._

## Modularizing Our Code

Adding or removing individual tiles is pretty easy, so let's step up the complexity and build the basis of a platformer:

![](./images/platformer-demo-optimized.gif)

While the tilemap-specific part of the demo code is pretty short, there's a bit more that goes into making it a functional demo. Thus far, we've gotten up and running fast with Phaser using a single file that has `preload`, `setup` and `update` functions. That's great for simple examples, but becomes a nightmare once you get to anything moderately complex.

We want to split up that monolithic file into easier-to-digest, isolated files called "modules." There are a lot of reasons go modular with your code. If used well, they help create portable & reusable chunks of code that are easier to think about. If you are using a modern browser (roughly anything late 2017 onward), you can use modules in your JS (without needing webpack, parcel, etc.) like this:

```html
<script src="./js/index.js" type="module"></script>
```

Note: you won't see this in the CodeSandbox demos since they use the [Parcel bundler](https://parceljs.org/) to enable module support, but you will see it in the source code for this series in [this repository](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/tree/master/examples/post-2).

Inside of index.js, you can now use [`import`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import) functions, objects, or primitive values from other files that have an [`export`](https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/export). `import` and `export` provide ways for us to split our single file code into separate files. If you aren't familiar with modules, check out the [modules chapter](https://eloquentjavascript.net/10_modules.html) from _Eloquent JavaScript_ or [this overview](https://blog.cloud66.com/an-overview-of-es6-modules-in-javascript/).

Here's what our new project structure looks like:

![](./images/directory-structure.png)

```
.
├── assets/
|
├── js/
|  |
|  ├── index.js
|  |     Creates the Phaser game from our config
|  |
|  ├── player.js
|  |     Handles player momvement and animations
|  |
|  └── platformer-scene.js
|        The new home of our scene (e.g. preload, create, update)
|
└── index.html
      Loads up index.js
```

Check out the code below, starting with index.js. From there, when you see an `import`, look at the file that's being referenced to follow the thread. This will be the basis we build upon for the next section.

[![Edit Phaser Tilemap Post 2: 02-modules-demo](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/p5pqqjk6q0?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1)

<!-- Embed link for medium: https://codesandbox.io/s/p5pqqjk6q0?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1 -->

_↳ Check out the [CodeSandbox](https://codesandbox.io/s/31xpvv85om?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1), [live example](https://www.mikewesthad.com/phaser-3-tilemap-blog-posts/post-2/02-modules-demo) or the source code [here](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/blob/master/examples/post-2/02-modules-demo)._

There's a lot more that we could do to make this code more modular (e.g. taking advantage of Phaser's event system), but this is modular enough without introducing too many new concepts.

## The Platformer

Whew, we've got a template that we can build upon now. Let's turn back to the idea of a platformer where you can draw tiles to get around obstacles:

![](./images/paint-platformer-demo-optimized.gif)

There are two important tilemap-specific extensions to the code that we should break down:

1.  Painting colliding tiles
2.  Adding the spikes with proper hitboxes

First up, painting tiles. This is like the first example, but with the added wrinkle is that we want
the tiles that we add to be colliding, so that the player can land on them. `putTileAtWorldXY` will
return the [`Tile`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tile) object that we
just manipulated. `Tile` objects are pretty simple. They hold the index and position of the tile
that they render, along with some physics information. We can use `tile.setCollision` to enable collisions:

```js
const pointer = this.input.activePointer;
const worldPoint = pointer.positionToCamera(this.cameras.main);
if (pointer.isDown) {
  const tile = groundLayer.putTileAtWorldXY(6, worldPoint.x, worldPoint.y);
  tile.setCollision(true);
}
```

Tiles have other useful properties & methods. `tile.x` and `tile.y` are the position in grid units. `tile.getLeft()`, `tile.getBottom()`, `tile.getCenterX()`, etc. will give you the position in world pixel units. Check out the [docs](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tile) for more information.

Perfect, we can paint tiles now. But we've got a spike problem:

![](./images/paint-with-spike-problem-optimized.gif)

One of the arcade physics (AP) limits in Phaser is that the physics body of colliding tiles is forced to be a rectangle that matches the tile width and tile height. Our spike is only 6px tall, but it's given a 32px x 32px hitbox. If we load up the map and render the colliding tiles using `this.groundLayer.renderDebug`, the issue is more apparent:

![](./images/spike-tile-collision-debug.gif)

There are a few ways to solve this. We could switch to using Matter.js for physics, but that's overkill. Instead, let's turn the spikes from tiles into sprites, which we can give custom sized physics bodies. (Which also gives us a convenient excuse to look deeper at the tilemap API.)

Tilemaps have methods for turning Tiled objects and tiles into sprites: [`createFromObjects`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tilemap.html#createFromObjects__anchor) & [`createFromTiles`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.Tilemap.html#createFromTiles__anchor) respectively. You can these to visually lay out where your game entities should be - e.g. place objects in the location of enemies in your level and then turn them into proper sprites. Here's an [example](https://labs.phaser.io/edit.html?src=src\game%20objects\tilemap\static\create%20from%20objects.js&v=3.9.0) breakdown that uses this strategy to place animated coins from Phaser:

![](./images/object-to-sprite/object-to-sprite-annotated.gif)

_↳ Left side is Phaser and right side is Tiled. Note how width/height/flip etc. are copied over to the sprite._

In the context of our code, turning spikes into sprites is a little more complicated, so we can roll our own custom version of tile to sprite logic by looping over all the `Tile` objects in a layer using [`forEachTile`](https://photonstorm.github.io/phaser3-docs/Phaser.Tilemaps.DynamicTilemapLayer.html#forEachTile__anchor):

```js
// Create a physics group - useful for colliding the player against all the spikes
this.spikeGroup = this.physics.add.staticGroup();

// Loop over each Tile and replace spikes with custom sprites
this.groundLayer.forEachTile(tile => {
  // The spike index is 77
  if (tile.index === 77) {
    // A sprite has its origin at the center, so place the sprite at the center of the tile
    const x = tile.getCenterX();
    const y = tile.getCenterY();
    const spike = this.spikeGroup.create(x, y, "spike");

    // The map has spike tiles that have been rotated in Tiled ("z" key), so parse out that angle
    // to the correct body placement
    spike.rotation = tile.rotation;
    if (spike.angle === 0) spike.body.setSize(32, 6).setOffset(0, 26);
    else if (spike.angle === -90) spike.body.setSize(6, 32).setOffset(26, 0);
    else if (spike.angle === 90) spike.body.setSize(6, 32).setOffset(0, 0);

    // And lastly, remove the spike tile from the layer
    this.groundLayer.removeTileAt(tile.x, tile.y);
  }
});
```

And we'll end up with nice hitboxes like:

![](./images/spike-sprite-collision-debug.gif)

Now for everything in context, where we reset the game when the player touches the spikes:

[![Edit Phaser Tilemap Post 2: 03-drawing-platformer](https://codesandbox.io/static/img/play-codesandbox.svg)](https://codesandbox.io/s/mo2j4nvkxy?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1)

<!-- Embed link for medium: https://codesandbox.io/s/mo2j4nvkxy?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1 -->

_↳ Check out the [CodeSandbox](https://codesandbox.io/s/mo2j4nvkxy?hidenavigation=1&module=%2Fjs%2Findex.js&moduleview=1), [live example](https://www.mikewesthad.com/phaser-3-tilemap-blog-posts/post-2/03-drawing-platformer) or the source code [here](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/blob/master/examples/post-2/03-drawing-platformer)._