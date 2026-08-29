---
sidebar_position: 6
---

# Part 5: Game Mechanics

In this part, we'll add game mechanics that make our platformer engaging: collectible coins, a scoring system, and user interface elements.

## What We'll Accomplish

By the end of this part, you'll have:
- Collectible coin objects with physics
- A scoring system that tracks collected coins
- UI elements displaying score and other game information
- Win condition when all coins are collected
- Sound effects for collecting coins
- Level completion detection

## Prerequisites

- Completed [Part 4: Platforms and Collision](./part-4-platforms)
- Understanding of UI elements in cocos2d-mono
- Basic game state management concepts

## Step 1: Collectible Assets

You'll need these assets for collectibles:
- `coin.png` - Coin sprite
- `coin_pickup.wav` - Collection sound effect (optional)

Add these to your Content folder. Find them in the [complete project reference](https://github.com/Cocos2D-Mono/cocos2d-mono-samples/tree/main/Tutorial%20Samples/Platformer/Final/Platformer/Content).

## Step 2: Create Collectible Class

Create a new file called `Collectible.cs`:

```csharp
using Cocos2D;
using Box2D.Dynamics;
using Box2D.Common;
using Box2D.Collision.Shapes;
using CocosDenshion;

namespace Platformer
{
    public class Collectible : CCSprite
    {
        private b2Body _body;

        public Collectible(b2World world, float x, float y) : base("coin")
        {
            Position = new CCPoint(x, y);

            // Create physics body
            b2BodyDef bodyDef = new b2BodyDef();
            bodyDef.type = b2BodyType.b2_staticBody;
            bodyDef.position = new b2Vec2(x / PhysicsHelper.PTM_RATIO, y / PhysicsHelper.PTM_RATIO);

            _body = world.CreateBody(bodyDef);

            // Create circle shape for collectible
            b2CircleShape shape = new b2CircleShape();
            shape.Radius = ContentSize.Width * 0.3f / PhysicsHelper.PTM_RATIO;

            b2FixtureDef fixtureDef = new b2FixtureDef();
            fixtureDef.shape = shape;
            fixtureDef.isSensor = true; // Make it a sensor (no collision response)
            fixtureDef.filter.categoryBits = PhysicsHelper.CATEGORY_COLLECTIBLE;
            fixtureDef.filter.maskBits = PhysicsHelper.CATEGORY_PLAYER;

            _body.CreateFixture(fixtureDef).UserData = this;

            // Add a rotation action
            RunAction(new CCRepeatForever(new CCRotateBy(2.0f, 360)));
        }

        public void Collect(GameLayer gameLayer)
        {
            // Remove from physics world
            _body.World.DestroyBody(_body);
            _body = null;

            // Play collection animation
            RunAction(new CCSequence(
                new CCScaleTo(0.2f, 1.5f),
                new CCScaleTo(0.2f, 0.0f),
                new CCCallFunc(() => RemoveFromParent())
            ));

            // Play collection sound
            CCSimpleAudioEngine.SharedEngine.PlayEffect("coin_pickup");

            // Increase score
            gameLayer.IncreaseScore(10);
        }
    }
}
```

## Step 3: Update Contact Listener

Update your `ContactListener.cs` to handle coin collection:

```csharp
// Add this to the BeginContact method in ContactListener.cs
public override void BeginContact(b2Contact contact)
{
    // ...existing code
            
    // Check for collectible contacts
    CheckCollectibleContact(contact.GetFixtureA(), contact.GetFixtureB());
    CheckCollectibleContact(contact.GetFixtureB(), contact.GetFixtureA());
}

private void CheckCollectibleContact(b2Fixture fixtureA, b2Fixture fixtureB)
{
    // Check if fixA is a collectible and fixB is the player
    Collectible collectible = fixtureA.UserData as Collectible;
    if (collectible != null &&
        fixtureB.Filter.categoryBits == PhysicsHelper.CATEGORY_PLAYER)
    {
        // Get the game layer from the player's parent
        Player.FootSensorUserData playerNode = fixtureB.UserData as Player.FootSensorUserData;
        if (playerNode != null && collectible.Parent is GameLayer gameLayer)
        {
            collectible.Collect(gameLayer);
        }
    }
}
```

## Step 4: Update GameLayer

Update your `GameLayer.cs` to include collectibles:

```csharp
private void CreateLevel()
{
    // ...existing code

    // Create collectibles
    for (int i = 0; i < 3; i++)
    {
        Collectible coin = new Collectible(_world, 200 + i * 200, 200);
        AddChild(coin);
    }
}
```

## 🎯 Checkpoint: Game Mechanics

At this point, you should have:
- ✅ Collectible coins placed throughout the level
- ✅ Working score system that updates when coins are collected
- ✅ UI showing current score and coin collection progress
- ✅ Sound effects for collecting coins (if audio files are included)

**Test your game**: Run the project and verify that:
1. Coins appear on platforms and animate (rotate and bob)
2. Walking into coins collects them and increases score
3. UI updates correctly showing score and coin count

## Troubleshooting

**Coins not collecting**: Check that contact listener is handling sensor collisions correctly.

**UI not updating**: Verify that event handlers are properly connected to GameManager events.

**Sound not playing**: Ensure audio files are added to Content folder with proper build action.

## Complete Code Reference

You can find the complete code for this part in the [tutorial samples repository](https://github.com/Cocos2D-Mono/cocos2d-mono-samples/tree/main/Tutorial%20Samples/Platformer/Checkpoints/Part%205).

## Congratulations! 🎉

You've successfully created a complete 2D platformer game using cocos2d-mono! Your game features:

- **Physics-based gameplay** with Box2D integration
- **Smooth character controls** with jump mechanics
- **Collectible system** with scoring
- **Professional polish** with effects and animations
- **Complete game states** from menu to gameplay
- **Audio integration** for immersive experience

## What's Next

Your game is complete — but it doesn't fight back yet. The series continues in **[Part 6: Enemies and Simple AI](./part-6-enemies)**, where patrolling enemies, stomp mechanics, and side-hit consequences get built from the exact sensor and filtering patterns you've already learned.
