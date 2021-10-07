## Before starting
Install http-server and run it to check the game:

```bash
npm install -g http-server

http-server
```

## Basic setup

The HTML:

```html
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <script src="https://cdn.jsdelivr.net/npm/phaser@3.15.1/dist/phaser-arcade-physics.min.js"></script>
  <title>My game</title>
</head>

<body>
  <script>
    // Code goes here
  </script>
</body>
</html>

```

The JS:

```js
var config = {
  type: Phaser.AUTO, // Phaser.CANVAS or Phaser.WEBGL or Phaser.AUTO
  width: 800,
  height: 600,
  scene: {
    preload: preload,
    create: create,
    update: update
  }
};

function preload() {}
function create() {}
function update() {}

var game = new Phaser.Game(config);
```

## Load the assets

```js
function preload() {
  this.load.image('sky', 'images/sky.png');
  this.load.image('floor', 'images/floor.png');
  this.load.image('table', 'images/table.png');
}
```

## Placing the background

```js
function create() {
  this.add.image(400, 300, 'sky'); // 400, 300 being the center of the image
  // or this.add.image(0, 0, 'sky').setOrigin(0, 0)
}
```

### Understand that the order we place objects matters

```js
  function create() {
    this.add.image(400, 300, 'sky');
    this.add.image(400, 300, 'table');
  }
```

### Add floors that the player can collide with

Before adding the floors we need to setup the physics of our game:

```js
var config = {
  ...
  physics: {
    default: 'arcade',
    arcade: {
      gravity: { y: 500 },
        debug: false
      }
    }
  }
};
```

Then we can start adding stuff:

```js
  var floors;
  // The bodies in phaser can be dynamic or static
  //
  // A dynamic body is one that can move around via forces such as velocity or acceleration
  // It can bounce and collide with other objects and that collision is influenced by the mass of
  // the body and other elements.
  //
  // A Static Body simply has a position and a size. It isn't touched by gravity, you cannot set
  // velocity on it and when something collides with it, it never moves. Static by name, static by
  // nature. And perfect for the ground and platforms that we're going to let the player run around
  // on.
  floors = this.physics.add.staticGroup();
  floors.create(600, 400, 'floor');
  floors.create(50, 250, 'floor');
  floors.create(750, 220, 'floor');
  floors.create(400, 300, 'table');
  floors.create(400, 590, 'floor'); // Bottom floor
```

## Add the player

Player will be a spritesheet. Spritesheets are images with a collection of different images for movement.

```js
// Add to preload
this.load.spritesheet('dude', 'images/dude.png', { frameWidth: 32, frameHeight: 48 });
```

```js
var player;
// In create():

player = this.physics.add.sprite(100, 450, 'dude');
player.setBounce(0.2);
player.setCollideWorldBounds(true);

this.anims.create({
  key: 'left',
  frames: this.anims.generateFrameNumbers('dude', { start: 0, end: 3 }),
  frameRate: 10,
  repeat: -1
});

this.anims.create({
  key: 'turn',
  frames: [ { key: 'dude', frame: 4 } ],
  frameRate: 20
});

this.anims.create({
  key: 'right',
  frames: this.anims.generateFrameNumbers('dude', { start: 5, end: 8 }),
  frameRate: 10,
  repeat: -1
});
```

### Make player collide

```js
this.physics.add.collider(player, floors);
```

## Movement

```js
var cursors;

// In create():
cursors = this.input.keyboard.createCursorKeys();
```

```js
function update() {
  if (cursors.left.isDown) {
    player.setVelocityX(-160);

    player.anims.play('left', true);
  }
  else if (cursors.right.isDown) {
    player.setVelocityX(160);

    player.anims.play('right', true);
  }
  else {
    player.setVelocityX(0);

    player.anims.play('turn');
  }

  if (cursors.up.isDown && player.body.touching.down) {
    player.setVelocityY(-330); // 330 px per second
  }
}
```

## Add stars

First load the star image:

```js
this.load.image('star', 'images/star.png');
```

```js
var stars;
// in create():
  //  Some stars to collect, 12 in total, evenly spaced 70 pixels apart along the x axis
  stars = this.physics.add.group({
    key: 'star',
    repeat: 11,
    setXY: { x: 12, y: 0, stepX: 70 }
  });

  // Give random bounce
  stars.children.iterate(function(child) {
    //  Give each star a slightly different bounce
    child.setBounceY(Phaser.Math.FloatBetween(0.4, 0.8));
  });
```

