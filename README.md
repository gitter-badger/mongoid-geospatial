Mongoid Geospatial
==================

A Mongoid Extension that simplifies the use of MongoDB spatial features.


** On beta again **

Removing some trash, improving and adding support for RGeo and GeoRuby.
Version 2+ is going to be beta testing, when it's ready I'll release v3,
So the major version stays the same as mongoid.


There are no plans to support MongoDB < 2.0
There are no plans to support Mongoid <= 2.0



Quick Start
-----------

This gem focus on (making helpers for) spatial features MongoDB has.
You can also use an external Geometric/Spatial alongside.

    # Gemfile
    gem 'mongoid_geospatial'


    # A place to illustrate Point, Line and Polygon
    class Place
      include Mongoid::Document
      include Mongoid::Geospatial

      field :name,     type: String
      field :location, type: Point, :spatial => true
      field :route,    type: Linestring
      field :area,     type: Polygon
    end


Geometry Helpers
----------------

We currently support GeoRuby and RGeo.
If you require one of those, a #to_geo method will be available to all
spatial fields, returning the external library corresponding object.
To illustrate:

    class Person
      include Mongoid::Document
      include Mongoid::Geospatial

      field :location, type: Point
    end

    me = Person.new(location: [8, 8])

    # Example with GeoRuby
    point.class # Mongoid::Geospatial::Point
    point.to_geo.class # GeoRuby::SimpleFeatures::Point

    # Example with RGeo
    point.class # Mongoid::Geospatial::Point
    point.to_geo.class # RGeo::Geographic::SphericalPointImpl


Configure
----------------

Assemble it as you need (use a initializer file):

With RGeo

    Mongoid::Geospatial.use_rgeo
    # Optional
    # Mongoid::Geospatial.factory = RGeo::Geographic.spherical_factory

With GeoRuby

    Mongoid::Geospatial.use_georuby

