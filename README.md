# gbs-EditActorActiveIndexPlugin
 GBStudio plugin that allows editing the order of the actor in the active actor list used for rendering and OAM order

 Note: The plugin and example project are updated for upcoming version 4.2 (latest dev build) will no longer work with 4.1

This plugin contains 3 events: Get actor active index, Set actor active index and Sort actors verticaly

Setting an actor active index will activate the actor if it was deactivated and will reposition the actor in the active actor list at the index specified.
Setting it to 0 will be at the start of the list, rendered last
Setting it to the current amount of active actor or more will put it at the end of the list, rendered first (if you put a big number it'll put it at the end of the list no problem)

In this example, we get the active index of an actor and assign it to the player, this makes it so that the player actor will be inserted before the actor in the active list, which will make the player render behind that actor
![image](https://github.com/user-attachments/assets/5a426393-584e-4f23-b652-16cc829d96bb)

NOTE: This plugin is only usable for color games, on the DMG, OAM order does not affect sprite render order.

Example of displaying a walking on grass effect over a player actor

https://github.com/user-attachments/assets/646fd337-f69d-41b3-a1ad-e2ace08a900f

Sort actor verticaly will sort all current active actors depending on their y position
![image](https://github.com/user-attachments/assets/daa6adb6-19a5-4dd7-b7e1-d24da7120b28)

Can be put in a loop to keep the sort updated, it is pretty optimized and fast but keep a reasonable wait time if your scene is busy


https://github.com/user-attachments/assets/1b4fded2-95ee-4c1a-ab5a-b3616659223c

