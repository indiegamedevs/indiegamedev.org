---
title: "Keeping windows on the wall in an isometric game"
date: 2026-11-20
articles_tags: ["isometric", "2d-art", "core-graphics", "spritekit"]
summary: "A worked example of drawing a window and its night light from the same wall coordinates."
---

A cottage window can have four straight sides and still look crooked. On a wall that slopes down one pixel for every two pixels across, a horizontal sill in screen space cuts across the brickwork. The trick used in the iPhone game StepHaven is to give each wall its own drawing coordinates.

Start with the screen position of a top corner. Call the distance across the wall *u* and the distance down the wall *v*. In a canvas whose y coordinate increases downward, the drawing position is:

```text
x = cornerX + u
y = cornerY + slope × u + v
```

For example, give the wall a corner at (20, 40) and a slope of +0.5. A window starting at (8, 12) in wall coordinates starts at (28, 56) on the canvas. Its top-right corner, 16 units farther along the wall, lands at (44, 64). Make it 10 units tall and the bottom corners fall at (28, 66) and (44, 74). The sill now follows the same slope as the bricks. Changing the slope to −0.5 lets the same window drawing sit on the opposite face.

The game uses the same projection for lights after sunset. The example window’s local center is (16, 17), which maps to (36, 65). Put the glow there. With separate lists of hand-placed glow positions, an edit to the daytime art can leave a lit patch on solid brick. One practical test is to render the same cottage once in daylight and once after dark, then compare the center of each glow with the panes underneath it.

The developer’s illustrated account shows the actual buildings and describes how this approach works alongside SpriteKit depth sorting and Core Graphics texture generation. [A cozy town with no image files](https://stephaven.app/journal/drawn-in-code/)
