# Code tự nhảy:

var IS_IOS = /iPad|iPhone|iPod/.test(window.navigator.platform);

function getTimeStamp() {
  return IS_IOS ? new Date().getTime() : performance.now();
}

function boxCompare(tRexBox, obstacleBox) {
  var crashed = false;
  var tRexBoxX = tRexBox.x;
  var tRexBoxY = tRexBox.y;

  var obstacleBoxX = obstacleBox.x;
  var obstacleBoxY = obstacleBox.y;

  // Axis-Aligned Bounding Box method.
  if (
    tRexBox.x < obstacleBoxX + obstacleBox.width &&
    tRexBox.x + tRexBox.width > obstacleBoxX &&
    tRexBox.y < obstacleBox.y + obstacleBox.height &&
    tRexBox.height + tRexBox.y > obstacleBox.y
  ) {
    crashed = true;
  }

  return crashed;
}

function CollisionBox(x, y, w, h) {
  this.x = x;
  this.y = y;
  this.width = w;
  this.height = h;
}

TRexCollisionBoxes = {
  DUCKING: [new CollisionBox(1, 18, 55, 25)],
  RUNNING: [
    new CollisionBox(22, 0, 17, 16),
    new CollisionBox(1, 18, 30, 9),
    new CollisionBox(10, 35, 14, 8),
    new CollisionBox(1, 24, 29, 5),
    new CollisionBox(5, 30, 21, 4),
    new CollisionBox(9, 34, 15, 4)
  ]
};

TrexAnimFrames = {
  WAITING: {
    frames: [44, 0],
    msPerFrame: 1000 / 3
  },
  RUNNING: {
    frames: [88, 132],
    msPerFrame: 1000 / 12
  },
  CRASHED: {
    frames: [220],
    msPerFrame: 1000 / 60
  },
  JUMPING: {
    frames: [0],
    msPerFrame: 1000 / 60
  },
  DUCKING: {
    frames: [262, 321],
    msPerFrame: 1000 / 8
  }
};

function createAdjustedCollisionBox(box, adjustment) {
  return new CollisionBox(
    box.x + adjustment.x,
    box.y + adjustment.y,
    box.width,
    box.height
  );
}

function checkForCollision(obstacle, tRex, opt_canvasCtx) {
  var obstacleBoxXPos = Runner.defaultDimensions.WIDTH + obstacle.xPos;

  // Adjustments are made to the bounding box as there is a 1 pixel white
  // border around the t-rex and obstacles.
  var tRexBox = new CollisionBox(
    tRex.xPos + 1,
    tRex.yPos + 1,
    tRex.config.WIDTH - 2,
    tRex.config.HEIGHT - 2
  );

  var obstacleBox = new CollisionBox(
    obstacle.xPos + 1,
    obstacle.yPos + 1,
    obstacle.typeConfig.width * obstacle.size - 2,
    obstacle.typeConfig.height - 2
  );

  // Debug outer box
  if (opt_canvasCtx) {
    drawCollisionBoxes(opt_canvasCtx, tRexBox, obstacleBox);
  }

  // Simple outer bounds check.
  if (boxCompare(tRexBox, obstacleBox)) {
    var collisionBoxes = obstacle.collisionBoxes;
    var tRexCollisionBoxes = tRex.ducking
      ? TRexCollisionBoxes.DUCKING
      : TRexCollisionBoxes.RUNNING;

    // Detailed axis aligned box check.
    for (var t = 0; t < tRexCollisionBoxes.length; t++) {
      for (var i = 0; i < collisionBoxes.length; i++) {
        // Adjust the box to actual positions.
        var adjTrexBox = createAdjustedCollisionBox(
          tRexCollisionBoxes[t],
          tRexBox
        );
        var adjObstacleBox = createAdjustedCollisionBox(
          collisionBoxes[i],
          obstacleBox
        );
        var crashed = boxCompare(adjTrexBox, adjObstacleBox);

        // Draw boxes for debug.
        if (opt_canvasCtx) {
          drawCollisionBoxes(opt_canvasCtx, adjTrexBox, adjObstacleBox);
        }

        if (crashed) {
          return [adjTrexBox, adjObstacleBox];
        }
      }
    }
  }
  return false;
}

