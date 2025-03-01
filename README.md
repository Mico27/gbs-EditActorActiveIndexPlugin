# gbs-EditActorActiveIndexPlugin
 GBStudio plugin that allows editing the order of the actor in the active actor list used for rendering and OAM order

This plugin contains 2 events: Get actor active index and Set actor active index

Setting an actor active index will activate the actor if it was deactivated and will reposition the actor in the active actor list at the index specified.
Setting it to 0 will be at the start of the list, rendered last
Setting it to the current amount of active actor or more will put it at the end of the list, rendered first (if you put a big number it'll put it at the end of the list no problem)

In this example, we get the active index of an actor and assign it to the player, this makes it so that the player actor will be inserted before the actor in the active list, which will make the player render behind that actor
![image](https://github.com/user-attachments/assets/5a426393-584e-4f23-b652-16cc829d96bb)

NOTE: This plugin is only usable for color games, on the DMG, OAM order does not affect sprite render order.
