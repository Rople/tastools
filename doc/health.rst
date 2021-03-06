Health, damage, other entities
==============================


Health and damage
-----------------

Suppose :math:`\mathcal{H}` and :math:`\mathcal{A}` are the health and armour
respectively, while :math:`\mathcal{H}'` and :math:`\mathcal{A}'` are new
health and armour.  Suppose :math:`D` is the original damage received by the
player.  Assuming the damage type is not ``DMG_FALL`` and not ``DMG_DROWN``,
then

.. math:: \mathcal{H}' = \mathcal{H} - \Delta\mathcal{H}
          \quad\quad\quad
          \mathcal{A}' =
          \begin{cases}
          0 & \text{if } \mathcal{A} = 0 \\
          \max(0, \mathcal{A} - 2D/5) & \text{otherwise}
          \end{cases}

where

.. math:: \Delta\mathcal{H} =
          \begin{cases}
          \operatorname{trunc}(D - 2\mathcal{A}) & \text{if } \mathcal{A}' = 0 \\
          \operatorname{trunc}(D/5) & \text{otherwise}
          \end{cases}

If the damage type if ``DMG_FALL`` or ``DMG_DROWN``, then the armour value will
remain the same, with :math:`\Delta\mathcal{H} = \operatorname{trunc}(D)`.
Sometimes the player velocity will be modified upon receiving damage, forming
the foundation for a variety of damage boosts.  First we have the concept of an
"inflictor" associated with a damage, which may or may not exist.  Drowning
damage, for example, does not have an inflictor.  Inflictor could be a grenade
entity, a satchel charge entity, a human grunt, or even the player himself (in
the case of selfgaussing).  It is the first argument to
``CBaseMonster::TakeDamage`` in ``dlls/combat.cpp``.

Suppose :math:`\mathbf{v}` is the player velocity and :math:`\mathbf{p}` the
player position.  If an inflictor with position
:math:`\mathbf{p}_\text{inflictor}` exists, then with

.. math:: \mathbf{d} = \mathbf{p} - \mathbf{p}_\text{inflictor} + \langle 0, 0, 10\rangle

we have

.. math:: \mathbf{v}' = \mathbf{v} +
          \begin{cases}
          \min(1000, 10\Delta\mathcal{H}) \mathbf{\hat{d}} & \text{if duckstate} = 2 \\
          \min(1000, 5\Delta\mathcal{H}) \mathbf{\hat{d}} & \text{otherwise}
          \end{cases}

We can immediately see that if the duckstate is 2 the change in velocity is
greater.  It is sad to see that the maximum possible boost given by a single
damage is 1000 ups and not infinite.


Object manoeuvre
----------------

Let :math:`V` the value of ``sv_maxvelocity``.  Define

.. math:: \operatorname{clip}(\mathbf{v}) := \left[ \min(v_x, V), \min(v_y, V), \min(v_z, V) \right]

which is basically ``PM_CheckVelocity``.  Assuming the player is not
accelerating, :math:`\lVert\mathbf{v}\rVert > E` and the use key is pressed
then with :math:`\mathbf{\tilde{v}}_0 = \mathbf{v}_0` the subsequence player
velocities :math:`\mathbf{v}_k` and object velocities :math:`\mathbf{u}_k` is
given by

.. math:: \begin{align*}
          \mathbf{v}_{k+1} &= (1 - k\tau) \operatorname{clip}(0.3\mathbf{\tilde{v}}_k) \\
          \mathbf{\tilde{v}}_{k+1} &= \mathbf{u}_k + \mathbf{v}_k \\
          \mathbf{u}_{k+1} &= (1 - k\tau) \operatorname{clip}(0.3\mathbf{\tilde{v}}_{k+1})
          \end{align*}

The physics of object boosting is well understood with trivial implementation.
A trickier technique is fast object manoeuvre, which is the act of "bringing"
an object with the player at extreme speed for a relatively long duration.

The general idea is to repeatedly activate ``+use`` for just one frame then
deactive it for subsequent :math:`n` frames while pulling an object.  Observe
that when ``+use`` is active the player velocity will be reduced significantly.
And yet, when ``+use`` is deactivated, the player velocity will be equal to the
object velocity, which may well be moving very fast.  The player will then
continue to experience friction.