Runner.instance_.__proto__.update = function() {
  this.updatePending = false;

  var now = getTimeStamp();
  var deltaTime = now - (this.time || now);
  this.time = now;

  if (this.playing) {
    this.clearCanvas();

    if (this.tRex.jumping) {
      this.tRex.updateJump(deltaTime);
    }

    this.runningTime += deltaTime;
    var hasObstacles = this.runningTime > this.config.CLEAR_TIME;

    // First jump triggers the intro.
    if (this.tRex.jumpCount == 1 && !this.playingIntro) {
      this.playIntro();
    }

    // The horizon doesn't move until the intro is over.
    if (this.playingIntro) {
      this.horizon.update(0, this.currentSpeed, hasObstacles);
    } else {
      deltaTime = !this.activated ? 0 : deltaTime;
      this.horizon.update(
        deltaTime,
        this.currentSpeed,
        hasObstacles,
        this.inverted
      );
    }

    // Predict
    for (let i = 0; i < this.horizon.obstacles.length; i++) {
      var needJump = false;
      var predictLength = this.currentSpeed * 10;
      var pushLength = this.currentSpeed;
      var distance;
      for (var step = 1; step < predictLength; step += 0.1) {
        this.tRex.xPos += step;
        if (this.tRex.jumping) {
          var msPerFrame = TrexAnimFrames[this.tRex.status].msPerFrame;
          var framesElapsed = step / msPerFrame;
          // Speed drop makes Trex fall faster.
          if (this.tRex.speedDrop) {
            this.tRex.yPos += Math.round(
              this.tRex.jumpVelocity *
                this.tRex.config.SPEED_DROP_COEFFICIENT *
                framesElapsed
            );
          } else {
            this.tRex.yPos += Math.round(
              this.tRex.jumpVelocity * framesElapsed
            );
          }
        }

        var collision =
          hasObstacles &&
          checkForCollision(this.horizon.obstacles[i], this.tRex);

        this.tRex.xPos -= step;
        if (this.tRex.jumping) {
          var msPerFrame = TrexAnimFrames[this.tRex.status].msPerFrame;
          var framesElapsed = step / msPerFrame;
          // Speed drop makes Trex fall faster.
          if (this.tRex.speedDrop) {
            this.tRex.yPos -= Math.round(
              this.tRex.jumpVelocity *
                this.tRex.config.SPEED_DROP_COEFFICIENT *
                framesElapsed
            );
          } else {
            this.tRex.yPos -= Math.round(
              this.tRex.jumpVelocity * framesElapsed
            );
          }
        }

        if (collision) {
          distance = step;
          needJump = true;
          break;
        }
      }
      if (needJump) {
        if (distance <= pushLength) this.horizon.obstacles[i].meetTRex = true;
        else {
          if (!this.tRex.ducking) this.tRex.startJump(this.currentSpeed);
        }
      }
    }

    // Check for collisions.
    var collision =
      hasObstacles && checkForCollision(this.horizon.obstacles[0], this.tRex);

    if (!collision) {
      this.distanceRan += (this.currentSpeed * deltaTime) / this.msPerFrame;

      if (this.currentSpeed < this.config.MAX_SPEED) {
        this.currentSpeed += this.config.ACCELERATION;
      }
    } else {
      this.gameOver();
    }

    var playAchievementSound = this.distanceMeter.update(
      deltaTime,
      Math.ceil(this.distanceRan)
    );

    if (playAchievementSound) {
      this.playSound(this.soundFx.SCORE);
    }

    // Night mode.
    if (this.invertTimer > this.config.INVERT_FADE_DURATION) {
      this.invertTimer = 0;
      this.invertTrigger = false;
      this.invert();
    } else if (this.invertTimer) {
      this.invertTimer += deltaTime;
    } else {
      var actualDistance = this.distanceMeter.getActualDistance(
        Math.ceil(this.distanceRan)
      );

      if (actualDistance > 0) {
        this.invertTrigger = !(actualDistance % this.config.INVERT_DISTANCE);

        if (this.invertTrigger && this.invertTimer === 0) {
          this.invertTimer += deltaTime;
          this.invert();
        }
      }
    }
  }

  if (
    this.playing ||
    (!this.activated && this.tRex.blinkCount < Runner.config.MAX_BLINK_COUNT)
  ) {
    this.tRex.update(deltaTime);
    this.scheduleNextUpdate();
  }
};

# Code được max điểm:

Runner.instance_.distanceMeter.setHighScore(3999950);

# Code phóng to, thu nhỏ con khủng long:

scale = 2; // Sửa tỉ lệ tại đây
Runner.instance_.tRex.config.HEIGHT_REAL = Runner.instance_.tRex.config.HEIGHT;
Runner.instance_.tRex.config.WIDTH_REAL = Runner.instance_.tRex.config.WIDTH;
Runner.instance_.tRex.config.WIDTH_DUCK_REAL =
  Runner.instance_.tRex.config.WIDTH_DUCK;
Runner.instance_.tRex.config.HEIGHT *= scale;
Runner.instance_.tRex.config.WIDTH *= scale;
Runner.instance_.tRex.config.HEIGHT_DUCK *= scale;
Runner.instance_.tRex.config.WIDTH_DUCK *= scale;
Runner.instance_.tRex.yPos -=
  Runner.instance_.tRex.config.HEIGHT -
  Runner.instance_.tRex.config.HEIGHT_REAL;
Runner.instance_.tRex.groundYPos -=
  Runner.instance_.tRex.config.HEIGHT -
  Runner.instance_.tRex.config.HEIGHT_REAL;
Runner.instance_.tRex.__proto__.draw = function(x, y) {
  var sourceX = x;
  var sourceY = y;
  var sourceWidth =
    this.ducking && this.status != "CRASHED"
      ? this.config.WIDTH_DUCK_REAL
      : this.config.WIDTH_REAL;
  var sourceHeight = this.config.HEIGHT_REAL;

  if (window.devicePixelRatio > 1) {
    sourceX *= 2;
    sourceY *= 2;
    sourceWidth *= 2;
    sourceHeight *= 2;
  }

  // Adjustments for sprite sheet position.
  sourceX += this.spritePos.x;
  sourceY += this.spritePos.y;

  // Ducking.
  if (this.ducking && this.status != "CRASHED") {
    this.canvasCtx.drawImage(
      Runner.imageSprite,
      sourceX,
      sourceY,
      sourceWidth,
      sourceHeight,
      this.xPos,
      this.yPos,
      this.config.WIDTH_DUCK,
      this.config.HEIGHT
    );
  } else {
    // Crashed whilst ducking. Trex is standing up so needs adjustment.
    if (this.ducking && this.status == "CRASHED") {
      this.xPos++;
    }
    // Standing / running
    this.canvasCtx.drawImage(
      Runner.imageSprite,
      sourceX,
      sourceY,
      sourceWidth,
      sourceHeight,
      this.xPos,
      this.yPos,
      this.config.WIDTH,
      this.config.HEIGHT
    );
  }
};
