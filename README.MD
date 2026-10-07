# 🚑 Riyadh Emergency Access Agent

An AI assistant that answers emergency-access planning questions for Riyadh by running **graph algorithms on the real road network**, then explaining the results in plain language.

> *"Where should Riyadh add 3 new emergency centers so more of the city is within 10 minutes of a hospital, and which 5 roads matter most for reaching hospitals?"*

## The problem

In an emergency, minutes matter. Some neighborhoods are much farther from a hospital than others, and planners need to know:

- **Where are the gaps?** Which areas are beyond an acceptable drive time to care?
- **Which roads matter most?** Which roads do the most locations depend on to reach a hospital?
- **Where should new facilities go?** Which sites would bring the most people within reach?

LLMs alone can't answer these reliably: they invent numbers and can't compute routes. This project gives an LLM agent exact graph-algorithm tools, so every number comes from a real computation.

## How it works

```
START ──▶ graph_analyst ──▶ writer ──▶ END
```

| Agent | Tools | Job |
|---|---|---|
| **graph_analyst** | `critical_roads`, `suggest_new_sites` | Decides which tools the question needs, calls them, and reports the exact results |
| **writer** | none | Turns the findings into a short Markdown report for a non-technical planner |

Both agents are built with LangChain's `create_agent` and connected with LangGraph's `StateGraph`.

### Data

- **Roads:** Riyadh's drivable road network from OpenStreetMap (via OSMnx), as a directed graph. Nodes are intersections, and edges are road segments weighted by free-flow travel time estimated from speed limits.
- **Hospitals:** OpenStreetMap features tagged `amenity=hospital`, each snapped to the nearest intersection.
- **Study area:** a 10 km radius around central Riyadh (configurable). The first run downloads the data; later runs load it from `cache/`.

### Graph algorithms

1. **Multi-source Dijkstra (precomputed once).** The road graph is reversed and a zero-cost "super source" is linked to every hospital. One Dijkstra run then gives every intersection its fastest drive time to the nearest hospital.
2. **`critical_roads`: targeted betweenness.** Each intersection's fastest route to care forms a tree. Counting how many intersections route through each road ranks the roads that emergency access depends on most.
3. **`suggest_new_sites`: greedy maximal coverage.** 150 uncovered intersections are sampled as candidate sites. Dijkstra with a time cutoff finds which locations each candidate could reach, then the algorithm repeatedly picks the candidate that covers the most still-uncovered locations.

### Settings
Change these in the setup cell:

| Setting | Default | Meaning |
|---|---|---|
| `CENTER` | `(24.7136, 46.6753)` | Center of the study area (lat, lon) |
| `RADIUS_M` | `10_000` | Study-area radius in meters |
| `DEFAULT_THRESHOLD_MIN` | `10` | Default acceptable drive time to a hospital |
| `MODEL_NAME` | `nvidia/nemotron-3.5-lightning:free` | Any OpenRouter model that supports tool calling |

## Ask your own question

```python
ask('Which 10 roads matter most for reaching hospitals?')
ask('Where should 2 new emergency centers go to bring more of the city within 5 minutes of care?')
```

The notebook also draws a map of the hospitals, critical roads and suggested sites.

## Notebook structure

| Section | Contents |
|---|---|
| 1. Setup | Imports, settings, LLM |
| 2. Data | Download or load roads and hospitals |
| 3. Graph algorithms | Dijkstra precompute, critical roads, site selection |
| 4. Tools | `@tool` wrappers the agent can call |
| 5. Agents | `graph_analyst` and `writer` |
| 6. LangGraph pipeline | State, nodes, edges |
| 7. Run it | `ask()` helper and an example question |
| 8. Map | Folium map of the results |

## Limitations

- **No traffic.** Drive times are free-flow estimates from speed limits; real rush-hour times in Riyadh can be much longer.
- **No population weighting.** Every intersection counts equally, so a dense neighborhood and an empty industrial area have the same weight.
- **All hospitals are treated the same.** A clinic counts the same as a trauma center, and capability (stroke, cardiac, pediatric) isn't modeled.
- **OpenStreetMap data is community-made.** Some hospitals may be missing or mis-tagged.
- **"Critical" means many routes pass through a road**, not that closing it would cut access. A nearby parallel street may make a closure harmless.
- **Site selection is approximate.** Greedy search over 150 sampled candidates, with a fixed random seed. A different sample can give different sites.
- **Results haven't been validated** against real travel times or ambulance data.

## Next steps

- Weight intersections by population (WorldPop or GHSL data).
- Add time-of-day traffic factors to edge travel times.
- Tag hospitals by capability and run the analysis per emergency type.
- Add a road-closure simulation tool to measure how "critical" a road really is.
- Compare greedy site selection against an exact solver on a small area.

---

Submitted by: <your name> — academy: @SDAIAAcademy