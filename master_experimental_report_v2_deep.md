# Master Experimental Report V2 Deep Integrated Edition: Truck-Drone EVRPTW-NL

> This integrated edition replaces the earlier front-loaded deep supplement. Development evidence is now embedded into the final method descriptions, RQ analysis, negative results, and discussion. No solver was rerun and no algorithm code was modified.

## Evidence Coverage Statement

This report uses three evidence levels. **Controlled final evidence** refers to the final 25-customer 108-run matrix and is the only basis for formal performance claims. **Developmental evidence** refers to 5/10/25/50-customer stage experiments, method logs, Hybrid audits, and smoke tests; it explains why the algorithms evolved but is not treated as a controlled ablation unless explicitly stated. **Diagnostic evidence** refers to operator calls, accepted/improved counts, violation fields, runtime outliers, candidate diagnostics, and petal/crossing metrics; it is used to explain mechanisms. The main sources are `final_model_freeze_spec.md`, final code, final CSV/JSONL results, `ga_development_report.md`, `alns_development_report.md`, `hybrid_development_report.md`, `hybrid_stage6_report.md`, `hybrid_final_audit.md`, `hybrid_final_25_smoke_audit.md`, `petal_route_design_report.md`, and `modeling_and_progress.md`.

## 1. Executive Summary

The research focus is **constraint-aware metaheuristic solution and computational analysis for a unified Truck-Drone EVRPTW-NL**. The final problem couples truck routing, drone service assignment, hard time windows, truck and drone energy propagation, launch-recovery synchronization, charging-station decisions, and four charging policies: LFC, LPC, NFC, and NPC.

The final 25-customer controlled experiment contains 108 runs across C101/R101/RC101, GA/ALNS/Hybrid, LFC/LPC/NFC/NPC, and three seeds. In the controlled final evidence, ALNS has the highest feasible rate at 75.0%, Hybrid follows at 72.222%, and GA reaches 66.667%. GA uses fewer feasible vehicles on average (7.666667) than Hybrid (7.884615) and ALNS (8.888889). ALNS obtains the shortest feasible total distance on average (874.735865), while Hybrid obtains the lowest feasible paper cost (1232.753151) and the most stable runtime median/mean profile (63.404685s / 62.724606s). These results do not support a universal winner; they support a search-behavior interpretation.

Developmental evidence explains why. Early algorithms generated rough routes and relied heavily on shared simulation/evaluation. That made feasibility less attributable to each algorithm and weakened Hybrid complementarity. GA therefore moved toward structured representation and constraint-aware decoding. ALNS moved from generic removal/insertion toward time, energy, drone, charging, spatial, and local diagnostic operators. Hybrid moved from naive GA-best refinement to Top-K, vehicle-preserving refinement, periodic/stagnation triggering, local neighborhoods, and finally diverse candidate generation. The final 108-run matrix tests these accumulated design decisions; it does not erase the negative development evidence. Hybrid is competitive, especially in paper cost, but it still does not dominate GA or ALNS.

## 2. Research Questions

RQ1 asks how GA and ALNS became constraint-aware rather than generic routing methods. RQ2 compares final search behavior and trade-offs. RQ3 studies C/R/RC sensitivity. RQ4 studies the 2 x 2 charging-policy design. RQ5 studies when Hybrid improves or fails. RQ6 studies computational robustness and runtime heavy tails. These questions are interpreted through the three evidence levels defined above.

## 3. Literature and Research Context

The project is grounded in Solomon VRPTW benchmarks, Schneider E-VRPTW, ETRD-NL, classic ALNS, genetic algorithms for routing, truck-drone routing, and EVRP charging literature. Current project evidence supplies the implementation facts; external literature supplies conceptual motivation for time windows, electric routing, nonlinear/partial charging, truck-drone synchronization, GA population search, ALNS neighborhood search, and hybrid metaheuristics. Bibliographic gaps are documented in `reference_gaps.md`; unresolved literature items remain marked `[REFERENCE NEEDED]` rather than fabricated.

## 4. Problem Motivation

A classical VRP decides customer visit order. VRPTW adds hard service intervals, so distance-short routes can become infeasible. EVRPTW adds battery state and charging-station decisions, so route feasibility depends on energy propagation and charging time. Truck-drone routing adds service assignment, launch and recovery nodes, drone-route construction, and synchronization. Nonlinear and partial charging add charging amount and SOC-dependent charging time. In this project these additions are not independent. Moving one customer can change truck arrival times, drone launch feasibility, drone SOC, charging needs, recovery waiting, and all downstream time-window feasibility. This is why early `route first + repair later` behavior was not enough and why the final algorithms had to move constraint awareness inside search.

## 5. Final Mathematical Model and Coupling Logic

The final model is conceptual rather than a MILP implementation: the actual solvers use structured solution objects, route simulation, and evaluator checks. A solution is represented as \(S=(R,M,G)\), where \(R\) is the set of truck routes, \(M\) the set of same-route drone missions, and \(G\) the truck/drone charging plan. Customers \(C\), charging stations \(F\), depot \(0\), truck routes \(r\), drone missions \(m\), and nodes \(N=C \cup F \cup \{0\}\) form the main sets.

Customer coverage is hard: every customer must be served exactly once by either truck or drone, written conceptually as \(x_i^{truck}+x_i^{drone}=1, \forall i\in C\). Without this constraint, a solver could reduce distance by omitting customers or duplicating service. Truck routes start and end at the depot: \(R_r=(0,...,0)\). Drone missions are same-route missions: \(m=(r, l, P_m, q)\), where launch \(l\) and recovery \(q\) both appear in truck route \(R_r\), launch precedes recovery, and \(P_m\) can contain multiple customers and charging stations. Cross-truck recovery is not part of the frozen model.

Time propagation is causal. If vehicle \(v\) travels from node \(i\) to \(j\), arrival is \(a_j^v=d_i^v+t_{ij}^v\), service starts at \(b_j^v=\max(a_j^v,e_j)\), and hard time-window feasibility requires \(b_j^v\le l_j\). Waiting is recorded but not itself infeasible unless it pushes downstream service past due times. Truck-drone synchronization at recovery is \(b_q^{sync}=\max(a_q^{truck},a_q^{drone})\), producing truck wait or drone wait. This constraint exists because the truck cannot continue with the drone unavailable.

Energy propagation is also causal. For truck or drone \(v\), SOC follows \(E_j^v=E_i^v-\rho_v d_{ij}+g_i^v\), with \(0\le E_j^v\le B_v\). A charging station visit can add \(g_i^v\), but only at existing generated station nodes. Capacity constraints require truck load and drone mission load to remain within configured capacity. Violations are checked by the shared simulator/evaluator; algorithms may anticipate them, but final feasibility is always judged by the evaluator.

