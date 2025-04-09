## Nintendo DS Geometry & Rendering Engine notes, the basics, and the vertex cache and edge finding for RE

As usual, while developing an emulator, I've found the existing documentation amazing - made by heroes ! - but lacking in some details or explanations. And as usual, I''ve had access to some really great people on the r/emudev Discord who've helped me out a lot. I'm writing some of these blog posts in order to share what I've learned.

In this case, some of it is a little more speculative, as not everything is known about the Nintendo DS. In writing my own implementation, I've attempted to constrain myself in the same way as the hardware, so that I can see what reasonable design constraints are, and extrapolate based on what we know. I'll try to be clear about what is speculation. If I don't remark on that, you can assume it's known info.

This post assumes a passing familiarity with 3d matrix math, vectors, and rasterization. There's literally hundreds of good tutorials on these topics, so I'll try to focus on the peculiarities of the DS. 

So starting from what we know:

* 256x192 output resolution, 15-bit RGB output.
* 18-bit RGB internal rendering.
* 144k of vertex RAM, 104k of polygon RAM: max 6144 vertices and 2048 polygons per frame, aka, 24 bytes per vertex and 52 bytes per polygon.
* The rendering engine starts rendering about 70 lines before its data might be needed; it has a 48-line buffer. 
* Polygons are ingested as triangles or quads, but then are clipped to have up to 10 sides.
* Vertices as the RE sees them (in the 24 bytes) support:
  * X, Y, Z, W at some precision
  * 6-bit R,G,B values
  * 12.4-bit U and V values for textures
* Polygons as the RE sees them (in the 52 bytes) support:
  * Texture parameters from TEXIMAGE_PARAM such as format, offset in RAM, etc.
  * Polygon attributes such as mark edges, toon shading, etc.
  * a list of y coordinates that each slope (should) begin at
  * A pointer to the first vertex it needs in VRAM, as well as how many it has
  * Speculation: Min_y and max_y for vertices, whether it is forward- or backward-facing 
  * a value that represents how to normalize the w-values
* There are 4 types of matrices kept internally, two of which have a 31-entry stack.
  * Projection matrix - for projecting coordinates to the view volume
  * Coordinate/position matrix
  * Direction/vector matrix
  * Texture matrix
  * "Clipmatrix" which is Coordinate * Projection, update any time either of those are 

### The life of a vertex
A vertex is submitted to the GE.

It is then added to a cache of vertex points that I plan to elaborate below.

The vertex is multiplied by the ClipMatrix, which is the Coordinate * Projection matrices.

If there are 3 points, and we're in the correct poly mode, a triangle is generated. If there are 4, in the correct mode, a quad is genereted.

Then, as part of the polygon ingestion, the vertex may be clipped against all 6 planes, potentially splitting 3? times, to two children.

Then it is transformed to screen coordinates and saved to vertex RAM.

Its data is then used to render the associated polygons.

### The vertex cache
The DS is very starved for vertex RAM. It can only fit 6144 total vertices. If we only used triangles, that would limit us to 2048 triangles, which is a limitation we have; however, it would limit us to only 1536 quads.

In order to minimize the amount of work needed for each vertex, as well as maximize use of vertex memory, the DS keeps previously transformed, clipped and split vertices if possible while ingesting a polygon strip.

I believe I've come across a simple algorithm it may employ, although it may also be a widely-used algorithm in computer science. It's fairly basic. It involves a singly forward-linked list with special properties.

To begin with, we have the container of the vertices, which is (conceptually) a polygon. It needs to keep track of the number of "original" points submitted to it.

As vertices are ingested, they are immediately multiplied by the clipmatrix and then added to the list at the end. List nodes have some special properties, namely: they can be marked as "processed" or "not processed," and also, "original vertex starts here" (this will be explained in a bit).

So. 

1) On vertex ingestion, add a new item to the end: the ingested vertex multiplied by clipmatrix
2) If correct number of points are present (3 for triangle or 4 for quad), ingest a polygon.
3) After polygon ingestion is done, if we are working on single triangles or quads, clear the list.
4) If we are working on triangle strip, delete all elements from the beginning up until the next encountered "original vertex started here" mark.. For quads, clear the first 2 instead of just 1.

Now, polygon ingestion looks like this:

1) Check for polygon clip & discard
2) Only clip edges where at least one vertex is unprocessed. This avoids processing edges that have already been processed.
3) Clipping "splits" a vertex, so insert a new vertex to the right of the one being clipped. Example: nodes   1 2 3 4 5, if 3 is split, you'll get 1 2 3.0 3.5 4 5 . 3.0 should retain its "original vertex started here" mark
3) After clipping, determine if we're going to overfill vertex RAM. If so, set the overflow flag and abort (I'm speculating that this goes here)
4) Commit vertices to RAM. To do this, traverse the list in order. If a node is not processed,
   1) Transform vertex to screen coordinates
   2) Emplace in vertex RAM, saving a pointer to where it is saved, into the list element
   3) As well, determine ymin/ymax vertices and w normalization values for the polygon
5) Fill out the rest of the info about a polygon that RE will need, such as number of vertices and pointer to first vertex, and commit to polygon RAM. Pointer to first vertex will be the pointer saved in the first list element.

This simple linked list algorithm is nice because it avoids duplicate matrix multiplication, clipping, and transforms to screen, as well as using the minimum possible amount of vertex RAM, all while making it super easy to manage the vertex cache with a very simple implementation and not too many transistors.

It's my supposition that the DS uses this or a close algorithm because it's so simple to implement and has so many benefits and so readily solves otherwise tough questions like "after clipping, where do I resume the polygon list?"

### Determining which edge to render
The MelonDS blog has lots of great info about the renderer. It shows very clearly examples of different polygons and how they are filled. It does not, however, share how to choose the two edges to lerp between.

As a scanline renderer that handles 2 lines in parallel, the RE has to answer this question: "which slopes do I lerp between, given that I am operating on line #y?"

This also is a very simple algorithm, that works for me, but took me an embarassing amount of time to come up with.

I'll describe it as it applies to finding one edge. Go in the opposite "direction" to find the next edge to LERP with.

1) Start with topmost vertex. Edge = topmost vertex and next vertex in that direction
2) Traverse around the polygon
3) Look for the first edge in this direction which intersects with the current scanline.
4) Reject it if this edge is also completely horizontal - this doesn't need ot be done if you properly interpolate differently for x-major vs. y-major edges. But it makes it a little simpler.
5) Once the edge is found, sort the two vertices by min-y

If you do this once in the +1 direction and once in the -1 direction, you'll end up with 2 edges. It doesn't matter which is "left" or "right:" when you interpolate x to the current scanline, THEN you choose a left or right and fill the span between them, following the rules the MelonDS blog outlines.