One important note to make is that despite the player velocity being scaled
down by 0.3 when ``+use`` is active, the object velocity will actually increase
in magnitude.  An implication of this is that the object will gradually
overtake the player, until it goes out of the player's use radius.  To put it
another way, we say that the *mean* object speed is greater than the mean
player speed.  To control the mean player speed, :math:`n` must be adjusted.
If :math:`n` is too low or too high, the mean player speed will be very low.
Therefore there must exist an optimal :math:`n` at which the mean player speed
is maximised.

However, we do not often use the optimal :math:`n` when carrying out this
trick.  Instead, we would use the smallest possible :math:`n` so that the
object mean speed will be as high as possible while ensuring the object stays
within the use radius.  This means the object will hit an obstruction as soon
as possible, so that we can change direction as soon as that happens.


Damage boosting
---------------

Handgrenades
~~~~~~~~~~~~

The handgrenade is one of the most useful weapons for damage boosting in
Half-Life.  It is versatile and can be used in many situations.  Interestingly,
the initial speed and direction of the grenade when it is tossed depend on the
player pitch in a subtle way.  For example, when :math:`\varphi = \pi/2`
(i.e. the player is facing straight down) the initial speed and direction are
:math:`0` and :math:`\pi/2` respectively.  However, when :math:`\varphi = 0`
the initial speed and direction now become :math:`400` and :math:`-\pi/18 =
-10^\circ` respectively.  Another notable aspect of handgrenades is that its
initial velocity depends on the player velocity at the instant of throwing.
This is unlike MP5 grenades.

In general, we can describe the initial velocity and direction of handgrenades
in the following way.  **Assuming all angles are in degrees**.  First of all,
the player pitch will be clamped within :math:`(-180^\circ, 180^\circ]`.  Let
:math:`\varphi_g` be the handgrenade's initial pitch, then we have

.. math:: \varphi_g = -10^\circ +
          \begin{cases}
          (8/9) \varphi & \text{if } \varphi < 0 \\
          (10/9) \varphi & \text{otherwise}
          \end{cases}

And if :math:`\mathbf{v}_g` is its initial velocity and
:math:`\mathbf{\hat{f}}_g` is the unit forward vector constructed using
:math:`\varphi_g`, then

.. math:: \mathbf{v}_g = \mathbf{v} + \min(500, 360 - 4\varphi_g)
          \mathbf{\hat{f}}_g

To visualise this equation, we plotted a graph of the handgrenade's horizontal
speed and vertical velocity relative to the player against the player pitch.

.. image:: _static/handgrenade-vel.png

TODO

Distribution of health
~~~~~~~~~~~~~~~~~~~~~~

Health is a scarce resource in any speedrun because medkits and health chargers are relatively rare. Despite this harsh constraint, it is common to want to perform multiple damage boosts using whatever health that is available until the health becomes too low. A natural question to ask is: what is the optimal way to distribute the limited health over these damage boosts, so that the total time taken to reach the destination is minimised?

Intuitively, this question seems to have a simple answer. Suppose there are two straight paths we need to travel to reach the destination. We want to perform damage boosts at the very beginning of each path. Let the lengths of these two paths be 250 and 750 units. Assume that the initial horizontal speed at the beginning of each path is 100 ups. For simplicity, we will assume that we can consume up to 100 HP in total without dying.

Now observe that the length ratio is 1:3, so it is natural to guess that the health should also be distributed in 1:3 proportion for each straight path. Namely, allocate 25 HP to the damage boost for the shorter path and 75 HP for the longer path. Thus, we calculate that the total time taken to travel both paths is 1.597 seconds. However, what if we allocate 34 HP for the shorter path and 66 HP for the longer path instead? Then the total time is 1.555 seconds. In fact, we claim that this is the optimal distribution which minimises the total time. Even though the difference is small in this particular scenario, it is not at all obvious why the 1:3 distribution is suboptimal.

To find out the optimal health distribution, we construct a model which closely reflects actual situations. We first assume that we are required to perform damage boosts for :math:`n` *distance segments*. We define a distance segment as a straight line path which directly benefits from a damage boost done at the beginning of the path. To take a concrete example, imagine an extremely narrow L-shaped path where the turn is extremely sharp. Since the turn is very sharp, the player's horizontal speed will be reduced to a *fixed* value after making the turn. Thus, we consider the L-shaped path to be comprised of two distance segments, one for each straight path. Notice that no matter how much health is allocated to the initial boost, the speed gained will be lost after making the turn. Thus, the two straight paths are of distinct distance segment: the time taken to travel across the second straight path is independent of whatever that happens while travelling in the first straight path.