The key coupling is sequential: customer assignment changes truck/drone route structure; route structure changes arrival times; arrival times change launch/recovery feasibility; launch/recovery timing changes synchronization waiting; waiting changes downstream time windows; route legs change SOC; SOC changes charging decisions; charging time again changes time windows. This coupling is the technical reason the final GA, ALNS, and Hybrid designs cannot be reduced to ordinary distance minimization.

## 6. Mathematical Charging Model

The frozen charging model treats LFC/LPC/NFC/NPC as a 2 x 2 design: linear vs nonlinear and full vs partial. Full charging sets target energy to capacity. Partial charging sets target energy to a required-energy estimate plus margin. Linear charging uses constant-rate cumulative time, so charging time is proportional to added energy. Nonlinear charging is implemented as an engineering segmented SOC approximation: charging in high SOC intervals is slower, so cumulative time increases faster near full charge.

| | Full target | Partial target |
|---|---|---|
| Linear | LFC | LPC |
| Nonlinear | NFC | NPC |

This model explains the final RQ4 pattern. In final controlled evidence, ALNS charging time falls from LFC 166.309045 to LPC 80.751952, and from NFC 242.731195 to NPC 76.090921. GA and Hybrid show the same partial-charging direction: GA LFC/LPC/NFC/NPC charging times are 200.759/110.421/248.166/124.416, while Hybrid values are 157.640/97.580/287.642/101.119. The mechanism is consistent with the model: partial policies avoid unnecessary high-SOC charging, especially under nonlinear charging. This is controlled final evidence for charging-time behavior, not proof of global optimality.

## 7. Unified Computational Framework

All final methods use the same final solution schema, simulator, evaluator, and CSV pipeline. This shared evaluator is important for fairness: GA, ALNS, and Hybrid can make different construction choices, but they are judged by the same feasibility and metric definitions. It also explains why early development had to move constraints into the algorithms. If all methods merely generated rough structures and then relied on the same downstream checks, method-specific search behavior became difficult to distinguish.

## 8. Final GA Method with Integrated Evolution

### 8.1 Structured representation and why it exists

Final GA represents customer order, service modes, truck route splits, drone priorities, charging policy preference, and multi-route solution structure. This component exists because the early customer-order GA could not express the final problem. A pure customer permutation cannot represent whether customer \(i\) is served by truck or drone, which route launches the drone, where recovery occurs, whether a drone route visits several customers, or where truck/drone charging occurs. Developmental evidence therefore motivated the transition from `customer_order` alone to `truck_routes`, `drone_tasks`, `charging_plan`, `service_mode`, `route_split_bias`, and drone-priority information. The final behavior in RQ2 is consistent with this design: GA has the lowest average feasible vehicle count (7.666667), which reflects its constructive route-splitting and service-assignment bias.

### 8.2 Constraint-aware decoder and why it exists

The final GA decoder constructs routes while considering time windows, truck energy, charging, drone feasibility, and synchronization. This component exists because early search-first/repair-later behavior produced structurally weak candidates. Development logs record that early R101-5 GA+NPC had only 40% time-window feasibility and total violation about 3.086 before awareness improved. That evidence is developmental, not a controlled ablation, because other settings evolved. It still explains the design decision: if feasibility is only discovered after a route is built, crossover and mutation spend many evaluations in irrelevant infeasible neighborhoods. Moving TW/Energy/Charging/Drone/Sync checks into decoding creates candidates that the evaluator can meaningfully compare.

### 8.3 Multi-customer drone and charging mechanism

Final GA supports multi-customer drone missions and drone station visits under the frozen same-route launch/recovery rule. This exists because early single-customer drone insertion was judged decorative: it could show a drone line on a plot but did not sufficiently alter truck workload. The final drone route `[launch, customer..., station..., recover]` gives GA a real service-assignment mechanism. The final controlled evidence shows GA feasible drone distance mean 101.991886, indicating drone missions are present in feasible solutions, although this alone does not prove they always improve cost.

### 8.4 Diversity generation and the Hybrid role

Final GA also acts as a diverse promising-start generator for Hybrid. This role did not exist in the earliest standalone GA objective. It emerged after Hybrid development showed that GA-best was not necessarily the best ALNS starting point and quality Top-K candidates could still be structurally homogeneous. Diverse candidate types such as distance-oriented, vehicle-oriented, time-window-oriented, drone-aggressive, drone-conservative, charging-oriented, petal-oriented, and balanced were introduced to give ALNS different neighborhoods. The 10-customer debug evidence supports this direction: `hybrid_diverse_topk` reached 100% feasibility, 2.667 average vehicles, 344.074 average distance, 494.442 paper cost, and 4.584s runtime. The later 25-customer smoke evidence warns that this did not become universal dominance.

## 9. Final ALNS Method with Integrated Evolution

Final ALNS uses an explicit state, independent initial construction, destroy/repair/local-search operators, layered candidate screening, full evaluator validation, and diagnostics. Its final operator families are not arbitrary. D-WorstTime and R-TWAware exist because early failures exposed time-window pressure. D-ChargingCritical, R-EnergyAware, and charging cleanup exist because generic insertion cannot reason about battery infeasibility and charging delay. DroneReassign and DroneRebuild exist because service assignment, launch/recovery, and synchronization are central coupling constraints. Crossing and petal operators exist because route geometry issues were observed during development and route quality could not be captured by distance alone.

The final controlled evidence shows ALNS has the highest feasible rate (75.0%) and the shortest feasible total distance mean (874.735865), but also the highest feasible vehicle count mean (8.888889) and highest paper cost mean (1319.028256). This matches its development history. ALNS was strengthened as a feasibility and local-restructuring method; the same local freedom can reduce distance while accepting more routes. Earlier Hybrid audits had already exposed this vehicle-distance conflict when ALNS refined a GA solution to shorter distance but more vehicles.

Layered evaluation was introduced after full evaluator calls became a runtime bottleneck. Developmental evidence includes R101-25 ALNS core cases around 45-47s with hundreds of evaluator calls and later runtime-pathology discussions. The final runtime evidence still shows ALNS heavy tails: median 90.530s, mean 456.634s, max 10929.262s, with two runs above 1000s. Thus the final ALNS is paper-comparable but computationally less stable than Hybrid.

## 10. ALNS Operator Analysis: Motivation, Outcome, Interpretation

