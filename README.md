# kite-lcoe

**Optimizing Static-Kite AWES Models for Individual Locations to Minimize LCOE**

A flexible modeling framework to optimize the placement and design of static kite-based airborne wind
energy systems (AWES). Given a kite design and a location, it estimates the Levelized Cost of Energy
(LCOE) and finds the design parameters that minimize it.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration](#configuration)
- [Assumptions](#assumptions)
- [Limitations](#limitations)
- [Data Sources](#data-sources)
- [License](#license)

## Overview

Airborne Wind Energy Systems (AWES) are an emerging class of renewable energy technologies designed to
harness high-altitude winds using tethered flying devices. They can offer stronger and more consistent
winds than tower-mounted turbines, often with reduced material use, lower noise, less risk to flying
wildlife and a smaller land footprint.

This project focuses on **static kites**: a kite that, at any particular moment, remains
aerodynamically stable at a fixed elevation and angle, generating power through turbines attached to the
kite or the tether. Unlike crosswind or dynamic kite systems, these are mechanically simpler, with fewer
moving parts and reduced control requirements.

**Features**

- Builds altitude-dependent wind speed and air density functions from Copernicus Climate Data Store (ERA5) data
- Simulates the kite and tether system to determine feasible flight conditions and power output
- Models location-based costs, such as onshore vs. offshore installation
- Minimizes LCOE over user-defined parameters (currently maximum tether length)
- Can be run over a grid of locations and plotted as a map
- Designed to be customized with a company's own kite, power and cost models

The climate data is global, so the framework can theoretically be applied anywhere. You choose the
locations and times to evaluate. Depending on how companies model costs, cost data may only be available
in limited regions.

## How It Works

1. **Climate data.** Altitude is calculated for each pressure level with the hypsometric equation, air
   density with the ideal gas law, and wind speed from the wind components. A cubic fit models air
   density against altitude, and a degree 6 polynomial models wind speed (an exponential profile
   deviates too much, especially under 1 km).
2. **Kite position.** The tether angle at which the net force on the kite (gravity, kite lift and drag,
   turbine lift and drag) points along the tether is found by minimization. The tether is then modeled
   as 10 segments (configurable) balancing tension, gravity and wind drag, to find the kite altitude
   for which the tether base is as close to the ground as possible. If no valid altitude exists, the
   kite would fall, and power for that time period is 0.
3. **Power.** For this project, power is proportional to the component of wind speed perpendicular to
   the kite cubed, multiplied by kite area and air density. It is 0 if no valid altitude is found or if
   the altitude exceeds a set maximum.
4. **Cost.** Component costs are summed, with the base structure cost multiplied by 5 if the site is
   offshore (determined from Natural Earth land polygons).
5. **LCOE and optimization.** Average power over the times you specify is multiplied by the expected
   operational hours and divided by lifetime costs to give the LCOE, which SciPy then minimizes over
   the chosen parameters at each location.

## Project Structure

```
.
├── src/kite_lcoe/
│   ├── __init__.py
│   ├── config.py
│   ├── climate.py
│   ├── kite_model.py
│   ├── energy.py
│   ├── cost.py
│   ├── land.py
│   └── lcoe.py
├── scripts/
│   ├── download_era5.py
│   ├── run_grid.py
│   └── plot_map.py
├── tests/
│   ├── test_model.py
│   └── test_land.py
├── data/
│   ├── README.md
│   └── .gitkeep
├── results/
│   └── .gitkeep
├── pyproject.toml
├── .gitignore
├── LICENSE
└── README.md
```

### Library (`src/kite_lcoe/`)

| File | Purpose |
|---|---|
| `__init__.py` | Marks the folder as a Python package and stores the version number. |
| `config.py` | All default parameters in one place: kite area and mass, tether properties, maximum altitude, efficiency, polynomial fit degrees, every component cost, operational hours, and the optimizer's starting guess. Edit this file to model a different system. |
| `climate.py` | Opens the ERA5 GRIB files and, for a given location and time, converts pressure, temperature and wind components into fitted wind speed and air density functions of altitude (`ClimateData.altitude_profiles`). Includes an optional plotting helper to check the fits. |
| `kite_model.py` | The kite and tether physics. Contains the lift and drag coefficient functions, the turbine force model, vector summation, and `find_alt_and_theta`, which returns the stable kite altitude, tether angle, maximum tension and a validity measure. Replace the aerodynamic functions with your own kite's data. |
| `energy.py` | `get_energy` computes power output at one location, tether length and time (0 if no valid flight). `get_average_energy` averages it over the sampled times, using the best tether length at each time. |
| `cost.py` | `get_cost` returns total system cost for a given maximum tether length and land/offshore flag. |
| `land.py` | `LandMask` loads a Natural Earth shapefile and reports whether a coordinate is on land. |
| `lcoe.py` | `get_lcoe` ties everything together: it computes LCOE and minimizes it over the chosen parameters. Add more design parameters in its `objective` function. |

### Scripts (`scripts/`)

| File | Purpose |
|---|---|
| `download_era5.py` | Downloads the two ERA5 datasets from the Climate Data Store (requires a free account and API key). |
| `run_grid.py` | Runs the LCOE calculation over a latitude/longitude grid and writes the results to a text file, two lines per point (coordinates, then LCOE). The grid bounds, spacing and times to average over are supplied by the user. |
| `plot_map.py` | Reads the results file and draws a map of colored points (green to red up to 30 ¢/kWh, dark grey above). |

### Tests (`tests/`)

| File | Purpose |
|---|---|
| `test_model.py` | Smoke tests that need no external data: vector summation, offshore vs. onshore cost, cost growth with tether length, and a valid output from the kite model on a synthetic wind profile. |
| `test_land.py` | Checks land vs. sea classification for known coordinates. Skipped unless the `KITE_LCOE_LAND_SHAPEFILE` environment variable points to a land shapefile. |

### Other files

| File | Purpose |
|---|---|
| `data/README.md` | Describes the three input files the project needs. The `data/` folder is a suggested place to keep them; its contents are git-ignored. |
| `data/.gitkeep`, `results/.gitkeep` | Empty placeholders so Git keeps the otherwise-empty (git-ignored) folders. |
| `pyproject.toml` | Package metadata, dependencies and optional extras (`download`, `dev`). |
| `.gitignore` | Keeps large data files (GRIB), generated results, IDE files and Python caches out of the repository. |
| `LICENSE` | MIT license. |

## Installation

```bash
git clone <repo-url>
cd <repo-folder>
pip install -e .
```

Optional extras: `pip install -e ".[download]"` for the CDS downloader and `pip install -e ".[dev]"` for
`pytest`.

Reading GRIB files with `cfgrib` requires the ECMWF ecCodes library. If `pip` does not provide it on your
platform, installing through conda is the easiest route:

```bash
conda install -c conda-forge cfgrib geopandas
```

## Usage

Three input files are needed (see [`data/README.md`](data/README.md)): an ERA5 pressure-level GRIB file,
an ERA5 single-level GRIB file, and a Natural Earth land-polygon shapefile. Paths are passed on the
command line.

**1. Download the climate data** (optional; needs a free CDS account and API key)

```bash
python scripts/download_era5.py \
    --pressure-out <pressure-level GRIB path> \
    --single-out <single-level GRIB path> \
    --area <north> <west> <south> <east> \
    --year <year(s)> --month <month(s)> --day <day(s)> --time <HH:MM time(s)>
```

**2. Compute LCOE over a grid of locations and times of your choice**

```bash
python scripts/run_grid.py \
    --pressure-file <pressure-level GRIB> \
    --single-file <single-level GRIB> \
    --land-shapefile <Natural Earth land shapefile> \
    --output <results text file> \
    --lat-min <deg> --lat-max <deg> --lon-min <deg> --lon-max <deg> --step <deg> \
    --times <YYYY-MM-DDTHH:MM> [<YYYY-MM-DDTHH:MM> ...]
```

The downloaded climate data must cover the grid and the times you specify.

**3. Plot the map**

```bash
python scripts/plot_map.py \
    --input <results text file> \
    --land-shapefile <Natural Earth land shapefile> \
    [--output <image path>]
```

Run any script with `--help` for all options.

**Running the tests**

```bash
pytest
```

## Configuration

| To change | Edit |
|---|---|
| Kite, tether and cost parameters | `src/kite_lcoe/config.py` |
| Lift/drag coefficients and turbine forces | `src/kite_lcoe/kite_model.py` |
| Power model | final expression in `get_energy`, `src/kite_lcoe/energy.py` |
| Cost model | `src/kite_lcoe/cost.py` |
| Parameters to optimize | `objective` in `src/kite_lcoe/lcoe.py` |

The default values are realistic (although sometimes semi-arbitrary) test inputs, many from patent
[US8602363B2](https://patents.google.com/patent/US8602363B2/en).

## Assumptions

| Assumption | Justification | How to fix (if necessary) |
|---|---|---|
| Molar mass of air is a constant | Virtually all modern kite systems operate well within the troposphere (bottom 12 km), where differences in air molar mass tend to be under 1% | Incorporate humidity data |
| Tether is non-elastic | Kite tethers are often made of materials that are not very elastic such as kevlar | Scale up the segment lengths as a function of the tension on them, the tension already being calculated |
| Turbine is at same altitude as kite | Higher altitude winds closer to the kite generate more energy | Use the model of the optimal tether to return the altitude and angle of the turbine |
| Mean virtual air temperature is the average of the air temperatures at the two data points | In non-tropical climates, a lack of extreme humidity usually results in less than 1% error (Guinn and Barry 2016) | Incorporate humidity data |

## Limitations

- Default cost values are placeholders (many from the patent above); replace them with your own cost
  models for meaningful absolute LCOE values.
- Average power is computed only from the times you supply, so sampling few times gives less
  meaningful averages. More times means longer run times.
- Large grids are computationally intensive, but grid points are independent, so the process can be
  parallelized by users with more computing resources.
- The degree 6 wind fit on a limited number of points can overfit.

## Data Sources

- [Climate Data Store](https://cds.climate.copernicus.eu/), Copernicus (ERA5)
- [Natural Earth](https://www.naturalearthdata.com/) land polygons

## License

Released under the MIT License. See [LICENSE](LICENSE).

## Author

Rishabh Gala. Developed as part of the Non-Trivial Fellowship.