In practice, there is, of course, no perfect distance segment. Turns are rarely so sharp that all boosts in the horizontal speed are nullified. Nevertheless, the concept of distance segments can serve as a helpful guide and approximation to practical situations. Note also that the distance segments need not be continuous as is the case in the L-shaped path example described previously. Indeed, distance segments are completely independent of each other.

Let :math:`s_1, \ldots, s_n` be the lengths of the distance segments. Let :math:`u_1, \ldots, u_n` be the initial horizontal speeds are the beginning of each distance segment before damage boosting. These initial speeds are assumed to be fixed, independent of previous damage boosts. They are typically approximated in practice. And let :math:`\Delta v_1, \ldots, \Delta v_n` be the change in horizontal speed as a result of the damage boost at the beginning of each distance segment. Now assume that the speed stays constant after boosting. We can then compute that the total time required to traverse all distance segments is

.. math:: T(\Delta v_1, \ldots, \Delta v_n) = \frac{s_1}{u_1 + \Delta v_1} + \cdots + \frac{s_n}{u_n + \Delta v_n}

Here, the total time is written as a function with parameters :math:`\Delta v_1, \ldots, \Delta v_n`. We want to minimise this quantity by finding the optimal values for each of :math:`\Delta v_i`. Note also that we have a constraint, namely the amount of health given at the beginning of everything, before any boosting is done. We may express this constraint simply as

.. math:: H(\Delta v_1, \ldots, \Delta v_n) = \Delta v_1 + \cdots + \Delta v_n = 10\mathcal{H}

where :math:`\mathcal{H}` is the total health amount that will be consumed. Here, the coefficient of :math:`10` reflects the assumption that the player will duck for each damage boosting. Indeed, recall that by ducking the player will receive twice the amount of speed boost compared to that received in upright position. By stating the optimisation problem this way, it may readily be solved via the method of Lagrange multipliers.

This optimisation method is particularly useful when we have a multivariate objective function and an equation constraining the parameters. In this optimisation problem, we want to solve the :math:`n + 1` equations consisting of the constraint along the equations encoded as :math:`\nabla T = -\lambda \nabla H` where :math:`\lambda` is the Lagrange multiplier. Writing out the latter explicitly, we have

.. math:: \frac{s_i}{(u_i + \Delta v_i)^2} = \lambda
   :label: explicit_lagrange

for all :math:`1 \le i \le n`.  To proceed, we introduce a temporary variable :math:`\mathcal{\tilde{H}}` such that

.. math:: 10\mathcal{H} = \mathcal{\tilde{H}} - u_1 - \cdots - u_n

As a result, the constraint equation may be written as

.. math:: (u_1 + \Delta v_1) + \cdots + (u_n + \Delta v_n) = \mathcal{\tilde{H}}

Using :eq:`explicit_lagrange`, we then eliminate all :math:`u_i + \Delta v_i`, yielding

.. math:: \sqrt{\frac{s_1}{\lambda}} + \cdots + \sqrt{\frac{s_n}{\lambda}} = \mathcal{\tilde{H}}

Or equivalently, by eliminating the temporary variable,

.. math:: \left( \frac{\sqrt{s_1} + \cdots + \sqrt{s_n}}{10\mathcal{H} + u_1 + \cdots + u_n} \right)^2 = \lambda

Eliminating :math:`\lambda` using :eq:`explicit_lagrange` again, we have the solution for each :math:`\Delta v_i` in the following form:

.. math:: \Delta v_i = \frac{\sqrt{s_i}}{\sum_{k=1}^n \sqrt{s_k}} \left(
          10\mathcal{H} + \sum_{k=1}^n u_k \right) - u_i

Looking at this equation, we observe the rather counterintuitive ratio. In particular, the ratio is *not* given by

.. math:: \frac{s_i}{\sum_{k=1}^n s_i}

as one would have guessed.

We want to remark that this model makes the assumption that the speed is constant after boosting. This is normally not true in practice. However, consider that the speed after a damage boost is typically very high, and recall from strafing physics that the acceleration at higher speeds is noticeably lower.




Gauss boost and quickgauss
--------------------------

Gauss is a very powerful weapon by virtue of its damage and recoil, which can
be exploited to change the player velocity significantly.  When the secondary
fire is shot, the new player velocity will become

.. math:: \mathbf{v}' = \mathbf{v} + 250t \langle \cos\vartheta, \sin\vartheta, 0\rangle \cos\varphi

where :math:`0.5 \le t \le 4` is the charging time in seconds.  The damage
produced is :math:`D = 50t`.