| operator_name | operator_type | runs | calls | accepted_results | improved_results | acceptance_rate | improvement_rate | average_runtime_weighted | effectiveness_class |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| D-DroneTask | destroy | 588 | 10701 | 8370 | 375 | 0.78217 | 0.035043 | 0.283368 | High-frequency / high-value |
| D-WorstTime | destroy | 610 | 7530 | 4396 | 210 | 0.583798 | 0.027888 | 0.14871 | High-frequency / high-value |
| D-ChargingCritical | destroy | 501 | 5350 | 2615 | 193 | 0.488785 | 0.036075 | 0.325508 | High-frequency / high-value |
| D-Crossing | destroy | 536 | 4815 | 2436 | 171 | 0.505919 | 0.035514 | 0.757803 | High-frequency / high-value |
| D-SyncCritical | destroy | 153 | 2086 | 1603 | 31 | 0.768456 | 0.014861 | 0.304434 | High-frequency / high-value |
| D-RouteRemoval | destroy | 237 | 4225 | 3377 | 28 | 0.79929 | 0.006627 | 0.156639 | High-frequency / low-value |
| D-AngleSector | destroy | 150 | 1498 | 938 | 27 | 0.626168 | 0.018024 | 0.363083 | High-frequency / high-value |
| D-Cluster | destroy | 157 | 1094 | 598 | 20 | 0.546618 | 0.018282 | 10.38884 | High-frequency / high-value |
| D-Random | destroy | 249 | 2717 | 1871 | 19 | 0.688627 | 0.006993 | 0.163954 | High-frequency / low-value |
| LS-DroneRebuildV2 | local_search | 353 | 18998 | 18550 | 2791 | 0.976419 | 0.14691 | 0.053495 | High-frequency / high-value |
| H-DroneReassign | local_search | 291 | 9098 | 9098 | 1433 | 1.0 | 0.157507 | 0.23665 | High-frequency / high-value |
| LS-DroneRebuild | local_search | 49 | 7320 | 7232 | 812 | 0.987978 | 0.110929 | 0.005128 | High-frequency / high-value |
| LS-RouteMergeV2 | local_search | 240 | 13701 | 13701 | 393 | 1.0 | 0.028684 | 1.2141 | High-frequency / high-value |
| LS-RelocateV2 | local_search | 389 | 21738 | 21738 | 254 | 1.0 | 0.011685 | 0.040734 | High-frequency / high-value |
| H-CrossRouteRelocateNoNewVehicle | local_search | 291 | 9098 | 9098 | 121 | 1.0 | 0.0133 | 0.00919 | High-frequency / high-value |
| LS-CrossingRemoval | local_search | 350 | 18261 | 18261 | 65 | 1.0 | 0.003559 | 0.019963 | High-frequency / low-value |
| LS-Relocate | local_search | 49 | 7320 | 7320 | 31 | 1.0 | 0.004235 | 0.011988 | High-frequency / low-value |
| H-SwapSameVehicle | local_search | 291 | 9098 | 9098 | 9 | 1.0 | 0.000989 | 0.007393 | High-frequency / low-value |
| H-RelocateSameVehicle | local_search | 291 | 9098 | 9098 | 7 | 1.0 | 0.000769 | 0.00805 | High-frequency / low-value |
| H-LaunchRecoverAdjust | local_search | 291 | 9098 | 9098 | 1 | 1.0 | 0.00011 | 0.001167 | High-frequency / low-value |

The operator diagnostics must be read as diagnostic evidence, not final performance proof. Operators with high call counts are not automatically useful; the relevant question is whether calls become accepted or improved solutions under the final ranking. Drone-aware operators were added because development repeatedly identified drone assignment and synchronization as coupled constraints. When DroneReassign or DroneRebuild variants show nonzero improvement, that supports the search-landscape interpretation that customer service mode and launch/recovery structure are meaningful neighborhoods. Conversely, ChargingPolish, WaitingReduction, PetalPolish, or RouteMerge variants can have low improvement even though they were theoretically motivated. This does not make their original hypotheses irrational; it shows that trigger conditions may be rare, other operators may already handle the improvement, or the move improves a secondary metric without improving the final ranking. In the paper, such operators should be reported as negative diagnostic findings, not hidden.

## 11. Final Hybrid GA-ALNS with Integrated Evolution

Final Hybrid consists of GA candidate generation, diversity-aware Top-K selection, conversion to ALNS state, ALNS refinement, vehicle-preserving acceptance, paper-cost/ranking comparison, and final evaluator validation. Every component corresponds to an observed historical failure.

| Final Hybrid component | Historical failure that motivated it | Final role |
|---|---|---|
| Top-K candidates | Single GA-best refinement was unstable | Test multiple promising neighborhoods |
| Candidate diversity | Quality Top-K remained structurally similar | Expose ALNS to different route/service patterns |
| Vehicle-preserving rule | ALNS sometimes reduced distance by adding vehicles | Prevent misleading improvement under vehicle-priority ranking |
| Paper-cost priority | Local improvement did not always match paper objective | Align refinement acceptance with final reporting metric |
| Stagnation/periodic tests | End-only refinement might be too late | Tested timing, then showed timing alone was not enough |
| Hybrid-local neighborhoods | Generic ALNS could be too disruptive | Refine near GA solutions without increasing vehicles |

Developmental evidence gives the causal chain. Naive Hybrid retained GA on R101-5, improved R101-10 by about 0.64%, and on R101-25 encountered the shorter-distance/higher-vehicle conflict. Top-K then selected rank-3 candidates in R101-10 and R101-25, showing GA rank was not identical to ALNS refinement potential. Vehicle-preserving refinement rejected high-vehicle short-distance candidates and accepted same-vehicle improvements in R101-10. Periodic and stagnation triggering showed that timing alone was not the bottleneck: R101-25 periodic ran about 97.2s with zero injections, and stagnation triggered but also injected zero. Hybrid-local neighborhoods then attempted safer local refinement, but Stage6 debug still showed `hybrid_topk` distance 371.125 and `hybrid_preserve` 358.810 compared with GA 349.970 under that debug setting. Finally diverse Top-K improved 10-customer debug behavior but 25-customer smoke evidence still showed Hybrid could trade fewer distance for more vehicles.

Final controlled evidence confirms conditional effectiveness rather than dominance. Hybrid feasible rate is 72.222%, between GA and ALNS. Hybrid paper cost mean is 1232.753151, lower than GA 1262.172734 and ALNS 1319.028256. Hybrid wins/ties/losses are 18/5/13 against GA and 25/0/11 against ALNS. Therefore the final claim should be: Hybrid is a competitive conditional synthesis with strong paper-cost behavior and stable runtime, not a universally superior algorithm.

## 12. Algorithm Development History as Causal Evidence

