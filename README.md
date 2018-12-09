## Earcutr

WARNING - in progress, doesnt work yet

Polygon triangulation library, translated into Rust computer language from
the original Earcut project from MapBox. https://github.com/mapbox/earcut

![image showing an outline of a building, with triangles inside of it](viz/building.png "Building, earcut")

#### Usage

```rust
var triangles = earcut([10,0, 0,50, 60,60, 70,10]); // returns [1,0,3, 3,2,1]
```

Signature: `earcut(vertices[, holes, dimensions = 2])`.

* `vertices` is a flat array of vertex coordinates like `[x0,y0, x1,y1, x2,y2, ...]`.
* `holes` is an array of hole _indices_ if any
  (e.g. `[5, 8]` for a 12-vertex input would mean one hole with vertices 5&ndash;7 and another with 8&ndash;11).
* `dimensions` is the number of coordinates per vertex in the input array (`2` by default).

Each group of three vertex indices in the resulting array forms a triangle.

```rust
// triangulating a polygon with a hole
earcut([0,0, 100,0, 100,100, 0,100,  20,20, 80,20, 80,80, 20,80], [4]);
// [3,0,4, 5,4,0, 3,4,7, 5,0,1, 2,3,7, 6,5,1, 2,7,6, 6,1,2]

// triangulating a polygon with 3d coords
earcut([10,0,1, 0,50,2, 60,60,3, 70,10,4], null, 3);
// [1,0,3, 3,2,1]
```

If you pass a single vertex as a hole, Earcut treats it as a Steiner point.

If your input is a multi-dimensional array (e.g. [GeoJSON Polygon](http://geojson.org/geojson-spec.html#polygon)),
you can convert it to the format expected by Earcut with `earcut.flatten`:

```rust
var data = earcut.flatten(geojson.geometry.coordinates);
var triangles = earcut(data.vertices, data.holes, data.dimensions);
```

After getting a triangulation, you can verify its correctness with `earcut.deviation`:

```rust
var deviation = earcut.deviation(vertices, holes, dimensions, triangles);
```

Returns the relative difference between the total area of triangles and the area of the input polygon.
`0` means the triangulation is fully correct.

#### How it works: The data

The main data structure is a list of lists, and each of those in turn is 
a sequence of points, with Cartesian coordinates. (x,y)

    Object = [ List1, List2, ... ]
    List1 = [[0,1], [2,4], [3,5],...]
    List2 = [[1,3], [0,0], [-2,-4],...]
    ...

Each List represents a polygon, also called a contour, or a ring of 
points. The first List is considered the 'body' or main shape. The rest 
of the lists are each a 'hole' within the main body.

For example a rectangle could be [ [ [0,0],[7,0],[7,4],[0,4] ] ]
This has a single contour, with four points.

    _______
    |     |
    |     |
    |     |
    |_____|
 
A square with a triangle shaped hole in the middle would be as follows
[ [ [0,0],[7,0],[7,4],[0,4] ],
  [ [1,1],[3,1],[3,3] ] ]

This has two contours, the first with four points, the second with three points.

    _______
    |     |
    |  /| |
    | /_| |
    |_____|

Now, we can convert these arrays of coordinates into a single array, by 
having some additional information stored alongside it.

For the rectangle, here is how the data looks: [ [ [0,0],[7,0],[7,4],[0,4] ] ]
After we convert it to a single array, it looks like this:

    data [ 0,0,7,0,7,4,0,4 ]
    holeindexes: []
    dimensions: 2

Notice we can still interpret the data as a sequence of points.

Now, lets try the same with the rectangle+hole:

    data [ 0,0,7,0,7,4,0,4,1,1,3,1,3,3  ]
    holeindexes: [ 8 ]
    dimensions: 2

Notice we can still interpret the data as a sequence of points,
and by looking at 'hole indexes' we can figure out where the first contour
ends and the next begins. If we had two holes, there'd be two holeindexes.

In some systems 'winding' of the holes differs from that of the 'body'. In
other words if your ead the points off in order, they will be clockwise
for the body, and counterclockwise for the holes (or vice versa).

Data examples are included under tests/fixtures in json files.

#### How it works: The algorithm

The library implements a modified ear slicing algorithm,
optimized by [z-order curve](http://en.wikipedia.org/wiki/Z-order_curve) hashing
and extended to handle holes, twisted polygons, degeneracies and self-intersections
in a way that doesn't _guarantee_ correctness of triangulation,
but attempts to always produce acceptable results for practical data.

It's based on ideas from
[FIST: Fast Industrial-Strength Triangulation of Polygons](http://www.cosy.sbg.ac.at/~held/projects/triang/triang.html) by Martin Held
and [Triangulation by Ear Clipping](http://www.geometrictools.com/Documentation/TriangulationByEarClipping.pdf) by David Eberly.

#### Why another triangulation library?

The aim of this project is to create a Rust language triangulation 
library that is very simple while being robust enough to handle most 
practical datasets without crashing or producing garbage.

If you want to get correct triangulation even on very bad data with lots of self-intersections
and earcutr is not precise enough, take a look at [libtess.js](https://github.com/brendankenny/libtess.js).

#### Install

You can copy the earcutr.rs file into your own project and use it.

To download the full library, with tests,

```bash
git clone github.com/donbright/earcutr
cd earcutr
cargo build
cargo test -- --nocapture # shows all output
cargo test -- --test-threads=1  # test-threads=1 will enable visualization
ls viz/testoutput.json # test in visualiztaion mode will create this file
cd viz                 # vizualisation code lives here, it's javascript/html
firefox viz.html       # view in your favorite web browser (circa 2018)
```

#### Ports to other languages

- https://github.com/mapbox/earcut (Original javascript, earcutr is a port)
- [mapbox/earcut.hpp](https://github.com/mapbox/earcut.hpp) (C++11)
