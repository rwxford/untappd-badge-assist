# Untappd Badge Assist

Untappd Badge Assist is a companion app concept focused on helping users optimize their badge progress. It highlights nearby badges, expiring opportunities, and the easiest paths to unlock badges based on location and available beers.

## Core Features

- **Nearby badge alerts**: Surface badges available near the user’s current location.
- **Expiring badge notifications**: Notify users when badges are close to expiring.
- **Expiration countdowns**: Show time remaining on expiring badges.
- **Easiest-to-earn insights**: Rank badges by effort based on proximity, inventory, and availability.
- **Inventory support**: Track beers the user has access to, enabling quick matches to badge requirements.
- **Menu scan or upload**: Scan or upload a menu photo to identify available drinks and determine which badges could be earned from the selection.

## Proposed MVP Flow

1. User grants location access.
2. App pulls active badges and nearby venues.
3. App ranks badges by proximity and estimated difficulty.
4. User adds inventory (beers or styles they have on hand).
5. User scans or uploads a menu photo to match drinks to badge requirements.
6. App recalculates easiest badges and notifies on expiring opportunities.

## Data Needs

- Badge catalog with requirements and expiration dates.
- Venue/beer availability data tied to geographic locations.
- User inventory (manual entry or import from Untappd check-ins).
- Menu image OCR and drink parsing to map menu items to badge requirements.

## Next Steps

- Validate data access options for Untappd badge and venue data.
- Define notification cadence and user controls.
- Sketch UI for map view, badge list, and inventory manager.
- Explore OCR libraries or APIs for menu photo ingestion and parsing.