| Stage | Method | Observed problem | Hypothesis | Modification | Experiment | Before result | After result | Observed improvement | Observed degradation | Final status |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| GA Stage 0 | GA | Customer-order-only encoding could not express truck/drone assignment, launch/recovery, charging, or multi-vehicle structure. | A richer solution representation is required before GA can search the final Truck-Drone EVRPTW-NL space. | Move from simple customer_order to structured solution with truck_routes, drone_tasks, charging_plan. | Developmental evidence - not controlled ablation | Early GA produced infeasible cases; time-window feasibility was reported around 20-40% in user-provided diagnostics. | Later GA output could represent multi-route truck-drone charging-capable solutions. | Representation no longer blocked drone/charging/multi-vehicle modeling. | Larger search space increased decoder complexity. | Superseded by final structured GA |
| GA Stage 1-3 | GA | Drone tasks were decorative when limited to single-customer missions without station visits. | Allowing multi-customer drone_route and drone charging should make drone service a real routing decision. | Upgrade drone task to route_index + launch + drone_route + recover + customers; allow station nodes and charging_plan vehicle='drone'. | Developmental evidence - not controlled ablation | Single-customer sortie could not express realistic drone contribution. | Final raw results contain drone_tasks and truck/drone distances in 25-customer solutions. | Drone service became representable and evaluable. | Drone construction required pruning to avoid runtime explosion. | Retained in final method |
| GA Stage 4-8 | GA | Multi-customer drone and charging search created a large candidate space and could threaten runtime. | Hierarchical pruning, evaluation cache, route rebalance, specialized mutation, route split, and diversity anchors can keep GA usable at 25/50 customers. | Add layered drone search, caching, Truck-Drone mutations, incremental vehicle expansion, route-preserving crossover, and time budget. | Developmental evidence - not controlled ablation | Unbounded drone construction risked slow or infeasible outputs. | GA development report states GA can generate feasible 25/50-customer solutions, but remains conservative. | Feasibility and scale handling improved. | Vehicles and distances can be conservative. | Retained in final GA |
| ALNS Phase 1 | ALNS | Initial ALNS scaffold was not a real independent ALNS; it could not explain destroy/repair/local operator effects. | An explicit ALNSState and independent initial construction can make ALNS a fair standalone comparison method. | Add ALNSState, initial solution, D-Random/D-WorstTime/D-RouteRemoval/D-DroneTask, R-Regret/TW/Energy/Drone, and local operators. | Developmental evidence - not controlled ablation | Scaffold behavior was not comparable to final GA. | R101-5 True, 3 vehicles, distance 161.151, runtime 0.845s; R101-10 True, 3 vehicles, distance 240.500, runtime 5.497s; C101-5 and RC101-5 also True. | ALNS became independently runnable and feasible on small cases. | Quality and scale robustness were not yet sufficient. | Modified later |
| ALNS layered/petal/post-stage | ALNS | ALNS was feasible but conservative; full evaluator calls and many weak operators increased runtime. | Layered candidate evaluation and targeted operators can improve distance, crossing, drone, charging, and vehicle behavior without uncontrolled runtime. | Add route merge V2, crossing removal, drone rebuild, charging cleanup, vehicle reduction, spatial/petal metrics, and quick/local/full evaluation layers. | Developmental evidence - not controlled ablation | R101-25 alns_core True, 9 vehicles, distance 710.524, runtime 47.507s, evaluator calls 965. | Later R101-25 alns_core True, 9 vehicles, distance 643.225, runtime 45.195s, crossing successes 7; post-stage alns_full True, 10 vehicles, distance 780.638, drone tasks 4 in one snapshot. | Crossing and candidate filtering could improve route structure and distance in some profiles. | Vehicle count and drone contribution remained uneven; some improvements traded distance, vehicles, and waiting. | Retained in final ALNS with limitations |
| Hybrid Stage 1 | Hybrid | Simple GA best -> ALNS refine often failed to improve final accepted solution. | GA global search plus ALNS local refinement should be complementary. | Implement single-best post-processing from GA solution to ALNS state. | Developmental evidence - not controlled ablation | No true integration; selector simply chose GA or ALNS. | R101-5 retained GA because ALNS used more vehicles; R101-10 ALNS refine replaced GA with improvement_percentage about 0.64%; R101-25 retained GA because ALNS distance was shorter but vehicles higher. | Real conversion/refinement worked and sometimes improved. | Vehicle-count side effect blocked many refinements. | Superseded by Top-K |
| Hybrid Stage 2-2.5 | Hybrid | GA rank-1 was not always the best ALNS starting point; ALNS could reduce distance by adding vehicles. | Diverse Top-K plus vehicle-preserving refinement improves robustness. | select_diverse_top_k and preserve_vehicle_count. | Developmental evidence - not controlled ablation | Single-best refine was blocked by vehicle increases. | R101-10 Top-K selected rank 3 GA candidate; R101-25 Top-K selected rank 3; preserve kept R101-5 and R101-25 vehicle counts while accepting same-vehicle improvement on R101-10. | Candidate choice and ranking became more defensible. | Preserve rule can reject distance gains. | Retained in final Hybrid logic |
| Hybrid Stage 3-6 | Hybrid | Post-processing might be too late, but fixed ALNS triggering did not guarantee injection. | Periodic/stagnation ALNS and hybrid-local operators can inject improvements during candidate evolution. | Periodic elite improvement, stagnation-triggered ALNS, H-local operators. | Developmental evidence - not controlled ablation | Top-K alone could be limited. | R101-5 periodic injected 1 candidate; R101-25 periodic completed in about 97.2s but injection count was 0. Stagnation R101-10/25 triggered but injected 0. Stage6 summary: GA 100% feasible, 3.000 vehicles, distance 349.970; hybrid_topk 371.125; hybrid_preserve 358.810. | Hybrid-local successes existed but did not consistently change final solution. | Runtime increased and injection was often zero. | Supporting evidence |
| Hybrid Stage 7-9 | Hybrid | GA candidate pool was too homogeneous; ALNS refined similar neighborhoods. | Multi-type high-diversity candidates plus paper-cost priority can give ALNS better starts and make final ranking consistent. | distance/vehicle/TW/drone/charging/petal/balanced candidate types, hybrid_diverse_topk, paper_cost_priority. | Developmental evidence plus final controlled evidence | C101-10 hybrid_diverse_topk once had about 532s runtime before fix. | Debug: hybrid_diverse_topk 100% feasible, 2.667 vehicles, distance 344.074, paper cost 494.442, runtime 4.584s; final 108-run matrix retained for controlled conclusions. | Diverse Hybrid became the final method and fixed a major runtime pathology in debug. | 25-customer smoke: Hybrid had 7 vehicles and distance 812.254 vs GA 7 vehicles and 810.202 in one case; final audit kept non-dominance caution. | Retained in final Hybrid |
| Petal/spatial development | GA/ALNS/Hybrid metrics | Distance and vehicle count alone did not describe route shape; routes could cross or overlap spatially. | Petal/spatial soft metrics help diagnose and guide route quality without becoming hard constraints. | Add petal_score, crossing_count, route_compactness, sector_coherence, depot_radial_consistency. | Developmental evidence - not controlled ablation | R101-25 alns_core crossing_count 19 and petal_score 967.846 in one report. | Updated alns_full snapshot showed crossing_count 8 and petal_score 411.927 but vehicle_count 10 and distance 780.638. | Spatial quality improved in some snapshots. | Better spatial shape can trade off against vehicles or distance. | Retained as soft metric/diagnostic |

This matrix is developmental evidence. It explains design decisions but does not replace the controlled 108-run experiment. Its most important lesson is that each final component was introduced after a concrete failure: route representation after expression limits, constraint-aware decoding after feasibility failures, drone charging after decorative drone tasks, layered evaluation after runtime pressure, vehicle preservation after ALNS vehicle inflation, and diverse Top-K after Hybrid candidate homogeneity.