Starting with the later versions of GoldSrc Half-Life, and including all Steam
versions, there is a magnificent exploit that makes the weapon fire in
secondary mode as though :math:`t = 4`, but taking only half a second to charge
plus one cell.  This technique is called quickgauss, so named because it is
quick to produce the maximum possible boost and damage, rather than having to
charge for a full 4 seconds and consuming a lot of cells.  This can be achieved
by charging up the weapon, then save and reload the game before the weapon
completes its minimum 0.5s charge.  Upon reloading, the weapon will complete
the 0.5s charge followed by a quickgauss shot.


Gauss reflect boost
-------------------

In secondary mode, explosions happen at the points at which the gauss beam
reflects.  Moreover, gauss beams only reflect if the angle of incidence
:math:`\alpha` (the smallest angle between the incoming beam and the plane
normal) is greater than 60 degrees.  Let :math:`D` the initial damage.  When
the beam reflects for the first time, then a blast damage is produced with
:math:`D\cos\alpha`.  The damage is then reduced to :math:`D(1 - \cos\alpha)`.
So on so forth.

The maximum blast damage is obviously 100.  This is true when the damage from
the gauss beam is 200 and :math:`\alpha` is 60 degrees.  Suppose a player is
ducked and shoots the flat floor beneath him with :math:`\alpha = \pi/3`.  The
height of the source of gauss beam will be :math:`18 + 12 = 30` units above the
ground.  Therefore the horizontal distance from the point of reflection to the
player is :math:`90/\sqrt{3}`, and it can be shown that the health loss is
75-77 (assuming no armour).  Interestingly, the direction of boost is
vertically upward.

Another more dramatic type of reflect boost is the following: facing the wall
(might need to adjust the yaw angle slightly) with 60 degrees pitch and shoot
while *unducked*.  When the gauss is shot, the beam will reflect at
:math:`16\sqrt{3}` units below the gun position.  Hence the blast damage is
around 86-91.  It turns out that the reflected beam will directly hit the
player, dealing another 100 damage.  The combined damage is therefore 186-191,
giving 930-955 ups of vertical boost.  If the player DWJ before boosting, the
resultant vertical speed would be approximately 1198-1223 ups, which is
enormous.  Note that normal jumping will not work, it has to be DWJ.  This is
to prevent the normal jumping animation from playing.  Presumably, movement of
the legs in the jumping animation causes the beam to miss the player hitboxes.


Selfgauss
---------

Selfgauss happens when a gauss beam hits certain obstructions.  The beam will
shoot out of the player's body, thereby hitting the head from within.  In
normal situations the beam will ignore the player unless it has been reflected
at least once.  In this case the damage inflictor is the player himself,
therefore :math:`\mathbf{\hat{d}} = \langle 0,0,1\rangle`, thus the boost is
vertically upward.  If the player is ducked, the pitch angle must be
sufficiently negative for the beam to headshot the player.

.. image:: _static/selfgauss-1.png

There are two possibilities that trigger selfgauss.  If a beam enters an
obstruction and exits it, the distance between the point of entry and the point
of exit must be numerically greater than the damage of the beam.  This distance
is :math:`\color{blue}{\ell}` in the figure above.  For example, if the
distance is 100 units and the damage is 110, then selfgauss will not be
triggered.  In addition, the beam must be able to exit the obstruction.  Thus
selfgaussing does not work if the obstruction touches a wall with no gap in
between.  On the other hand, if the beam crosses at least one non-worldspawn
entities (such as ``func_wall``) aside from the first obstruction, the distance
between the point of entry into the first obstruction and the point of exit out
of the *last non-worldspawn entity* is now considered.

Unfortunately, in some occasions if the obstruction is very thick selfgauss may
not be triggered, even if the conditions described above are fulfilled.  The
exact reason remains a mystery.

Selfgaussing can be very powerful.  It is possible to obtain a 1000 ups upward
boost using just 67 damage.  However, if the pitch angle is too high or too
low, the beam might not hit the head, hence reducing the boost significantly.


Quick weapon switch and gauss rapid fire
----------------------------------------

Weapons in Half-Life does moderate damage by default.  The quick weapon switch
(QWS) trick is one way to dramatically boost one's damage rate.  There is a
small delay immediately after switching to any weapon before one could fire it.
To eliminate this delay, one saves the game and reloads it.  This is the essence
of the QWS trick.

