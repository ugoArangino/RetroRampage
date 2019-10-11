## Change Log

Occasionally bugs happen, and given the episodic nature of this tutorial, it is difficult to address these retrospectively without changing the Git commit history.

This file is a record of bugs that have been found and fixed since the tutorial started. The dates next to each bug indicate when the fix was merged. If you completed the relevant tutorial(s) after the date listed for a given bug, you can safely ignore it.

### Unused Property in Player struct (2020/02/05)

When we originally wrote the Player weapon code in [Part 8](Tutorial/Part8.md) we added a `lastAttackTime` property which was not actually used in the implementation.

This has now been removed.

### Monsters Can See Through Push-walls (2020/01/28)

When push-walls were introduced in [Part 11](Tutorial/Part11.md), the `World.hitTest()` method was not updated to detect ray intersections with the `Pushwall` billboards, with the result that the monster in the second room in the first level can see (and be shot by) the player through the push-wall.

The fix was to replace the following lines in the `World.hitTest()` method:

```swift
for door in doors {
    guard let hit = door.billboard.hitTest(ray) else {
```

with:

```swift
let billboards = doors.map { $0.billboard } +
    pushwalls.flatMap { $0.billboards(facing: ray.origin) }
for billboard in billboards {
    guard let hit = billboard.hitTest(ray) else {
```

### Bitmap Bounds Error (2019/10/11)

The original `drawColumn()` method introduced in [Part 4](Tutorial/Part4.md) had an unsafe upper bound that could potentially cause a crash by trying to read beyond the end of the source bitmap.

The fix was to replace the following line in the `drawColumn()` method in `Bitmap.swift`:

```swift
let start = Int(point.y), end = Int(point.y + height) + 1
```

with:

```swift
let start = Int(point.y), end = Int((point.y + height).rounded(.up))
```

### Inverted Bitmap Width and Height (2019/10/11)

The original logic in [Part 9](Tutorial/Part9.md) that switched to column-first pixel order had a bug where the width and height were swapped on output, causing the result to be corrupted for non-square images. Since the game used square textures for all the walls and sprites, the bug wasn't immediately apparent.

The fix was to change the last line in the `Bitmap.init()` function in `UIImage+Bitmap.swift` from:

```swift
self.init(height: cgImage.width, pixels: pixels)
```

to:

```swift
self.init(height: cgImage.height, pixels: pixels)
```

### Flipped Floor and Ceiling (2019/09/27)

The original logic in [Part 9](Tutorial/Part9.md) for rotating the textures to compensate for switching to column-first pixel order had the side-effect of flipping the Z-axis. This resulted in the floor texture being drawn on the ceiling, and vice-versa (thanks to [Adam McNight](https://twitter.com/adamcnight/status/1174323711710781442?s=20) for reporting).

The fix for this was to change two lines in `UIImage+Bitmap.swift`. First, in `UIImage.init()` change: 

```swift
self.init(cgImage: cgImage, scale: 1, orientation: .left)
```

to:

```swift
self.init(cgImage: cgImage, scale: 1, orientation: .leftMirrored)
```

Then in `Bitmap.init()` change:

```swift
UIImage(cgImage: cgImage, scale: 1, orientation: .rightMirrored).draw(at: .zero)
```

to:

```swift
UIImage(cgImage: cgImage, scale: 1, orientation: .left).draw(at: .zero)
```

### Wall Collisions (2019/08/19)

The original wall collision detection code described in [Part 2](Tutorial/Part2.md) had a bug that could cause the player to stick when sliding along a wall (thanks to [José Ibañez](https://twitter.com/jose_ibanez/status/1163225777401401344?s=20) for reporting).

The fix for this was to return the largest intersection detected between any wall segment, rather than just the first intersection detected. The necessary code changes are in `Actor.intersection(with map:)`, which should now look like this:

```swift
func intersection(with map: Tilemap) -> Vector? {
    let minX = Int(rect.min.x), maxX = Int(rect.max.x)
    let minY = Int(rect.min.y), maxY = Int(rect.max.y)
    var largestIntersection: Vector?
    for y in minY ... maxY {
        for x in minX ... maxX where map[x, y].isWall {
            let wallRect = Rect(
                min: Vector(x: Double(x), y: Double(y)),
                max: Vector(x: Double(x + 1), y: Double(y + 1))
            )
            if let intersection = rect.intersection(with: wallRect),
                intersection.length > largestIntersection?.length ?? 0 {
                largestIntersection = intersection
            }
        }
    }
    return largestIntersection
}
```

### Sprite Rendering (2019/08/02)

In the original version of [Part 5](Tutorial/Part5.md) there were a couple of bugs in the sprite texture coordinate calculation. In your own project, check if the `// Draw sprites` section in `Renderer.swift` contains the following two lines:

```swift
let textureX = Int(spriteX * Double(wallTexture.width))
let spriteTexture = textures[sprite.texture]
```

If so, replace them with:

```swift
let spriteTexture = textures[sprite.texture]
let textureX = min(Int(spriteX * Double(spriteTexture.width)), spriteTexture.width - 1)
```