## 13. Experimental Design and Data Quality

The formal main experiment is the final 25-customer matrix: C101/R101/RC101, methods GA/ALNS/Hybrid, charging policies LFC/LPC/NFC/NPC, seeds 1987/42/128, station count 8, and nominal time budget 90s, for 108 runs if the CSV is complete. Supporting experiments at 5/10/25/50 customers are used for mechanism validation and method development, not mixed into final averages. Truck-only OR-Tools/PyVRP references remain historical because they do not solve the complete Truck-Drone EVRPTW-NL.

Data quality issues are retained rather than deleted:

| detected issue | affected rows | whether corrected | correction rule |
| --- | --- | --- | --- |
| main row count | 108 | not corrected | Expected 108 from config; observed 108. |
| duplicate runs | 0 | not corrected | Duplicates are reported, not deleted. |
| missing expected combinations | 0 | not corrected | No imputation. |
| infeasible runs | 31 | not corrected | Retained for feasible-rate analysis. |
| runtime > 500s | 4 | not corrected | Outliers retained. |
| feasible-total_violation inconsistency | 0 | not corrected | Reported only. |

## 14. RQ1 Constraint Adaptation Evidence

Early state: feasibility was heavily downstream. Generic customer ordering and generic destroy/repair could create routes, but many Truck-Drone EVRPTW-NL constraints were only discovered during simulation/evaluation. Developmental evidence includes early GA time-window failures and ALNS scaffold limitations. This motivated moving constraints inside search: GA added service modes, route splits, drone priorities, and TW/Energy/Charging/Sync-aware decoding; ALNS added explicit state, problem-aware operators, and layered feasibility estimation; Hybrid added vehicle-preserving and paper-cost aligned acceptance.

Final state: the shared evaluator still remains the final hard-feasibility judge. That is important. The algorithms do not replace simulator verification; they generate better candidates before verification. The research implication is mechanism-supported rather than purely statistical: moving constraint awareness inside search improves method independence and makes Hybrid complementarity possible. However, because not every development stage is a controlled ablation, this should be written as a design-supported conclusion, not as a theorem.

## 15. RQ2 Overall Algorithm Behavior

| method | runs | feasible_rate | feasible_vehicle_count_mean | feasible_total_distance_mean | feasible_charging_time_mean | feasible_waiting_time_mean | feasible_paper_cost_mean | runtime_median | runtime_mean | runtime_max |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ALNS | 36 | 75.0 | 8.888889 | 874.735865 | 137.720392 | 1114.748686 | 1319.028256 | 90.529717 | 456.633936 | 10929.261842 |
| GA | 36 | 66.667 | 7.666667 | 906.231298 | 170.940348 | 683.615107 | 1262.172734 | 76.122086 | 402.306639 | 10922.699444 |
| Hybrid | 36 | 72.222 | 7.884615 | 891.437547 | 156.253175 | 684.790325 | 1232.753151 | 63.404685 | 62.724606 | 73.821376 |

**Controlled final evidence.** ALNS has the highest feasible rate (75.0%), GA has the lowest feasible vehicle count (7.666667), ALNS has the shortest total distance (874.735865), and Hybrid has the lowest paper cost (1232.753151) and most stable runtime. These patterns follow the development history. GA's constructive decoder and vehicle-aware route splitting bias it toward fewer vehicles. ALNS's local route restructuring biases it toward shorter distance and higher feasibility but can increase vehicle count. Hybrid's paper-cost alignment and vehicle-preserving rule were designed exactly because earlier Hybrid stages exposed vehicle-distance conflicts.

Counter-evidence matters. Hybrid does not have the best feasible rate, the fewest vehicles, or the shortest distance. Therefore the correct conclusion is not `Hybrid dominates`; it is that Hybrid combines competitive solution quality with lower runtime variance and lower average paper cost under the final ranking.

## 16. RQ3 Instance Sensitivity

