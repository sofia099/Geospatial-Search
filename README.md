# Geospatial-Search on Airbnb Data
<img width="1237" alt="Screenshot 2023-12-05 at 4 07 09 AM" src="https://github.com/sofia099/Geospatial-Search/assets/59860423/e5155c17-142f-4f78-86e9-09044c8fae86">

Geospatial search is an incredibly powerful tool for data exploration, analysis, and visualization. This project aims to showcase the power of geospatial search by providing an interactive and ***visual*** experience via a webpage. Users will be able to draw polygons, polylines, and place markers (points) on a map to run geospatial searches against those values. The following actions are supported:
- Find Airbnb locations within multiple polygon
- Find Airbnb locations within multiple polygon with holes
- Find Airbnb locations within a certain distance from a polygon, polyline, or point
- Any combinations of the above

The interactive map is powered by [Leaflet](https://leafletjs.com/), an open-source JavaScript library for mobile-friendly interactive maps. The dataset used in this project is available for public download on [Inside Airbnb](http://insideairbnb.com/get-the-data/). I recommend hosting your dataset in [Rockset](https://rockset.com/) which is optimized for low-latency queries and has a built-in `geography` data type. You will need to create an account on Rockset to get an API key for this project. To create an account on Rockset go [here](https://rockset.com/create/).<br /><br />

## Step 1: Download Data
[Inside Airbnb](http://insideairbnb.com/get-the-data/) hosts quarterly data over the last year by each region. In this example, I use the San Francisco `Detailed Listings data`. Download the zip file for free on their site. <br /><br />

## Step 2: Upload Data to Rockset
In order to execute fast Geospatial Search queries over the dataset, we will need a low-latency database to host our collection. If you are using the San Francisco dataset I shared above, you can ingest right away by following these steps:
  1. In the Rockset Console, go to the "Collections" tab and then select "Create a Collection"
  2. Scroll down and select "File Upload" then "Start"
  3. Upload the dataset. Be sure to select "CSV/TSV" for File Format, then click "Next"
  5. Do NOT use the default ingest transformation. The [ingest transformation](https://docs.rockset.com/documentation/docs/ingest-transformation) is a powerful tool available in Rockset. It allows you to execute a SQL query on all incoming data _before_ it is stored in Rockset. You can choose to omit certain fields & correctly cast the rest. I recommend using the ingest transformation below as a starting point:

  ```
  SELECT
      _input.id as _id,
      _input.name,
      CAST(_input.latitude as FLOAT) as latitude,
      CAST(_input.longitude as FLOAT) as longitude,
      _input.picture_url,
      _input.listing_url,
      TRY_CAST(REPLACE(_input.price, '$') as float) as price,
      TRY_CAST(_input.review_scores_rating as float) as review_scores_rating,
      TRY_CAST(_input.number_of_reviews as int) as number_of_reviews,
      TRY_CAST(_input.beds as int) as beds,
      TRY_CAST(_input.accommodates as int) as accommodates,
      _input.bathrooms_text,
      _input.description,
      JSON_PARSE(_input.amenities) as amenities
  FROM
      _input
  ```

  6. In the next page, type a workspace name and collection name. I used workspace=`airbnb` and collection=`sf_listings`.
  8. Final step is to click "Create" and wait for the data to ingest into your Rockset collection. This will only take a few minutes with the San Francisco dataset.<br /><br />

## Step 3: Write & Save a Query Lambda
To run a geospatial search, we'll utilize the following Rockset-native geography functions:
- `ST_GEOGPOINT(longitude, latitude)` Constructs a new point with the given longitude and latitude.
- `ST_GEOGFROMTEXT(well_known_text)` Converts a [well known text](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry) to a geography.
- `ST_CONTAINS(geography_a, geography_b)` Returns true if and only if `geography_b` is entirely contained within `geography_a.`
- `ST_DISTANCE(geography_a, geography_b)` Returns the distance, in meters, between the closest points in the two geographies.

In the Query Editor in Rockset, save the following SQL query as a Query Lambda named `airbnbSearch` under the `airbnb` workspace:
```
SELECT
    _id,
    latitude,
    longitude,
    name,
    picture_url,
    listing_url,
    price,
    review_scores_rating,
    number_of_reviews,
    beds,
    accommodates,
    bathrooms_text,
    description,
    amenities
FROM
    airbnb.sf_airbnb
WHERE
    (
        NOT :containsSearch
        OR ST_CONTAINS(
            ST_GEOGFROMTEXT(:wktContains),
            ST_GEOGPOINT(longitude, latitude)
        )
    )
    AND (
        NOT :distanceSearch
        OR ST_DISTANCE(
            ST_GEOGFROMTEXT(:wktDistance),
            ST_GEOGPOINT(longitude, latitude)
        ) < :distanceMeters
    )
LIMIT
  10
```

Be sure to add the following parameters & default values:
```
{
  "name": "containsSearch",
  "type": "bool",
  "value": "false"
},
{
  "name": "distanceSearch",
  "type": "bool",
  "value": "false"
},
{
  "name": "wktContains",
  "type": "string",
  "value": "POINT(0 0)"
},
{
  "name": "wktDistance",
  "type": "string",
  "value": "POINT(0 0)"
},
{
  "name": "distanceMeters",
  "type": "float",
  "value": "0"
}
```

In short, this query runs a spatial search to find Airbnb data points that are contained in denoted polygons and/or are within a certain distance from denoted shapes (could be points, lines, or polygons). It will return **10** results at random. You can easily add additional metadata filtering in the `WHERE` clause and sort by adding a `ORDER BY` clause.<br /><br />

## Step 4: Create an API Key & Grab your Region
Create an API key in the [API Keys tab of the Rockset Console](https://console.rockset.com/apikeys). The region can be found in the dropdown menu at the top of the page. For more information, refer to [Rockset's API Reference](https://docs.rockset.com/documentation/reference/rest-api).<br /><br />

## Step 5: Update `geospatial-search.html`
Before running the .html file, check the following lines & update as needed:
- line 89: `center: [37.7749, -122.4194]`<br />
  These are coordinates to San Francisco. Update if using another location dataset
- line 281: `const apiKey = "YOUR_ROCKSET_API_KEY";`<br />
  Update with your Rockset API Key from Step 4.
- line 282: `const apiServer = "YOUR_ROCKSET_REGION_URL"`<br />
  Update with your Rockset Region URL (ex: "https://api.usw2a1.rockset.com")
- line 283: `const qlWorkspace = 'airbnb'`<br />
  If you saved the Query Lambda from Step 3 in a different workspace, update here.
- line 284: `const qlName = 'airbnbSearch'`<br />
  If you saved the Query Lambda from Step 3 under a different name, update here.
<br /><br />

## Step 6: Run the .html file and start drawing shapes!
Now we're ready to play! The tools are color-coded to denote the geographic function being used:
- Green polygons are equivalent to contain (using `ST_CONTAINS`)
- Red polygons are equivalent to not contain (aka holes).
- Blue is used for distance (suing `ST_DISTANCE`).

Draw a Green polygon (remember to click 'Finish' to close the shape). Draw a Red polygon inside the Green polygon to denote a hole. Draw a Blue polygon, line, or place a blue marker down to signify a point of distance. Below the map, enter the distance in meters from that shape that you wish to search for Airbnb data. Click 'Search' to view the results:

<img width="1237" alt="Screenshot 2023-12-05 at 4 07 09 AM" src="https://github.com/sofia099/Geospatial-Search/assets/59860423/c1897182-9688-4553-b3c7-77ab05d8ec04">


<br /><br />

### Known Limitations & Bugs
- Clicking on the first point does not close the Polyline/Polygon shape in Leaflet ([Github Issue](https://github.com/Leaflet/Leaflet.draw/issues/660))
- When drawing a hole in a polygon, it must be drawn immediately after the outer polygon. Fix in the works.
- Currently only 1 blue shape can be used for a distance search.
- Leaflet can be finicky when `allowIntersection` is set to `true` for polygons. Currently set to `false` for smoother user experience, but its important to note that polygon lines can **not** cross to ensure correct behavior.

### Leaflet Documentation:
https://leafletjs.com/reference.html <br />
https://leaflet.github.io/Leaflet.draw/docs/leaflet-draw-latest.html

### Rockset Documentation:
https://docs.rockset.com/documentation/reference/data-types#geography <br />
https://docs.rockset.com/documentation/reference/geographic-functions#st_intersects

### Interested in learning more?
Check out this recording of a workshop I hosted on Geospatial Search: <br />
https://www.youtube.com/watch?v=Jjg2J4r4zEE

Check out these slides from said workshop: <br />
https://docs.google.com/presentation/d/12qcMZfhTJlRMe8_Cutvqua6Z95Bbx5sJGxPLjbCtUNg/edit?usp=sharing

Check out this cool blog on another Airbnb Geospatial Use Case: <br />
https://rockset.com/blog/outside-lands-airbnb-prices-and-rocksets-geospatial-queries/)https://rockset.com/blog/outside-lands-airbnb-prices-and-rocksets-geospatial-queries/