There is an instance variable of ``float`` type in the ``CBasePlayer`` class
named ``m_flNextAttack``.  Upon switching to a new weapon, this variable is set
by ``CBasePlayerWeapon::DefaultDeploy`` to ``0.5``.  This effectively adds a
delay of half a second to any weapon switching.  However, this value is not
written to the savestate when the game is saved.  After reloading, the variable
will be set to zero and therefore the weapon can start firing immediately.
Contrary to popular beliefs, the trick does not work by cancelling the
animation, because the delay is independent of the animation.

Though there is quick weapon switch, we do not have "weapon rapid fire" to
eliminate the delay between consecutive fires.  This is because this delay is
set by ``m_flNextPrimaryAttack`` or ``m_flNextSecondaryAttack``, which are
written to the savestate.  Fortunately, the gauss weapon is a remarkable
exception to this. It is widely known that there is no firing delay in secondary
mode.  More importantly, the delay in primary mode is enforced by
``m_flNextAttack`` instead of ``m_flNextPrimaryAttack``.  This allows one to
save and reload the game immediately after a primary fire to bypass the delay.
This is called gauss rapid fire (GRF) or fastfire.

The gauss weapon in rapid fire has an impressive damage rate.  One primary fire
inflicts a damage of 20.  Thus at 1000 fps, the damage rate is 20000 HP per
second, dealing a total damage of 1000 HP limited by the ammunition.  The damage
rate is significantly higher than that of any other weapon, including gauss with
the quickgauss trick.  With GRF most enemies will be annihilated within a
fraction of a second.  This can be useful for clearing paths in confined spaces
and destroying monsters that block the progress of the game, such as the
Nihilanth.


Box item duplication
--------------------

Box item duplication is a trick useful for duplicating items dropped by crates.
To perform this trick, we simply fire a shotgun simultaneously at two adjacent
crates, one of which is explosive and the other will spawn the desired item
upon destruction.  Consequently, the desired item will be duplicated.  It seems
straightforward to understand how this trick works: the crate in question
breaks twice due to damages inflicted simultanously by the explosive crate and
the shotgun pellets.  Such explanation implies that any type of simultaneous
damages inflicted in the same frame can trigger the glitch.  Unfortunately,
this is false.  For instance, if we shoot the crate while a hand grenade
explodes in the same frame, the box items will not be duplicated.

Building blocks
~~~~~~~~~~~~~~~

Understanding the correct explanation requires a detailed knowledge of the
Half-Life damage system and the sequence of events when performing the trick.
Often, damages in Half-Life are not inflicted immediately.  Instead, a series
of damages may be accumulated by the *multidamage* mechanism.  There are three
important functions associated with this mechanism: ``ClearMultiDamage``,
``AddMultiDamage`` and ``ApplyMultiDamage``.  Each of these functions works on
a global variable called ``gMultiDamage`` which has the following type

.. code-block:: cpp

   typedef struct
   {
     CBaseEntity *pEntity;
     float amount;
     int type;
   } MULTIDAMAGE;

The ``pEntity`` field is the entity on which damages are inflicted while the
``amount`` field is the accumulated damage.  The ``type`` field is not
important.

``ClearMultiDamage`` is the simplest function out of the three.  It simply
assigns ``NULL`` to ``gMultiDamage->pEntity`` and zeros out
``gMultiDamage->amount`` and ``gMultiDamage->type``.  This function accepts no
parameter.

``ApplyMultiDamage`` is straightforward.  When called, it will invoke
``gMultiDamage->pEntity->TakeDamage`` with the damage specified by
``gMultiDamage->amount``.  As the name suggests, ``TakeDamage`` simply
subtracts the entity's health by the given damage.  When the entity is a
breakable crate and its health is reduced to below zero, it will turn into a
``SOLID_NOT``, which renders itself invisible to any tracing functions.  Then,
the crate will fire any associated targets, schedule its removal from memory
after 0.1s, then spawn its associated item.  At this point you may be confused:
if the crate becomes ``SOLID_NOT``, then how can any further damages be dealt
to it if the crate cannot be found by tracing functions?  Continue reading.

``AddMultiDamage`` is slightly trickier.  One of the parameters accepted by
this function is the target entity on which damages are to be inflicted.  When
this function is invoked, it checks whether the current
``gMultiDamage->pEntity`` differs from the supplied entity.  If so, it will
call ``ApplyMultiDamage`` to deal the currently accumulated damages on the
current ``gMultiDamage->pEntity``.  After that, it assigns the supplied entity
to ``gMultiDamage->pEntity`` and the supplied damage to
``gMultiDamage->amount``.  On the other hand, if the supplied entity is the
same as the current ``gMultiDamage->pEntity``, then the supplied damage will
simply be added to ``gMultiDamage->amount``.