Defaults (change if you know what you're doing)

    Mongoid::Geospatial.lng_symbol = :x
    Mongoid::Geospatial.lat_symbol = :y
    Mongoid::Geospatial.earth_radius = EARTH_RADIUS



Model Setup
-----------

You can create Point, Line, Circle, Box and Polygon on your models:


    class River
      include Mongoid::Document
      include Mongoid::Geospatial

      field :name,              type: String
      field :length,            type: Integer
      field :discharge,         type: Integer

      field :source,            type: Point,    spatial: true
      field :mouth,             type: Point,    spatial: true
      field :course,            type: Line
      field :boundings,         type: Box

      # spatial indexing
      spatial_index :mouth

      # default mongodb options
      spatial_index :mouth, {bit: 24, min: -180, max: 180}
    end


Use
---

Generate indexes on MongoDB:


    rake db:mongoid:create_indexes


Points
------

* an unordered hash with the lat long string keys defined when setting the field (only applies for setting the field)
* longitude latitude array in that order - [long,lat] ([x, y])
* an unordered hash with latitude key(:lat, :latitude) and a longitude key(:lon, :long, :lng, :longitude)
* an ordered hash with longitude as the first item and latitude as the second item
  This hash does not have include the latitude and longitude keys
  \*only works in ruby 1.9 and up because hashes below ruby 1.9 because they are not ordered
* anything with the method to_lng_lat that converts it to a [long,lat]

We store data in the DB as a [lng,lat] array then reformat when it is returned to you


    hudson = River.create(
      name: 'Hudson',
      length: 315,
      discharge: 21_400,
      # when setting array LNG (x) MUST BE FIRST LAT (y) MUST BE SECOND
      # source: [-73.935833,44.106667],
      # but we can use hash in any order
      source: {:lat => 44.106667, :lng => -73.935833},
      mouth: {:latitude => 40.703056, :longitude => -74.026667}

Now to access this spatial information we can do this

    hudson.mouth  # => [-74.026667, 40.703056]

Distance and other geometrical calculations are delegated to the external
library you choosed. More info about using RGeo or GeoRuby below.
Some built in helpers:

    # Returns middle point + radius
    # Useful to search #within_circle
    hudson.mouth.radius(5)        # [[-74.., 40..], 5]
    hudson.mouth.radius_sphere(5) # [[-74.., 40..], 0.00048..]

    # Returns hash if needed
    hudson.mounth.to_hsh              # {:x => -74.., :y => 40..}
    hudson.mounth.to_hsh(:lon, :lat)  # {:lon => -74.., :lat => 40..}


Query
--------

Before you read about mongoid_spatial have sure you read this:

http://mongoid.org/en/origin/docs/selection.html#standard

All MongoDB queries are handled by Mongoid.


You can use Geometry instance directly on any query:

* near
  * Bar.where(:location.near => person.house)

* near_sphere
  * Bar.where(:location.near_sphere => person.house)

* within_box
  * Bar.where(:location.within_box => hood.area)

* within_circle
  * Bar.where(:location.within_box => hood.area)

* within_circle_sphere
  * Bar.where(:location.within_circle_sphere => hood.area)

* within_polygon
  * Bar.where(:location.within_polygon => city.area)


External Libraries
------------------

Use RGeo?
https://github.com/dazuma/rgeo

RGeo is a Ruby wrapper for Proj/GEOS.
It's perfect when you need to work with complex calculations and projections.
It'll require more stuff installed to compile/work.


Use GeoRuby?
https://github.com/nofxx/geo_ruby

GeoRuby is a pure Ruby Geometry Library.
It's perfect if you want simple calculations and/or keep your stack in pure ruby.
Albeit not full featured in maths it has a handful of methods and good import/export helpers.

Use Nothing?

This lib won't stand in your way.
Write your own wrapper if you want.


Class Methods
-------------

* MISSING: this is the last part I'm working on...wait and comment *

Some method are added to your class when you define a field as spatial.

    field :location, type: Point, spatial: true



Geometry
--------

You can also store Circle, Box, Line (LineString) and Polygons.
Some helper methods are available to them:


    # Returns a geometry bounding box
    # Useful to query #within_box
    polygon.bbox
    polygon.bounding_box

    # Returns a geometry calculated middle point
    # Useful to query for #near
    polygon.center

    # Returns middle point + radius
    # Useful to search #within_circle
    polygon.radius(5)        # [[1.0, 1.0], 5]
    polygon.radius_sphere(5) # [[1.0, 1.0], 0.00048..]




Mongo DB 1.9+ New Geo features
---------

Multi-location Documents v.1.9+

MongoDB now also supports indexing documents by multiple locations. These locations can be specified in arrays of sub-objects, for example:

```
> db.places.insert({ addresses : [ { name : "Home", loc : [55.5, 42.3] }, { name : "Work", loc : [32.3, 44.2] } ] })
> db.places.ensureIndex({ "addresses.loc" : "2d" })
```

Multiple locations may also be specified in a single field:

```
> db.places.insert({ lastSeenAt : [ { x : 45.3, y : 32.2 }, [54.2, 32.3], { lon : 44.2, lat : 38.2 } ] })
> db.places.ensureIndex({ "lastSeenAt" : "2d" })
```

By default, when performing geoNear or $near-type queries on collections containing multi-location documents, the same document may be returned multiple times, since $near queries return ordered results by distance. Queries using the $within operator by default do not return duplicate documents.

  v2.0
In v2.0, this default can be overridden by the use of a $uniqueDocs parameter for geoNear and $within queries, like so:

```
> db.runCommand( { geoNear : "places" , near : [50,50], num : 10, uniqueDocs : false } )
> db.places.find( { loc : { $within : { $center : [[0.5, 0.5], 20], $uniqueDocs : true } } } )
```

  Currently it is not possible to specify $uniqueDocs for $near queries
Whether or not uniqueDocs is true, when using a limit the limit is applied (as is normally the case) to the number of results returned (and not to the docs or locations).  If running a geoNear query with uniqueDocs : true, the closest location in a document to the center of the search region will always be returned - this is not true for $within queries.

In addition, when using geoNear queries and multi-location documents, often it is useful to return not only distances, but also the location in the document which was used to generate the distance.  In v2.0, to return the location alongside the distance in the geoNear results (in the field loc), specify includeLocs : true in the geoNear query. The location returned will be a copy of the location in the document used.

  If the location was an array, the location returned will be an object with "0" and "1" fields in v2.0.0 and v2.0.1.

```
> db.runCommand({ geoNear : "places", near : [ 0, 0 ], maxDistance : 20, includeLocs : true })
{
  "ns" : "test.places",
  "near" : "1100000000000000000000000000000000000000000000000000",
  "results" : [
    {
      "dis" : 5.830951894845301,
      "loc" : {
        "x" : 3,
        "y" : 5
      },
      "obj" : {
        "_id" : ObjectId("4e52672c15f59224bdb2544d"),
        "name" : "Final Place",
        "loc" : {
          "x" : 3,
          "y" : 5
        }
      }
    },
    {
      "dis" : 14.142135623730951,
      "loc" : {
        "0" : 10,
        "1" : 10
      },
      "obj" : {
        "_id" : ObjectId("4e5266a915f59224bdb2544b"),
        "name" : "Some Place",
        "loc" : [
          [
            10,
            10
          ],
          [
            50,
            50
          ]
        ]
      }
    },
    {
      "dis" : 14.142135623730951,
      "loc" : {
        "0" : -10,
        "1" : -10
      },
      "obj" : {
        "_id" : ObjectId("4e5266ba15f59224bdb2544c"),
        "name" : "Another Place",
        "loc" : [
          [
            -10,
            -10
          ],
          [
            -50,
            -50
          ]
        ]
      }
    }
  ],
  "stats" : {
    "time" : 0,
    "btreelocs" : 0,
    "nscanned" : 5,
    "objectsLoaded" : 3,
    "avgDistance" : 11.371741047435734,
    "maxDistance" : 14.142157540259815
  },
  "ok" : 1
}
```

The plan is to include this functionality in a future release. Please help out ;)

This Fork
---------

This fork is not backwards compatible with 'mongoid_spatial'.
This fork delegates calculations to the external libs and use Moped.

Change in your models:

    include Mongoid::Spacial::Document

to

    include Mongoid::Geospatial


And for the fields:


    field :source,  type: Array,    spacial: true

to

    field :source,  type: Point,    spatial: true


Beware the 't' and 'c' issue. It's spaTial.



Troubleshooting
---------------

**Mongo::OperationFailure: can't find special index: 2d**

Indexes need to be created. Execute command:

    rake db:mongoid:create_indexes


Thanks
------

* Thanks to Kristian Mandrup for creating the base of the gem and a few of the tests
* Thanks to CarZen LLC. for letting me release the code we are using


Contributing
------------

* Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet
* Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it
* Fork the project
* Start a feature/bugfix branch
* Commit and push until you are happy with your contribution
* Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
* Please try not to mess with the Rakefile, version, or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.


Copyright
-----------

Copyright (c) 2011 Ryan Ong. See LICENSE.txt for further details.
