---
description: A shortlist of Airbnb stays for a destination, dates and party size
---

Build a shortlist of places to stay.

Ask me for the destination, the dates and who is travelling if I have not given them. Guest counts matter to the price, so do not assume two adults.

Then:

1. Call `hasdata_airbnb_listing_getAirbnbListings` with `location`, `checkIn`, `checkOut` and the guest split across `adults`, `children`, `infants` and `pets`.
2. Report how many stays came back before filtering. An empty result means those dates, not that place, and I need to hear that plainly.
3. Shortlist five: title, the stay total from `price.originalPrice` with the window from `price.qualifier`, the `discountedPrice` when the listing carries one, `rating` with `reviews`, and the listing URL. Do not report bed or bathroom counts here, because the search does not return them.
4. Flag anything rated under 4.5 or with fewer than ten reviews. Treat a listing with no `rating` at all as unrated and say so, rather than letting it pass the filter because the field was missing.
5. Open `hasdata_airbnb_property_getAirbnbPropertyDetails` only for the stays I pick. Report `overview`, which is where the guest, bedroom, bed and bath counts live, the amenities whose `available` is true, anything in `safetyAndPropertyInfo`, the host record, and what the host says in the description.

Quote prices for the window I asked about, and say so. A nightly rate without its dates is a number I cannot use.