When an explosive crate detonates, damage is dealt to the surrounding entities.
The function responsible of inflicting this blast damage is ``RadiusDamage``.
This function looks for entities within a given radius.  For each entity, it
usually does a ``ClearMultiDamage``, followed by ``TraceAttack`` (which simply
calls ``AddMultiDamage`` on the target entity) and then ``ApplyMultiDamage``.

Finally, we come to the final building block toward understanding the trick:
``FireBulletsPlayer``.  This function is called whenever a shotgun is fired.
At the very beginning of this function, ``ClearMultiDamage`` is called,
followed by a loop in which each pellet is randomly assigned a direction to
simulate spread, then a tracing function is called for each pellet to determine
what entity has been hit.  Then, this entity's ``TraceAttack`` is called.
After the loop ends, the function concludes with a call to
``ApplyMultiDamage``.

Process
~~~~~~~

We can now make use of the knowledge we learnt above to understand how the
trick works.  Suppose we have two crates, one explosive and the other carrying
the desired item.  To perform the trick we fire the shotgun so that both crates
are simultaneously broken.  First of all, ``FireBulletsPlayer`` will be called.
The ``ClearMultiDamage`` at the beginning of the function ensures that any
leftover multidamage will not interfere with our current situation.  Suppose
the first few pellets strike the explosive crate.  For each of these pellets,
``TraceAttack`` is being called on the explosive crate.  This in turns call
``AddMultiDamage`` which accumulates the damage dealt to the explosive crate.
Suppose now the loop comes to a pellet that is set to deal damage on the
desired crate.  As a result, ``TraceAttack`` and so ``AddMultiDamage`` is
called on the desired crate, which is a *different entity* than the explosive
crate.  Since the desired crate is not the same as ``gMultiDamage->pEntity``,
``AddMultiDamage`` will call ``ApplyMultiDamage`` to inflict the accumulated
damage against the explosive crate.  This is the moment where the explosive
crate explodes.

The explosive crate calls ``RadiusDamage`` which in turn inflicts damage onto
the desired crate.  When this happens, the ``TakeDamage`` associated with the
desired crate will be called, which causes the associated item to spawn.  The
desired crate now turns into ``SOLID_NOT``.  Once ``RadiusDamage`` returns, we
go back to the last ``AddMultiDamage`` call mentioned in the previous
paragraph.  Here, ``gMultiDamage->pEntity`` will be made to point to the
desired crate, and the damage for the current pellet will be assigned to
``gMultiDamage->amount``.

Remember the ``FireBulletsPlayer`` at the beginning of this series of events?
The loop in this function will continue to iterate.  However, since the desired
crate is of ``SOLID_NOT`` type, the tracing functions will completely miss the
crate.  In other words, the rest of the shotgun pellets will not hit the
desired crate, and that in total only one pellet hits the crate.  When the loop
finally completes, the final ``ApplyMultiDamage`` then inflicts the damage
dealt by the one pellet onto the desired crate.  Since ``ApplyMultiDamage``
does not rely on tracing functions to determine the target entity, but rather,
it uses ``gMultiDamage->pEntity`` set a moment ago, the damage will be
successfully inflicted which triggers the second ``TakeDamage`` call for the
desired crate.  This will again causes it to spawn the associated item.

One assumption we made in the description above is that the loop in
``FireBulletsPlayer`` breaks the explosion crate first.  If this is not the
case, then the item will not be duplicated.  To see this, notice that the
desired crate becomes ``SOLID_NOT`` as soon as the first set of pellets breaks
it, which causes the later explosion to miss the crate.

Limited applicability
~~~~~~~~~~~~~~~~~~~~~

So why does shooting the target crate when a grenade explodes not work?  To see
this, suppose the grenade explodes first.  The grenade will call
``RadiusDamage`` to inflict blast damage onto the target crate.  After that,
the crate becomes ``SOLID_NOT``.  The bullets will therefore miss the crate.
On the other hand, suppose the bullets hit the crate first.  The crate will
then break and becomes ``SOLID_NOT`` again.  When the grenade later calls
``RadiusDamage``, the tracing functions within ``RadiusDamage`` will again miss
the crate.

To put it simply, this trick does not work in cases like this because usually
there is no way for the second damage to find the crate, since they depend on
tracing functions and they do not save the pointer to the desired crate
*before* the crate becomes ``SOLID_NOT``.
