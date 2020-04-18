# LowSpec
Since last year HOTS stopped supporting weaker PC configurations and some people were unable to play the game. For this reason I created a utility that improves the performance of the game and you can download it [here].

I didn’t implement all optimizations that I wanted, but I can add them if there is the interest or if the devs could provide me a better access to the renderer since not all optimizations were possible to implement from “outside” (see Devs Appendix).
 At this moment, the utility can:
- Decrease the texture quality more than the game offers (keys F6/F7 in game for increase/decrease).
- Decrease the geometry quality more than the game offers (keys F8/F9 in game for increase/decrease). 
- Decrease the screen resolution more than the game offers (see the utility user guide). 
- Switch between singlethreaded/multithreaded rendering (keys F11/F12 in game).

There are many obvious bottlenecks like high poly models and high resolutions, which decreasing directly improves the performance, but degrades also the visual quality. Switching between singlethreaded/multithreaded rendering brings a huge speed up (my laptop gets 15-25% increase) with no visual quality los, but currently this works only on some integrated Intel graphics cards (you can try your luck with any graphics card, it will be probably just slower).

![geometry_comparison](https://user-images.githubusercontent.com/11290866/79631249-9c755480-8160-11ea-8604-c0a879ace2bd.png)

“0.15 Geometry quality” means that the game renders only 15% triangles of the original geometry, which is enormous reduction, while you can still clearly recognize all the characters. This is because HOTS doesn’t optimize the characters geometry for the game, but uses the same characters as for the menu and therefore all the characters have way more polygons than necessary. Fortunately I found a decimation algorithm that can reduce all the geometry with a nice quality on the background thread almost instantly, so it doesn’t even need file caching. However, it’s not perfect, as you can see some holes on Azmodan model.
  
  
![texture_comparison](https://user-images.githubusercontent.com/11290866/79631252-a434f900-8160-11ea-9c64-c553d6b80468.png)

“0.5 Texture quality” means that the texture mip-map level is biased by 5 levels and also switching to faster point filtering, while “1.0 Texture quality” means there is no mip-map level bias (therefore original texture quality) and it also uses original filtering.
  
Is this some kind of hacking? Well, technically yes, but it doesn’t go any further than any your video recording program similar to FRAPS. When FRAPS connects to your game, it searches the DirectX library in your running game process and tells the DirectX to capture the last frame of your game and displays the additional fps info. My utility connects the game the same way as FRAPS and then just tells the DirectX to decrease the quality and/or to distribute the work over multiple cores.

Is this allowed? Modifying software in general is illegal, but this utility does NOT modify any Blizzard software nor any assets, only communicates with the DirectX. The utility doesn’t even give you any game play advantage for cheating.

Will this work with the next game patch? Yes it will. The utility really doesn’t do anything else than communicates with the DirectX library. The DirectX is always the same on all the PCs. The only way to break the utility would be, if the game significantly reworks the renderer, which will probably not happen any soon.

I don’t want to install any crap! You don’t have to! The whole utility is just 182KB big, you can extract it anywhere you want and if you don’t like it, just delete it! That’s it! 

Now the game looks like a crap, but you can also play it on a piece of crap! 

# Devs Appendix
Optimizing this game without any source code is very time consuming and I decided to stop and simply ask for help either from the devs or some other hard core graphics programmers. The game is protected against debuggers and graphics profilers, so I had to write my own graphics analyzer and I discovered some spaces for optimizations without affecting the visual quality:

The game heavily suffers from uploading the data from CPU to GPU, while there are 2 x 128B constant buffers that are doing cca 300-450 uploads every frame, while more than half of these uploads are the very same data! I bet these are some material properties, that are exactly the same between different objects and therefore don't need to be uploaded multiple times, but without modifying the game I couldn't make this more efficient. I bet that the game sorts the draw calls based on the materials, but maybe it could consider also the material sorting based on the constant buffer content and not upload it if the previous material had the same one (this should cut cca 200-300 uploads to GPU per frame).

After switching to the lowest graphics settings, all my tested machines became CPU bounded. The game issues render draw calls via DX11 immediate context from the main thread, which is later blocking everything else. This gives a huge space for multithreaded optimizations. I injected the creation of the immediate context and created a deferred context instead. It might sound crazy, but since both contexts have the very same interface, this perfectly works. Additionally I had to synchronize map/unmap reads + start with write discard if you are doing writes with no overwrite to index/vertex buffer. After this, I implemented some logic that finishes and executes the deferred command lists on the background thread and that’s literally all you have to do to enable multithreaded rendering. However, the cost of the creation of the rendering commands on the deferred/immediate context should be the same, so this doesn’t give you much benefit, so why to even do that? Well simply because you can now move the execution of the command buffer to the rendering thread and better control when to wait for its finishing (immediate context blocks your main thread unpredictably, because you have no control over when the execution starts/ends). Unfortunately because of the poor driver implementations of DX11 deferred contexts the commands creation takes longer than on the immediate context (so far it seems that only the integrated Intel graphics cards do this properly). This gave me a next idea that worked on all tested PCs, but for time reasons I didn’t finish it.

To reduce the commands creation cost, I implemented my own super lightweight command buffer. Because HOTS uses only a limited subset of DX11 api and I know the game limits, I could do a lot of presumptions that the native command buffers couldn’t. This approach was implemented by many game engines during DX11 era. Unfortunately there is a problem that I had to make extra copies of all map/unmap writes, which wasn’t that an issue. The real problem was during the writes with no overwrite to index/vertex buffers – in this case the map/unmap doesn’t allocate a new buffer and just uses the existing one and the game writes into it without providing me the information which part was overwritten and therefore I didn’t know which part I should copy. Since this type of writes is allowed only for index/vertex buffers, I solved this problem by checking the start/end of DrawIndexedInstanced/DrawIndexed calls (obviously only the range of indices and vertices that is drawn needs the copying). However this detection had its own problems and in the end I implemented it only on a paused replay scene and it resulted in cca 20% boost on all tested machines (obviously this depends on how much the machine is CPU bounded). However since this would be way easier to implement directly with the game source code and the DX11 deferred context worked for me already, I decided to stop the development and continue only if people demand it.

The whole implementation took around 3-4 weeks counting writing own performance GPU analyzer and blind guessing since I had no source code and couldn’t even attach the debugger. I think the multithreaded rendering and constant buffer upload reduction is something worth to implement, so if someone is into this stuff, I would be very happy assisting.

# Testing configurations:

- Laptop Win10, 2.20GHz AMD A8-7410APU, 8GB RAM, integrated AMD Radeon R5 Graphics
- Laptop Win10, 2.30GHz i7-3610QM, 16GB RAM, integrated Intel HD Graphics 4000
- Desktop Win8, 3.4GHz i7-6700, 16GM RAM, GeForce GTX 745 (just to confirm that CPU is still bottleneck on lowest graphics settings on a better HW)

For any questions, feel free to contact me at gamer9xxx@gmail.com



