Author: chopan050

## `Scene.run()`

This is the equivalent of `Scene.render()` in ManimCE. Naming it `Scene.run()` avoids confusion with `Camera.render()`.
If we're porting this to ManimCE, a good idea is to have both a `Scene.run()` method (the main logic would be implemented there) and a deprecated `Scene.render()` method just for backwards compatibility, which would call `Scene.run()`.

Some differences:
- Before calling `Scene.setup()`, it sets the "virtual animation" and "real animation" start time **(TODO: they're apparently related to the "interactive mode", but what are they exactly?)**, and it calls its `SceneFileWriter.begin()` method.
- Like in ManimCE, it calls `Scene.setup()` and `Scene.construct()`. But then it calls another method, `Scene.interact()`, AKA the "live rendering".
- It then calls `Scene.tear_down()`, which unlike ManimCE's `Scene.tear_down()` is already implemented and probably not meant to be implemented in Scene subclasses.
- Calling `Scene.tear_down()` is the last thing ManimGL's `Scene.run()` does. ManimCE's `Scene.render()` does some final logging for the current animation.

## `Scene.play()`

This would be the most important method, which plays one animation.
ManimGL's version does not have this entanglement between the scene, a renderer and animations. In fact, there is no renderer class in ManimGL!

Our ManimCE version of `Scene.play()` is a chaos. It was already described by Benjamin Hackl in the thematic guide "A deep dive into Manim's internals" in https://docs.manim.community, but the main thing is that there are renderer objects and a lot of mutual callings between `Scene` and the renderers.
Instead, ManimGL's `Scene.play()` is much less convoluted.

First, it directly prepares all of its animations via the `prepare_animation()` function in manimlib/animation/animation.lib. This function takes an object, and if it's already an `Animation` it just returns it, but if it's an `AnimationBuilder` it builds it into an animation. `Animation`).

Then, it calls their `Animation.update_rate_info()`, which is just a setter.

Finally it calls the following 5 important functions in order:

### 1. `Scene.pre_play()`

The first thing `Scene.pre_play()` does is to verify if this is an interactive scene (and this is the first animation being played). If it is, it keeps the animation running indefinitely until the presenter ends it by pressing Space or right arrow.

- To mark a `Scene` as "interactive", it must be initialized with `presenter_mode=True`. By default, `Scene.presenter_mode` is set to False.
- If `Scene.presenter_mode == True`, and this is the first time `Scene.play()` was called (`Scene.num_plays` represents the number of times `Scene.play()` or more specifically `Scene.post_play()` was called, so if it's 0 it means it was never called before), `Scene.pre_play()` calls `Scene.hold_loop()`, which represents the interactive scene loop.
- What `Scene.hold_loop()` does is: while the `Scene.hold_on_wait` property remains True (it was already set to True in `Scene.__init__()` if `Scene.presenter_mode` was True), it repeatedly calls `Scene.update_frame()` which would keep the animation running. The only way to break this loop is defined in `Scene.on_key_press()`, where a series of events triggered by keyboard presses are defined. In particular, if the presenter presses Space or right key, `Scene.hold_on_wait` is set to False, breaking the loop in `Scene.hold_loop()` and ending this "interactive mode". Regardless. `Scene.hold_on_wait` is immediately set to True again. **(TODO: an `EventListener` must be calling `Scene.on_key_press()`, but where is that `EventListener` being instantiated?)**

Then, it calls `Scene.update_skipping_status()`, which decides whether this animation should be skipped or not. This decision is mainly based on 3 properties: `Scene.original_skipping_status` (defined by the original value of `Scene.skip_animations`), `Scene.start_at_animation_number` and `Scene.end_at_animation_number`.

- `Scene.skip_animations` is a boolean which decides whether to skip animations (i.e. not render them) or not. Its value may vary over time, but its original value when passed to `Scene.__init__()` is remembered by `Scene.original_skipping_status`. If `skip_animations=True` is passed to `Scene.__init__()`, **all** animations will be skipped and `Scene.skip_animations` will never be set to False. Else, only some animations might be skipped depending on other variables, meaning `Scene.skip_animations` could temporarily be set to True, but can eventually be set again to False thanks to `Scene.original_skipping_status`. Many, many things depend on this property and may change its value.
- `Scene.start_at_animation_number` and `Scene.end_at_animation_number` are optional integers which represent where to start and where to end rendering animations based on their numbers (Animation 0, Animation 1, etc).
- If `Scene.start_at_animation_number` was set to a number, `Scene.skip_animations` would be set to False in `Scene.__init__()` immediately after saving its original value in `Scene.original_skipping_status`. This would mean that, until that animation number X is reached, all the previous animations would be skipped.
- In the current method we're discussing, `Scene.update_skipping_status()`, if `Scene.start_at_animation_number` is set and we just reached that animation number (`if self.num_plays == self.start_at_animation_number`), we stop skipping animations (of course, unless we said explicitly "let's skip **all** animations" by passing `skip_animations=True` to `Scene.__init__`, which would be remembered by `Scene.original_skipping_status`). When we "stop skipping animations", not only we set `Scene.skip_animations` to False, but we also set the `Scene.virtual_animation_start_time` to the current ellapsed time. This is the time where the virtual animation would, well, start.
- This would also set `Scene.skip_time` (the time spent skipping animations) equal to `Scene.time` (total time, the time spent playing or skipping animations in general), which makes sense since all we've been doing until now is just skipping animations.
- On the other hand, if `Scene.end_at_animation_number` is set and we've already reached that animation number, it raises an `EndScene` exception. This makes sense because why keep doing all this process of skipping animations until the end? It's best just to raise an exception to end this immediately. This exception would propagate from `Scene.update_skipping_status` to `Scene.pre_play`, to `Scene.play`, to `Scene.construct`, and would be finally caught by `Scene.run`. This would mean `Scene.construct` ends there, `Scene.interact` gets skipped, and it would immediately call `Scene.tear_down`.

If the current animation won't be skipped, `Scene.pre_play()` then prepares its `SceneFileWriter` to begin writing the animation into a file by calling `SceneFileWriter.begin_animation()` (don't confuse with `SceneFileWriter.begin()`, which was used in `Scene.run()`!). Explaining how it works is beyond the scope of this guide.

Then, if "preview mode" is activated, we set the real animation and virtual animation start times. **(TODO: still find out what are those times)**

- "Preview mode" is activated by default, and occurs when a Scene is instantiated with `preview=True`, which in turn creates a `Window` object (defined in maninlib/window) which is assigned to `Scene.window`.

Finally, it calls `Scene.refresh_static_mobjects()` which in turn calls `Camera.refresh_static_mobjects()` for its `Scene.camera`. This falls out of the scope of this guide (and honestly I don't understand it yet).


### 2. `Scene.begin_animations()`

This method is quite simple. It takes the animations passed to `Scene.play()` and for each one of them it calls their `Animation.begin()` method. Also, if the mobject the `Animation` is acting on was not previously in the scene (usually when playing animations which introduce these mobjects into scene, like `ShowCreation` (the equivalent of ManimCE's `Create`), `DrawBorderThenFill`, `Write`, etc.), it adds it to the scene.


### 3. `Scene.progress_through_animations()`


