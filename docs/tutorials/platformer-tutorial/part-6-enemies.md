---
sidebar_position: 7
---

# Part 6: Enemies and Simple AI

Our platformer has movement, physics, collectibles, and scoring — but nothing pushes back. In this part we'll add **patrolling enemies** the player can stomp (or be sent home by), and the best part: every new mechanic is built from a pattern you've already learned in this series.

## What We'll Accomplish

By the end of this part, you'll have:
- An `Enemy` class with patrol AI that walks between bounds and turns around
- A **head sensor** on enemies — the mirror image of the player's foot sensor from Part 3
- Stomp-to-defeat: land on an enemy for points and a bounce
- Side-hit consequences: touch an enemy and the player respawns
- A new collision category wired into the filtering system from Part 2
- A classic Box2D lesson: why bodies can't be destroyed during contact callbacks, and the deferred-cleanup pattern

## Prerequisites

- Completed [Part 5: Game Mechanics](./part-5-mechanics) (or start from the [Part 5 checkpoint](https://github.com/Cocos2D-Mono/cocos2d-mono-samples/tree/main/Tutorial%20Samples/Platformer/Checkpoints/Part%205))

## Step 1: A New Collision Category

Enemies need their own lane in the collision-filtering system. Add one line to `PhysicsHelper.cs`:

```csharp
// Categories for collision filtering
public const ushort CATEGORY_PLAYER = 0x0001;
public const ushort CATEGORY_PLATFORM = 0x0002;
public const ushort CATEGORY_COLLECTIBLE = 0x0004;
public const ushort CATEGORY_ENEMY = 0x0008;
```

And let the player's body fixture collide with it — in `Player.cs`, extend the mask:

```csharp
// Set collision filtering
fixtureDef.filter.categoryBits = PhysicsHelper.CATEGORY_PLAYER;
fixtureDef.filter.maskBits = PhysicsHelper.CATEGORY_PLATFORM | PhysicsHelper.CATEGORY_COLLECTIBLE | PhysicsHelper.CATEGORY_ENEMY;
```

Platforms need the same treatment. In `Platform.cs`, the fixture's mask currently accepts only the player — and Box2D filtering only allows a collision when **each side's mask accepts the other's category**. Leave this out and enemies fall straight through the floor at spawn:

```csharp
// Set collision filtering
b2Fixture fixture = _body.FixtureList;
b2Filter filter = fixture.Filter;
filter.categoryBits = PhysicsHelper.CATEGORY_PLATFORM;
filter.maskBits = PhysicsHelper.CATEGORY_PLAYER | PhysicsHelper.CATEGORY_ENEMY;
fixture.SetFilterData(filter);
```

## Step 2: The Enemy Class

Create `Enemy.cs`. Structurally it's a cousin of `Player`: a dynamic Box2D body under a sprite, plus a thin sensor box — except the sensor sits on the enemy's **head**, because that's where stomps happen.

```csharp
using System;
using Cocos2D;
using Box2D.Dynamics;
using Box2D.Common;
using Box2D.Collision.Shapes;
using CocosDenshion;

namespace Platformer
{
    public class Enemy : CCSprite
    {
        // Physics body
        private b2Body _body;

        // Patrol parameters
        private const float PATROL_SPEED = 2.0f;
        private readonly float _patrolMinX;
        private readonly float _patrolMaxX;
        private int _direction = 1; // 1 = right, -1 = left

        private bool _isDefeated;

        public bool IsDefeated { get { return _isDefeated; } }

        public Enemy(b2World world, float x, float y, float patrolMinX, float patrolMaxX)
            : base("player_idle")
        {
            // Placeholder art: reuse the player sprite with a red tint.
            // In a real game, swap in dedicated enemy artwork.
            Color = new CCColor3B(230, 60, 60);

            _patrolMinX = patrolMinX;
            _patrolMaxX = patrolMaxX;

            // Start the sprite at the spawn point - otherwise the first frame
            // draws at (0,0) before Update syncs it to the physics body
            Position = new CCPoint(x, y);

            // Create physics body (dynamic, like the player: it walks and falls)
            b2BodyDef bodyDef = new b2BodyDef();
            bodyDef.type = b2BodyType.b2_dynamicBody;
            bodyDef.fixedRotation = true;
            bodyDef.allowSleep = false;
            bodyDef.position = new b2Vec2(x / PhysicsHelper.PTM_RATIO, y / PhysicsHelper.PTM_RATIO);

            _body = world.CreateBody(bodyDef);

            // Body fixture - collides with platforms and the player.
            // Sized to the VISIBLE art, not the texture: player_idle's 64x64
            // canvas is mostly transparent padding, with the ~26x31 character
            // sitting bottom-center. Boxing the whole texture would let the
            // empty air beside the enemy hit the player.
            b2PolygonShape shape = new b2PolygonShape();
            shape.SetAsBox(
                ContentSize.Width * 0.2f / PhysicsHelper.PTM_RATIO,
                ContentSize.Height * 0.25f / PhysicsHelper.PTM_RATIO,
                new b2Vec2(0, -ContentSize.Height * 0.25f / PhysicsHelper.PTM_RATIO),
                0);

            b2FixtureDef fixtureDef = new b2FixtureDef();
            fixtureDef.shape = shape;
            fixtureDef.density = 1.0f;
            fixtureDef.friction = 0.2f;
            fixtureDef.restitution = 0.0f;
            fixtureDef.filter.categoryBits = PhysicsHelper.CATEGORY_ENEMY;
            fixtureDef.filter.maskBits = PhysicsHelper.CATEGORY_PLATFORM | PhysicsHelper.CATEGORY_PLAYER;

            _body.CreateFixture(fixtureDef).UserData = this;

            // Head sensor - the player defeats the enemy by landing on this.
            // The body box is bottom-aligned, so its top face - the visible
            // head - sits at the sprite's vertical center (y = 0). The sensor
            // is WIDER than the body box: wide enough that every position
            // where the player can physically stand on the enemy registers
            // as a stomp, leaving no edge to perch on.
            b2PolygonShape headShape = new b2PolygonShape();
            headShape.SetAsBox(
                ContentSize.Width * 0.35f / PhysicsHelper.PTM_RATIO,
                3f / PhysicsHelper.PTM_RATIO,
                new b2Vec2(0, 0),
                0);

            b2FixtureDef headFixtureDef = new b2FixtureDef();
            headFixtureDef.shape = headShape;
            headFixtureDef.isSensor = true;
            // Explicit filter: an unset filter defaults to category 0x0001 -
            // the same value as CATEGORY_PLAYER - with mask 0xFFFF, so the
            // head would masquerade as the player in category checks and
            // generate contacts with platforms, coins, and other enemies.
            headFixtureDef.filter.categoryBits = PhysicsHelper.CATEGORY_ENEMY;
            headFixtureDef.filter.maskBits = PhysicsHelper.CATEGORY_PLAYER;

            b2Fixture headSensor = _body.CreateFixture(headFixtureDef);
            headSensor.UserData = new HeadSensorUserData(this);
        }

        public void Update(float dt)
        {
            if (_isDefeated)
                return;

            // Sync the sprite with the physics body
            Position = PhysicsHelper.ToCocosVector(_body.Position);

            // Patrol AI: walk in the current direction, turn around at the bounds
            if (Position.X <= _patrolMinX)
                _direction = 1;
            else if (Position.X >= _patrolMaxX)
                _direction = -1;

            _body.LinearVelocity = new b2Vec2(PATROL_SPEED * _direction, _body.LinearVelocity.y);

            // Face the direction of travel
            FlipX = _direction < 0;
        }

        public void Defeat(GameLayer gameLayer)
        {
            if (_isDefeated)
                return;

            _isDefeated = true;

            // Defeat is resolved from GameLayer.Update, AFTER the physics
            // step - never from inside a contact callback, where the world
            // is locked and silently ignores DestroyBody. That makes it
            // safe to destroy the body right here.
            RemoveFromWorld();

            // Squash, fade, and remove the sprite
            RunAction(new CCSequence(
                new CCScaleTo(0.15f, 1.3f, 0.4f),
                new CCFadeOut(0.25f),
                new CCCallFunc(() => RemoveFromParent())
            ));

            // Reuse the landing sound as a defeat thump; a real game would
            // use a dedicated effect here.
            CCSimpleAudioEngine.SharedEngine.PlayEffect("land");

            gameLayer.IncreaseScore(25);
        }

        public void RemoveFromWorld()
        {
            if (_body != null)
            {
                _body.World.DestroyBody(_body);
                _body = null;
            }
        }

        // User data for the head sensor - lets the contact listener recognize
        // "the player landed on an enemy" (see ContactListener.CheckEnemyContact).
        public class HeadSensorUserData
        {
            public Enemy Enemy { get; private set; }

            public HeadSensorUserData(Enemy enemy)
            {
                Enemy = enemy;
            }
        }
    }
}
```

The "AI" here is deliberately simple — walk until a bound, turn around — and that's the point: enemy behavior is just **per-frame logic driving a physics body**, exactly like the player's movement in Part 3. Chasing, jumping, or line-of-sight behaviors are all upgrades to this one `Update` method.

:::warning The golden rule: record in callbacks, resolve after the step

`BeginContact` runs *inside* `world.Step`, while the world is **locked**. Two things follow:

1. **You can't mutate the world there.** Box2D silently ignores `DestroyBody` on a locked world — destroying a body in the callback would *appear* to work while actually leaving an invisible solid body behind: a ghost platform hanging where the enemy died.
2. **You can't trust contact order.** A stomp's sensor contact and the accompanying body-to-body contact arrive in the same step, in unspecified order — acting on whichever the callback sees first makes the outcome random.

So our contact listener only *records* what happened (Step 4), and `GameLayer.Update` resolves the results *after* the step (Step 5) — which is also why `Defeat` can safely destroy the body.

:::

## Step 3: Player Helpers

The player needs a few small additions in `Player.cs` — a reusable respawn (extract the existing fall-off-screen reset), a bounce for successful stomps, and a falling check the stomp logic will use:

```csharp
public void Update(float dt)
{
    // Update sprite position based on physics body
    Position = PhysicsHelper.ToCocosVector(_body.Position);

    // Check if player fell off the screen
    if (Position.Y < -100)
    {
        Respawn();
    }
}

public void Respawn()
{
    // Reset to the starting position
    _body.SetTransform(new b2Vec2(100 / PhysicsHelper.PTM_RATIO, 300 / PhysicsHelper.PTM_RATIO), 0);
    _body.LinearVelocity = b2Vec2.Zero;

    // Fresh spawn state - a player hit in mid-air shouldn't carry
    // exhausted jumps into the respawn
    _jumpCount = 0;
    _canJump = false;
}

public void Bounce()
{
    // A small hop, used after stomping an enemy
    _body.LinearVelocity = new b2Vec2(_body.LinearVelocity.x, JUMP_FORCE * 0.6f);
}

public bool IsFalling
{
    get { return _body.LinearVelocity.y <= 0; }
}
```

While we're in `Player.cs`, fix a long-standing bug that enemy testing makes obvious: the double jump. The old `Jump()` re-derived `_canJump` from the jump count, but the foot sensor leaving the ground immediately sets `_canJump = false` — so the mid-air second jump never fired. Make the air jump an explicit rule instead:

```csharp
public void Jump()
{
    // First jump needs the ground under the foot sensor; the one
    // mid-air follow-up is the double jump. (_canJump is the grounded
    // flag; _jumpCount resets when the player lands.)
    bool canAirJump = _jumpCount > 0 && _jumpCount < MAX_JUMPS;

    if (_canJump || canAirJump)
    {
        _body.LinearVelocity = new b2Vec2(_body.LinearVelocity.x, JUMP_FORCE);
        _jumpCount++;

        _isRunning = false; // Stop running animation when jumping
        // Play jump animation
        StopAllActions();
        RunAction(new CCAnimate(_jumpAnimation));

        // Play jump sound
        PlayJumpSound();
    }
}
```

The other half of that fix lives in `GameLayer`: `Jump()` was called on *every frame* Space was held, which burned both jumps back to back. Jump on the key-press edge instead (new field `_wasJumpPressed` next to the other input flags):

```csharp
// Jump on the key-press edge, not every held frame - otherwise
// holding Space would burn both jumps back to back
if (_isJumpPressed && !_wasJumpPressed)
    _player.Jump();
_wasJumpPressed = _isJumpPressed;
```

## Step 4: Teaching the Contact Listener About Enemies

This is where the design pays off. Add enemy checks to `BeginContact` in `ContactListener.cs`:

```csharp
// Check for enemy contacts (stomp or side hit)
CheckEnemyContact(contact.GetFixtureA(), contact.GetFixtureB());
CheckEnemyContact(contact.GetFixtureB(), contact.GetFixtureA());
```

Per the golden rule from Step 2, the listener never *acts* on a contact — it records the result for `GameLayer` to resolve after the step. Give it somewhere to record (this also needs `using System.Collections.Generic;` at the top of the file):

```csharp
public class ContactListener : b2ContactListener
{
    // Contact callbacks run in the middle of world.Step, while the world
    // is locked and the order of same-step contacts is unspecified. So
    // the callbacks below only RECORD what happened; GameLayer.Update
    // resolves the results after the step.
    private readonly List<Enemy> _pendingStomps = new List<Enemy>();
    private readonly List<Enemy> _pendingSideHits = new List<Enemy>();
    private readonly List<Collectible> _pendingCollections = new List<Collectible>();

    public List<Enemy> PendingStomps { get { return _pendingStomps; } }
    public List<Enemy> PendingSideHits { get { return _pendingSideHits; } }
    public List<Collectible> PendingCollections { get { return _pendingCollections; } }

    // ...existing code
```

And the handler — note that it only records:

```csharp
private void CheckEnemyContact(b2Fixture fixtureA, b2Fixture fixtureB)
{
    // Stomp: the player's FOOT sensor touched an enemy's HEAD sensor -
    // the two sensor patterns from Parts 3 and 6 meeting in the middle.
    Enemy.HeadSensorUserData headData = fixtureA.UserData as Enemy.HeadSensorUserData;
    Player.FootSensorUserData footData = fixtureB.UserData as Player.FootSensorUserData;

    if (headData != null && footData != null)
    {
        // Only a falling player scores a stomp - the same contact
        // fires when jumping UP past the head zone, and that shouldn't
        // count.
        if (footData.Player.IsFalling &&
            !_pendingStomps.Contains(headData.Enemy))
        {
            _pendingStomps.Add(headData.Enemy);
        }
        return;
    }

    // Side hit: an enemy BODY fixture touched the player's BODY fixture.
    // The IsSensor check matters: without it, the player's foot sensor
    // brushing the enemy body would count as damage during a stomp.
    Enemy enemy = fixtureA.UserData as Enemy;
    if (enemy != null && !enemy.IsDefeated &&
        !fixtureB.IsSensor &&
        fixtureB.Filter.categoryBits == PhysicsHelper.CATEGORY_PLAYER)
    {
        if (!_pendingSideHits.Contains(enemy))
        {
            _pendingSideHits.Add(enemy);
        }
    }
}
```

The same rule retrofits Part 5's coins — their `Collect` also destroyed a body from inside the callback (silently ignored, leaking a ghost sensor per pickup). Record instead, and while here, collect on *any* PLAYER-category fixture rather than requiring the foot sensor specifically:

```csharp
private void CheckCollectibleContact(b2Fixture fixtureA, b2Fixture fixtureB)
{
    // Record a pickup when any PLAYER-category fixture touches a
    // coin. Both player fixtures qualify (the body, and the foot
    // sensor's default filter) - requiring a specific one would make
    // pickups depend on which fixture happens to overlap first.
    Collectible collectible = fixtureA.UserData as Collectible;
    if (collectible != null && !collectible.IsCollected &&
        fixtureB.Filter.categoryBits == PhysicsHelper.CATEGORY_PLAYER)
    {
        if (!_pendingCollections.Contains(collectible))
        {
            _pendingCollections.Add(collectible);
        }
    }
}
```

`Collectible` needs the matching guard — a `_isCollected` flag mirroring the enemy's `_isDefeated` (public `IsCollected` property, set first thing in `Collect`, early-return if already set). Since `Collect` now runs after the step, its existing `DestroyBody` call finally works as written.

One more listener change: **qualify the ground**. Since Part 3, the foot sensor has enabled jumping on *any* contact — fine when platforms were the only thing to touch, but the foot sensor now also brushes coins and enemy head sensors, and neither should hand the player a mid-air jump reset. Replace the foot-sensor blocks in `BeginContact`/`EndContact` with a shared helper that only counts solid platforms:

```csharp
public override void BeginContact(b2Contact contact)
{
    // Check for foot sensor contacts to enable jumping
    CheckFootContact(contact.GetFixtureA(), contact.GetFixtureB(), true);
    CheckFootContact(contact.GetFixtureB(), contact.GetFixtureA(), true);

    // ...existing collectible and enemy checks
}

public override void EndContact(b2Contact contact)
{
    // Check for foot sensor contacts to disable jumping
    CheckFootContact(contact.GetFixtureA(), contact.GetFixtureB(), false);
    CheckFootContact(contact.GetFixtureB(), contact.GetFixtureA(), false);
}

private void CheckFootContact(b2Fixture fixtureA, b2Fixture fixtureB, bool began)
{
    Player.FootSensorUserData footData = fixtureA.UserData as Player.FootSensorUserData;
    if (footData == null)
        return;

    // Only solid platform ground counts as standing on something.
    // The foot sensor also brushes coins and enemy head sensors,
    // and those must not reset the player's jumps mid-air.
    if (fixtureB.IsSensor ||
        fixtureB.Filter.categoryBits != PhysicsHelper.CATEGORY_PLATFORM)
        return;

    footData.Player.SetCanJump(began);
}
```

:::tip The IsSensor guard

A stomp and a side hit can happen in the *same physics step* — the player's foot sensor overlaps the enemy's head sensor while the foot sensor also brushes the enemy's body box. The `!fixtureB.IsSensor` check ensures only the player's **solid body** fixture counts as taking a hit, so a clean stomp never punishes the player.

Record-and-resolve handles the other half: the player's *body* also touches the enemy's body during a stomp, and Box2D reports same-step contacts in an unspecified order. Because both events are only recorded, and the resolver processes stomps first, the stomp wins no matter which contact arrived first. Subtle contact-handling details like these are where platformers live or die.

:::

## Step 5: Wiring Enemies Into the Level

In `GameLayer.cs`, track them, spawn them, and update them. Add the list next to the other game objects:

```csharp
private List<Enemy> _enemies = new List<Enemy>();
```

Spawn two at the end of `CreateLevel()` — one patrolling the floor, one holding a platform:

```csharp
// Create enemies: one patrolling the floor, one on a platform.
// Patrol bounds keep each enemy on its own ground.
Enemy floorEnemy = new Enemy(_world, 400, 110, 300, 550);
_enemies.Add(floorEnemy);
AddChild(floorEnemy);

Enemy platformEnemy = new Enemy(_world, 600, 160, 520, 680);
_enemies.Add(platformEnemy);
AddChild(platformEnemy);
```

In `Update`, right after `_world.Step(...)`, resolve what the listener recorded — stomps first, so a stomp always beats a side hit from the same step:

```csharp
// Update physics world
_world.Step(dt, 8, 3);

// Resolve enemy contacts recorded during the step. Stomps resolve
// first, so when a stomp and a side hit arrive in the same step
// the stomp always wins - regardless of Box2D's contact order.
foreach (Enemy enemy in _contactListener.PendingStomps)
{
    if (!enemy.IsDefeated)
    {
        enemy.Defeat(this);
        _player.Bounce();
    }
}
_contactListener.PendingStomps.Clear();

foreach (Enemy enemy in _contactListener.PendingSideHits)
{
    if (!enemy.IsDefeated)
        OnPlayerHit();
}
_contactListener.PendingSideHits.Clear();

foreach (Collectible coin in _contactListener.PendingCollections)
{
    if (!coin.IsCollected)
        coin.Collect(this);
}
_contactListener.PendingCollections.Clear();
```

Then drive the enemies, right after the player:

```csharp
foreach (Enemy enemy in _enemies)
    enemy.Update(dt);
```

Handle the side hit, and clear the list on restart:

```csharp
public void OnPlayerHit()
{
    // Touching an enemy from the side sends the player back to the start.
    // Extension ideas: health/lives, invincibility frames, knockback.
    _player.Respawn();
}
```

Restarting deserves care, because **sprites and physics bodies have different lifetimes**: `RemoveAllChildren` only clears the scene graph, while every body the level created — player, platforms, coins, enemies — would live on invisibly in the world. The simplest airtight restart rebuilds the physics world itself. Extract the world setup from the constructor:

```csharp
private void CreatePhysicsWorld()
{
    // Initialize physics world with gravity
    _world = new b2World(new b2Vec2(0, -10.0f));
    _world.SetContactListener(_contactListener);
}
```

(the constructor now just creates `_contactListener` and calls `CreatePhysicsWorld()`), and restart becomes:

```csharp
private void RestartGame(object sender)
{
    // Reset score
    _score = 0;

    // Sprites and physics bodies have different lifetimes:
    // RemoveAllChildren only clears the scene graph, and bodies the
    // level created (player, platforms, coins, enemies) would live
    // on invisibly in the old world. Rebuilding the physics world
    // from scratch guarantees no orphaned bodies survive a restart.
    CreatePhysicsWorld();
    _enemies.Clear();
    _platforms.Clear();

    // Drop any contact results recorded for the old level
    _contactListener.PendingStomps.Clear();
    _contactListener.PendingSideHits.Clear();
    _contactListener.PendingCollections.Clear();

    RemoveAllChildren();
    CreateLevel();
}
```

## 🎯 Checkpoint: Enemies

Run the game. You should see:
- Two red enemies patrolling — one on the floor, one on a platform — turning around at their bounds
- Landing on an enemy squashes it, plays a thump, awards **+25**, and bounces you
- Touching an enemy from the side sends you back to the start
- Restart brings the enemies back

Compare against the [Part 6 checkpoint](https://github.com/Cocos2D-Mono/cocos2d-mono-samples/tree/main/Tutorial%20Samples/Platformer/Checkpoints/Part%206) if anything differs.

## Troubleshooting

**Enemies fall through the floor** — `Platform.cs` must include `CATEGORY_ENEMY` in its `maskBits` (Step 1). A collision only happens when each side's mask accepts the other's category — the enemy accepting platforms isn't enough.

**An invisible platform floats where a defeated enemy stood** — the body was "destroyed" during a contact callback. The world is locked mid-step and silently ignores `DestroyBody`, so the solid body survived. Defer destruction to after the step (see the warning in Step 2).

**The player passes through enemies** — check that the player's `maskBits` includes `CATEGORY_ENEMY` *and* the enemy's `maskBits` includes `CATEGORY_PLAYER`; filtering must agree from both sides.

**The player can stand on top of a living enemy** — the head sensor must cover (and slightly overhang) the enemy's entire top. If it's narrower than the body box, a landing on an uncovered edge is physically supported by the solid body but never triggers the stomp — the player perches mid-air on an invisible ledge, and gets carried along as the enemy walks.

**Enemies die when the player jumps up past them** — the stomp branch is missing the `IsFalling` check. The foot-over-head contact also fires while rising through the head zone; only a falling player should stomp.

**A clean stomp sometimes also counts as a side hit** — the listener is acting inside `BeginContact` instead of recording. Same-step contact order is unspecified, so the body contact can be processed before the stomp; record both and resolve stomps first after the step (Steps 4–5).

**The player gets extra jumps by brushing coins or enemy heads mid-air** — `SetCanJump` is firing on every foot-sensor contact. Only solid `CATEGORY_PLATFORM` fixtures count as ground (see `CheckFootContact` in Step 4).

**The double jump never fires** — two causes, both fixed in Step 3: `Jump()` must allow an explicit mid-air jump (`_jumpCount > 0 && _jumpCount < MAX_JUMPS`) rather than relying on `_canJump`, which the foot sensor clears the moment the player leaves the ground; and jumping must trigger on the key-press *edge*, or holding the key burns both jumps instantly.

**Empty air beside the enemy hurts the player** — the fixtures are sized from `ContentSize`, but the texture has transparent padding around the visible art. Measure the art and build the boxes around it (see the body fixture in Step 2); a fixture sized to the full canvas reaches ~13px past each visible edge of this sprite.

**Invisible obstacles appear after restarting** — the restart is rebuilding the scene but not the physics world. Bodies aren't removed by `RemoveAllChildren`; recreate the world (or destroy every body) before `CreateLevel` (see `RestartGame` in Step 5).

**Stomping also respawns the player** — the `!fixtureB.IsSensor` guard is missing from the side-hit branch (see the tip in Step 4).

**Enemies walk off their platform** — patrol bounds are level-design data: keep `patrolMinX`/`patrolMaxX` inside the platform's horizontal extent. (Edge *detection* — turning around at a drop automatically — is a great extension exercise.)

**Enemies never move** — make sure `enemy.Update(dt)` is called from `GameLayer.Update` and that `bodyDef.allowSleep = false` is set.

## Complete Code Reference

The complete project through Part 6 is in the [tutorial samples repository](https://github.com/Cocos2D-Mono/cocos2d-mono-samples/tree/main/Tutorial%20Samples/Platformer/Final).

## Congratulations! 🎉

Your platformer now fights back. More importantly, you extended a real game *using only the patterns the game already taught you* — sensors, categories, user data, and per-frame logic. That's how features get added to shipping games too.

## Next Steps for Enhancement

Ideas for your next parts:
- Smarter AI: edge detection, chasing the player on sight, jumping enemies
- Health and lives instead of instant respawn
- Multiple levels with different layouts
- Power-ups and special abilities
- Better sprite artwork and animations
- Achievements and high scores
- Mobile touch controls

Thank you for following this tutorial series! You now have the foundation — and the patterns — to build complete 2D games with cocos2d-mono.
