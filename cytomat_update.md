# Cytomat Update Endpoint

## Project Objective
Currently Titan does not track the locations of plates within the Cytomat. There is however a 
database with the plates' barcodes and their locations stored in GBG. The goal of this project
is to update Titan with that GBG database. It will do this by patching one large JSON body with 
the 504 barcodes of every filled shelf or an empty string '' if the shelf is empty.

## Step by step

0. **GBG creates a JSON object with the locations and barcodes of the plates on all 504 shelves **

   "id": "XYZ123456789",

0. **The python app automatically grabs the new csv and parses**

   The Python app parses the csv data into a json format.

0. **The python app makes an api request to Titan to update the locations**

   Within Titan, the 96 tube rack is considered a `Location` and the
   tubes are considered `Container`s.  First the rack location `id` needs to be
   retrieved via an api call:

   ```
   GET https://api.notablelabs.com/v1/barcodes/XYZ123456789

   =>

   {
     "id": "XYZ123456789",
     "item_type": "Location",
     "item": {
       "id": "5f6baff1-35f2-4564-9f4e-8c3284c42eff",
       "type": null,
       "name": null,
       "parent_location_id": null,
       "created_at": "2016-02-19T22:42:21.952Z",
       "updated_at": "2016-02-19T22:42:21.952Z",
       "location_type": "96_tube_rack",
       "number": null
     }
   }
   ```
   NOTE: Check with an engineer to ensure that the above endpoint exists


   Now that the location object is retrieved, we can make an API call on
   the location object to update the containers.

   ```
   PATCH https://api.notablelabs.com/v1/locations/u5f6baff1-35f2-4564-9f4e-8c3284c42eff

   body:
   {
     "location": {
        "container_barcode_ids": {
          "1": "kl238d98j", 
          "2": "9823dj02d",
          "3": null, // for empty
          ...
          "96": "98cc3dj02d"
        }
      }
   }

   =>

   {
     "id": "5f6baff1-35f2-4564-9f4e-8c3284c42eff",
     "type": null,
     "name": null,
     "parent_location_id": null,
     "created_at": "2016-02-19T22:42:21.952Z",
     "updated_at": "2016-02-19T22:42:21.952Z",
     "location_type": "96_tube_rack",
     "number": null
   }
   ```

   All existing containers under this location will be removed and the
   new set of containers corresponding to the barcodes passed in will be
   saved.

   Note that `container_barcode_ids` should be the barcodes as strings
   in order from A1 -> A12, then B1 -> B12 etc

   If a barcode is not found, the the response will be:

   ```
   {
     "Error": "Barcode 123456789023 not found"
   }
   ```
   With status code `404`

0. **The python app indicates that the location update was successful**

   A quick success message lets the scientist know that she can remove
   the rack.

Scanning a 96 tube rack should always be indempotent so if a scientist
is ever unsure if a rack has been updated, she'll simply scan to be
sure.