| source_instance | method | runs | feasible_runs | feasible_rate | infeasible_runs | feasible_vehicle_count_mean | feasible_vehicle_count_median | feasible_vehicle_count_std | feasible_total_distance_mean | feasible_total_distance_median | feasible_total_distance_std | feasible_truck_distance_mean | feasible_truck_distance_median | feasible_truck_distance_std | feasible_drone_distance_mean | feasible_drone_distance_median | feasible_drone_distance_std | feasible_completion_time_mean | feasible_completion_time_median | feasible_completion_time_std | feasible_charging_count_mean | feasible_charging_count_median | feasible_charging_count_std | feasible_charging_time_mean | feasible_charging_time_median | feasible_charging_time_std | feasible_waiting_time_mean | feasible_waiting_time_median | feasible_waiting_time_std | feasible_truck_waiting_time_mean | feasible_truck_waiting_time_median | feasible_truck_waiting_time_std | feasible_drone_waiting_time_mean | feasible_drone_waiting_time_median | feasible_drone_waiting_time_std | feasible_petal_score_mean | feasible_petal_score_median | feasible_petal_score_std | feasible_crossing_count_mean | feasible_crossing_count_median | feasible_crossing_count_std | feasible_paper_cost_mean | feasible_paper_cost_median | feasible_paper_cost_std | runtime_mean | runtime_median | runtime_std | runtime_max | runtime_gt_500_count | runtime_gt_1000_count |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C101 | ALNS | 12 | 4 | 33.333 | 8 | 9.0 | 9.0 | 0.0 | 1056.818485 | 1058.851069 | 15.393399 | 974.335976 | 977.738435 | 15.552481 | 82.48251 | 83.852385 | 2.73975 | 1098.540926 | 1098.540926 | 0.0 | 11.25 | 11.5 | 3.201562 | 269.844498 | 238.971975 | 127.191769 | 4539.629881 | 4562.304258 | 68.979285 | 3921.670878 | 3944.345255 | 68.979285 | 617.959003 | 617.959003 | 0.0 | 964.302393 | 900.374197 | 191.287211 | 18.75 | 17.5 | 3.774917 | 2616.060205 | 2601.441167 | 103.521097 | 997.810454 | 91.100706 | 3127.622727 | 10929.261842 | 1 | 1 |
| C101 | GA | 12 | 4 | 33.333 | 8 | 5.75 | 6.0 | 0.5 | 1094.484442 | 1119.745399 | 54.060522 | 912.72603 | 919.925111 | 63.427977 | 181.758411 | 155.016622 | 64.162513 | 1101.743772 | 1102.811388 | 2.135231 | 12.5 | 12.0 | 2.645751 | 346.960727 | 331.658069 | 106.752543 | 1995.472884 | 1926.529087 | 225.139112 | 1759.208597 | 1836.185255 | 168.684764 | 236.264287 | 70.150568 | 384.722148 | 1379.127988 | 1370.391326 | 332.051957 | 26.75 | 26.5 | 6.652067 | 1999.379462 | 1996.810027 | 116.725121 | 73.002336 | 72.301811 | 17.043708 | 95.948919 | 0 | 0 |
| C101 | Hybrid | 12 | 4 | 33.333 | 8 | 5.5 | 5.5 | 0.57735 | 1088.223049 | 1094.915896 | 50.923112 | 853.657205 | 869.933013 | 88.892709 | 234.565843 | 237.048399 | 89.862225 | 1102.811388 | 1102.811388 | 0.0 | 11.25 | 11.5 | 2.061553 | 312.003117 | 280.418283 | 123.057804 | 2090.113044 | 2088.126321 | 214.474588 | 1752.446814 | 1778.361235 | 196.477461 | 337.66623 | 272.954454 | 403.969818 | 1223.315032 | 1302.328964 | 447.065282 | 23.5 | 25.0 | 8.962886 | 2007.170985 | 1971.865014 | 89.744385 | 63.488062 | 63.441074 | 5.108074 | 73.30088 | 0 | 0 |
| R101 | ALNS | 12 | 12 | 100.0 | 0 | 9.583333 | 10.0 | 0.514929 | 782.233274 | 778.51378 | 34.062 | 696.979144 | 678.394644 | 56.644102 | 85.25413 | 96.420389 | 44.418408 | 210.442636 | 212.263362 | 12.840739 | 3.5 | 3.0 | 1.445998 | 77.238857 | 53.955169 | 53.820774 | 657.385572 | 692.137567 | 164.521189 | 644.527228 | 692.137567 | 151.121358 | 12.858344 | 0.0 | 27.755376 | 646.70632 | 610.175152 | 228.33284 | 12.666667 | 12.0 | 4.53939 | 1027.033111 | 1011.127137 | 51.867869 | 90.437677 | 90.359068 | 0.411141 | 91.442813 | 0 | 0 |
| R101 | GA | 12 | 8 | 66.667 | 4 | 8.25 | 8.5 | 0.886405 | 777.78524 | 766.008996 | 52.19677 | 670.863732 | 660.926768 | 63.310834 | 106.921509 | 104.397188 | 93.016804 | 221.356686 | 219.041631 | 7.403451 | 4.125 | 4.0 | 0.991031 | 95.472886 | 93.245536 | 51.731753 | 577.031342 | 553.578149 | 73.917305 | 563.113771 | 534.389421 | 81.388923 | 13.91757 | 0.0 | 31.761841 | 928.495492 | 749.597769 | 506.458719 | 18.125 | 14.5 | 10.119818 | 1020.995354 | 991.894894 | 59.563598 | 984.440674 | 87.413971 | 3129.768372 | 10922.699444 | 1 | 1 |
| R101 | Hybrid | 12 | 10 | 83.333 | 2 | 8.8 | 9.0 | 1.032796 | 814.379064 | 820.813016 | 72.437473 | 702.602054 | 696.841327 | 80.048638 | 111.77701 | 109.877505 | 96.667775 | 213.073111 | 215.373714 | 14.501623 | 4.9 | 5.0 | 2.078995 | 84.373407 | 61.542254 | 58.581484 | 558.884263 | 564.8124 | 110.982025 | 554.539492 | 547.694107 | 108.935516 | 4.344772 | 0.0 | 9.986547 | 979.992049 | 898.585937 | 299.585335 | 19.1 | 17.5 | 5.989806 | 1039.55973 | 1010.121615 | 107.027294 | 63.640798 | 63.733504 | 4.421513 | 69.323954 | 0 | 0 |
| RC101 | ALNS | 12 | 11 | 91.667 | 1 | 8.090909 | 8.0 | 0.700649 | 909.43592 | 924.840228 | 73.629384 | 851.116463 | 869.919655 | 64.291133 | 58.319457 | 61.420922 | 29.62705 | 238.038482 | 233.520027 | 15.079344 | 6.181818 | 7.0 | 1.88776 | 155.65512 | 117.376227 | 107.46053 | 368.278921 | 378.983692 | 93.957927 | 333.240266 | 306.186105 | 109.219212 | 35.038655 | 3.54102 | 53.247504 | 528.786444 | 567.087553 | 229.701031 | 10.272727 | 11.0 | 4.540725 | 1165.920433 | 1093.080516 | 145.675259 | 281.653678 | 90.190632 | 662.257111 | 2384.601612 | 1 | 1 |
| RC101 | GA | 12 | 12 | 100.0 | 0 | 7.916667 | 8.0 | 0.792961 | 929.110956 | 923.358632 | 66.715208 | 856.994327 | 873.670032 | 59.001263 | 72.116628 | 59.159599 | 59.403963 | 243.856834 | 242.45571 | 18.027753 | 6.583333 | 7.0 | 2.274696 | 162.57853 | 154.031932 | 65.460157 | 317.385025 | 291.039793 | 134.191301 | 292.63968 | 291.039793 | 107.245973 | 24.745345 | 0.0 | 46.143354 | 367.028454 | 368.378599 | 102.536658 | 7.0 | 7.0 | 2.044949 | 1177.222078 | 1211.219104 | 122.393649 | 149.476907 | 61.442232 | 294.818908 | 1083.48245 | 1 | 1 |
| RC101 | Hybrid | 12 | 12 | 100.0 | 0 | 7.916667 | 8.0 | 0.668558 | 890.057783 | 864.521232 | 74.842267 | 833.974972 | 826.846009 | 31.26539 | 56.082811 | 33.796906 | 70.501426 | 241.305735 | 237.713551 | 18.94838 | 6.25 | 6.5 | 1.864745 | 164.236333 | 127.797937 | 86.944973 | 321.271138 | 323.695136 | 95.914488 | 317.285178 | 323.695136 | 93.539442 | 3.98596 | 0.0 | 9.388706 | 351.858538 | 318.765231 | 152.102564 | 6.75 | 6.0 | 3.048845 | 1135.608391 | 1083.95923 | 130.874021 | 61.044957 | 59.834212 | 8.109481 | 73.821376 | 0 | 0 |

C101 is the critical family because final evidence reports low feasibility across methods in that structure. Development logs also contain runtime and candidate-search difficulty around C101, including the C101-10 `hybrid_diverse_topk` runtime pathology before budget fixes. The plausible mechanism is that clustered/time-window structure can narrow feasible insertion positions and amplify waiting/charging consequences. This is an interpretive inference, not a fully verified causal proof, because the current report does not include a complete per-violation C101 diagnostic decomposition for all 108 final runs. The correct paper wording should therefore distinguish observed evidence from unverified mechanism.

## 17. RQ4 Charging Policy Analysis

