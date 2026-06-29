# Domain Glossary

The shared language of **Weather POC**. Definitions only — what each term *is* and what
distinguishes it from neighbouring terms. No implementation or process detail.

> Factory process vocabulary (Story, Feature, Spec, Plan, ADR, HITL, AFK, orchestrator)
> is **not** defined here — it belongs to the Enate SDLC Factory, not to this product's
> domain. This glossary covers only the weather product itself.

## Actors

### End User
A person who uses the application to look up weather for a City. The only actor in the
product — there is no operator, administrator, or API consumer role.

## Locations

### City
A named place, supplied by the End User as free text, that a weather lookup is performed
for. The single unit of location in the product — there are no regions, postcodes, or
coordinates. A City is the input to both the Search and the Weather View.

## Screens

### Search
The first screen the End User sees, whose sole purpose is to accept a City and start a
lookup. Distinct from the Weather View, which displays results; Search collects the City
but shows no weather.

### Weather View
The screen shown after a lookup, displaying the Current Conditions and Forecast for the
chosen City. Unlike the Search screen it both shows weather *and* accepts a new City, so
the End User can switch cities without returning to Search.

## Weather data

### Current Conditions
The weather for the present hour in the chosen City — the default selection shown when the
Weather View first loads. Not a separate dataset from the Forecast but its first Time
Period; "right now" is simply day one's current hour.

### Forecast
The full set of weather data for a City covering **four calendar days including today**,
at Hourly granularity. Encompasses the Current Conditions (its first Time Period) through
the end of the fourth day. Not a daily summary — every Time Period within it is an hour.

### Hourly
The granularity at which the Forecast is expressed: one set of weather Attributes per hour,
rather than one per day. Distinguishes the Forecast's resolution from a coarser day-level
outlook.

### Time Period
A single hour within the Forecast, carrying its own set of weather Attributes. The unit the
End User selects between in the Weather View; the current-hour Time Period is the default
(the Current Conditions).

## Weather attributes

The four measures shown for each Time Period. Collectively the "key weather information"
the product reports; nothing outside this set is displayed.

### Temperature
The air temperature for a Time Period.

### Wind Speed
The speed of the wind for a Time Period.

### Chance of Rain
The probability of rain for a Time Period, expressed as a percentage. A likelihood, not a
measured rainfall amount.

### Weather Condition
The overall state of the weather for a Time Period (e.g. clear, cloudy, rain), shown to the
End User as an icon. The qualitative summary that accompanies the three numeric Attributes
(Temperature, Wind Speed, Chance of Rain).