The stars should collide with the floors

```js
// in create():
this.physcs.add.collider(stars, floors);
```

### Player can pickup stars physics.add.overlap

```js
// in create():
this.physics.add.overlap(player, stars, collectStar, null, this);
```

Then add the function collectStar (check [disableBody](https://photonstorm.github.io/phaser3-docs/Phaser.Physics.Arcade.Image.html#disableBody__anchor) in the Phaser documentation for more info):

```js
function collectStar(player, star) {
  star.disableBody(true, true);
}
```

## New level

When all the stars are picked up we want to spawn new ones. We basically need to enable those stars again (check [enableBody](https://photonstorm.github.io/phaser3-docs/Phaser.Physics.Arcade.Image.html#enableBody__anchor) in the Phaser documentation for more info):

```js
// in collectStar():
if (stars.countActive(true) === 0) {
  stars.children.iterate(function(child) {
    child.enableBody(true, child.x, 0, true, true);
  });
}
```

## Add bombs

```js
// in preload():
this.load.image('bomb', 'images/bomb.png');
```

```js
var bombs;
var gameOver = false;

// in create():
bombs = this.physics.add.group();
bombs.create(400, 100, 'bomb'); // just to test

this.physics.add.collider(bombs, floors);

this.physics.add.overlap(player, bombs, hitBomb, null, this);
```

Then add the function hitBomb():

```js
function hitBomb(player, bomb) {
  this.physics.pause();

  player.setTint(0xff0000);
  player.anims.play('turn');
  gameOver = true;
}
```

Lastly, add a condition for game over:

```js
// in update();
if (gameOver) {
  return;
}
```


```js
// in collectStar(), inside the if (stars.countActive(true) === 0)

// Create a bomb in the half screen not containing the player
var x = (player.x < 400) ? Phaser.Math.Between(400, 800) : Phaser.Math.Between(0, 400);

var bomb = bombs.create(x, 16, 'bomb');
bomb.setBounce(1);
bomb.setCollideWorldBounds(true);
bomb.setVelocity(Phaser.Math.Between(-200, 200), 20);
bomb.allowGravity = false;
```

The game will be pretty much done! Really! Now it's all about details!

## Make the level more like a level

Let's change the placement of the platforms for a more interesting level:

```js
floors.create(400, 580, 'table');
floors.create(400, 300, 'floor').setScale(0.2, 1).refreshBody();
floors.create(-300, 400, 'floor');
floors.create(1000, 400, 'floor');
```

## Add sound

```js
// in preload():
this.load.audio('music', 'sound/music.mp3');
this.load.audio('jump', 'sound/jump.mp3');
this.load.audio('collect', 'sound/collect.mp3');
this.load.audio('dying', 'sound/dying.mp3');
this.load.audio('nextlevel', 'sound/threetone.ogg');
this.load.audio('explosion', 'sound/explosion.mp3');
```

Then in create():

```js
// in create():
this.jumpSound = this.sound.add('jump');
this.collectSound = this.sound.add('collect');
this.dyingSound = this.sound.add('dying');
this.nextLevelSound = this.sound.add('nextlevel');
this.explosionSound = this.sound.add('explosion');

// By the end of the method:
this.music = this.sound.add('music');
this.music.play();
```

Also add all the other sounds where it makes sense to add them:

```js
this.jumpSound.play();
this.collectSound.play();
this.dyingSound.play();
this.nextLevelSound.play();
this.explosionSound.play();
```

## Setup some score

```js
// in create():
var gameFont = {
  fontSize: '30px',
  fontFamily: 'Georgia, "Goudy Bookletter 1911", Times, serif'
};

this.score = 0;
this.scoreText = this.add.text(10, 10, 'Score: 0', gameFont);
```

Then, when the player picks up a star:

```js
this.score += 10;
this.scoreText.setText(`Score: ${this.score}`);
```

## Game over text


```js
this.gameOverText = this.add.text(240, 150, '', gameFont);
this.gameOverText.setFontSize(50);
this.gameOverText.setTint(0xff0000, 0xff0000, 0xffaa00, 0xffaa00);
```

Then, on explosion:

```js
this.gameOverText.setText('GAME OVER');
```