| method | charging_policy | runs | feasible_rate | feasible_charging_time_mean | feasible_completion_time_mean | feasible_paper_cost_mean | runtime_median |
| --- | --- | --- | --- | --- | --- | --- | --- |
| ALNS | LFC | 9 | 77.778 | 166.309045 | 347.996813 | 1345.546328 | 90.491424 |
| GA | LFC | 9 | 66.667 | 200.75867 | 375.692237 | 1274.279824 | 71.003212 |
| Hybrid | LFC | 9 | 66.667 | 157.639818 | 373.191635 | 1205.885795 | 62.854488 |
| ALNS | LPC | 9 | 77.778 | 80.751952 | 343.707607 | 1267.473768 | 90.493707 |
| GA | LPC | 9 | 66.667 | 110.420603 | 374.745151 | 1240.383916 | 64.859852 |
| Hybrid | LPC | 9 | 77.778 | 97.579998 | 346.082273 | 1194.859496 | 65.494222 |
| ALNS | NFC | 9 | 66.667 | 242.731195 | 380.916144 | 1422.167524 | 90.404375 |
| GA | NFC | 9 | 66.667 | 248.166275 | 392.378315 | 1319.676731 | 82.295583 |
| Hybrid | NFC | 9 | 66.667 | 287.641738 | 393.434386 | 1361.623411 | 61.427072 |
| ALNS | NPC | 9 | 77.778 | 76.090921 | 344.352976 | 1255.659586 | 90.798352 |
| GA | NPC | 9 | 66.667 | 124.415843 | 374.536061 | 1214.350464 | 78.539776 |
| Hybrid | NPC | 9 | 77.778 | 101.119031 | 345.044777 | 1183.215746 | 63.190038 |

The final charging result is consistent with the mathematical mechanism and development history. Partial charging lowers charging time for all three methods: ALNS LFC/LPC = 166.309/80.752 and NFC/NPC = 242.731/76.091; GA LFC/LPC = 200.759/110.421 and NFC/NPC = 248.166/124.416; Hybrid LFC/LPC = 157.640/97.580 and NFC/NPC = 287.642/101.119. Full nonlinear charging is costly because it can push charging into high-SOC slow segments. NPC avoids part of that penalty by charging only to a required target. Developmental evidence explains why charging-aware operators and charging-plan fields were introduced, but final evidence shows the dominant policy effect comes from full vs partial more clearly than from a single algorithm's charging operator success.

## 18. RQ5 Hybrid Effectiveness

| method | vs | wins | ties | losses | feasible_paired_delta_paper_cost_mean | feasible_paired_delta_paper_cost_median |
| --- | --- | --- | --- | --- | --- | --- |
| Hybrid | GA | 18 | 5 | 13 | -15.170217 | -6.00501 |
| Hybrid | ALNS | 25 | 0 | 11 | -111.195871 | -34.200711 |

Hybrid effectiveness is the point where development and final evidence meet. Naive Hybrid failed to prove automatic synergy. Top-K showed that GA's rank-best candidate was not always the best ALNS starting point. Vehicle-preserving refinement was introduced because ALNS could reduce distance by adding vehicles. Periodic and stagnation experiments showed that calling ALNS earlier did not solve the problem if the neighborhood or candidate diversity was insufficient. Hybrid-local operators reduced disruption but did not guarantee improvement. Diverse Top-K finally addressed candidate homogeneity.

Final paired evidence supports a conditional claim: Hybrid beats GA in 18 matched cases, ties 5, and loses 13; Hybrid beats ALNS in 25 and loses 11. Its average paired paper-cost delta is negative against both GA and ALNS. But the loss counts and the 25-customer smoke case prevent an overclaim. Hybrid works when diverse GA candidates expose an ALNS-refinable neighborhood that improves paper cost without increasing vehicles. It fails when ALNS refinement conflicts with vehicle count, when candidates are too homogeneous, or when local changes improve secondary metrics but not the final ranking.

## 19. RQ6 Computational Robustness

| method | runs | runtime_median | runtime_mean | runtime_std | runtime_max | runtime_gt_500_count | runtime_gt_1000_count |
| --- | --- | --- | --- | --- | --- | --- | --- |
| GA | 36 | 76.122086 | 402.306639 | 1811.42546 | 10922.699444 | 2 | 2 |
| ALNS | 36 | 90.529717 | 456.633936 | 1835.505163 | 10929.261842 | 2 | 2 |
| Hybrid | 36 | 63.404685 | 62.724606 | 6.03889 | 73.821376 | 0 | 0 |

Runtime behavior reflects development complexity. GA and ALNS both have extreme final outliers above 1000s, while Hybrid has no >500s final outliers and a runtime median of 63.404685s. This is not because Hybrid is simpler; it is because final Hybrid was forced into bounded candidate generation and refinement after development exposed runtime pathologies. More drone enumeration, charging checks, local candidates, operator loops, and full evaluator calls increase computational burden. Caching, pruning, Top-K, time budgets, and diverse but limited candidate pools were engineering responses. Extreme cases are retained as evidence of heavy-tail risk rather than removed.

Extreme runtime cases:

| instance | source_instance | method | charging_policy | seed | runtime_seconds | feasible | vehicle_count | total_distance | paper_cost |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| RC101_25_seed1987_td | RC101 | GA | LFC | 1987 | 1083.48245 | True | 9 | 964.869714 | 1226.576039 |
| R101_25_seed42_td | R101 | GA | NFC | 42 | 10922.699444 | False | 7 | 992.996183 | 1292.663018 |
| C101_25_seed42_td | C101 | ALNS | NFC | 42 | 10929.261842 | False | 7 | 792.28255 | 1807.82874 |
| RC101_25_seed42_td | RC101 | ALNS | NPC | 42 | 2384.601612 | True | 8 | 970.163119 | 1157.186058 |

## 20. Supporting Experiments as Methodological Evidence

### 20.1 5-customer mechanism validation

Developmental 5-customer runs were used to verify mechanics: feasible truck/drone route construction, charging, synchronization, and trace diagnostics. Examples include ALNS R101-5 NPC feasible with 3 vehicles, distance 161.151, and runtime 0.845s, and C101-5/RC101-5 feasible checks around 0.9-1.0s. These are not final ranking evidence; they show that the simulator/evaluator and method-specific construction could produce valid small solutions.

### 20.2 10-customer method evolution

10-customer experiments exposed algorithm behavior without the runtime risk of full 25/50 runs. R101-10 Hybrid refine achieved about 0.64% improvement in one stage, Top-K later selected rank-3 candidates, and Stage6 debug showed GA distance 349.970, hybrid_topk 371.125, hybrid_preserve 358.810, hybrid_diverse_topk 344.074, and hybrid_diverse_stagnation 342.752. Because these used development configurations, they support design decisions but do not prove final superiority.

### 20.3 25-customer development

25-customer development was the bridge to the final experiment. It revealed the Hybrid vehicle-distance conflict: R101-25 GA had 7 vehicles and distance 810.202, while hybrid_diverse_topk had 8 vehicles and distance 771.314 in the smoke/audit case. This evidence directly motivated vehicle-preserving and paper-cost priority rules.

### 20.4 50-customer stress evidence

50-customer experiments were treated as stress/development evidence, not as the main controlled benchmark. Their value is to expose scalability and runtime behavior. They should not be averaged with the final 25-customer matrix.

## 21. Representative Solutions

Route figures must come from saved raw solutions. If a particular raw result does not contain enough route/drone/charging data for reconstruction, the report should state `Route reconstruction unavailable` rather than rerunning solvers or redrawing a route by hand.

## 22. Negative Results and Failure Analysis

### Hybrid non-dominance

Final W/T/L and development stages both show non-dominance. Naive refinement often retained GA; periodic/stagnation triggered ALNS but often injected zero; Stage6 local refinement did not consistently outperform GA. Final evidence still has 13 Hybrid losses against GA. Lesson: complementarity requires structural diversity and aligned acceptance, not just serial composition.

### Vehicle-distance conflict

Development warning: ALNS sometimes reduced distance by increasing vehicles. Final manifestation: ALNS has the shortest distance but highest vehicle count. Design response: vehicle-preserving Hybrid refinement. Limitation: preserving vehicles can reject legitimate distance improvements.

### Ineffective ALNS operators

Some operators were added for valid reasons but showed limited improvement. This is not a failure to hide; it identifies a narrow feasible neighborhood and objective mismatch. Operators should be discussed by motivation, tested diagnostic outcome, and final interpretation.

### Runtime pathology

The C101-10 ~532s development runtime and final GA/ALNS >1000s outliers show that runtime risk existed before and survived into the final experiment for standalone methods. Hybrid's bounded runtime is a design achievement, but not proof of superior solution quality.

### Early repair/evaluator dependence

Early dependence on shared evaluator/repair reduced method differentiation. This explains why the project moved toward constraint-aware decoders, states, and operators. It is also a warning for future work: if feasibility mostly comes from a shared downstream repair, algorithm comparison becomes less meaningful.

## 23. Discussion

### 23.1 Constraint-aware search matters

The development history and final results together support the central argument. Generic routing operators were insufficient because Truck-Drone EVRPTW-NL constraints are coupled. Moving constraint awareness into GA decoding and ALNS operators made the methods more independent and interpretable, even though the evaluator remains the final judge.

### 23.2 Search bias differs by method

GA is construction-biased and tends to preserve fewer vehicles. ALNS is local-restructuring-biased and tends to improve feasibility/distance but can use more vehicles. Hybrid is synthesis-biased: it can improve paper cost when candidate diversity and vehicle-preserving refinement align, but it does not dominate all metrics.

### 23.3 Hybridization requires structural complementarity

The project spent many stages on Hybrid because the theoretical complementarity is real: GA supplies global population diversity, ALNS supplies local restructuring. The evidence shows that this complementarity is conditional. If GA candidates are homogeneous or ALNS neighborhoods conflict with the ranking, Hybrid becomes equal to GA or worse. This is why final Hybrid emphasizes diverse candidate types and paper-cost aligned acceptance.

### 23.4 Candidate quality vs candidate diversity

Development results showed that the GA-best candidate is not always the best ALNS start, and quality Top-K is not enough if structures are similar. Diverse Top-K is therefore not a cosmetic addition; it is the final response to the observed failure mode that ALNS had no useful neighborhood to exploit.

### 23.5 Charging changes the search landscape

Partial charging reduces charging time across methods, and nonlinear full charging tends to be costly. This changes not only charging metrics but also time-window and completion behavior because charging time propagates through routes and synchronization.

### 23.6 Development and final evidence can disagree

Some debug-stage improvements did not become final dominance. That is normal in heuristic research: small controlled mechanics can validate an idea, while larger formal matrices test robustness. The report should present these contradictions as scientific evidence, not as embarrassment.

### 23.7 Negative results as design knowledge

Ineffective operators, vehicle-distance conflict, Hybrid losses, C101 difficulty, and runtime tails all provide design knowledge. They define where the current frozen method is reliable and where future work should focus.

## 24. Limitations

The main formal scale is 25 customers. 50-customer evidence is stress/developmental. OR-Tools and PyVRP are truck-only references, not complete-model competitors. Truck and drone use Euclidean distance; road networks, airspace, wind, no-fly zones, and cross-truck drone recovery are not modeled. Charging curves are engineering approximations. No global optimality is proven. Hybrid is competitive but not universally dominant. Runtime outliers remain and are retained.

## 25. Contribution-Evidence Mapping

| Contribution | RQ | Evidence | Table/Figure | Limitation |
| --- | --- | --- | --- | --- |
| Unified Truck-Drone EVRPTW-NL computational model | RQ1 | final model, simulator, evaluator | model-code map, problem schematic | engineering model, not complete real-world physics |
| Constraint-aware GA adaptation | RQ1/RQ2 | solve_ga.py, ga_tools.py, final results | GA method table, overall summary | not globally optimal |
| Independent ALNS with diagnostics | RQ1/RQ2 | ALNS code and operator summary | operator bubble/heatmap | operator effects uneven |
| Diverse Top-K GA-ALNS Hybrid | RQ5 | hybrid code and diagnostics | win/tie/loss, paired deltas | conditional, not universal dominance |
| Charging policy analysis | RQ4 | charging.py and final 108 rows | 2x2 charging figure | segmented approximation |
| Runtime robustness analysis | RQ6 | runtime fields | runtime ECDF/outlier table | no profiling proof |

## 26. References

References are maintained in `references.bib` and `reference_gaps.md`. Items still marked `[REFERENCE NEEDED]` require manual bibliographic verification before submission.

## 27. Integrated Completion Audit

| Check | Status | Evidence |
| --- | --- | --- |
| development narrative enters Final Method | PASS | GA/ALNS/Hybrid final sections each include why each component exists. |
| historical results enter RQ analysis | PASS | RQ2-RQ6 explicitly connect final numbers with development evidence. |
| 108-run remains formal performance basis | PASS | Controlled final evidence is separated from developmental evidence. |
| GA final behavior explained by GA history | PASS | Structured representation, decoder, drone/charging, and diversity are linked to failures. |
| ALNS final behavior explained by operator/history | PASS | Operator families are linked to historical motivation and diagnostics. |
| Hybrid final behavior connected to stages | PASS | Naive, Top-K, preserve, periodic/stagnation, local, diverse Top-K are connected to final W/T/L. |
| charging connects model/history/final result | PASS | 2x2 policy model and final charging-time numbers are integrated. |
| runtime connects development complexity | PASS | Runtime pathology and final outliers are discussed together. |
| supporting experiments have numbers and roles | PASS | 5/10/25/50 roles include concrete development numbers where available. |
| avoid overclaiming developmental evidence | PASS | Developmental evidence is repeatedly marked as non-controlled where appropriate. |
| no second front-loaded deep section | PASS | Report starts with normal report sections and integrates deep material in place. |
